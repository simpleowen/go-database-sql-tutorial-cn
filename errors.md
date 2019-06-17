# 错误处理

几乎所有 `database/sql` 的操作都以错误作为最后一个返回值. 你应该始终检查这些错误, 永远不要忽略它们

在某些地方, 错误行为是特殊情况, 这里有些你需要知道的事情

## 遍历结果集产生的错误

考虑以下代码:

```go
for rows.Next() {
	// ...
}
if err = rows.Err(); err != nil {
	// handle the error here
}

```

`rows.Err()` 返回的错误是 `rows.Next()` 循环产生的错误集合. 循环可能因某些提前退出, 所有你得检查循环是否是正常结束. 循环异常结束会自动调用 `rows.Close()`, 当然多次调用 `rows.Close()` 是无害的

## 关闭结果集产生的错误

如前所述, 循环提前结束的时候, 你应该总是显示关闭 `sql.Rows`. 循环正常和非正常退出 rows 都会自动关闭, 但是你可能会错误的做这件事:

```go
for rows.Next() {
	// ...
	break; // whoops, rows is not closed! memory leak...
}
// do the usual "if err = rows.Err()" [omitted here]...
// it's always safe to [re?]close here:
if err = rows.Close(); err != nil {
	// but what should we do if there's an error?
	log.Println(err)
}

```

`rows.Close()` 返回的错误是一般规则的唯一例外，你最好捕获并检查所有数据库操作中的错误. 如果 `rows.Close()` 返回一个错误, 你可能不知道要如何处理. 记录错误消息或 `panicing` 可能是唯一明智的事情，如果这不合理，那么也许你应该忽略错误

## QueryRow() 产生的错误

考虑以下检索单行数据的代码:

```go
var name string
err = db.QueryRow("select name from users where id = ?", 1).Scan(&name)
if err != nil {
	log.Fatal(err)
}
fmt.Println(name)

```

若没有 `id = 1` 的用户怎么办? 结果集中将不会有任何行存在, `Scan()` 函数也不会将值扫描进 `name` 变量中, 这会发生什么事?

Go 定义了一个特殊的错误常量 `sql.ErrNoRows`. 当 `QueryRow()` 返回结果集为空时, `sql.ErrNoRows` 错误会产生. 在大多数情况下, 这需要作为一种特殊情况来处理. 空结果集通常不会被应用程序认为是一种异常情况, 所以如果你不处理空结果集的情况, 可能导致出乎意料的应用程序代码错误

`query` 中的错误将被推迟，直到调用 `Scan()`，然后从中返回. 上述代码更好的方式如下:

```go
var name string
err = db.QueryRow("select name from users where id = ?", 1).Scan(&name)
if err != nil {
	if err == sql.ErrNoRows {
		// there were no rows, but otherwise no error occurred
	} else {
		log.Fatal(err)
	}
}
fmt.Println(name)

```

有人可能会问为什么空结果集被认为是错误, 空集没有任何错误. 那是由于 `QueryRow()` 函数需要用这种特殊方式告诉调用者区分有没有检索到数据. 没有 `sql.ErrNoRows`，`Scan()` 不会做任何事情，你可能没有意识到你的变量根本没有从数据库中获得任何值

你应该只会在使用 `QueryRow()` 时碰到这种错误, 如果在其他地方也遇见了, 估计是你做错了某些事

## 识别特定的数据库错误

像下面这样写代码是非常诱人的:

```go
rows, err := db.Query("SELECT someval FROM sometable")
// err contains:
// ERROR 1045 (28000): Access denied for user 'foo'@'::1' (using password: NO)
if strings.Contains(err.Error(), "Access denied") {
	// Handle the permission-denied error
}

```

这不是最好的方式, 例如，字符串值可能会有所不同，具体取决于服务器用于发送错误消息的语言. 比较错误号来识别特定错误是更好的方式.

但是，执行此操作的机制因驱动程序而异，因为这不是 `database/sql` 自身的一部分. 本教程着重介绍 MySQL 驱动, 你可以像这样写代码:

```go
if driverErr, ok := err.(*mysql.MySQLError); ok { // Now the error number is accessible directly
	if driverErr.Number == 1045 {
		// Handle the permission-denied error
	}
}
```

同样，此处的 `MySQLError` 类型由此特定驱动程序提供，并且 `.Number` 字段可能因驱动程序而异. `number` 的值来自于 MySQL 数据库的错误消息, 所以它是特定于数据库的, 而不是特定于驱动程序的.

上面的代码还是不太优美. 与 `1045` 这个神奇的数字相比, 是一种代码的味道. 一些驱动提供了错误代码列表, 像 Postgresql 数据库驱动 `pq` 就是这样, 在 [error.go](https://github.com/lib/pq/blob/master/error.go) 页面提供了错误代码列表. 还有一个由 VividCortex 维护的 [MySQL 错误代码列表](https://github.com/VividCortex/mysqlerr). 通过这些错误代码列表, 可以这样改进代码:

```go
if driverErr, ok := err.(*mysql.MySQLError); ok {
	if driverErr.Number == mysqlerr.ER_ACCESS_DENIED_ERROR {
		// Handle the permission-denied error
	}
}

```

## 处理连接错误

如果与数据库的连接被抛弃, 被杀死, 亦或发生了异常要怎么办?

即使这种情况发生了, 你不需要去实现重试的逻辑. 作为 `database/sql` [连接池](connection-pool.md)的一部分, 连接失败的处理程序是内置的. 当你在执行查询或其他数据库操作发生连接异常时, Go 会重新打开一个新的连接(或从连接池中取出一个新的连接)并且自动重连数据库, 重连次数可达 10 次

但是，这也可能会产生一些意想不到的后果. 当发生其他错误情况时，可能会重试某些类型的错误. 这可能也是特定于驱动程序的. 一个 `MySQL` 驱动程序的例子是使用 `KILL` 取消不需要的语句(例如,长时间运行的查询)会导致语句被重试最多10次
