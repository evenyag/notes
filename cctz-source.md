# cctz 源码学习

主要关注 civil_time 的逻辑

civil_time 的主要实现在 `include/cctz/civil_time_detail.h` 内，而 `include/cctz/civil_time.h` 主要是对里面实现的 reimport
```c++
#include "cctz/civil_time_detail.h"

using civil_year = detail::civil_year;
using civil_month = detail::civil_month;
using civil_day = detail::civil_day;
using civil_hour = detail::civil_hour;
using civil_minute = detail::civil_minute;
using civil_second = detail::civil_second;
```

而 civil_year/civil_month 等类其实都是基于 civil_time 实现的
```c++
struct second_tag {};
struct minute_tag : second_tag {};
struct hour_tag : minute_tag {};
struct day_tag : hour_tag {};
struct month_tag : day_tag {};
struct year_tag : month_tag {};

using civil_year = civil_time<year_tag>;
using civil_month = civil_time<month_tag>;
using civil_day = civil_time<day_tag>;
using civil_hour = civil_time<hour_tag>;
using civil_minute = civil_time<minute_tag>;
using civil_second = civil_time<second_tag>;
```

## 构造
civil_time 可以通过传入 y:m:d:hh:mm:ss 来构造，而默认构造出来的是 `1970:01:01 00:00:00` 对应的 civil_time
```c++
explicit CONSTEXPR_M civil_time(year_t y, diff_t m = 1, diff_t d = 1,
                              diff_t hh = 0, diff_t mm = 0,
                              diff_t ss = 0) noexcept
  : civil_time(impl::n_sec(y, m, d, hh, mm, ss)) {}

CONSTEXPR_M civil_time() noexcept : f_{1970, 1, 1, 0, 0, 0} {}
```

可以看到，实际上会调用 `impl::n_sec()` ，拿到一个 fields 结构然后调用底下的私有构造函数将 civil_time 构造出来
```c++
// The designated constructor that all others eventually call.
explicit CONSTEXPR_M civil_time(fields f) noexcept : f_(align(T{}, f)) {}
```

而 civil_time 本身的成员就是这个 fields 结构体，这个就是 normalized 之后的 y/m/d/hh/mm/ss ，可以看到其数值范围都是标准化后的。这里通过 typedef 里一系列类型（如 `month_t`/`day_t`）来表示标准化后的数据，当看到这里类型之后就表示对应的数字是标准化后的了
```
// Type aliases that indicate normalized argument values.
using month_t = std::int_fast8_t;   // [1:12]
using day_t = std::int_fast8_t;     // [1:31]
using hour_t = std::int_fast8_t;    // [0:23]
using minute_t = std::int_fast8_t;  // [0:59]
using second_t = std::int_fast8_t;  // [0:59]

// Normalized civil-time fields: Y-M-D HH:MM:SS.
struct fields {
  CONSTEXPR_M fields(year_t year, month_t month, day_t day,
                     hour_t hour, minute_t minute, second_t second)
      : y(year), m(month), d(day), hh(hour), mm(minute), ss(second) {}
  std::int_least64_t y;
  std::int_least8_t m;
  std::int_least8_t d;
  std::int_least8_t hh;
  std::int_least8_t mm;
  std::int_least8_t ss;
};
```

