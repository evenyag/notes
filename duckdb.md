# DuckDB

## 查询
```c++
#include "duckdb.hpp"

using namespace duckdb;

int main() {
	DuckDB db(nullptr);
	Connection con(db);

	con.Query("CREATE TABLE integers(i INTEGER)");
	con.Query("INSERT INTO integers VALUES (3)");
	auto result = con.Query("SELECT * FROM integers");
	result->Print();
}
```

### Connection::Query()
```c++
unique_ptr<MaterializedQueryResult> Connection::Query(const string &query) {
	auto result = context->Query(query, false);
	D_ASSERT(result->type == QueryResultType::MATERIALIZED_RESULT);
	return unique_ptr_cast<QueryResult, MaterializedQueryResult>(move(result));
}
```

### ClientContext::Query()
ClientContext::Query()
- ParseStatements()
- PendingQueryInternal() 得到 PendingQueryResult
- ExecutePendingQueryInternal() 执行 PendingQueryResult 得到 QueryResult
    - PendingQueryResult::ExecuteInternal()

注意这里只有 is_last_statement 才允许使用 stream_result
```c++
//! Issue a query, returning a QueryResult. The QueryResult can be either a StreamQueryResult or a
//! MaterializedQueryResult. The StreamQueryResult will only be returned in the case of a successful SELECT
//! statement.
unique_ptr<QueryResult> ClientContext::Query(const string &query, bool allow_stream_result) {
    // ...
	vector<unique_ptr<SQLStatement>> statements;
	if (!ParseStatements(*lock, query, statements, error)) {
		return make_unique<MaterializedQueryResult>(move(error));
	}
	// ...

	unique_ptr<QueryResult> result;
	QueryResult *last_result = nullptr;
	for (idx_t i = 0; i < statements.size(); i++) {
		auto &statement = statements[i];
		bool is_last_statement = i + 1 == statements.size();
		bool stream_result = allow_stream_result && is_last_statement;
		auto pending_query = PendingQueryInternal(*lock, move(statement));
		unique_ptr<QueryResult> current_result;
		if (!pending_query->success) {
			current_result = make_unique<MaterializedQueryResult>(pending_query->error);
		} else {
			current_result = ExecutePendingQueryInternal(*lock, *pending_query, stream_result);
		}
		// now append the result to the list of results
		if (!last_result) {
			// first result of the query
			result = move(current_result);
			last_result = result.get();
		} else {
			// later results; attach to the result chain
			last_result->next = move(current_result);
			last_result = last_result->next.get();
		}
	}
	return result;
}
```

#### ClientContext::ParseStatements()
```c++
vector<unique_ptr<SQLStatement>> ClientContext::ParseStatements(const string &query) {
	auto lock = LockContext();
	return ParseStatementsInternal(*lock, query);
}

vector<unique_ptr<SQLStatement>> ClientContext::ParseStatementsInternal(ClientContextLock &lock, const string &query) {
	Parser parser(GetParserOptions());
	parser.ParseQuery(query);

	PragmaHandler handler(*this);
	handler.HandlePragmaStatements(lock, parser.statements);

	return move(parser.statements);
}
```

#### ClientContext::PendingQueryInternal()
ClientContext::PendingQueryInternal()
    - PendingStatementOrPreparedStatement()
        - PendingStatementInternal()

```c++
unique_ptr<PendingQueryResult> ClientContext::PendingQueryInternal(ClientContextLock &lock,
                                                                   unique_ptr<SQLStatement> statement, bool verify) {
	auto query = statement->query;
	shared_ptr<PreparedStatementData> prepared;
	if (verify) {
		return PendingStatementOrPreparedStatementInternal(lock, query, move(statement), prepared, nullptr);
	} else {
		return PendingStatementOrPreparedStatement(lock, query, move(statement), prepared, nullptr);
	}
}
```

#### ClientContext::ExecutePendingQueryInternal()
ClientContext::ExecutePendingQueryInternal()
- PendingQueryResult::ExecuteInternal()
```c++
unique_ptr<QueryResult> ClientContext::ExecutePendingQueryInternal(ClientContextLock &lock, PendingQueryResult &query,
                                                                   bool allow_stream_result) {
	return query.ExecuteInternal(lock, allow_stream_result);
}
```

#### ClientContext::PendingStatementInternal()
ClientContext::PendingStatementInternal()
- CreatePreparedStatement()
- PendingPreparedStatement()
```c++
unique_ptr<PendingQueryResult> ClientContext::PendingStatementInternal(ClientContextLock &lock, const string &query,
                                                                       unique_ptr<SQLStatement> statement) {
	// prepare the query for execution
	auto prepared = CreatePreparedStatement(lock, query, move(statement));
	// by default, no values are bound
	vector<Value> bound_values;
	// execute the prepared statement
	return PendingPreparedStatement(lock, move(prepared), move(bound_values));
}
```

