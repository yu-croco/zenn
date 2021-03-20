---
title: "GolangでDecimalをJSON化した際にStringになる問題に対処する"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang"]
published: true
---

# 問題
GolangでDecimalを使いたい場合には [shopspring/decimal](https://github.com/shopspring/decimal) を利用するかと思いますが、これをJSONで返す際に値がStringとなる問題があります。

```go
import (
    "encoding/json"
    "fmt"
    "github.com/shopspring/decimal"
)

type User struct {
    Score decimal.Decimal `json:"score"`
}

func main() {
    score, _ := decimal.NewFromString("136.02")
    u := User{Score: score}

    jsonUser, _ := json.Marshal(u) // こんな感じになる {"score":"136.02"}
}
```

## 解決方法
[Values of the Decimal type get marshaled to JSON as strings, not numbers #21](https://github.com/shopspring/decimal/issues/21) というISSUEにもあがっていますが、[decimal.go](https://github.com/shopspring/decimal/blob/master/decimal.go#L53) に以下の設定があり、 `true` にすることでJSON変換した際にnumberとして処理されるようにできます。
ただしWARNINGとして書かれている通り桁数が大きすぎると問題になるとのことで、そこは注意が必要です。

```go
// MarshalJSONWithoutQuotes should be set to true if you want the decimal to
// be JSON marshaled as a number, instead of as a string.
// WARNING: this is dangerous for decimals with many digits, since many JSON
// unmarshallers (ex: Javascript's) will unmarshal JSON numbers to IEEE 754
// double-precision floating point numbers, which means you can potentially
// silently lose precision.
var MarshalJSONWithoutQuotes = false
```

# やってみよう
素直に `MarshalJSONWithoutQuotes = true` を設定するとdecimal型のデータをnumberとしてJSON化できました。

```go
import (
    "encoding/json"
    "fmt"
    "github.com/shopspring/decimal"
)

type User struct {
    Score decimal.Decimal `json:"score"`
}

func main() {
    decimal.MarshalJSONWithoutQuotes = true

    score, _ := decimal.NewFromString("136.02")
    u := User{Score: score}

    jsonUser, _ := json.Marshal(u) // // こんな感じになる {"score":136.02}
}
```

上記のようにmain関数に書いたりするのはちょっとアレなので、init functionに設定を混ぜるのが便利かと思います。
*init functionについては [The init Function](https://tutorialedge.net/golang/the-go-init-function/) が参考になりました。

個人的には `decimal.Decimal` を使っているstructと同じパッケージに定義するのが良さそうな気がしています(例えば以下の感じ)。

```
.
├── go.mod
├── go.sum
├── main.go
└── pkg
    └── model
        └── user
            ├── init.go // ここで設定
            └── user.go
```