实际计算出 fields 的逻辑就在 `n_sec()` 了，而 `n_sec()` 就是对 sec 以及以上的数据都做标准化
```c++
CONSTEXPR_F fields n_sec(year_t y, diff_t m, diff_t d, diff_t hh, diff_t mm,
                         diff_t ss) noexcept {
  // Optimization for when (non-constexpr) fields are already normalized.
  // 优先针对标准化后（normalized）的数值进行处理，大部分情况这些值应该都是标准化后的
  if (0 <= ss && ss < 60) {
    // 这里变量都是 n 开头应该就是取标准化之意， ss 范围不标准
    const second_t nss = static_cast<second_t>(ss);
    if (0 <= mm && mm < 60) {
      // mm 范围不标准
      const minute_t nmm = static_cast<minute_t>(mm);
      if (0 <= hh && hh < 24) {
        // hh 范围不标准
        const hour_t nhh = static_cast<hour_t>(hh);
        if (1 <= d && d <= 28 && 1 <= m && m <= 12) {
          // d 和 m 的范围也标准
          const day_t nd = static_cast<day_t>(d);
          const month_t nm = static_cast<month_t>(m);
          // 所有值都标准，直接构造 fields
          return fields(y, nm, nd, nhh, nmm, nss);
        }
        // 否则 d 和 m 可能范围不标准，用 n_mon 对 m 进行标准化
        return n_mon(y, m, d, 0, nhh, nmm, nss);
      }
      // hh 可能不标准， 用 m_hour 对 hh 进行标准化
      // `hh / 24` 计算 cd 即天的增量， `hh % 24` 计算转成天后余下几小时
      return n_hour(y, m, d, hh / 24, hh % 24, nmm, nss);
    }
    // mm 不标准，用 m_min 对 mm 做标准化
    // `mm / 60` 计算 ch 即小时的增量，然后 `mm % 60` 拿到余下几分钟
    return n_min(y, m, d, hh, mm / 60, mm % 60, nss);
  }
  // 这里 ss 是不标准的，先对 ss 做标准化
  // 首先计算 ss 有几分钟， cm 感觉就是类似 count_minute 的意思
  diff_t cm = ss / 60;
  // 接下来是 ss 还余下几秒
  ss %= 60;
  if (ss < 0) {
    // 当然 ss 可能是负数，如果 ss 取模后还是负的，我们这里需要将 ss 变为正数来进行标准化，这时候 cm 要减去 1 分钟， ss 加上 60s
    cm -= 1;
    ss += 60;
  }
  // 然后也是调用 n_min 对 mm 标准化
  // 这里 `mm / 60 + cm / 60` 是计算这个分钟值对应有多少小时，而 `mm % 60 + cm % 60` 是计算转为小时后余下多少分钟
  // 分开计算再加起来应该是为了避免 overflow ，但是这也存在 mm + cm 加起来反而超过 60min 的情况没有处理
  return n_min(y, m, d, hh, mm / 60 + cm / 60, mm % 60 + cm % 60,
               static_cast<second_t>(ss));
}
```

`n_min()` 对分钟及以上的数据做标准化
```c++
// ch 是 hh 的增量
CONSTEXPR_F fields n_min(year_t y, diff_t m, diff_t d, diff_t hh, diff_t ch,
                         diff_t mm, second_t ss) noexcept {
  // 计算 mm 对应了多少小时
  ch += mm / 60;
  // 转为小时后余下多少分钟
  mm %= 60;
  if (mm < 0) {
    // 如果分钟是负的，我们要转为正数做标准化，因此先减 1h ，然后加 60 分钟
    ch -= 1;
    mm += 60;
  }
  // 接下来用 n_hour 对 hh 部分做标准化
  // 同理， `hh / 24 + ch / 24` 算出 cd 即对 d 的增量，然后 `hh % 24 + ch % 24`
  // 得到余下的小时数
  return n_hour(y, m, d, hh / 24 + ch / 24, hh % 24 + ch % 24,
                static_cast<minute_t>(mm), ss);
}
```

`n_hour()` 对小时及以上数据做标准化，逻辑大同小异，不过到了天以后，处理就会慢慢复杂
```c++
// cd 是对 d 的增量
CONSTEXPR_F fields n_hour(year_t y, diff_t m, diff_t d, diff_t cd,
                          diff_t hh, minute_t mm, second_t ss) noexcept {
  // 计算 hh 对应多少天
  cd += hh / 24;
  // 余下多少小时
  hh %= 24;
  if (hh < 0) {
    // 如果小时还是负的，需要转为正数，减一天然后加 24h
    cd -= 1;
    hh += 24;
  }
  // 用 n_mon 对余下部分做标准化
  return n_mon(y, m, d, cd, static_cast<hour_t>(hh), mm, ss);
}
```

