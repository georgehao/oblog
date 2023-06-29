---
title: "当 go-sql-driver/mysql 遇到 mysql timestamp 的离奇 bug"
date: 2022-12-08T21:27:21+08:00
authors: hhf
tags: [Go, mysql, o-sql-driver/mysql] 
categories: [Go, mysql, go-sql-driver/mysql]
toc: true
---

hi，大家好，我是 haohongfan。

对于 Go CURD-Boy 来说，相信 `github.com/go-sql-driver/mysql` 这个库都不会陌生。或许有些人可能没太留意，直接就复制粘贴了 import。比如我们使用 gorm 的时候，如果不加 `_ "github.com/go-sql-driver/mysql"` 的话，就会报：`panic: sql: unknown driver "mysql" (forgotten import?)`。基本上 Go 的 CURD 都离不开这个特别重要的库。

不过最近在使用 go-sql-driver/mysql 查询 mysql 的时候，就出现一个很有意思的 bug, 这里分享出来给大家看看。

## Demo 准备

### 数据库准备

```sql
CREATE TABLE `Test1` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `create_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=101 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```
丛这个 sql 语句中可以看出来， create_time 是 `timestamp` 类型，这里要特别留意 timestamp 这个类型。

现在插入一条数据，然后查看刚插入的数据的值。

```sql
insert into Test1 values (1, '2022-01-01 00:00:00')
```

查看下 msyql 当前的时区：

```sql
 show VARIABLES like '%time_zone%';
```

结果是：

| Variable_name | Value |
| :--  | :-- | 
| system_time_zone | CST |
| time_zone | +08:00 |

接下来使用 mysql unix_timestamp 查看 create_time 的时间戳

```sql
SELECT unix_timestamp(create_time) from Test1 where id = 1;
```

结果是：

| unix_timestamp(create_time) | 
| :--: | 
| 1640966400 |


### 使用 go-sql-driver 读取 create_time 的值

```go
package main

import (
	"database/sql"
	"fmt"
	_ "github.com/go-sql-driver/mysql"
	"time"
)

func main() {
  var user = "user"
  var pwd = "password"
  var dbName = "dbname"
  dsn := fmt.Sprintf("%s:%s@tcp(localhost:3306)/%s?timeout=100s&parseTime=true&interpolateParams=true", user, pwd, dbName)
  db, err := sql.Open("mysql", dsn)
  if err != nil {
    panic(err)
  }
  defer db.Close()

  rows, err := db.Query("select create_time from Test1 limit 1")
  if err != nil {
    panic(err)
  }
  for rows.Next() {
    t := time.Time{}
    rows.Scan(&t)
    fmt.Println(t)
    fmt.Println(t.Unix())
  }
}
```

我们运行个程序会输出下面的结果:

```
2022-01-01 00:00:00 +0000 UTC
1640995200
```

你发现问题所在了吗, 可能顺着看下来不太直观，我用一张图把结果粘在一块就看清楚了。

![](https://cdn.jsdelivr.net/gh/georgehao/img/20221208221921.png)

看图中红色箭头指向的两个结果，用 go-sql-driver 读取的结果和在 mysql 中用 unix_timestamp 获取的结果明显是不一样的。

通过截图可以看出来，现在 create_time 在数据库的 `2022-01-01 00:00:00` 是东八区的时间，也就是北京时间，这个时间对应的时间戳就是 1640966400. 但是 go 程序读出来的却不是, go-sql-driver 读取到 1640995200 是什么呢？其实细心的小伙伴已经看出来了，这是 0 时区的 2022-01-01 00:00:00。

再直接点，mysql 的 create_time 是 `2022-01-01 00:00:00 +008`，而读取到的是 `2022-01-01 00:00:00 +000`，他俩压根就不是一个值。

基本能看出来 bug 是如何发生的了。下面接着剖析下 go-sql-driver 源码，看看是哪里导致的结果的错误。

## go-sq-driver 源码

源码我就不贴详细的源码了，只贴关键的路径，有兴趣的小伙伴可以顺着这个路径去看。

![](https://cdn.jsdelivr.net/gh/georgehao/img/go-sql-driver-callpath.jpg)

我们详细关注调用路径中红色的两个方块的内存中的值。
![](https://cdn.jsdelivr.net/gh/georgehao/img/go-sql-driver-source.jpg)

```go 
// https://github.com/go-sql-driver/mysql/blob/master/packets.go#L788-L798

