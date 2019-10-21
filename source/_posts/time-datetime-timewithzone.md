---
title: "Time, DateTime and TimeWithZone"
date: 2018-03-03 12:39:10
categories:
  - tech
---

如果你开始在 Rails 中开始处理时间，这显然是在实现一个应用中无法避免的，你一定很快就会遇到第一个问题，Rails 中有三个可以表示时间的类 - Time，DateTime and ActiveSupport::TimeWithZone, 他们之间有什么不同？

当然，让我们从一些老生常谈开始

## Time vs. DateTime

相对于 TimeWithZone 随 Ruby 一同安装后开箱即用的 Time 和 DateTime 可能是你第一个使用过的时间相关的类，你应该和我一样困惑过，他们有什么区别，在应用中我应该使用哪一个？

地位上，Time 来自 Ruby 核心库(core)，无需`require`加载即可使用，DateTime 属于 Date 的子类，一同来自于 Ruby 标准库(std-lib)，需要额外`require 'date'`。

如果查阅到一些过时的资料，可能会看到诸如 “Time 是对 [POSIX time](https://en.wikipedia.org/wiki/Unix_time) 的简单封装，因此只能表示`1970-01-01 00:00:00 +00:00`之后的时间”，但实际上自 Ruby 1.9.2，Time 可以表示时间不在受到限制。

> Since Ruby 1.9.2, Time implementation uses a signed 63 bit integer, Bignum or Rational. The integer is a number of nanoseconds since the Epoch which can represent 1823-11-12 to 2116-02-20. When Bignum or Rational is used (before 1823, after 2116, under nanosecond), Time works slower as when integer is used.

在日常使用上 Time 与 DateTime 没有太大的差别，可以处理历史中的年月日，时分秒，星期和时区，已经能涵盖大多是的使用案例，但倘若你要同时考虑一些历史上的历法转变 DateTime 将是你的不二之选，以下是来自 API 文档中的一段示例，如果你看完之后震惊之余，苦思冥想也无法参透其中缘由，别想那么多，那说明你根本用不上他。

```ruby
shakespeare = DateTime.iso8601('1616-04-23', Date::ENGLAND)
 #=> Tue, 23 Apr 1616 00:00:00 +0000
cervantes = DateTime.iso8601('1616-04-23', Date::ITALY)
 #=> Sat, 23 Apr 1616 00:00:00 +0000
```

## DateTime vs. TimeWithZone

ActiveSupport::TimeWithZone（以下简称 TimeWithZone）作为 ActiveRecord 中 datetime 数据库类型的默认对应类型，旨在让 Rails 开发者更方便的处理时区。尽管 Time 和 DateTime 同样保存了时区的信息，但仅依赖于系统的`ENV['TZ']`环境变量，最多也仅可以切换至 GMT 或 UTC。[^1] 仅可以应付本地化网站、应用的使用场景。一旦涉及国际化场景，多时区的时间处理，Ruby 内置的时间库便显得不太够用。

[^1]: GMT 格林威治标准时间，UTC 协调世界时，在大多数用途上两者并没有区别，Ruby 中的 Time/DateTime 也将两者一视同仁。

大多时候你可能根本不会注意到自己正在处理的是 TimeWithZone 还是 DateTime，经过 Rails 对 DateTime 的扩充后，他们有极其相似的实例方法，都可以进行时间区间的加减，也可以实现互相转换。但若因此判断无需纠结 Rails 中时间类型具体的实现，则有些天真了。

上文提到，两者同样可以处理时间区间的加减，但这其中还是有微妙的区别，当计算不确定长度的时间区间（days, months, years 等）时，夏令时。DateTime 不会考虑任何夏令时规则，因此使用 DateTime 实例加减这些时间区间，即使跨过 夏令时边界，其时间部分及时区部分（%T %z）也不会变化。相应的，当 TimeWithZone 实例的时区处在有夏令时规则的时区时，在加减时间区间中如果跨越了夏令时边界，时区部分会相应的变化，已达到与人类直觉上的一致。

如果应用中对时间区间计算有精确度要求的话，这个问题就格外需要重视了。他甚至会让你不知不觉得中招，如上文所说，ActiveRecord 对 datetime 数据库类型的对应类型为 TimeWithZone，以下代码我们假设一个 periods 表，记录了一段时间的开始时间和结束时间，并同时假设，某条记录中，根据当前 app 时区，这个开始时间和结束时间跨越了夏令时边界。那么因为 DateTime 和 TimeWithZone 计算上的不统一，在存入表前的计算结果，和从表中取出后重新计算的结果，显然是不一致的。

```ruby
start_time = DateTime.current
end_time = start_time + 1.month

Period.create!(start_time: start_time, end_time: end_time)
period = Period.last

period.end_time == period.start_time + 1.month # => not sure!
```

想避免以上情况有以下可选措施：

1. 避免自然日，自然月的时间区间计算，尽可能使用准确的时分秒。（推荐）
2. 在统一的时区下，使用统一的类型计算，如进行计算前统一转换为 DateTime 的 utc 时间，根据显示的需要再对时区进行必要的转换。

## ENV['TZ'] vs. config.time_zone

结论性而言，`Time.now`和`DateTime.now`依赖于`ENV['TZ']`所设定的时区，而从 DB 中取出的 TimeWithZone, 以及 ActiveSupport 引入的`Time.current`和`DateTime.current`得到的时间的默认时区，是依赖于由`config.time_zone`所设定的应用时区。

通常而言，虽然时区不同，但两者所对应的时间应该是统一的，同时切换为 UTC 时间后是完全相等的。但不同时区在一定程度上却会造成一些意想不到的混乱。以下是我所遇到过的一个由时区不同所产生的 bug。我们假设现在处于 2018-03-01 10:00:00 +0800，系统时区为上海时区（+0800），应用时区为太平洋时区（-0800），并再次引入我们之前定义过的 periods 表。

```ruby
start_time = DateTime.now
end_time = start_time + 2.years

Period.create!(start_time: start_time, end_time: end_time)
period = Period.last

period.starts_at.to_datetime + 2.years == period.ends_at.to_datetime # => false, 1 day difference
```

如果我说这个 bug 大多数情况下只会四年发生一次，你可能很快就能找到方向。
实际原因也确实很傻，因为按照上海时间 2018 年 3 月 1 号前进 2 年为 2020 年 3 月 1 号，而当经过 DB 的储存读取，开始时间的时区根据应用时区，被转化为 -0800，即使转为 DateTime 也不会有所改变。时区转变后，日期由 2018 年 3 月 1 日相应的变为了 2018 年 2 月 28 日，这实际上也没有任何不妥。但当再次前进 2 年后，时间变为 2020 年 2 月 28 日，看似依然没有任何问题，但 2020 年是闰年，2 月 28 日到 3 月 1 日之间，有别于 2018 年，多了 2 月 29 日这一天，这一天的时间差也是由此而来。

解决方案事实上前文也有提到的，如果你对时间计算有严格的要求，还是尽量避免进行自然日，自然月，自然年这一类不确定时间的计算。如果一定需要，也尽量统一在 utc 时间下进行计算，再根据时间显示的需求转换时区。
