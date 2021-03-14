---
title: "Golangのslice header入門"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang"]
published: false
---

Golangのsliceを関数の引数として渡した際の挙動が良く分からなかったので調べてみました。

# 結論
ひとまず雑な結論から

- sliceの変数を関数の引数に渡すと `slice header` が値渡しがなされる
    - そのため、呼び出された関数内でlenやcapを変更しても、元の値には影響を及ぼさない
- `slice header` は値渡しされるが、sliceの要素の書き換えは可能
    - オリジナルの `slice header` も、値渡しで渡された `slice header` も、オリジナルのsliceの値へのポインタを見ているため

# 実験
sliceは参照系なので元の変数を更新できるでしょ、というわけで以下のようなコードを動かしてみる。

```go
import (
    "fmt"
    "strings"
)

func toLower(s []string) {
    for idx, str := range s {
        s[idx] = strings.ToLower(str)
    }
}

func addC(s []string) {
    s = append(s, "C")
}

func main() {
    s1 := []string{"A","B"}
    fmt.Println(s1) // [A B]

    toLower(s1)
    fmt.Println(s1) // [a b] <- わかる

    s2 := []string{"A","B"}
    fmt.Println(s2) // [A B]

    addC(s2)
    fmt.Println(s2) // [A B] <- ほう!?
}
```

`addC()` の際に変数s2に"C"が加わっていない...。

謎なのでみんな大好きprintデバッグをしてみよう。

```go
import (
  "fmt"
  "strings"
)

func toLower(s []string) {
    for idx, str := range s {
        s[idx] = strings.ToLower(str)
    }
    fmt.Printf("[toLower] ptr=%v, len=%v, cap=%v\n", &s[0], len(s), cap(s))
}

func addC(s []string) {
    s = append(s, "C")
    fmt.Printf("[addC] ptr=%v, len=%v, cap=%v\n", &s[0], len(s), cap(s))
}

func main() {
    s1 := []string{"A","B"}
    
    fmt.Printf("[s1その1] ptr=%v, len=%v, cap=%v\n", &s1[0], len(s1), cap(s1))
    toLower(s1)
    fmt.Printf("[s1その2] ptr=%v, len=%v, cap=%v\n", &s1[0], len(s1), cap(s1))
    
    fmt.Println("=======================")
    s2 := []string{"A","B"}
    fmt.Printf("[s2]その1 ptr=%v, len=%v, cap=%v\n", &s2[0], len(s2), cap(s2))
    addC(s2)
    fmt.Printf("[s2]その2 ptr=%v, len=%v, cap=%v\n", &s2[0], len(s2), cap(s2))
}
```

結果は以下の通り。
`s1` `s2` ともにsliceの0番目の値のポインタ(1番目の要素を検証しても同じような感じの結果になる)は、mainで生成したものと関数側で呼び出されたものは同じとなっている。
同じポインタに対して値を操作してるんだから、 `s1` に関しては「そうっすね」って感じではある。

`s2` は `[toLower]` のところでは `len=3, cap=4` なのに、関数を実行したあとのmainでは `len=2, cap=2` のままである（は？？）。

```
[s1その1] ptr=0xc0000b6000, len=2, cap=2
[toLower] ptr=0xc0000b6000, len=2, cap=2
[s1その2] ptr=0xc0000b6000, len=2, cap=2
=======================
[s2]その1 ptr=0xc0000b6020, len=2, cap=2
[addC] ptr=0xc0000b4040, len=3, cap=4
[s2]その2 ptr=0xc0000b6020, len=2, cap=2
```

この理由は `sliceが関数の引数に渡される際、sliceのslice headerを値渡しで渡しているから` であるらしい。

# slice header is 何？
sliceの変数には以下の値を保持している。一般的に `slice header` と呼ばれるらしい。

- ptr : 0番目の要素へのポインタ
- len:そのスライスの要素数（長さ）
- cap : そのスライスの容量（キャパシティ）

sliceを関数に渡す際、`slice header` を値渡しで渡す（参照渡しではない！）。イメージとしては以下の感じである。

![](https://storage.googleapis.com/zenn-user-upload/2k11y10zz1ntjdorslbiblk7xiut)

`s2` に `addC` した際には、値渡しされた `slice header` に対して要素を追加したため、元の `s2` には影響を与えない。

![](https://storage.googleapis.com/zenn-user-upload/lh359di4m9p6pa91c3s2am76f97p)

# sliceは何故元データを更新できるのか？
「sliceは参照系なので元の値が更新できる」といった感じの説明をよく聞きます（実際、値は書き換えられている）が、先ほどの説明では`slice header` は値渡しなのでどういうこっちゃとなりそうです。
何でやと思いますが、実際は先程の図が示しているように、 オリジナルの `slice header` もコピーされた `slice header` も、slice内の値は同じものを参照しています。そのため、コピーされている `slice header` を通してsliceの値を更新すると、オリジナルのslice変数にも影響を与えることができるのである。

# 雑感
sliceｶﾝｾﾞﾝﾆﾘｶｲｼﾀ!

# 参考
- [GolangのSliceを関数の引数に渡した時の挙動](https://christina04.hatenablog.com/entry/2017/09/26/190000)
- [Golang tips: why pointers to slices are useful and how ignoring them can lead to tricky bugs](https://medium.com/swlh/golang-tips-why-pointers-to-slices-are-useful-and-how-ignoring-them-can-lead-to-tricky-bugs-cac90f72e77b)
- [Effective Go #Slice](https://golang.org/doc/effective_go#slices)
- [【Go】スライス変数の仕組み -スライスヘッダについて](https://www.ohitori.fun/entry/how-slice-variables-work-about-slice-header)
- [The Go Blog Arrays, slices (and strings): The mechanics of 'append'](https://blog.golang.org/slices)