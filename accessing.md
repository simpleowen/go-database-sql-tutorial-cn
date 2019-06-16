# 访问数据库

导入驱动包后, 就可以创建一个数据库对象 `sql.DB` 了

使用 `sql.Open()` 来创建 `sql.DB`, 它返回一个 `*sql.DB` 指针:

```go
func main() {
	db, err := sql.Open("mysql",
		"user:password@tcp(127.0.0.1:3306)/hello")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
}
```

在上述示例中, 我们说明几件事:

1. `sql.Open` 的第一个参数是驱动名称. 这是驱动程序用于向 `database/sql` 注册自己的字符串, 为了防止混乱, 通常它与驱动包名称一致. 例如, `mysql` 是 [Go MySQL Driver](https://github.com/go-sql-driver/mysql) 包的驱动名称, 当然有些驱动也不遵循此约定, 比如, `sqlite3` 对于 [sqlite3 driver](https://github.com/mattn/go-sqlite3), `postgres` 对于 [Pure Go Postgres driver](https://github.com/lib/pq)
2. 第二个参数是连接数据库的特定语法, 在上述示例中, 我们在本地连接一个名为 `hello` 的 MySQL 数据库实例
3. 你应该始终检查和处理所有 `database/sql` 操作返回的错误. 但是我们稍后将会讨论一些例外的案例
4. 如果 `sql.DB` 在函数外没有被引用, 那么使用 `defer db.Close()` 来关闭 db 是 go 程序员经常使用的方式

与直觉相反, `sql.Open()` 并不会与数据库建立连接, 也不会验证连接参数. 它只是单纯地为后面的数据库操作建立了一个数据库对象的抽象. 在第一次需要的时候, 它才与数据库建立实际连接. 在这种情况下, 如果你想确认能否正常访问数据库(比如, 检查能否建立网络连接并登录), 可使用 `db.Ping()` 来做. 记得处理返回的错误:

```go
err = db.Ping()
if err != nil {
	// do something here
}
```

尽管结束数据库操作时使用 `Close()` 来关闭是惯用的方式, 但 `sql.DB` 对象被设计为长期存在. 不用频繁地打开和关闭数据库, 而要为每一个你要访问的数据库创建一个 `sql.DB` 对象, 并且在程序结束访问数据库之前都不要关闭它. 根据需要传递它, 或者将它作为全局变量来使用. 在短命函数中不要使用 `Open()` 和 `Close()`, 推荐的方式是, 将 `sql.DB` 作为一个其一个参数传递给它

如果不把 `sql.DB` 当成一个长期对象, 你可能会碰到一些糟糕的问题, 比如, 糟糕的重用和共享连接, 网络资源耗尽和由于 TCP 连接状态 `TIME_WAIT` 导致的偶发故障. 这些问题的出现表明你没有按 `database/sql` 的设计来使用它

现在万事俱备, 你可以开始使用 `sql.DB` 对象了...