`n_month` 对月的数据做标准化
```c++
// cd 是对 d 的增量
CONSTEXPR_F fields n_mon(year_t y, diff_t m, diff_t d, diff_t cd,
                         hour_t hh, minute_t mm, second_t ss) noexcept {
  if (m != 12) {
    // m 的范围是 1 ~ 12 ，如果 m 不等于 12
    // 首先计算 m 对应了多少年，将这部分加到 y
    y += m / 12;
    // 去掉年的部分后计算 m 余下几个月
    m %= 12;
    if (m <= 0) {
      // 如果为负数，则需要标准化为正数，扣去一年，然后加上 12 月
      y -= 1;
      m += 12;
    }
  }
  // 调用 n_day 继续处理，注意这里 cd 还会继续往下传
  return n_day(y, static_cast<month_t>(m), d, cd, hh, mm, ss);
}
```

然后就是最为复杂的 `n_day()` 的逻辑了
```c++
// cd 是对 d 的增量，到这里除了 d 部分，其他部分都标准化了
// 需要注意底下用的的常量
// - 每 400 年就正好有 146097 天 (Gregorian cycle of 400 years has exactly 146,097 days)
CONSTEXPR_F fields n_day(year_t y, month_t m, diff_t d, diff_t cd,
                         hour_t hh, minute_t mm, second_t ss) noexcept {
  // 计算 y 模 400 年余下多少年，记录到 ey 中， ey 后续会用来累计 cd 和 d 都合并并转成标准化的 d 后有多少年，
  // 而不需要对原来的 y 做计算，这其实也是一种避免 overflow 的手段
  // 详见 https://github.com/google/cctz/pull/152
  year_t ey = y % 400;
  // 记录原来 y 模 400 年余下多少年到 oey ，大意应该就是 original ey
  const year_t oey = ey;
  // 计算 cd 可以对应到多少个 400 年，并累加到 ey 中
  ey += (cd / 146097) * 400;
  // 计算 cd 扣掉若干 400 年后剩下多少天
  cd %= 146097;
  if (cd < 0) {
    // 如果 cd 为负了，需要保证是正数，即从 ey 中减去 400 年，加上 146097 天
    ey -= 400;
    cd += 146097;
  }
  // 同样的逻辑，从 d 中扣除掉所有可扣的 400 年，加到 ey 中
  ey += (d / 146097) * 400;
  // d 中余下的天数再加上 cd 得到所有余下的天数
  d = d % 146097 + cd;
  // 以下逻辑对 d 进行处理，将 d 减少到 146097 以下
  if (d > 0) {
    // 如果 d 大于 0
    if (d > 146097) {
      // 如果 d 大于 146097 ，我们知道 d 和 cd 之前都应该小于 146097 ，因此加起来最多不会超过 2 * 146097
      // 扣掉 400 年对应的天数， ey 加上 400 年
      ey += 400;
      d -= 146097;
    }
  } else {
    // 否则说明 d 小于 0
    if (d > -365) {
      // We often hit the previous year when stepping a civil time backwards,
      // so special case it to avoid counting up by 100/4/1-year chunks.
      // 如果 d 比 -365 要大
      // 这里是一个特殊优化，因为使用的时候经常看能需要看上一年，比如去年的今天，这时我们走一个特殊路径
      // 避免走到后面的各种复杂逻辑
      // ey 减 1 往前一年
      ey -= 1;
      // 计算前一年这个月距离今年有几天， d 补上该年的天数
      d += days_per_year(ey, m);
    } else {
      // 否则，需要将 d 转为正数， ey 扣掉 400 年， d 加上 146097 天
      ey -= 400;
      d += 146097;
    }
  }
  // 到这里 d 应该都是 146097 以下了，同时也保证是正数
  if (d > 365) {
    // 如果 d 大于 365 ，则说明肯定超过 1 年
    // 计算 ey 这个月在 400 年周期里对应的位置（index），其实就是方便后面打表，计算从 ey 往后走经过多少天
    int yi = year_index(ey, m);  // Index into Gregorian 400 year cycle.
    // 循环处理，每次计算是否可以再往后一个世纪（100 年）
    for (;;) {
      // 计算 yi 往后 100 年（一个世纪）过去了多少天
      int n = days_per_century(yi);
      // 如果 d 小于 n 说明没法往后前进 100 年了，跳出循环
      if (d <= n) break;
      // d 减去 n ，也就是过去 100 年的天数
      d -= n;
      // ey 往后 100 年，即一个世纪
      ey += 100;
      // yi 也需要增加 100
      yi += 100;
      // 如果 yi >= 400 了再将下标对齐到 400 以内
      if (yi >= 400) yi -= 400;
    }
    // 接下来循环处理，每次往前 4 年
    for (;;) {
      // 计算往后 4 年会过去多少天
      int n = days_per_4years(yi);
      // 如果 d 小于 n 则跳出去
      if (d <= n) break;
      // d 减去 n ，即 4 年的天数
      d -= n;
      // ey 往后走 4 年
      ey += 4;
      // yi 往后 4 年
      yi += 4;
      // 对齐到 400 内
      if (yi >= 400) yi -= 400;
    }
    // 接下来每次往前 1 年
    for (;;) {
      int n = days_per_year(ey, m);
      if (d <= n) break;
      d -= n;
      ++ey;
    }
  }
  // 到这里 d 已经小于 365 天了
  if (d > 28) {
    // 如果 d > 28 则超过了一个月，按月往前进
    for (;;) {
      // 计算 ey 这个月有多少天
      int n = days_per_month(ey, m);
      // 如果 d 已经小于 n 了，说明 d 已经位于这一月内了
      if (d <= n) break;
      // 减去这个月的天数
      d -= n;
      // 月份递增，如果超过 12 月则增加一年
      if (++m > 12) {
        ++ey;
        m = 1;
      }
    }
  }
  // 通过前面的计算，我们计算出 y 的增量并加到了 ey 中，因此通过 ey - oey 就能拿到实际
  // 应该累加多少年到 y ，然后构造出最终的 fields
  return fields(y + (ey - oey), m, static_cast<day_t>(d), hh, mm, ss);
}
```

