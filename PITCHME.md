---

### golangのゼロ値ではまった話し
### @tap1ra

---

### 自己紹介
- GMOペパボ インフラチーム
- キャリア
  + SIerで保険屋さん向けSE
  + ペパボでロリポ・ヘテムル
- 釣り

---

### golangのゼロ値について

ある型の変数に値を代入して初期化しなかった時の値

| 型        | ゼロ値                    |
| --------- | ------------------------- |
| boonlean  | false                     |
| int       | 0                         |
| float     | 0.0                       |
| string    | ""               |
| pointer   | nil |
| function  | nil |
| interface | nil |
| slice     | nil |
| channel   | nil |
| map       | nil |

---

### nilじゃない・・・？
boolean: false
int: 0

---

### 今回はまったライブラリ

https://github.com/jinzhu/gorm

> The fantastic ORM library for Golang, aims to be developer friendly.

---

### gormの基本
userテーブルのid=1を更新する場合

```golang
type User struct {
  ID uint
  Name string
  Anyflg bool
}

user = User{ID: 1, Name: "Tro", Anyflg: true}

db, _ := gorm.Open("mysql", "xxxxxxxxxxx")
db.Model(&user).Where('id = 1').UpdateColumns(user)
```

---

### 以下の内容に更新される
| id  | name | anyflg |
| --- | ---- | ------ |
| 1   | taro | true   |


---

### 意図と違う動作するパターン

```golang
type User struct {
  ID uint
  Name string
  Anyflg bool
}

user = User{ID: 1, Name: "Tro", Anyflg: false} ※ Anyflg: falseに注目

db, _ := gorm.Open("mysql", "xxxxxxxxxxx")
db.Model(&user).Where('id = 1').UpdateColumns(user)
```

---

### 実際にはanyflgが更新されなかった
anyflgはtrueのまま

| id  | name | anyflg |
| --- | ---- | ------ |
| 1   | taro | true   |

---

### なぜ？？？

gormの内部実装でboolean型のfalseは値が指定されていないという扱いをしていた

---

### 詳細
以下のコード

```golang
func isBlank(value reflect.Value) bool {
  switch value.Kind() {
  case reflect.String:
    return value.Len() == 0
  case reflect.Bool:
    return !value.Bool()
  case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
    return value.Int() == 0
 略
}
```

---

### 細かい話は以下のブログで！
- [gormのUpdateColumnsでモデル内のboolゼロ値(false)を持ったカラムが更新されなかった - カメニッキ](http://tapira.hatenablog.com/entry/2017/08/09/173718)
- [gormのUpdateColumnsでモデル内のboolゼロ値(false)を持ったカラムが更新されなかった の続き - カメニッキ](http://tapira.hatenablog.com/entry/2017/08/11/182249)

---

### どうすればいいか

