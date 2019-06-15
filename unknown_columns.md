# 使用未知列

`Scan()` 函数要求传入确切数量的目标变量, 如果你不知道 `query` 返回的内容怎么办?

如果不知道 `query` 会返回多少列, 你可以使用 `Columns()` 函数来获取一个列名的列表. 通过检查列表的长度就能知道 `query` 返回了多少列, 然后你就可以传递正确的参数 slice 给 `Scan()` 函数. 例如, MySQL 的一些分支在使用 `SHOW PROCESSLIST` 命令时返回不同的列, 所以你必须对此做好准备, 否则将引发一个异常. 以下是可行的处理方式:

```go
cols, err := rows.Columns()
if err != nil {
	// handle the error
} else {
	dest := []interface{}{ // Standard MySQL columns
		new(uint64), // id
		new(string), // host
		new(string), // user
		new(string), // db
		new(string), // command
		new(uint32), // time
		new(string), // state
		new(string), // info
	}
	if len(cols) == 11 {
		// Percona Server
	} else if len(cols) > 8 {
		// Handle this case
	}
	err = rows.Scan(dest...)
	// Work with the values in dest
}
```

如果你不知道列名和它们的类型, 你应该使用 `sql.RawBytes`

```go
cols, err := rows.Columns() // Remember to check err afterwards
vals := make([]interface{}, len(cols))
for i, _ := range cols {
	vals[i] = new(sql.RawBytes)
}
for rows.Next() {
	err = rows.Scan(vals...)
	// Now you can check each element of vals for nil-ness,
	// and you can use type introspection and type assertions
	// to fetch the column into a typed variable.
}
```
