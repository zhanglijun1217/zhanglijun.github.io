---
title: java8中的LocalDateTime
copyright: true
date: 2018-11-11 21:51:44
tags:
	- Java8
categories:
	- Java语法
	- Java8
---

## 背景

最近在项目中遇到了一些时间进行转化的小需求，比如一个时间添加多少天之后，两个时间的比较之类的。这里要去了解一下java8中的新增的时间API--LocalDateTime。

 参考博客：[](http://rensanning.iteye.com/blog/2034622)

<!--more-->

## 一些用法

### 系统时间

```java
// now方法获取系统时间
LocalDate date = LocalDate.now();
// getMonth：英文  getMonthValue : 数字
System.out.println(date.getYear() + "/" + date.getMonthValue() + "/" + date.getDayOfMonth());

LocalTime time = LocalTime.now();
System.out.println(time.getHour() + ":" + time.getMinute() + ":" + time.getSecond());

// 没有提供 getMiles()方法 可以这样获取mile
System.out.println(time.get(ChronoField.MILLI_OF_SECOND));

// 日期和时间
LocalDateTime dateTime = LocalDateTime.now();
System.out.println(dateTime.getYear() + "/" + dateTime.getMonthValue() + "/" + dateTime.getDayOfMonth()
        + " " + dateTime.getHour() + ":" + dateTime.getMinute() + ":" + dateTime.getSecond());

// 时区 获取时间戳
Clock clock = Clock.systemDefaultZone();
System.out.println(clock.millis());
```

### 特定日期

```java
// of方法获取特定日期
LocalDate myDate = LocalDate.of(2018, 11, 6);
System.out.println(myDate.getYear() + "/" + myDate.getMonthValue() + "/" + myDate.getDayOfMonth());

// 获取特定日期对应的属性
LocalDate independenceDay = LocalDate.of(2018, Month.JUNE, 4);
// 获取周几
System.out.println(independenceDay.getDayOfWeek());

// 构造LocalTime
LocalTime myTime = LocalTime.of(10, 30, 45);
System.out.println(myTime.getHour() + ":" + myTime.getMinute() + ":" + myTime.getSecond());

// 同样，LocalDateTime也是可以通过of方法创建特定日期
LocalDateTime myDateTime = LocalDateTime.of(2018, Month.JUNE, 4, 10, 30, 45);
System.out.println(myDateTime.getYear() + "/" + myDateTime.getMonthValue() + "/" + myDateTime.getDayOfMonth()
+ " " + myDateTime.getHour() + ":" + myDateTime.getMinute() + ":" + myDateTime.getSecond());

// 也提供了LocalDate 和 LocalTime组合而成的LocalDateTime
LocalDateTime myDateTime2 = LocalDateTime.of(myDate, myTime);
System.out.println(myDateTime2.getYear() + "/" + myDateTime2.getMonthValue() + "/" + myDateTime2.getDayOfMonth()
        + " " + myDateTime2.getHour() + ":" + myDateTime2.getMinute() + ":" + myDateTime2.getSecond());
```

### 格式化

```java
 */
// date --> String
LocalDate formatDate1 = LocalDate.of(2014, 3, 3);
String dateString = formatDate1.format(DateTimeFormatter.ofPattern("yyyy/MM/dd"));
System.out.println(dateString);

// String --> date
LocalDate formatDate2 = LocalDate.parse(dateString, DateTimeFormatter.ofPattern("yyyy/MM/dd"));
System.out.println(formatDate2);
```

### 日期转换

```java
// LocalDate --> LocalDateTime
LocalDate changeDate = LocalDate.of(2018, 12, 4);
LocalDateTime changeDateTime = changeDate.atTime(10, 20, 30);
System.out.println(changeDateTime);
// LocalTime --> LocalDateTime
LocalTime changeTime = LocalTime.of(10, 20, 30);
LocalDateTime changeDateTime2 = changeTime.atDate(LocalDate.of(2018, 12, 4));
System.out.println(changeDateTime2);
// LocalDateTime --> LocalDate,LocalTime 有这样的api toLocalDate  toLocalTime
System.out.println(changeDateTime.toLocalDate());
System.out.println(changeDateTime2.toLocalTime());
```

### 日期加减

```java
LocalDate now = LocalDate.now();
// 2天后
System.out.println(now.plusDays(2L));
// 3天前
System.out.println(now.minusDays(3L));
// 一年后
System.out.println(now.plusYears(1));

// 2周前
System.out.println(now.minus(2L, ChronoUnit.WEEKS));
// 3年2月1天后
System.out.println(now.plus(Period.of(3, 2, 1)));
```

### 计算间隔

```java
LocalDateTime before = LocalDateTime.of(2011, 2, 11, 11, 11, 11);
LocalDateTime after = LocalDateTime.of(2014, 2, 11, 11, 11, 11);

// Duration 来表示间隔
Duration between = Duration.between(before, after);
// 间隔的天
System.out.println("间隔：" + between.toDays());
// 间隔的分钟
System.out.println("间隔：" + between.toMinutes());
```

### 日期比较

```java
LocalDate compareDate1 = LocalDate.of(2011, 1, 1);
LocalDate compareDate2 = LocalDate.of(2012, 1, 1);
System.out.println(compareDate1.isBefore(compareDate2));
int i = compareDate1.compareTo(compareDate2);
System.out.println("c1 compareTo c2 is " + i);
```

### 和java.util.Date的转换

```java
// LocalDateTime --> Instant --> Date
LocalDateTime currentLocalDateTime = LocalDateTime.now();
Instant instant = currentLocalDateTime.atZone(ZoneId.systemDefault()).toInstant();
Date date1 = Date.from(instant);
System.out.println(date1);

// Date --> Instant --> LocalDateTime
Date date2 = new Date();
Instant instant1 = date2.toInstant();
LocalDateTime dateTime1 = LocalDateTime.ofInstant(instant1, ZoneId.systemDefault());
System.out.println(dateTime1);

// Calendar --> Instant --> LocalDateTime
Calendar calendar = Calendar.getInstance();
Instant instant2 = calendar.toInstant();
LocalDateTime dateTime2 = LocalDateTime.ofInstant(instant2, ZoneId.systemDefault());
System.out.println(dateTime2);
```