而 `n_day()` 的计算也依赖了若干辅助函数和打表逻辑
```c++
// 是否闰年
CONSTEXPR_F bool is_leap_year(year_t y) noexcept {
  return y % 4 == 0 && (y % 100 != 0 || y % 400 == 0);
}
CONSTEXPR_F int year_index(year_t y, month_t m) noexcept {
  // 如果超过 2 月则 y 需要 + 1 ，这个处理逻辑和 days_per_year() 一致，这里还不太理解为啥 ?
  // 可能主要是一种打表的逻辑，方便后面计算从当年当月到现在过去了多少天，因此就跟计算 days_per_year 一样，考虑 2 月
  // y 计算模 400 后余下多少，得到 yi
  const int yi = static_cast<int>((y + (m > 2)) % 400);
  // yi 保证是正数，如果小于 0 会加上 400 ，因为是下标，负数直接加 400 得到正数的下标即可
  return yi < 0 ? yi + 400 : yi;
}
// 这里最早是一个数组，直接根据 yi 作为下标取数组里的值，里面的值只会是 0 或者 1
// 当时注释如下
// > The number of days in the 100 years starting in the mod-400 index year,
// > stored as a 36524-deficit value (i.e., 0 == 36524, 1 == 36525).
// 这里改为用 const fn 实现了
CONSTEXPR_F int days_per_century(int yi) noexcept {
  return 36524 + (yi == 0 || yi > 300);
}
// 同理是计算往后走 4 年过去了多少天
// > The number of days in the 4 years starting in the mod-400 index year,
// > stored as a 1460-deficit value (i.e., 0 == 1460, 1 == 1461).
// 这里也是用 const fn 实现了
CONSTEXPR_F int days_per_4years(int yi) noexcept {
  return 1460 + (yi == 0 || yi > 300 || (yi - 1) % 100 < 96);
}
// 计算从 y 这年 m 月到下一年这个月过了多少天，需要考虑月份
CONSTEXPR_F int days_per_year(year_t y, month_t m) noexcept {
  // 如果 2 月以后，则 m > 2 是 1 ， y + 1 即今年是闰年的话，就过了 366 天
  return is_leap_year(y + (m > 2)) ? 366 : 365;
}
CONSTEXPR_F int days_per_month(year_t y, month_t m) noexcept {
  CONSTEXPR_D int k_days_per_month[1 + 12] = {
      -1, 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31  // non leap year
  };
  return k_days_per_month[m] + (m == 2 && is_leap_year(y));
}
```

