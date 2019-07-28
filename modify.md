# 修改数据, 使用事务

Now we’re ready to see how to modify data and work with transactions. The distinction might seem artificial if you’re used to programming languages that use a “statement” object for fetching rows as well as updating data, but in Go, there’s an important reason for the difference.

现在你已经准备好了解如何修改数据和处理事务. 如果你习惯于编写使用 `语句` 对象来获取行以及更新数据的编程语言，那么区别似乎是假的，但在Go中，存在差异的重要原因.

## 修改数据的语句

Use Exec(), preferably with a prepared statement, to accomplish an INSERT, UPDATE, DELETE, or another statement that doesn’t return rows. The following example shows how to insert a row and inspect metadata about the operation:

使用 Exec(), 最好配合预处理语句来完成诸如 `INSERT`, `UPDATE`, `DELETE` 等不需要返回行的任务. 下面的例子展示了如何插入一条语句并获取操作元数据:

```go
stmt, err := db.Prepare("INSERT INTO users(name) VALUES(?)")
if err != nil {
	log.Fatal(err)
}
res, err := stmt.Exec("Dolly")
if err != nil {
	log.Fatal(err)
}
lastId, err := res.LastInsertId()
if err != nil {
	log.Fatal(err)
}
rowCnt, err := res.RowsAffected()
if err != nil {
	log.Fatal(err)
}
log.Printf("ID = %d, affected = %d\n", lastId, rowCnt)
```

Executing the statement produces a sql.Result that gives access to statement metadata: the last inserted ID and the number of rows affected.

执行 Exec() 后返回了一个 sql.Result 对象, 它有权访问语句的元数据, 包括最后一行的 ID 和此次操作影响的行数.

What if you don’t care about the result? What if you just want to execute a statement and check if there were any errors, but ignore the result? Wouldn’t the following two statements do the same thing?

如果你并不关心结果怎么办? 如果你只想执行一条语句并检查是否产生异常而不想知道执行结果怎么办? 以下两条语句不会做同样的事情吗?

```go
_, err := db.Exec("DELETE FROM users")  // OK
_, err := db.Query("DELETE FROM users") // BAD
```

The answer is no. They do not do the same thing, and you should never use Query() like this. The Query() will return a sql.Rows, which reserves a database connection until the sql.Rows is closed. Since there might be unread data (e.g. more data rows), the connection can not be used. In the example above, the connection will never be released again. The garbage collector will eventually close the underlying net.Conn for you, but this might take a long time. Moreover the database/sql package keeps tracking the connection in its pool, hoping that you release it at some point, so that the connection can be used again. This anti-pattern is therefore a good way to run out of resources (too many connections, for example).

答案是否. 它们不是做同一件事, 你永远也不要像这样使用 Query(). Query() 会返回一个 sql.Rows 对象, 它保存着一个与数据库的连接直到 sql.Rows 并关闭. 由于可能存在未读数据(如, 更多的行), 无法使用此连接. 上面这个例子中, 连接永远也不会被再次释放. 当然垃圾回收机制最终会帮你关闭底层 net.Conn, 但可能耗时较长. 此外, database/sql 包会一直跟踪池中的连接, 希望你会在某个时刻释放它, 以便重复利用该连接. 因此这种反模式是耗尽资源的一个好方法(比如, 太多的连接数).

Working with Transactions

## 使用事务

In Go, a transaction is essentially an object that reserves a connection to the datastore. It lets you do all of the operations we’ve seen thus far, but guarantees that they’ll be executed on the same connection.

在 Go 中, 事务本质上是一个保留与数据库连接的对象. 它允许你做目前为止你接触到的所有数据库操作, 而且是使用同一个连接去执行这些操作.

You begin a transaction with a call to db.Begin(), and close it with a Commit() or Rollback() method on the resulting Tx variable. Under the covers, the Tx gets a connection from the pool, and reserves it for use only with that transaction. The methods on the Tx map one-for-one to methods you can call on the database itself, such as Query() and so forth.

Prepared statements that are created in a transaction are bound exclusively to that transaction. See prepared statements for more.

You should not mingle the use of transaction-related functions such as Begin() and Commit() with SQL statements such as BEGIN and COMMIT in your SQL code. Bad things might result:

- The Tx objects could remain open, reserving a connection from the pool and not returning it.
- The state of the database could get out of sync with the state of the Go variables representing it.
- You could believe you’re executing queries on a single connection, inside of a transaction, when in reality Go has created several connections for you invisibly and some statements aren’t part of the transaction.

While you are working inside a transaction you should be careful not to make calls to the db variable. Make all of your calls to the Tx variable that you created with db.Begin(). db is not in a transaction, only the Tx object is. If you make further calls to db.Exec() or similar, those will happen outside the scope of your transaction, on other connections.

If you need to work with multiple statements that modify connection state, you need a Tx even if you don’t want a transaction per se. For example:

- Creating temporary tables, which are only visible to one connection.
- Setting variables, such as MySQL’s SET @var := somevalue syntax.
- Changing connection options, such as character sets or timeouts.

If you need to do any of these things, you need to bind your activity to a single connection, and the only way to do that in Go is to use a Tx.