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

---

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
db.Model(&user).UpdateColumns(user)
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
db.Model(&user).UpdateColumns(user)
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

boolポインタ型を使用する

---

### 以下のようにモデルの定義を変更

```golang
type User struct {
  ID uint
  Name string
  Anyflg *bool
}
```

こうするとゼロ値はfalseではなくnilになる

---

### gormのゼロ値判定処理においても、Ptr型であれば以下のように判定される

```golang
func isBlank(value reflect.Value) bool {
  switch value.Kind() {
略
  case reflect.Interface, reflect.Ptr:
    return value.IsNil()
  }
```

---

### 先程意図と違う動作するパターンは以下のように書き換える

```golang
type User struct {
  ID uint
  Name string
  Anyflg *bool
}

// falseのポインタを渡す必要があるため予め定義が必要
var PFalse = &[]bool{false}[0]

user = User{ID: 1, Name: "Tro", Anyflg: &PFalse}

db, _ := gorm.Open("mysql", "xxxxxxxxxxx")
db.Model(&user).UpdateColumns(user)
```

---

### 意図したとおりに更新された
| id  | name | anyflg |
| --- | ---- | ------ |
| 1   | taro | false   |

---

### 同様のことがuint型などでも言える
id=0の更新が行えないため、同様にuintポインタ型にしてあげなくてはいけない

---

### 問題はここだけじゃなく更新対象絞込でも同じことが

以下のような場合

| id  | name | anyflg |
| --- | ---- | ------ |
| 1   | taro | false   |
| 2   | jiro | false   |
| 3   | hanako | false   |

---

```golang
type User struct {
  ID uint
  Name string
  Anyflg *bool
}

// IDにuintのゼロ値である0を指定
user = User{ID: 0}

db, _ := gorm.Open("mysql", "xxxxxxxxxxx")
// id=0で絞込しているつもりが、未指定となり全件更新が走る！！！！！！
db.Model(&user).Update('anyflg=true')
```

---

### 更新された・・・
| id  | name | anyflg |
| --- | ---- | ------ |
| 1   | taro | true   |
| 2   | jiro | true   |
| 3   | hanako | true   |

---

### まとめ
- golangのゼロ値が、型によっては他言語と違う形式になってる
- ライブラリのゼロ値判定によって、想像と異なる動作することがあるので注意が必要

---

### 何か他にいいやり方あれば教えてほしいです！

---

### おしまい