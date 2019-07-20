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
Second, as long as there’s an open result set (represented by rows), the underlying connection is busy and can’t be used for any other query. That means it’s not available in the connection pool. If you iterate over all of the rows with rows.Next(), eventually you’ll read the last row, and rows.Next() will encounter an internal EOF error and call rows.Close() for you. But if for some reason you exit that loop – an early return, or so on – then the rows doesn’t get closed, and the connection remains open. (It is auto-closed if rows.Next() returns false due to an error, though). This is an easy way to run out of resources.
rows.Close() is a harmless no-op if it’s already closed, so you can call it multiple times. Notice, however, that we check the error first, and only call rows.Close() if there isn’t an error, in order to avoid a runtime panic.
You should always defer rows.Close(), even if you also call rows.Close() explicitly at the end of the loop, which isn’t a bad idea.
Don’t defer within a loop. A deferred statement doesn’t get executed until the function exits, so a long-running function shouldn’t use it. If you do, you will slowly accumulate memory. If you are repeatedly querying and consuming result sets within a loop, you should explicitly call rows.Close() when you’re done with each result, and not use defer.

How Scan() Works

## Scan() 如何工作

When you iterate over rows and scan them into destination variables, Go performs data type conversions work for you, behind the scenes. It is based on the type of the destination variable. Being aware of this can clean up your code and help avoid repetitive work.

For example, suppose you select some rows from a table that is defined with string columns, such as VARCHAR(45) or similar. You happen to know, however, that the table always contains numbers. If you pass a pointer to a string, Go will copy the bytes into the string. Now you can use strconv.ParseInt() or similar to convert the value to a number. You’ll have to check for errors in the SQL operations, as well as errors parsing the integer. This is messy and tedious.

Or, you can just pass Scan() a pointer to an integer. Go will detect that and call strconv.ParseInt() for you. If there’s an error in conversion, the call to Scan() will return it. Your code is neater and smaller now. This is the recommended way to use database/sql.

Preparing Queries

## 预处理查询

You should, in general, always prepare queries to be used multiple times. The result of preparing the query is a prepared statement, which can have placeholders (a.k.a. bind values) for parameters that you’ll provide when you execute the statement. This is much better than concatenating strings, for all the usual reasons (avoiding SQL injection attacks, for example).

In MySQL, the parameter placeholder is ?, and in PostgreSQL it is $N, where N is a number. SQLite accepts either of these. In Oracle placeholders begin with a colon and are named, like :param1. We’ll use ? because we’re using MySQL as our example.

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