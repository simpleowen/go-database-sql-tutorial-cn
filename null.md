# 使用空值

可空列不太招人喜欢, 它会造成大量丑陋的代码. 如果可以, 应该尽量避开它们. 如果不行, 你需要使用 `database/sql` 包中特殊的类型或自定义类型来处理.

可空的类型包括: 布尔, 字符串, 整型, 浮点型等. 以下是使用方法:

```go
for rows.Next() {
	var s sql.NullString
	err := rows.Scan(&s)
	// check err
	if s.Valid {
	   // use s.String
	} else {
	   // NULL value
	}
}
```

可空类型的限制，以及在需要更有说服力的情况下避免可空列的原因：

1. 没有 `sql.NullUint64` 或 `sql.NullYourFavoriteType` 类型, 你需要自己定义.
2. 可空性可能很棘手, 而且在未来某个时刻可能会出现问题. 当你误以为某些东西不是空的时候, 你的程序将会崩溃, 或许很少, 以至于你在交付之前都没能捕获异常.
3. Go 的一个好处是它为每个变量提供了一个有用的默认零值. 但这不是使得可空类型更安全的有效方式.

如果你想自定义类型来处理空值, 可以参考 `sql.NullString` 的设计.

如果你无法避免在数据库中使用空值, 有一个大多数数据库都支持的方法, 即 `COALESCE()`. 像下面这样的代码可能是你可以使用的，而不会引入无数的 `sql.Null*` 类型。

```go
rows, err := db.Query(`
	SELECT
		name,
		COALESCE(other_field, '') as otherField
	WHERE id = ?
`, 42)

for rows.Next() {
	err := rows.Scan(&name, &otherField)
	// ..
	// If `other_field` was NULL, `otherField` is now an empty string. This works with other data types as well.
}
```