func (rows *textRows) readRow(dest []driver.Value) error {
	
	// ... 

	// Parse time field
	switch rows.rs.columns[i].fieldType {
	case fieldTypeTimestamp,
		fieldTypeDateTime,
		fieldTypeDate,
		fieldTypeNewDate:
		if dest[i], err = parseDateTime(dest[i].([]byte), mc.cfg.Loc); err != nil {
			return err
		}
	}
}
```

```go
func parseDateTime(b []byte, loc *time.Location) (time.Time, error) {
	const base = "0000-00-00 00:00:00.000000"
	switch len(b) {
	case 10, 19, 21, 22, 23, 24, 25, 26: // up to "YYYY-MM-DD HH:MM:SS.MMMMMM"

		year, err := parseByteYear(b)

		month := time.Month(m)

		day, err := parseByte2Digits(b[8], b[9])

		hour, err := parseByte2Digits(b[11], b[12])

		min, err := parseByte2Digits(b[14], b[15])

		sec, err := parseByte2Digits(b[17], b[18])

		// https://github.com/go-sql-driver/mysql/blob/master/utils.go#L166-L168
		if len(b) == 19 {
			return time.Date(year, month, day, hour, min, sec, 0, loc), nil
		}
	}
}
```

看到这里基本上就能明白，go-sql-driver 把数据库读出来的 create_time timestamp 值当做一个字符串，然后按照 mysql timestamp 的标准格式 "0000-00-00 00:00:00.000000" 去解析，分别得到 year, month, day, hour, min, sec。最后依赖传入 time.Location 值，调用 go 系统库 time.Date() 再去生成对应的值。

这里表面看起来没有问题，其实这里严重依赖了传入的 time.Location。这个 time.Location 是如何得到的呢？

通过阅读源码，可以明显的看出来，是通过解析传入的 DSN 的 Loc 获取。

https://github.com/go-sql-driver/mysql/blob/master/dsn.go#L467-L474
![](https://cdn.jsdelivr.net/gh/georgehao/img/go-sql-driver-loc.jpg)

那如果我们传入的 dsn 串不带 loc 时，Loc 就是 默认的 UTC 时区。
![](https://cdn.jsdelivr.net/gh/georgehao/img/go-sql-driver-default-loc.jpg)

### 读取数值不一致的原因分析

再回头看开头的程序，我们初始化 go-sql-driver 的 dsn 是 `user:password@tcp(localhost:3306)/dbname?timeout=100s&parseTime=true&interpolateParams=true`. 该 dsn 里面并不包含 loc 信息，所以 go-sql-driver 用使用了默认的 UTC 时区。 然后解析从 mysql 中获取的 timestamp 字段了，也就用默认的 UTC 时区去生成 Date，结果也就错了。

其实问题的主要原因就是：go-sql-driver 并没有按照数据库的时区去解析 timestamp 字段，而且依赖了开发者生成的 dsn 传入的 loc。当开发者传入的 loc 和 数据库的 time_zone 不匹配的时候，所有的 timestamp 字段都会解析错误。

有些人可能有疑问，如果 go-sql-driver 为什么不直接使用 mysql 的时区去解析 timestamp 呢，我觉得是一个bug。

## 结论

所以说，现阶段使用 go-sql-driver 的时候一定要特别注意，生成的 dsn 字符串一定要和数据的时区保持一致，不然的话就会导致 timestamp 字段解析错误。

