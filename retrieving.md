# 检索结果集

There are several idiomatic operations to retrieve results from the datastore.
这里有一些从数据库检索数据的惯用操作.

Execute a query that returns rows.
- 执行一个返回行的查询
- 预处理一个准备重复执行的语句, 多次执行, 最后销毁
- 一次性执行一条语句, 不对它进行预处理
- 执行一个返回单行的查询. 这种特殊情况有快捷方式
Prepare a statement for repeated use, execute it multiple times, and destroy it.
Execute a statement in a once-off fashion, without preparing it for repeated use.
Execute a query that returns a single row. There is a shortcut for this special case.

Go’s database/sql function names are significant. If a function name includes Query, it is designed to ask a question of the database, and will return a set of rows, even if it’s empty. Statements that don’t return rows should not use Query functions; they should use Exec().

database/sql 包的函数名都是有意义的, 如果一个函数名包含 Query 字样, 可以推断它是被用来执行数据库查询的, 即使查询结果为空, 它也会返回一个数据行的集合. 不返回数据行的语句(存储过程等)不应该使用 Query 函数, 而应该使用 Exec()


Fetching Data from the Database

## 从数据库提取数据

Let’s take a look at an example of how to query the database, working with results. We’ll query the users table for a user whose id is 1, and print out the user’s id and name. We will assign results to variables, a row at a time, with rows.Scan().

让我们看一个如何从数据库查询数据的例子. 我们将从 users 表查询 id 为 1 的用户, 然后打印出这个用户的 id 和名字. 最后我们将通过 rows.Scan() 将结果集一次一行赋值给变量.


```go
var (
	id int
	name string
)
rows, err := db.Query("select id, name from users where id = ?", 1)
if err != nil {
	log.Fatal(err)
}
defer rows.Close()
for rows.Next() {
	err := rows.Scan(&id, &name)
	if err != nil {
		log.Fatal(err)
	}
	log.Println(id, name)
}
err = rows.Err()
if err != nil {
	log.Fatal(err)
}

```

Here’s what’s happening in the above code:

以下对上述代码进行解释:

- 使用 db.Query() 发送查询到数据库, 像通常一样检查错误.
- 使用 defer rows.Close() 延迟关闭 rows, 这很重要.
- 使用 rows.Next() 循环读取行.
- 使用 rows.Scan() 读取每行中的列到变量中.
- 遍历完行以后, 再次检查错误.

在 Go 中, 这几乎是唯一的数据库处理方式. 比如, 你不能提取一行到一个 map 中. 这是由于一切都是强类型的, 你需要创建正确类型的变量来接收这些值, 并将指针传递给 rows.Scan().

We’re using db.Query() to send the query to the database. We check the error, as usual.
We defer rows.Close(). This is very important.
We iterate over the rows with rows.Next().
We read the columns in each row into variables with rows.Scan().
We check for errors after we’re done iterating over the rows.
This is pretty much the only way to do it in Go. You can’t get a row as a map, for example. That’s because everything is strongly typed. You need to create variables of the correct type and pass pointers to them, as shown.

A couple parts of this are easy to get wrong, and can have bad consequences.

一些过程很容易出现异常, 可能导致不好的结果.

You should always check for an error at the end of the for rows.Next() loop. If there’s an error during the loop, you need to know about it. Don’t just assume that the loop iterates until you’ve processed all the rows.

rows.Next() 循环结尾处, 你应该总是检查异常. 你需要知道在循环期间是否发生了异常, 不要只是假设循环会处理所有行.

Second, as long as there’s an open result set (represented by rows), the underlying connection is busy and can’t be used for any other query. That means it’s not available in the connection pool. If you iterate over all of the rows with rows.Next(), eventually you’ll read the last row, and rows.Next() will encounter an internal EOF error and call rows.Close() for you. But if for some reason you exit that loop – an early return, or so on – then the rows doesn’t get closed, and the connection remains open. (It is auto-closed if rows.Next() returns false due to an error, though). This is an easy way to run out of resources.

第二, 只要存在一个结果集, 底层连接就是忙碌状态,并且不能被其他查询语句使用. 这意味着在连接池中它是不可用的. 使用 rows.Next() 遍历完所有行后, 最终你会读取最后一行, rows.Next() 将会遇到内部 EOF 错误, 并且为你调用 rows.Close(). 如果由于某些原因你提前退出了循环, rows 将不会被关闭, 连接也是打开状态(如果 rows.Next() 因为异常返回false, 它将被自动关闭). 这是一个耗尽资源的简易方法.

rows.Close() is a harmless no-op if it’s already closed, so you can call it multiple times. Notice, however, that we check the error first, and only call rows.Close() if there isn’t an error, in order to avoid a runtime panic.

就算rows 已经关闭了, rows.Close() 也是一个无害的无操作, 所以你可以多次调用它. 尽管如此, 也要注意, 为了避免运行时错误, 确认没有异常发生后再调用 rows.Close().

You should always defer rows.Close(), even if you also call rows.Close() explicitly at the end of the loop, which isn’t a bad idea.
Don’t defer within a loop. A deferred statement doesn’t get executed until the function exits, so a long-running function shouldn’t use it. If you do, you will slowly accumulate memory. If you are repeatedly querying and consuming result sets within a loop, you should explicitly call rows.Close() when you’re done with each result, and not use defer.