在构造 civil_time 的过程中，还会将 fields 对齐到 ciliv_time 的实际精度
```c++
// Aligns the (normalized) fields struct to the indicated field.
CONSTEXPR_F fields align(second_tag, fields f) noexcept {
  return f;
}
CONSTEXPR_F fields align(minute_tag, fields f) noexcept {
  return fields{f.y, f.m, f.d, f.hh, f.mm, 0};
}
CONSTEXPR_F fields align(hour_tag, fields f) noexcept {
  return fields{f.y, f.m, f.d, f.hh, 0, 0};
}
CONSTEXPR_F fields align(day_tag, fields f) noexcept {
  return fields{f.y, f.m, f.d, 0, 0, 0};
}
CONSTEXPR_F fields align(month_tag, fields f) noexcept {
  return fields{f.y, f.m, 1, 0, 0, 0};
}
CONSTEXPR_F fields align(year_tag, fields f) noexcept {
  return fields{f.y, 1, 1, 0, 0, 0};
}
```

至此大致理清了构造出一个标准的 civil_time 的处理逻辑

## 自增自减
自增自减的逻辑都是通过 step 实现的
```c++
friend CONSTEXPR_F civil_time operator+(civil_time a, diff_t n) noexcept {
  // 加操作通过 step 实现，进行 step 时会传入 T 来控制精度
  return civil_time(step(T{}, a.f_, n));
}
friend CONSTEXPR_F civil_time operator+(diff_t n, civil_time a) noexcept {
  return a + n;
}
friend CONSTEXPR_F civil_time operator-(civil_time a, diff_t n) noexcept {
  // 减操作实际上可以转换为反过来 step() ，这里考虑了 min 转为正数会溢出，因此当等于 min 时需要特殊处理
  return n != (std::numeric_limits<diff_t>::min)()
             ? civil_time(step(T{}, a.f_, -n))
             : civil_time(step(T{}, step(T{}, a.f_, -(n + 1)), 1));
}
```

而 step 实际上就是重新构造一次 fields
```c++
// Increments the indicated (normalized) field by "n".
// step() 会对 fields 进行递增
CONSTEXPR_F fields step(second_tag, fields f, diff_t n) noexcept {
  // 递增 n 秒实际上重新根据 n_sec() 计算 fields
  return impl::n_sec(f.y, f.m, f.d, f.hh, f.mm + n / 60, f.ss + n % 60);
}
CONSTEXPR_F fields step(minute_tag, fields f, diff_t n) noexcept {
  return impl::n_min(f.y, f.m, f.d, f.hh + n / 60, 0, f.mm + n % 60, f.ss);
}
CONSTEXPR_F fields step(hour_tag, fields f, diff_t n) noexcept {
  return impl::n_hour(f.y, f.m, f.d + n / 24, 0, f.hh + n % 24, f.mm, f.ss);
}
CONSTEXPR_F fields step(day_tag, fields f, diff_t n) noexcept {
  // 递增 n 天直接将 n 作为 cd 传入 n_day() 即可
  return impl::n_day(f.y, f.m, f.d, n, f.hh, f.mm, f.ss);
}
CONSTEXPR_F fields step(month_tag, fields f, diff_t n) noexcept {
  return impl::n_mon(f.y + n / 12, f.m + n % 12, f.d, 0, f.hh, f.mm, f.ss);
}
CONSTEXPR_F fields step(year_tag, fields f, diff_t n) noexcept {
  return fields(f.y + n, f.m, f.d, f.hh, f.mm, f.ss);
}
```

