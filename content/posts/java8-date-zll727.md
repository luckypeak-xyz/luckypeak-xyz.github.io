---
title: Java8 Date
slug: java8-date-zll727
url: /post/java8-date-zll727.html
date: '2024-04-20 08:47:40'
lastmod: '2024-04-20 15:19:43'
toc: true
isCJKLanguage: true
---



https://www.baeldung.com/java-8-date-time-intro

## 概述

Java8 之前使用 `java.util.Date`​ 和 `java.util.Calender`​ 来处理时间。

Java8 引入了新的日期和时间 API，由 `java.time`​ 包下的 LocalDate、LocalTime、LocalDateTime、ZonedDateTime、Period、Duration 等类支持。解决了之前 Date/Time API 的一些问题：

* Date 和 Calendar 不是线程安全的
* Date 和 Calendar API 无法支持常见时间操作
* 时区逻辑处理繁琐

关于 Date 和 Calendar 使用就不展开了，直接用 Java8 引入的新版时间 API 即可。

## LocalDate, LocalTime 和 LocalDateTime

当不需要显式指定时区时使用这些类。

### LocalDate

LocalDate 表示 ISO 格式（yyyy-MM-dd）的日期，不包括时间。

```java
System.out.println(LocalDate.now()); // 2024-04-20
```

使用 of 或 parse 获取指定年、月、日的 LocalDate。

```java
System.out.println(LocalDate.of(2024, 3, 31)); // 2024-03-31
```

明天。

```java
System.out.println(LocalDate.now().plusDays(1)); // 2024-04-21
```

上个月的同一天。

```java
System.out.println(LocalDate.of(2024, 5, 31).minus(1, ChronoUnit.MONTHS)); // 2024-04-30
System.out.println(LocalDate.of(2024, 5, 30).minusMonths(1)); // 2024-04-30
System.out.println(LocalDate.of(2024, 3, 31).minusMonths(1)); // 2024-02-29
```

如果当月比上月长，当月多出来的都会对应当上月的最后一天。

‍
