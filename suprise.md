# 意外, 反模式, 限制

当熟悉 `database/sql` 库以后, 尽管你会觉得它非常简单, 你可能会被它支持的用例的精妙所震惊, 但这在 `Go` 核心库里面是很普通的.

## 资源耗尽

正如一直强调的, 如果你没按预期使用 `database/sql` 库, 可能会给你自己带来麻烦, 麻烦的表现方式通常是资源被消耗或低效的资源重用:

- 打开和关闭数据库会消耗资源
- 读取所有行失败, 或者调用 `rows.Close()` 失败, 池中的连接将保留
- 使用 `Query()` 没有返回任何行, 池中的连接将保留
- 不清楚预编译语句如何工作将导致大量的额外数据库操作

Large uint64 Values

## 大 unit64 值

这里有个可能出乎你意料之外的错误, 你不能将一个设置了高字节内容的很大的无符号整数作为参数传递给语句:

```go
_, err := db.Exec("INSERT INTO users(id) VALUES", math.MaxUint64) // Error

```

上述代码将抛出一个异常. 使用 `uint64` 类型的值时要注意, 它们可能在值为一个比较小的数可以正常工作, 但随着时间推移, 它们会增长到很大, 并抛出异常.

## 连接状态不匹配

有些操作可以改变连接状态, 因此这些操作可能会因为以下两个原因导致一些问题:

1. 某些连接状态, 比如你是否在事务中, 应该通过 Go 类型来处理
2. 你可能假定查询是基于同一个连接执行的, 实际上并不是

比如, 很多人使用 `USE` 语句来设置当前数据库. 但是在 Go 中, 这只会影响到你正在使用的连接. 除非你在一个事务中, 否则你认为在同一个连接上执行的语句实际上可能执行在池中不同的连接上, 只是程序感知不到这些影响罢了.

另外, 你更改连接后, 当它被释放到池中时, 可能会污染其他代码的状态. 这也是为什么你永远不要将 `BEGIN` 和 `COMMIT` 语句直接作为 `SQL` 命令的原因之一.

## 特定数据库语法

`database/sql` API 提供了关系型数据库的抽象, 但是特定的数据库和驱动程序可能有不同的行为和语法, 比如[预编译语句中的占位符](prepared.md).

## 多结果集

Go 驱动不支持任何形式的通过单个查询获取多个结果集的方式, 尽管有一个[支持批量操作的功能申请](https://github.com/golang/go/issues/5171), 如批量复制, 然而多结果集事也似乎依然没有戏.

这意味着返回多结果集的存储过程将无法正常执行.

## 调用存储过程

存储过程调用是区别于驱动程序的, 目前 MySQL 驱动还没有实现该功能. 不过你可以通过执行类似如下的代码来调用只返回单结果集的存储过程:

```go
err := db.QueryRow("CALL mydb.myprocedure").Scan(&result) // Error

```

事实上, 它不能正常工作. 它会抛出异常 `Error 1312: PROCEDURE mydb.myprocedure can’t return a result set in the given context`. 这是由于 MySQL 想要把连接设置成多语句模式, 但是 MySQL 驱动程序目前还不支持(看此 [Issue](https://github.com/go-sql-driver/mysql/issues/66))

## 多语句支持

The database/sql doesn’t explicitly have multiple statement support, which means that the behavior of this is backend dependent:

`database/sql` 包没有明确的多语句特性, 这意味着是否支持该特性取决于后端, 即数据库:

```go
_, err := db.Exec("DELETE FROM tbl1; DELETE FROM tbl2") // Error/unpredictable result
```

数据库会按自己的方式解析该语句, 可能只返回一个错误, 或只执行第一条语句, 或两个都执行.

同样, 也没有在事务中批量执行语句的方法. 每一条语句都要按顺序执行, 结果中的资源, 比如单行或多行数据, 必须被扫描或关闭, 以便底层连接被下一条语句使用. 使用事务与不使用事务的操作是有区别的. 在不使用事务的场景中, 执行查询, 遍历结果集, 循环中再次执行查询等操作完全有可能使用一个全新的连接:

```go
rows, err := db.Query("select * from tbl1") // Uses connection 1
for rows.Next() {
	err = rows.Scan(&myvariable)
	// The following line will NOT use connection 1, which is already in-use
	db.Query("select * from tbl2 where id = ?", myvariable)
}

```

但事务却是绑定在同一个连接上的, 所以一个连接如果还被占用着, 是无法执行下一个查询操作的:

```go
tx, err := db.Begin()
rows, err := tx.Query("select * from tbl1") // Uses tx's connection
for rows.Next() {
	err = rows.Scan(&myvariable)
	// ERROR! tx's connection is already busy!
	tx.Query("select * from tbl2 where id = ?", myvariable)
}
```

但是 Go 不会阻止你尝试这样做.  基于此, 在你尝试执行另一条语句前, 如果上一条语句所占用的资源没有被释放或清理, 此连接可能也会变得不可用. 这也意味着在事务中每条语句的执行都依赖单独的网络接口与数据库交互.