## 差值
差值的计算通过 difference() 实现
```c++
friend CONSTEXPR_F diff_t operator-(civil_time lhs, civil_time rhs) noexcept {
  return difference(T{}, lhs.f_, rhs.f_);
}
```

difference() 对于不同精度有不同实现
```c++
// Returns the difference between fields structs using the indicated unit.
CONSTEXPR_F diff_t difference(year_tag, fields f1, fields f2) noexcept {
  // 年之间直接计算差值
  return f1.y - f2.y;
}
CONSTEXPR_F diff_t difference(month_tag, fields f1, fields f2) noexcept {
  // 计算 f1 和 f2 年之间的差距，换算成月，然后再加上月之间的差距
  return impl::scale_add(difference(year_tag{}, f1, f2), 12, (f1.m - f2.m));
}
CONSTEXPR_F diff_t difference(day_tag, fields f1, fields f2) noexcept {
  // 调用 day_difference() 计算差距的天数，天数的差距计算麻烦写，需要专门的函数
  return impl::day_difference(f1.y, f1.m, f1.d, f2.y, f2.m, f2.d);
}
CONSTEXPR_F diff_t difference(hour_tag, fields f1, fields f2) noexcept {
  // 计算差距的天数并换算成小时，然后再加上小时之间的差
  return impl::scale_add(difference(day_tag{}, f1, f2), 24, (f1.hh - f2.hh));
}
CONSTEXPR_F diff_t difference(minute_tag, fields f1, fields f2) noexcept {
  // 计算差距的小时数并换算成分钟，然后再加上分钟之间的差
  return impl::scale_add(difference(hour_tag{}, f1, f2), 60, (f1.mm - f2.mm));
}
CONSTEXPR_F diff_t difference(second_tag, fields f1, fields f2) noexcept {
  // 计算差距的分钟数并换算成秒数，然后再加上秒之间的差
  return impl::scale_add(difference(minute_tag{}, f1, f2), 60, f1.ss - f2.ss);
}
```

`scale_add()` 就是用于计算 `(v * f + a)` 的
```c++
// Returns (v * f + a) but avoiding intermediate overflow when possible.
// 实际上就是计算 (v * f + a) ，不过用了一些技巧避免溢出
CONSTEXPR_F diff_t scale_add(diff_t v, diff_t f, diff_t a) noexcept {
  return (v < 0) ? ((v + 1) * f + a) - f : ((v - 1) * f + a) + f;
}
```