ClientContext::CreatePreparedStatement()
- Planner::CreatePlan() 生成逻辑计划
- Optimizer::Optimize() 优化
- PhysicalPlanGenerator::CreatePlan() 生成物理计划
```c++
shared_ptr<PreparedStatementData> ClientContext::CreatePreparedStatement(ClientContextLock &lock, const string &query,
                                                                         unique_ptr<SQLStatement> statement) {
	StatementType statement_type = statement->type;
	auto result = make_shared<PreparedStatementData>(statement_type);

    // ...
	Planner planner(*this);
	planner.CreatePlan(move(statement));
    // ...

	auto plan = move(planner.plan);
    // ...
	// extract the result column names from the plan
	result->read_only = planner.read_only;
	result->requires_valid_transaction = planner.requires_valid_transaction;
	result->allow_stream_result = planner.allow_stream_result;
	result->names = planner.names;
	result->types = planner.types;
	result->value_map = move(planner.value_map);
	result->catalog_version = Transaction::GetTransaction(*this).catalog_version;

	if (config.enable_optimizer) {
        // ...
		Optimizer optimizer(*planner.binder, *this);
		plan = optimizer.Optimize(move(plan));
        // ...
	}

    // ...
	// now convert logical query plan into a physical query plan
	PhysicalPlanGenerator physical_planner(*this);
	auto physical_plan = physical_planner.CreatePlan(move(plan));

    // ...
	result->plan = move(physical_plan);
	return result;
}
```

ClientContext::PendingPreparedStatement()
- PreparedStatementData::Bind()
- Executor::Initialize()
```c++
unique_ptr<PendingQueryResult> ClientContext::PendingPreparedStatement(ClientContextLock &lock,
                                                                       shared_ptr<PreparedStatementData> statement_p,
                                                                       vector<Value> bound_values) {
    // ...
	auto &statement = *statement_p;

	// bind the bound values before execution
	statement.Bind(move(bound_values));

	active_query->executor = make_unique<Executor>(*this);
	auto &executor = *active_query->executor;
	if (config.enable_progress_bar) {
		active_query->progress_bar = make_unique<ProgressBar>(executor, config.wait_time);
		active_query->progress_bar->Start();
		query_progress = 0;
	}
	executor.Initialize(statement.plan.get());
	auto types = executor.GetTypes();
    // ...

	auto pending_result = make_unique<PendingQueryResult>(shared_from_this(), *statement_p, move(types));
	active_query->prepared = move(statement_p);
	active_query->open_result = pending_result.get();
	return pending_result;
}
```

#### PendingQueryResult::ExecuteInternal()
PendingQueryResult::ExecuteInternal()
- CheckExecutableInternal()
- ExecuteTaskInternal()
    - ClientContext::ExecuteTaskInternal()
- ClientContext::FetchResultInternal()
- Close()
    - ClientContext::reset()
```c++
//! Returns the result of the query as an actual query result.
//! This returns (mostly) instantly if ExecuteTask has been called until RESULT_READY was returned.
unique_ptr<QueryResult> PendingQueryResult::Execute(bool allow_streaming_result) {
	auto lock = LockContext();
	return ExecuteInternal(*lock, allow_streaming_result);
}

unique_ptr<QueryResult> PendingQueryResult::ExecuteInternal(ClientContextLock &lock, bool allow_streaming_result) {
	CheckExecutableInternal(lock);
	while (ExecuteTaskInternal(lock) == PendingExecutionResult::RESULT_NOT_READY) {
	}
	if (!success) {
		return make_unique<MaterializedQueryResult>(error);
	}
	auto result = context->FetchResultInternal(lock, *this, allow_streaming_result);
	Close();
	return result;
}

PendingExecutionResult PendingQueryResult::ExecuteTaskInternal(ClientContextLock &lock) {
	CheckExecutableInternal(lock);
	return context->ExecuteTaskInternal(lock, *this);
}
```

ClientContext::ExecuteTaskInternal()
- Executor::ExecuteTask()
- ProgressBar::Update()
- ProgressBar::GetCurrentPercentage()

ClientContext::FetchResultInternal()
- StreamQueryResult 则直接返回
- MaterializedQueryResult
    - ClientContext::FetchInternal()
        - Executor::FetchChunk()
    - ChunkCollection::Append()
```c++
unique_ptr<QueryResult> ClientContext::FetchResultInternal(ClientContextLock &lock, PendingQueryResult &pending,
                                                           bool allow_stream_result) {
    // ...
	auto &prepared = *active_query->prepared;
	bool create_stream_result = prepared.allow_stream_result && allow_stream_result;
	if (create_stream_result) {
		active_query->progress_bar.reset();
		query_progress = -1;

		// successfully compiled SELECT clause and it is the last statement
		// return a StreamQueryResult so the client can call Fetch() on it and stream the result
		auto stream_result =
		    make_unique<StreamQueryResult>(pending.statement_type, shared_from_this(), pending.types, pending.names);
		active_query->open_result = stream_result.get();
		return move(stream_result);
	}
	// create a materialized result by continuously fetching
	auto result = make_unique<MaterializedQueryResult>(pending.statement_type, pending.types, pending.names);
	while (true) {
		auto chunk = FetchInternal(lock, GetExecutor(), *result);
		if (!chunk || chunk->size() == 0) {
			break;
		}
        // ...
		result->collection.Append(*chunk);
	}
	return move(result);
}
```

#### Executor::FetchChunk()
Executor::FetchChunk()
```c++
unique_ptr<DataChunk> Executor::FetchChunk() {
	D_ASSERT(physical_plan);

	auto chunk = make_unique<DataChunk>();
	root_executor->InitializeChunk(*chunk);
	while (true) {
		root_executor->ExecutePull(*chunk);
		if (chunk->size() == 0) {
			root_executor->PullFinalize();
			if (NextExecutor()) {
				continue;
			}
			break;
		} else {
			break;
		}
	}
	return chunk;
}
```
