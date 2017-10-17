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


### はまったこと