比较麻烦的是计算天之间的差值，主要实现见 `day_difference()`
```c++
// Returns the difference in days between two normalized Y-M-D tuples.
// ymd_ord() will encounter integer overflow given extreme year values,
// yet the difference between two such extreme values may actually be
// small, so we take a little care to avoid overflow when possible by
// exploiting the 146097-day cycle.
// 计算 y1/m1/d1 - y2/m2/d2
CONSTEXPR_F diff_t day_difference(year_t y1, month_t m1, day_t d1,
                                  year_t y2, month_t m2, day_t d2) noexcept {
  // 计算 y1/y2 模 400 年余下的部分， c4 应该是推测 4 century 的意思
  const diff_t a_c4_off = y1 % 400;
  const diff_t b_c4_off = y2 % 400;
  // 将 y1 和 y2 对齐到 400 年之后计算之间的差值到 c4_diff
  diff_t c4_diff = (y1 - a_c4_off) - (y2 - b_c4_off);
  // 用模 400 年余下的部分，通过 `ymd_ord()` 转换成天数然后计算天数之间的差值得到 delta
  diff_t delta = ymd_ord(a_c4_off, m1, d1) - ymd_ord(b_c4_off, m2, d2);
  if (c4_diff > 0 && delta < 0) {
    // 如果 c4_diff 大于 0 ， delta 小于 0 ，则说明是大减小需要将 delta 转为正数
    // 不过个人感觉这里和下面加 1 个 146097 应该就够了，这里为何是 2 倍呢 ???
    delta += 2 * 146097;
    c4_diff -= 2 * 400;
  } else if (c4_diff < 0 && delta > 0) {
    // 如果 c4_diff 小于 0 ， delta 大于 0 ，则说明是小减大，将 delta 转为负数
    delta -= 2 * 146097;
    c4_diff += 2 * 400;
  }
  // 用 c4_diff 和 delta 重新计算得到最终的差值
  return (c4_diff / 400 * 146097) + delta;
}
```

而 `day_difference()` 最复杂的地方就在 `ymd_ord()` 的实现，里面的原理也很精妙
```c++
// Map a (normalized) Y/M/D to the number of days before/after 1970-01-01.
// Probably overflows for years outside [-292277022656:292277026595].
// 将 y/m/d 转为 1970-01-01 以后的天数
//
// 这里的计算逻辑非常复杂，详细原理可以参考 https://howardhinnant.github.io/date_algorithms.html
// 实现还是非常精妙的，这里有一些需要注意的地方:
// > These algorithms internally assume that March 1 is the first day of the year.
// > This is convenient because it puts the leap day, Feb. 29 as the last day of the year,
// > or actually the preceding year. That is, Feb. 15, 2000, is considered by this algorithm
// > to be the 15th day of the last month of the year 1999.
//
// 一些概念
// - era: In these algorithms, an era is a 400 year period. As it turns out, the civil
// calendar exactly repeats itself every 400 years. And so these algorithms will first
// compute the era of a year/month/day triple, or the era of a serial date,
// and then factor the era out of the computation.
// - yoe: The year of the era (yoe). This is always in the range [0, 399].
// - doe: The day of the era. This is always in the range [0, 146096].
//
// era 和日期之间的规律
// ```
// era start date  end date
// -2  -0800-03-01 -0400-02-29
// -1  -0400-03-01 0000-02-29
//  0  0000-03-01  0400-02-29
//  1  0400-03-01  0800-02-29
//  2  0800-03-01  1200-02-29
//  3  1200-03-01  1600-02-29
//  4  1600-03-01  2000-02-29
//  5  2000-03-01  2400-02-29
// ```
// 可以看到对于非负的年份， era = y/400 ，而对于负数，则需要注意 [-400, -1] 对应 -1 的 era
CONSTEXPR_F diff_t ymd_ord(year_t y, month_t m, day_t d) noexcept {
  // 这里认为 3 月 1 日是一年的开始，所以这里做了处理，如果 m <=2 则 y - 1 得到 eyear
  const diff_t eyear = (m <= 2) ? y - 1 : y;
  // 计算 era ，这里负数减 399 保证计算得到正确的 era
  const diff_t era = (eyear >= 0 ? eyear : eyear - 399) / 400;
  // yoe 只要将 eyear 减下 era * 400 就能得到了，对照上面 era 和日期的规律还是不难理解的
  // 需要注意的是前面的计算方式保证这里 yoe 是 >= 0 的
  const diff_t yoe = eyear - era * 400;
  // 这个公式比较复杂，这里不做太多展开
  const diff_t doy = (153 * (m + (m > 2 ? -3 : 9)) + 2) / 5 + d - 1;
  // 拿到了 yoe (year-of-era) 和 doy (day-of-year) 之后，对于每年 * 365 ，然后处理下闰年，即每
  // 4 年 + 1 ，然后每 100 年 - 1
  const diff_t doe = yoe * 365 + yoe / 4 - yoe / 100 + doy;
  // 这里减 719468 是为了让时间从 1970-01-01 开始而不是从 0000-03-01 开始
  return era * 146097 + doe - 719468;
}
```

其中 doy 的计算公式直接看公式是很难理解的，需要结合[这里的解释](https://howardhinnant.github.io/date_algorithms.html)理解
```c++
const diff_t doy = (153 * (m + (m > 2 ? -3 : 9)) + 2) / 5 + d - 1;
```

首先我们需要一个公式将 03-01 映射到 0 ， 03-02 映射到 1 等等一直到一年的所有天数。这个计算可以分为几步
- 将月份的值 [1, 12] 即 1 - 12 月映射到 `[0, 11]` （3 月到 2 月）
- 计算从 03-01 到该月第一天的天数
- 加上当日离当月 1 号的天数

首先看下第二条，将月 m' （范围在 `[0, 11]` ， 3 到 2 月） 映射到一个数字，表示 m' 的第一天到 03-01 的天数
```
month	m'	days after m'-01
Mar	    0	0
Apr	    1	31
May	    2	61
Jun	    3	92
Jul	    4	122
Aug	    5	153
Sep	    6	184
Oct	    7	214
Nov	    8	245
Dec	    9	275
Jan	    10	306
Feb	    11	337
```
结果如上面所示，这样有一个好处是闰年的处理会变得很简单

这里问题会变成我们要寻找一个线性表达式 `a1 m' + a0` 来拟合这个趋势。这个斜率其实就是 337/11 = 30.63636363636364 。然而，计算机计算时的取整机制，因此事情变得复杂不少。我们实际上可以用一个单测来模拟这个场景
```c++
#include <cassert>

// Returns day of year for 1st day of month mp, mp == 0 is Mar, mp == 11 is Feb.
int
doy_from_month(int mp)
{
    return a1 * mp + a0;
}

int
main()
{
    int a[12] = {0, 31, 61, 92, 122, 153, 184, 214, 245, 275, 306, 337};
    for (int mp = 0; mp < 12; ++mp)
        assert(doy_from_month(mp) == a[mp]);
}
```

