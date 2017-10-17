---

### golangのゼロ値ではまった話し
### @tap1ra

#### Fukuoka.go#9 LT

---

### 自己紹介
- GMOペパボ インフラチーム
- キャリア
  + SIerで保険屋さん向けSE
  + ペパボでロリポ・ヘテムル
- 釣り
- ここ半年golangを触る機会があって使ってます

---

### golangのゼロ値について

ある型の値を初期化しなかった時の値

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

### nilにならない型がある!!

| 型        | ゼロ値                    |
| --------- | ------------------------- |
| boonlean  | false                     |
| int       | 0                         |
| float     | 0.0                       |
| string    | ""               |

---

### 本題: 今回はまったライブラリ

https://github.com/jinzhu/gorm

> The fantastic ORM library for Golang, aims to be developer friendly.

golangでORマッパーを探すと上位にでてくる有名所のライブラリ

---

### gormの基本的な使い方
例) userテーブルのid=1のレコードを更新する場合

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

### 以下のように更新される
| id  | name | anyflg |
| --- | ---- | ------ |
| 1   | taro | true   |


---

### 意図した動作にならない例

boolで定義した列をfalseに更新したい

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

### 更新されなかった
anyflgはtrueのまま

| id  | name | anyflg |
| --- | ---- | ------ |
| 1   | taro | true   |

---

### なぜか？？？

gormでは差分更新可能をサポートしており、モデルに更新したい列だけ指定して渡すことができる。  
例) Name列のみ更新したい　など  
その時 `値が指定されているかいないか` を内部実装で `ゼロ値かどうか` で判定している。  
よって、bool型のfalseはゼロ値であるため `更新する列として指定されていない` ものとして扱われていた  

---

### 詳細
以下のコード

```golang
// isBlankのものはUPDATEクエリ組立時にスキップされる
func isBlank(value reflect.Value) bool {
  switch value.Kind() {
  case reflect.String:
    return value.Len() == 0
  case reflect.Bool:
    return !value.Bool()  // falseの場合trueになる
  case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
    return value.Int() == 0
 略
}
```

---

### 細かい話は以下のブログに書きました
- [gormのUpdateColumnsでモデル内のboolゼロ値(false)を持ったカラムが更新されなかった - カメニッキ](http://tapira.hatenablog.com/entry/2017/08/09/173718)
- [gormのUpdateColumnsでモデル内のboolゼロ値(false)を持ったカラムが更新されなかった の続き - カメニッキ](http://tapira.hatenablog.com/entry/2017/08/11/182249)

---

### どうすればいいか？

boolポインタ型を使用する

---

### 以下のようにモデルの定義を変更

```golang
type User struct {
  ID uint
  Name string
  Anyflg *bool  //ここ
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

### さらに、問題はここだけじゃなく更新対象を絞込む場合にも

以下のような場合

| id  | name | anyflg |
| --- | ---- | ------ |
| 0   | tahira | false   |
| 1   | taro | false   |
| 2   | jiro | false   |
| 3   | hanako | false   |

---

id=0のユーザを更新するつもりで以下のようなコードにすると
```golang
type User struct {
  ID uint
  Name string
  Anyflg *bool
}

// IDにuintのゼロ値である0を指定
user = User{ID: 0}

db, _ := gorm.Open("mysql", "xxxxxxxxxxx")
db.Model(&user).Update('anyflg=true')
```

---

### 全件更新された・・・
| id  | name | anyflg |
| --- | ---- | ------ |
| 0   | tahira | true   |
| 1   | taro | true   |
| 2   | jiro | true   |
| 3   | hanako | true   |

---

### まとめ
- golangのゼロ値が型によっては他言語と違う形式になってる
- ライブラリのゼロ値判定によって、想像と異なる動作することがあるので注意が必要

---

### 何か他にいいやり方あれば教えてほしいです！

---

### おしまい