你应该总是使用 defer rows.Close(), 即使你在循环结尾处显示调用了 rows.Close(), 不要在循环内部 defer. defer 语句在函数退出时才会被执行, 所以不应该在一个长时间运行的函数里使用它. 如果你这么做了, 你将慢慢地耗费内存. 如果你在循环内部重复检索和使用结果集, 你应该在取出每个结果后显式调用 rows.Close(), 而不是使用 defer.

How Scan() Works

## Scan() 如何工作

When you iterate over rows and scan them into destination variables, Go performs data type conversions work for you, behind the scenes. It is based on the type of the destination variable. Being aware of this can clean up your code and help avoid repetitive work.

当你遍历行并将它们扫描进目标变量时, Go 会在后台帮你执行数据类型转换. 它基于目标变量的类型, 意识到这点能让你的代码变得清爽, 并避免重复性工作.

For example, suppose you select some rows from a table that is defined with string columns, such as VARCHAR(45) or similar. You happen to know, however, that the table always contains numbers. If you pass a pointer to a string, Go will copy the bytes into the string. Now you can use strconv.ParseInt() or similar to convert the value to a number. You’ll have to check for errors in the SQL operations, as well as errors parsing the integer. This is messy and tedious.

例如, 假设你从表中查询了一些字符串数据, VARCHAR(45) 或相似类型. 但是你恰好知道该表总是包含数值数据. 如果你传递一个指针给一个字符串, Go 将会复制字节到字符串中. 现在你可以使用 strconv.ParseInt() 或将值转换数值. 你必须检查 SQL 操作中和整型转换的错误. 这是麻烦和乏味的.

Or, you can just pass Scan() a pointer to an integer. Go will detect that and call strconv.ParseInt() for you. If there’s an error in conversion, the call to Scan() will return it. Your code is neater and smaller now. This is the recommended way to use database/sql.

或者, 你只传递一个整型指针给 Scan(). Go 将为你检测并调用 strconv.ParseInt(). 如果在转换中发生了异常, Scan() 将会返回它. 你的代码现在变得更简洁, 这是推荐使用 database/sql 包的方式.

Preparing Queries

## 预处理查询

You should, in general, always prepare queries to be used multiple times. The result of preparing the query is a prepared statement, which can have placeholders (a.k.a. bind values) for parameters that you’ll provide when you execute the statement. This is much better than concatenating strings, for all the usual reasons (avoiding SQL injection attacks, for example).

对于需要多次执行的查询, 通常你应该总是进行预处理. 对查询进行预处理的结果是产生一条提供占位符的预处理语句. 这比连接字符串好多了

In MySQL, the parameter placeholder is ?, and in PostgreSQL it is $N, where N is a number. SQLite accepts either of these. In Oracle placeholders begin with a colon and are named, like :param1. We’ll use ? because we’re using MySQL as our example.

MySQL 中, 参数占位符使用 `?`, PostgreSQL中是$N, N代表一个数字. SQLite 接受两种形式. Oracle中占位符以一个冒号开始, 后跟参数名, 像 `:param1`. 我们将在使用 MySQL 中的例子中使用 `?`.


```go
stmt, err := db.Prepare("select id, name from users where id = ?")
if err != nil {
	log.Fatal(err)
}
defer stmt.Close()
rows, err := stmt.Query(1)
if err != nil {
	log.Fatal(err)
}
defer rows.Close()
for rows.Next() {
	// ...
}
if err = rows.Err(); err != nil {
	log.Fatal(err)
}
```

Under the hood, db.Query() actually prepares, executes, and closes a prepared statement. That’s three round-trips to the database. If you’re not careful, you can triple the number of database interactions your application makes! Some drivers can avoid this in specific cases, but not all drivers do. See prepared statements for more.

在底层, `db.Query()` 实际上包含 `预处理`, `执行`, `关闭` 等操作. 这是到数据库的三次往返操作. 如果你不注意, 应用程序与数据库的交互次数可能增加 3 倍. 某些驱动在特定情况下能避免发送这种情形, 但不是每个驱动都能做到, 详情见 `预处理语句` 一节.

Single-Row Queries

## 单行查询

If a query returns at most one row, you can use a shortcut around some of the lengthy boilerplate code:

如果返回最多只有一行, 你可以使用比较简短快捷的代码来实现:

```go
var name string
err = db.QueryRow("select name from users where id = ?", 1).Scan(&name)
if err != nil {
	log.Fatal(err)
}
fmt.Println(name)
```

Errors from the query are deferred until Scan() is called, and then are returned from that. You can also call QueryRow() on a prepared statement:

检索数据产生的错误直到调用 Scan()时, 才能检测并返回. 你也可以在一个预处理语句中调用 QueryRow().

```go
stmt, err := db.Prepare("select name from users where id = ?")
if err != nil {
	log.Fatal(err)
}
defer stmt.Close()
var name string
err = stmt.QueryRow(1).Scan(&name)
if err != nil {
	log.Fatal(err)
}
fmt.Println(name)
```