不管 a1 取 30 还是 31 ，我们实际上都找不到一个合适的 a0 来让这个单测通过。但是，如果函数可以改为，我们可以对 y 轴的截距做更方便的调整了，例如只要 b0 比 a0 小，则 mp 是 0 的时候结果还是可以是 0
```c++
return (b1 * mp + b0) / a0;
```

首先一个比较直观的是试下 b1 == 337 和 a0 == 11 (毕竟上面用他们算了斜率)，不过这个组合找不到合适的 b0 。不过如果 b1 == 306 ， a0 == 10 的话 (306/10 = 30.63636363636364) 我们可以找到两个符合条件的 b0
```c++
return (306 * mp + 4) / 10;
return (306 * mp + 5) / 10;
```

而第一个实际上可以简化为
```c++
return (153 * mp + 2) / 5;
```

文章作者经过测试之后发现第一个公式要快一些（快 2%） ，因此选了第一个

然后接下来是将 m' 映射到一个月份的数字，其中从 3 月开始，即 3 月表示 0 ，这个可以通过加 9 模 12 计算得到
```c++
mp = (m + 9) % 12;
```

然后作者发现上面的写法改写成这样来避免 % 操作要稍微快一些
```c++
mp = m + (m > 2 ? -3 : 9);
```

上面组合起来得到下面的公式，可以计算得到一个月份距离 03-01 多少天
```
(153*(m + (m > 2 ? -3 : 9)) + 2)/5
```

然后加上在该月的天数再减 1 （因为 1 号在前面计算距离 03-01 天数时已经纳入计算了）就能得到 doy 了
```c++
const unsigned doy = (153*(m + (m > 2 ? -3 : 9)) + 2)/5 + d - 1;
```

