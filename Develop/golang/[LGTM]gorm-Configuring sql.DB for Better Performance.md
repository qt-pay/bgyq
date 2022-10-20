## [LGTM]gorm-Configuring sql.DB for Better Performance

原文：https://www.alexedwards.net/blog/configuring-sqldb

There are a lot of good tutorials which talk about Go's [`sql.DB`](https://golang.org/pkg/database/sql/#DB) type and how to use it to execute SQL database queries and statements. But most of them gloss over the [`SetMaxOpenConns()`](https://golang.org/pkg/database/sql/#DB.SetMaxOpenConns), [`SetMaxIdleConns()`](https://golang.org/pkg/database/sql/#DB.SetMaxIdleConns) and [`SetConnMaxLifetime()`](https://golang.org/pkg/database/sql/#DB.SetConnMaxLifetime) methods — which you can use to configure the behavior of `sql.DB` and alter its performance.

In this post I'd like to explain exactly what these settings do and demonstrate the (positive and negative) impact that they can have.

## DB instance and Pool

Is a new connection pool created every time I call it or is each call to `gorm.Open()` sharing the same connection pool?

**TLDR:** yes, try to reuse the returned DB object.

[gorm.Open](https://github.com/jinzhu/gorm/blob/v1.9.12/main.go#L58) does the following: (more or less):

1. lookup the driver for the given dialect
2. call `sql.Open` to return a `DB` object
3. call `DB.Ping()` to force it to talk to the database

This means that one `sql.DB` object is created for every `gorm.Open`. Per the [doc](https://golang.org/pkg/database/sql/#DB), **this means one connection pool for each DB object.**

The returned DB is safe for concurrent use by multiple goroutines and maintains its own pool of idle connections. **Thus, the Open function should be called just once.** It is rarely necessary to close a DB.

## Open and idle connections

I'll begin with a little background.

A `sql.DB` object is a ***pool** of many database connections* which contains both 'in-use' and 'idle' connections. A connection is marked as in-use when you are using it to perform a database task, such as executing a SQL statement or querying rows. When the task is complete the connection is marked as idle.

When you instruct `sql.DB` to perform a database task, it will first check if any idle connections are already available in the pool. If one is available then Go will reuse this existing connection and mark it as in-use for the duration of the task. If there are no idle connections in the pool when you need one, then Go will create an additional new additional connection.

## The SetMaxOpenConns method

By default there's no limit on the number of open connections (in-use + idle) at the same time. But you can implement your own limit via the `SetMaxOpenConns()` method like so:

```go
// Initialise a new connection pool
// One connection pool for each DB object.

db, err := sql.Open("postgres", "postgres://user:pass@localhost/db")
if err != nil {
    log.Fatal(err)
}

// Set the maximum number of concurrently open connections (in-use + idle)
// to 5. Setting this to less than or equal to 0 will mean there is no 
// maximum limit (which is also the default setting).
db.SetMaxOpenConns(5)
```

In this example code the pool now has a maximum limit of 5 concurrently open connections. If all 5 connections are already marked as in-use and another new connection is needed, then the application will be forced to wait until one of the 5 connections is freed up and becomes idle.

To illustrate the impact of changing `MaxOpenConns` I ran a benchmark test with the maximum open connections set to 1, 2, 5, 10 and unlimited. The benchmark executes parallel `INSERT` statements on a PostgreSQL database and you can find the code in [this gist](https://gist.github.com/alexedwards/5d1db82e6358b5b6efcb038ca888ab07). Here's the results:

```
BenchmarkMaxOpenConns1-8         500  3129633 ns/op   478 B/op         10 allocs/op
BenchmarkMaxOpenConns2-8         1000 2181641 ns/op   470 B/op         10 allocs/op
BenchmarkMaxOpenConns5-8         2000 859654 ns/op    493 B/op         10 allocs/op
BenchmarkMaxOpenConns10-8        2000 545394 ns/op    510 B/op         10 allocs/op
BenchmarkMaxOpenConnsUnlimited-8 2000 531030 ns/op    479 B/op          9 allocs/op
PASS
```

***Edit:** To make clear, the purpose of this benchmark is not to simulate 'real-life' behaviour of an application. It's solely to help illustrate how `sql.DB` behaves behind the scenes and the impact of changing `MaxOpenConns` on that behaviour.*

For this benchmark we can see that the more open connections that are allowed, the less time is taken to perform the `INSERT` on the database (3129633 ns/op with 1 open connection compared to 531030 ns/op for unlimited connections — about 6 times quicker). This is because the more open connections that are permitted, the more database queries can be performed concurrently.

## The SetMaxIdleConns method

By default `sql.DB` allows a maximum of 2 idle connections to be retained in the connection pool. You can change this via the `SetMaxIdleConns()` method like so:

```go
// Initialise a new connection pool
db, err := sql.Open("postgres", "postgres://user:pass@localhost/db")
if err != nil {
    log.Fatal(err)
}

// Set the maximum number of concurrently idle connections to 5. Setting this
// to less than or equal to 0 will mean that no idle connections are retained.
db.SetMaxIdleConns(5)
```

In theory, allowing a higher number of idle connections in the pool will improve performance because it makes it less likely that a new connection will need to be established from scratch — therefore helping to save resources.

Lets take a look at the same benchmark with the maximum idle connections is set to none, 1, 2, 5 and 10 (and the number of open connections is unlimited):

```
BenchmarkMaxIdleConnsNone-8  300   4567245 ns/op       58174 B/op        625 allocs/op
BenchmarkMaxIdleConns1-8    2000   568765 ns/op        2596 B/op         32 allocs/op
BenchmarkMaxIdleConns2-8     2000  529359 ns/op         596 B/op         11 allocs/op
BenchmarkMaxIdleConns5-8     2000  506207 ns/op         451 B/op          9 allocs/op
BenchmarkMaxIdleConns10-8    2000  501639 ns/op         450 B/op          9 allocs/op
PASS
```

When `MaxIdleConns` is set to none, a new connection has to be created from scratch for each `INSERT` and we can see from the benchmarks that the average runtime and memory usage is comparatively high.

Allowing just 1 idle connection to be retained and reused makes a massive difference to this particular benchmark — it cuts the average runtime by about 8 times and reduces memory usage by about 20 times. Going on to increase the size of the idle connection pool makes the performance even better, although the improvements are less pronounced.

So should you maintain a large idle connection pool? The answer is *it depends on the application*.

It's important to realise that keeping an idle connection alive comes at a cost — it takes up memory which can otherwise be used for both your application and the database.

It's also possible that if a connection is idle for too long then it may become unusable. For example, MySQL's [`wait_timeout`](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_wait_timeout) setting will automatically close any connections that haven't been used for 8 hours (by default).

When this happens `sql.DB` handles it gracefully. Bad connections will automatically be retried twice before giving up, at which point Go will remove the connection from the pool and create a new one. So setting `MaxIdleConns` too high may actually result in connections becoming unusable and more resources being used than if you had a smaller idle connection pool (with fewer connections that are used more frequently). So really you only want to keep a connection idle if you're likely to be using it again soon.

One last thing to point out is that `MaxIdleConns` should always be less than or equal to `MaxOpenConns`. Go enforces this and will automatically reduce `MaxIdleConns` if necessary.

## The SetConnMaxLifetime method

Let's now take a look at the `SetConnMaxLifetime()` method which sets the maximum length of time that a connection can be reused for. This can be useful if your SQL database also implements a maximum connection lifetime or if — for example — you want to facilitate gracefully swapping databases behind a load balancer.

You use it like this:

```go
// Initialise a new connection pool
db, err := sql.Open("postgres", "postgres://user:pass@localhost/db")
if err != nil {
    log.Fatal(err)
}

// Set the maximum lifetime of a connection to 1 hour. Setting it to 0
// means that there is no maximum lifetime and the connection is reused
// forever (which is the default behavior).
db.SetConnMaxLifetime(time.Hour)
```

In this example all our connections will 'expire' 1 hour after they were first created, and cannot be reused after they've expired. But note:

- This doesn't guarantee that a connection will exist in the pool for a whole hour; it's quite possible that the connection will have become unusable for some reason and been automatically closed before then.
- A connection can still be in use more than one hour after being created — it just cannot ***start** to be reused* after that time.
- This isn't an idle timeout. The connection will expire 1 hour after it was first created — not 1 hour after it last became idle.
- Once every second a cleanup operation is automatically run to remove 'expired' connections from the pool.

In theory, the shorter `ConnMaxLifetime` is the more often connections will expire — and consequently — the more often they will need to be created from scratch.

To illustrate this I ran the benchmarks with `ConnMaxLifetime` set to 100ms, 200ms, 500ms, 1000ms and unlimited (reused forever), with the default settings of unlimited open connections and 2 idle connections. These time periods are obviously much, much shorter than you'd use in most applications but they help illustrate the behaviour well.

```
BenchmarkConnMaxLifetime100-8               2000        637902 ns/op        2770 B/op         34 allocs/op
BenchmarkConnMaxLifetime200-8               2000        576053 ns/op        1612 B/op         21 allocs/op
BenchmarkConnMaxLifetime500-8               2000        558297 ns/op         913 B/op         14 allocs/op
BenchmarkConnMaxLifetime1000-8              2000        543601 ns/op         740 B/op         12 allocs/op
BenchmarkConnMaxLifetimeUnlimited-8         3000        532789 ns/op         412 B/op          9 allocs/op
PASS
```

In these particular benchmarks we can see that memory usage was more than 3 times greater with a 100ms lifetime compared to an unlimited lifetime, and the average runtime for each `INSERT` was also slightly longer.

If you do set `ConnMaxLifetime` in your code, it is important to bear in mind the frequency at which connections will expire (and subsequently be recreated). For example, if you have 100 total connections and a `ConnMaxLifetime` of 1 minute, then your application can potentially kill and recreate up to 1.67 connections (on average) every second. You don't want this frequency to be so great that it ultimately hinders performance, rather than helping it.

100个请求connect，一个连接持续 1 minute/60 second，这就要求app 一秒钟要处理将近2个请求，这样才能避免由于connMaxLifetime设置时间过短导致，需要recreate connect，浪费机器性能。

## Exceeding connection limits

Lastly, this article wouldn't be complete without mentioning what happens if you exceed a hard limit on the number of database connections.

As an illustration, I'll change my `postgresql.conf` file so only a total of 5 connections are permitted (the default is 100)...

```
max_connections = 5
```

And then rerun the benchmark test with unlimited open connections...

```
BenchmarkMaxOpenConnsUnlimited-8    --- FAIL: BenchmarkMaxOpenConnsUnlimited-8
    main_test.go:14: pq: sorry, too many clients already
    main_test.go:14: pq: sorry, too many clients already
    main_test.go:14: pq: sorry, too many clients already
FAIL
```

As soon as the hard limit of 5 connections is hit my database driver ([pq](https://github.com/lib/pq)) immediately returns a `sorry, too many clients already` error message instead of completing the `INSERT`.

To prevent this error we need to set the *total* maximum of open connections (in-use + idle) in `sql.DB` to comfortably below 5. Like so:

```go
// Initialise a new connection pool
db, err := sql.Open("postgres", "postgres://user:pass@localhost/db")
if err != nil {
    log.Fatal(err)
}

// Set the number of open connections (in-use + idle) to a maximum total of 3.
db.SetMaxOpenConns(3)
```

Now there will only ever be a maximum of 3 connections created by `sql.DB` at any moment in time, and the benchmark should run without any errors.

But doing this comes with a big caveat: when the open connection limit is reached, and all connections are in-use, any new database tasks that your application needs to execute will be forced to wait until a connection becomes free and marked as idle. In the context of a web application, for example, the user's HTTP request would appear to 'hang' and could potentially even timeout while waiting for the database task to be run.

To mitigate this you should always pass in a `context.Context` object with a fixed, fast, timeout when making database calls, using the context-enabled methods like [`ExecContext()`](https://golang.org/pkg/database/sql/#DB.ExecContext). An example can be seen in the [gist here](https://gist.github.com/alexedwards/5d1db82e6358b5b6efcb038ca888ab07#file-main_test-go-L12-L15).

## In Summary

1. As a rule of thumb, you should explicitly set a `MaxOpenConns` value. This should be comfortably below any hard limits on the number of connections imposed by your database and infrastructure.
2. In general, higher `MaxOpenConns` and `MaxIdleConns` values will lead to better performance. But the returns are diminishing, and you should be aware that having a too-large idle connection pool (with connections that are not re-used and eventually go bad) can actually lead to reduced performance.
3. To mitigate the risk from point 2 above, you may want to set a relatively short `ConnMaxLifetime`. But you don't want this to be so short that leads to connections being killed and recreated unnecessarily often.
4. `MaxIdleConns` should always be less than or equal to `MaxOpenConns`.

For small-to-medium web applications I typically use the following settings as a starting point, and then optimize from there depending on the results of load-testing with real-life levels of throughput.



```
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(25)
db.SetConnMaxLifetime(5*time.Minute) 
```



If you enjoyed this article, you might like to check out my [recommended tutorials list](https://www.alexedwards.net/blog) or check out my books [Let's Go](https://lets-go.alexedwards.net/) and [Let's Go Further](https://lets-go-further.alexedwards.net/), which teach you everything you need to know about how to build professional production-ready web applications and APIs with Go.

Filed under:[golang](https://www.alexedwards.net/blog/category/golang) [tutorial](https://www.alexedwards.net/blog/category/tutorial)