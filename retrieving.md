# 检索结果集

有几种从数据库检索数据的惯用操作.

1. 执行一个返回行的查询
2. 预处理一个准备重复执行的语句, 多次执行, 最后销毁
3. 一次性执行一条语句, 不对它进行预处理
4. 执行一个返回单行的查询. 这种特殊情况有一个捷径

`database/sql` 包的函数名是有意义的, 如果一个函数名包含 Query 字样, 可以推断它是被用来执行数据库查询的, 即使查询结果为空, 它也会返回一个数据行的集合. 不返回数据行的语句(存储过程等)不应该使用 Query 函数, 而应该使用 Exec()

## 从数据库提取数据

看一个如何从数据库查询数据的例子. 我们将从 `users` 表查询 `id` 为 `1` 的用户, 然后打印出这个用户的 `id` 和名字. 最后我们将通过 `rows.Scan()` 将结果集一次一行赋值给变量.

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

代码分析:

- 使用 db.Query() 发送查询命令到数据库, 记得要检查错误.
- 使用 defer rows.Close() 延迟关闭 rows, 这很重要.
- 使用 rows.Next() 循环读取行.
- 使用 rows.Scan() 读取每行中的值到变量中.
- 遍历完行以后, 再次检查错误.

在 Go 中, 这几乎是唯一的数据库处理方式. 比如, 你不能提取一行到一个 `map` 中. 这是由于一切都是强类型的, 你需要创建正确类型的变量来接收这些值, 并将指针传递给 `rows.Scan()`, 如下所示.

某些过程很容易出现异常, 进而导致不好的结果.

- 你应该始终在 `rows.Next()` 循环结束时检查异常. 你需要知道在循环期间是否发生了异常, 发生什么异常, 不要只是假设循环迭代, 直到你处理完所有行.
- 其次, 只要有一个打开的结果集, 底层连接就是忙碌状态,并且不能被其他查询语句使用. 这意味着在连接池中它是不可用的. 使用 `rows.Next()` 遍历完所有行后, 最终你会读取最后一行, `rows.Next()` 将会遇到内部 `EOF` 错误, 并且为你调用 `rows.Close()`. 如果由于某些原因你提前退出了循环, `rows` 将不会被关闭, 连接也是打开状态(如果 `rows.Next()` 因为异常返回 `false`, 它将被自动关闭). 这是一个耗尽资源的简易方法.
- 如果 `rows` 已经关闭了, `rows.Close()` 是一个无害的空白操作, 所以你可以多次调用它. 尽管如此, 也要注意, 为了避免运行时错误, 首先要检查错误, 如果没有错误发生, 则只调用 `rows.Close()`.
- 即使你在循环结束时显式调用了 `rows.Close()`, 也应该总是使用 `defer rows.Close()`.
- 不要在循环内部使用 `defer`. `defer` 语句在函数退出时才会被执行, 所以不应该在一个长时间运行的函数里使用它. 如果你这么做了, 内存将被慢慢地消耗掉. 如果你在循环内部重复检索和使用结果集, 你应该在取出每个结果集后显式调用 `rows.Close()`, 而不是使用 `defer`.

## Scan() 如何工作

当你遍历行并将它们的值扫描进目标变量时, Go 会在后台帮你执行数据类型转换. 它基于目标变量的类型, 意识到这点能让你的代码变得清爽, 并避免重复性工作.

例如, 假设你从表中查询了一些字符串数据(VARCHAR(45) 或相似类型). 但是你知道该表总是包含数值数据. 如果你传递一个指针给一个字符串, Go 将会复制字节到字符串中. 现在你可以使用 `strconv.ParseInt()` 或类似的方法将字符串值转换为数值. 在 `SQL` 操作和整型转换时, 你必须检查错误, 当然这是麻烦和乏味的.

或者, 你可以只传递 `Scan()` 给一个整型指针. Go 将为你检测并调用 `strconv.ParseInt()`. 如果在转换中发生了异常, `Scan()` 将会返回它. 你的代码现在变得更简洁, 这正是使用 `database/sql` 包的推荐方式.

## 预处理查询

对于需要多次执行的查询, 通常你应该总是对其进行预处理. 对查询进行预处理的结果是产生一条提供占位符的预处理语句. 这比连接字符串好多了(比如避免了 SQL 注入攻击).

MySQL 中, 参数占位符使用 `?`, PostgreSQL中是 `$N`, `N` 代表一个数字. SQLite 接受以上两种占位符形式. Oracle 中占位符则以一个冒号开始, 后跟参数名, 像 `:param1`. 下面的 MySQL 中的例子中使用 `?`.


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

在底层, `db.Query()` 实际上包含 `预处理`, `执行`, `关闭` 等操作. 这是到数据库的三次往返操作. 如果你不注意, 应用程序与数据库的交互次数可能增加 3 倍. 某些驱动在特定情况下能避免发送这种情形, 但不是每个驱动都能做到, 详情见 `预处理语句` 一节.

## 单行查询

如果查询最多只返回一行, 你可以使用比较简短快捷的代码来实现:

```go
var name string
err = db.QueryRow("select name from users where id = ?", 1).Scan(&name)
if err != nil {
	log.Fatal(err)
}
fmt.Println(name)
```

检索数据产生的错误直到调用 `Scan()` 时, 才能检测并返回. 你也可以在一个预处理语句中调用 QueryRow().

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
