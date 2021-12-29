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

