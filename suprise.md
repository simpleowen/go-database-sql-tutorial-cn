# 惊喜, 反模式, 限制

Although database/sql is simple once you’re accustomed to it, you might be surprised by the subtlety of use cases it supports. This is common to Go’s core libraries.
尽管当熟悉 database/sql 库以后, 你会觉得它非常简单, 你可能会被它支持的用例的精妙震惊, 但这在 Go 核心库里面是很平常的.


Resource Exhaustion
## 资源枯竭
As mentioned throughout this site, if you don’t use database/sql as intended, you can certainly cause trouble for yourself, usually by consuming some resources or preventing them from being reused effectively:

正如整篇文章提到的, 如果你没按预期使用 database/sql 库, 可能会给你自己带来麻烦, 麻烦的表现方式通常是资源被消耗和低效的资源重用:

Opening and closing databases can cause exhaustion of resources.
Failing to read all rows or use rows.Close() reserves connections from the pool.
Using Query() for a statement that doesn’t return rows will reserve a connection from the pool.
Failing to be aware of how prepared statements work can lead to a lot of extra database activity.

- 打开和关闭数据库会消耗资源
- 读取所有行失败, 或者使用 rows.Close() 保留池中的连接
- 使用 Query() 没有返回任何行将保留池中的连接
- 不知道预处理语句如何工作将导致大量的额外数据库活动

Large uint64 Values

## 大 unit64 值
Here’s a surprising error. You can’t pass big unsigned integers as parameters to statements if their high bit is set:

这里有个令人惊讶的错误, 你不能将一个设置了高位的很大的无符号整数作为参数传递给语句:

```go
_, err := db.Exec("INSERT INTO users(id) VALUES", math.MaxUint64) // Error

```

This will throw an error. Be careful if you use uint64 values, as they may start out small and work without error, but increment over time and start throwing errors.

上述代码将抛出一个异常. 使用 uint64 类型的值时要注意, 它们可能在值为一个比较小的数可以正常工作, 但随着时间推移, 它们会增长到很大, 并抛出异常.

Connection State Mismatch

## 连接状态不匹配
Some things can change connection state, and that can cause problems for two reasons:

有些东西可以改变连接状态, 因以下两个原因可能会导致一些问题:

Some connection state, such as whether you’re in a transaction, should be handled through the Go types instead.
1. 一些连接状态, 比如你是否在事务中, 应该通过 Go 类型来处理
2. 你可能假定查询运行在单个连接上, 实际上并不是


You might be assuming that your queries run on a single connection when they don’t.

For example, setting the current database with a USE statement is a typical thing for many people to do. But in Go, it will affect only the connection that you run it in. Unless you are in a transaction, other statements that you think are executed on that connection may actually run on different connections gotten from the pool, so they won’t see the effects of such changes.

比如, 很多人使用 USE 语句来设置当前数据库. 但是在 Go 中, 这只会影响你运行它的连接. 除非你在一个事务中, 否则你认为在一个连接上执行的语句实际上可能执行在不同的连接上, 所以它们看不到这类变化的影响.

Additionally, after you’ve changed the connection, it’ll return to the pool and potentially pollute the state for some other code. This is one of the reasons why you should never issue BEGIN or COMMIT statements as SQL commands directly, too.

另外, 你更改连接后, 连接将被放回到池中, 可能会影响其他代码的状态. 这也是为什么你永远不要将 BEGIN 和 COMMIT 语句直接作为 SQL 命令的原因之一.

Database-Specific Syntax

## 特定数据库语法

The database/sql API provides an abstraction of a row-oriented database, but specific databases and drivers can differ in behavior and/or syntax, such as prepared statement placeholders.

database/sql API 提供了面向行数据库的一个抽象, 特定的数据库和驱动程序可能有不同的行为和/或语法, 比如预处理语句中的占位符.

Multiple Result Sets

## 多重结果集
The Go driver doesn’t support multiple result sets from a single query in any way, and there doesn’t seem to be any plan to do that, although there is a feature request for supporting bulk operations such as bulk copy.

This means, among other things, that a stored procedure that returns multiple result sets will not work correctly.

Invoking Stored Procedures

## 调用存储过程

Invoking stored procedures is driver-specific, but in the MySQL driver it can’t be done at present. It might seem that you’d be able to call a simple procedure that returns a single result set, by executing something like this:

```go
err := db.QueryRow("CALL mydb.myprocedure").Scan(&result) // Error

```

In fact, this won’t work. You’ll get the following error: Error 1312: PROCEDURE mydb.myprocedure can’t return a result set in the given context. This is because MySQL expects the connection to be set into multi-statement mode, even for a single result, and the driver doesn’t currently do that (though see this issue).

Multiple Statement Support

## 多语句支持
The database/sql doesn’t explicitly have multiple statement support, which means that the behavior of this is backend dependent:

```go
_, err := db.Exec("DELETE FROM tbl1; DELETE FROM tbl2") // Error/unpredictable result
```

The server is allowed to interpret this however it wants, which can include returning an error, executing only the first statement, or executing both.

Similarly, there is no way to batch statements in a transaction. Each statement in a transaction must be executed serially, and the resources in the results, such as a Row or Rows, must be scanned or closed so the underlying connection is free for the next statement to use. This differs from the usual behavior when you’re not working with a transaction. In that scenario, it is perfectly possible to execute a query, loop over the rows, and within the loop make a query to the database (which will happen on a new connection):

```go
rows, err := db.Query("select * from tbl1") // Uses connection 1
for rows.Next() {
	err = rows.Scan(&myvariable)
	// The following line will NOT use connection 1, which is already in-use
	db.Query("select * from tbl2 where id = ?", myvariable)
}

```

But transactions are bound to just one connection, so this isn’t possible with a transaction:

```go
tx, err := db.Begin()
rows, err := tx.Query("select * from tbl1") // Uses tx's connection
for rows.Next() {
	err = rows.Scan(&myvariable)
	// ERROR! tx's connection is already busy!
	tx.Query("select * from tbl2 where id = ?", myvariable)
}
```

Go doesn’t stop you from trying, though. For that reason, you may wind up with a corrupted connection if you attempt to perform another statement before the first has released its resources and cleaned up after itself. This also means that each statement in a transaction results in a separate set of network round-trips to the database.

