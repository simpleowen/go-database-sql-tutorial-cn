# 使用预处理语句

使用 `预处理` 语句能获得 `Go` 语言中通常有的所有好处: 安全, 高效, 方便. 但是它们实现的方式可能跟你以前的经验不太一样, 特别是关于如何与 `database/sql` 内部交互的部分

## 预处理语句和连接

在数据库级别, 一条 `预处理` 语句是绑定到一个单独的数据库连接上的. 标准的流程是:

- 首先, 客户端发送一条带占位符形参的 `SQL` 语句到服务端做预处理
- 然后, 服务端返回一个该语句 `ID` 号来响应客户端
- 最后, 客户端通过发送这个 `ID` 号带上相关的实参来执行这条语句

但是在 `Go` 中，连接不会直接暴露给 `database/sql` 包的用户. 你不会在一个连接上 `预处理` 一条语句, 而是在 `DB` 或 `Tx` 上. 并且 `database/sql` 有一些很方便的特性, 如自动重试. 由于这些原因，在驱动程序层面存在的预处理语句和连接之间的底层关联对代码是不可见的

以下是它的工作原理:

- `预处理` 一条语句时, 它是在连接池中的连接上进行 `预处理` 的
- `Stmt` 对象记录哪条连接被使用了
- 执行 `Stmt` 时，它会尝试使用该连接. 如果由于它已关闭或忙于执行其他操作而无法使用，它将从池中获取另一个连接，并在另一个连接上使用数据库重新预处理语句

由于语句将在原始连接繁忙时根据需要重新预处理，因此数据库的高并发使用可能会使很多连接繁忙，从而创建大量预处理语句. 这可能导致语句的泄漏，正在 `预处理` 和重新 `预处理` 的语句比你想象的更频繁，甚至语句数量会达到服务端的限制数

## 避免预处理语句

Go 在幕后为你创建了 `预处理` 语句, 比如一个简单的 `db.Query(sql, param1, param2)` 调用, 通过 `预处理` SQL, 然后带上参数执行并最终关闭来工作

但是有时一条 `预处理` 语句不是你想要的那样, 可能有以下几种原因:

- 数据库不支持 `预处理` 语句. 比如使用 `MySQL` 驱动时, 你能连上 `MemSQL` 和 `Sphinx`, 因为它们支持 `MySQL` 线程协议. 但是它们不支持包含 `预处理` 语句的二进制协议, 所以他们执行 `预处理` 语句会失败
- 语句的重用不足以使它们变得有价值，并且安全问题以其他方式处理了，而且也不希望出现性能开销. 可以在 [VividCortex 博客](https://www.vividcortex.com/blog/2014/11/19/analyzing-prepared-statement-performance-with-vividcortex/)上看到这方面的一个例子

如果不想使用 `预处理` 语句, 你需要使用 `fmt.Sprint()` 或者类似方式来组装 `SQL`, 并且将其作为参数传递给 `db.Query()` 或 `db.QueryRow()`. 并且你使用的驱动库需要支持 `Go 1.1` 中通过 `Execer` 和 `Queryer` 接口新增的明文查询执行, [文档在这](https://golang.org/pkg/database/sql/driver/#Execer)

## 事务中的预处理语句

在 `Tx` 中创建的预处理语句仅与其绑定，因此之前关于重新 `预处理` 的注意事项不适用。当你对 `Tx` 对象进行操作时，你的操作将直接映射到其下的唯一连接

这也意味着在 `Tx` 中创建的预处理语句不能与它分开使用. 同样, 在 `DB` 上创建的 `预处理` 语句不能在事务中使用，因为它们将绑定到不同的连接

要在 `Tx` 中使用在事务外部 `预处理` 的语句，可以使用 `Tx.Stmt()`，它将从事务外部 `预处理` 的语句创建新的特定于事务的语句. 它通过获取现有的 `预处理` 语句，将连接设置为事务的连接并在每次执行时重新表示所有语句来完成此操作. 这种行为及其实现是不可取的，在数据​​库 `database/sql` 源代码中甚至还有一个 `TODO` 来改进它, 我们建议不要使用它

在事务中处理 `预处理` 好的语句时必须谨慎行事, 考虑以下例子:

```go
tx, err := db.Begin()
if err != nil {
    log.Fatal(err)
}
defer tx.Rollback()
stmt, err := tx.Prepare("INSERT INTO foo VALUES (?)")
if err != nil {
    log.Fatal(err)
}
defer stmt.Close() // danger!
for i := 0; i < 10; i++ {
    _, err = stmt.Exec(i)
    if err != nil {
        log.Fatal(err)
    }
}
err = tx.Commit()
if err != nil {
    log.Fatal(err)
}
// stmt.Close() runs here!
```

`Go 1.4` 之前的版本, 关闭 `*sql.Tx` 会将与其关联的连接释放回连接池中, 但是在已经发生的事件之后执行了对 `预处理` 好的语句的延迟调用关闭，这可能导致并发访问底层连接，导致连接状态不一致, 如果你使用 `Go 1.4` 及其后续版本, 应该确保在事务提交或回滚前, 语句总是被关闭的. 在 `Go 1.4` 以后的版本中已经被[修复](https://codereview.appspot.com/131650043)了[这个问题](https://github.com/golang/go/issues/4459)

## 预处理语句中参数占位符

`预处理` 语句中的参数占位符是跟数据库相关的, 以下是 `MySQL`, `PostgreSQL`, 和 `Oracle` 3 个常用数据库占位符参数语法的对比:

```
MySQL               PostgreSQL            Oracle
=====               ==========            ======
WHERE col = ?       WHERE col = $1        WHERE col = :col
VALUES(?, ?, ?)     VALUES($1, $2, $3)    VALUES(:val1, :val2, :val3)
```