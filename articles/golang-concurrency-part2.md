---
title: "Golang並行処理入門part2"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "並行処理"]
published: false
---

# コレは何？
Golangの並行処理に入門したときのメモであり、以下の記事の続きです。普段はHCLやyamlを中心に扱っているため、久しぶりにGolangのお勉強がてらまとめました。
https://zenn.dev/nameless_gyoza/articles/golang-concurrency-part1

# 前提
- `並行処理 is 何？`は書いてない
- Golangの並行処理の一つであるContextについてサラッと書いてる

# Context
## どんなものか？
goroutineとchannelだけで頑張ろうとするとタイムアウトやgoroutineのcancelハンドリングなどに対する考慮が難しいことが多い。
contextを使うことでこれらの考慮をよりお手軽に対応できるようになる（しかも標準パッケージとして！）。

[Package context](https://tip.golang.org/pkg/context/)でも以下のような説明がある。
>Package context defines the Context type, which carries deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.

[The Go Blog - Go Concurrency Patterns: Context](https://blog.golang.org/context)にはこのように書かれている。
> At Google, we developed a context package that makes it easy to pass request-scoped values, cancelation signals, and deadlines across API boundaries to all the goroutines involved in handling a request. The package is publicly available as context. This article describes how to use the package and provides a complete working example.


Contextのコアとなるのは `Context` typeである。
※コードは[Source file src/context/context.go](https://golang.org/src/context/context.go?s=2460:6019#L52)から拝借

```go
type Context interface {
    // Deadline returns the time when work done on behalf of this context
    // should be canceled. Deadline returns ok==false when no deadline is
    // set. Successive calls to Deadline return the same results.
    Deadline() (deadline time.Time, ok bool)

    // Done returns a channel that's closed when work done on behalf of this
    // context should be canceled. Done may return nil if this context can
    // never be canceled. Successive calls to Done return the same value.
    // The close of the Done channel may happen asynchronously,
    // after the cancel function returns.
    //
    // WithCancel arranges for Done to be closed when cancel is called;
    // WithDeadline arranges for Done to be closed when the deadline
    // expires; WithTimeout arranges for Done to be closed when the timeout
    // elapses.
    //
    // Done is provided for use in select statements:
    //
    //  // Stream generates values with DoSomething and sends them to out
    //  // until DoSomething returns an error or ctx.Done is closed.
    //  func Stream(ctx context.Context, out chan<- Value) error {
    //  	for {
    //  		v, err := DoSomething(ctx)
    //  		if err != nil {
    //  			return err
    //  		}
    //  		select {
    //  		case <-ctx.Done():
    //  			return ctx.Err()
    //  		case out <- v:
    //  		}
    //  	}
    //  }
    //
    // See https://blog.golang.org/pipelines for more examples of how to use
    // a Done channel for cancellation.
    Done() <-chan struct{}

    // If Done is not yet closed, Err returns nil.
    // If Done is closed, Err returns a non-nil error explaining why:
    // Canceled if the context was canceled
    // or DeadlineExceeded if the context's deadline passed.
    // After Err returns a non-nil error, successive calls to Err return the same error.
    Err() error

    // Value returns the value associated with this context for key, or nil
    // if no value is associated with key. Successive calls to Value with
    // the same key returns the same result.
    //
    // Use context values only for request-scoped data that transits
    // processes and API boundaries, not for passing optional parameters to
    // functions.
    //
    // A key identifies a specific value in a Context. Functions that wish
    // to store values in Context typically allocate a key in a global
    // variable then use that key as the argument to context.WithValue and
    // Context.Value. A key can be any type that supports equality;
    // packages should define keys as an unexported type to avoid
    // collisions.
    //
    // Packages that define a Context key should provide type-safe accessors
    // for the values stored using that key:
    //
    // 	// Package user defines a User type that's stored in Contexts.
    // 	package user
    //
    // 	import "context"
    //
    // 	// User is the type of value stored in the Contexts.
    // 	type User struct {...}
    //
    // 	// key is an unexported type for keys defined in this package.
    // 	// This prevents collisions with keys defined in other packages.
    // 	type key int
    //
    // 	// userKey is the key for user.User values in Contexts. It is
    // 	// unexported; clients use user.NewContext and user.FromContext
    // 	// instead of using this key directly.
    // 	var userKey key
    //
    // 	// NewContext returns a new Context that carries value u.
    // 	func NewContext(ctx context.Context, u *User) context.Context {
    // 		return context.WithValue(ctx, userKey, u)
    // 	}
    //
    // 	// FromContext returns the User value stored in ctx, if any.
    // 	func FromContext(ctx context.Context) (*User, bool) {
    // 		u, ok := ctx.Value(userKey).(*User)
    // 		return u, ok
    // 	}
    Value(key interface{}) interface{}
}
```

# Contextを利用する上でのお作法
[Package context](https://golang.org/pkg/context/)には以下のようなルールが記載されている。
- Do not store Contexts inside a struct type; instead, pass a Context explicitly to each function that needs it. The Context should be the first parameter, typically named ctx:
  - 詳細はこのブログがわかりやすかった: [The Go Blog - Contexts and structs](https://blog.golang.org/context-and-structs)
```go
// Bad
type User struct{
 ctx    context.Context
 userID string
 ...
}

// Good
func DoSomething(ctx context.Context, arg Arg) error {
// ... use ctx ...
}
```

- Do not pass a nil Context, even if a function permits it. Pass context.TODO if you are unsure about which Context to use.
- Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.
  - 型安全ではないため、APIリクエストに伴って付与される情報に限り利用する
    - 詳細はこのブログがわかりやすかった: [Peter Bourgon - Context](https://peter.bourgon.org/blog/2016/07/11/context.html)
    > Good examples of request scoped data include user IDs extracted from headers, authentication tokens tied to cookies or session IDs, distributed tracing IDs, and so on.

```go
// Don't do this.
func MyMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func (w http.ResponseWriter, r *http.Request) {
		// 型安全じゃないから無闇に使うものじゃない
		db := r.Context().Value(DatabaseContextKey).(Database)
		// Use db somehow 
		next.ServeHTTP(w, r) 
	})
}
```

- The same Context may be passed to functions running in different goroutines; Contexts are safe for simultaneous use by multiple goroutines.



# 参考
- [Go言語による並行処理](https://www.amazon.co.jp/Go%E8%A8%80%E8%AA%9E%E3%81%AB%E3%82%88%E3%82%8B%E4%B8%A6%E8%A1%8C%E5%87%A6%E7%90%86-Katherine-Cox-Buday/dp/4873118468/ref=sr_1_1?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=14JT6C98S85Q7&dchild=1&keywords=go%E8%A8%80%E8%AA%9E%E3%81%AB%E3%82%88%E3%82%8B%E4%B8%A6%E8%A1%8C%E5%87%A6%E7%90%86&qid=1624781084&sprefix=go%E8%A8%80%E8%AA%9E%E3%81%AB%E3%82%88%E3%82%8B%2Caps%2C262&sr=8-1)
- [The Go Blog - Go Concurrency Patterns: Context](https://blog.golang.org/context)
- [The Go Blog - Contexts and structs](https://blog.golang.org/context-and-structs)
- [Package context](https://tip.golang.org/pkg/context/)
- [Go1.7のcontextパッケージ](https://deeeet.com/writing/2016/07/22/context/)
- [Peter Bourgon - Context](https://peter.bourgon.org/blog/2016/07/11/context.html)
