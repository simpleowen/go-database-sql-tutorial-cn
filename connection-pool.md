# 连接池

一个基本的连接池在 [database/sql](https://golang.org/pkg/database/sql/) 包中已经实现了, 无需对它做过多的控制和检查, 但是这里有些有用的东西你需要知道:

- 连接池意味着在一个数据库上可能会打开两个连接来分别独立执行两条连续的语句, 这个特征使得一个现象普遍发生, 那就是程序员们时常对他们的代码异常感到困惑. 比如, `LOCK TABLES` 语句后跟 `INSERT` 语句会阻塞, 因为 `LOCK TABLES` 和 `INSERT` 两个操作可能被两个连接独立执行, 而 `INSERT` 可能正处于那条没有保持表锁连接上

- 连接是在需要时被创建的, 在池中并没有一条空闲的连接在等待你

- 默认情况下, 连接的数量没有限制. 如果你一次要做大量的操作, 你可以创建任意数量的连接. 但是这可能引起数据库返回一个 `too many connections` 的异常

- Go 1.1 以后, 你可以使用 `db.SetMaxIdleConns(N)` 来限制池中空闲连接数, 但是这并不会限制池的大小

- Go 1.2.1 以后, 你可以使用 `db.SetMaxOpenConns(N)` 来限制数据库的总打开连接数. 不幸的是,  一个死锁的 [bug](https://groups.google.com/forum/#!msg/golang-dev/jOTqHxI09ns/x79ajll-ab4J) (现已被修复)会阻止 `db.SetMaxOpenConns(N)` 安全地在 1.2 中使用

- 回收连接的速度相当快. 使用 `db.SetMaxIdleConns(N)` 设置一个比较大的空闲连接数能减少连接的流失, 并且这有助于重复利用连接

- 长时间保持一个空闲连接可能会导致一些问题(比如这个 [Issue #257](https://github.com/go-sql-driver/mysql/issues/257) 提到的 Microsoft Azure 上使用 MySQL 的情况). 如果由于连接空闲太久导致你获取连接超时, 请尝试使用 `db.SetMaxIdleConns(0)`

- 由于重用长期连接可能会带来一些网络问题, 你也可以通过设置 `db.SetConnMaxLifetime(duration)` 来指定一个连接的最大重用次数, 这个函数会关闭未使用的连接, 比如那些可能被延迟关闭的过期连接

**链接**(译者注)

- [LOCK TABLE statement](https://www.ibm.com/support/knowledgecenter/en/SSEPGG_10.5.0/com.ibm.db2.luw.sql.ref.doc/doc/r0000972.html)
- [LOCK TABLE](https://docs.oracle.com/cd/B19306_01/server.102/b14200/statements_9015.htm)
- [Oracle LOCK TABLE语句（锁表）_Oracle 教程_w3cschool](https://m.w3cschool.cn/oraclejc/oraclejc-vxaw2r2n.html)
- [MySQL中lock tables和unlock tables浅析 - 潇湘隐者 - 博客园](https://www.cnblogs.com/kerrycode/p/6991502.html)