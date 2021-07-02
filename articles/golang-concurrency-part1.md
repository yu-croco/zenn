---
title: "Golang並行処理入門part1"
emoji: "🥷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "並行処理"]
published: true
---

# コレは何？
Golangの並行処理に入門したときのメモです。普段はHCLやyamlを中心に扱っているため、久しぶりにGolangのお勉強がてらまとめました。

part2は以下です。
https://zenn.dev/nameless_gyoza/articles/golang-concurrency-part2

# 前提
- `並行処理 is 何？`は書いてない
- Golangでよく出てくる並行処理(主にsync.WaitGroupとchannel)についてサラッと書いてる
    - 理論とか丁寧な解説は書いてない

# Golangでの並行処理

## WaitGroup
以下の場合に有効な並行処理
- 並行処理の結果を気にしない
- 他に結果を収集する手段がある

ドキュメントはこちら：[WaitGroup](https://golang.org/pkg/sync/#WaitGroup)


```go
func main() {
	var wg sync.WaitGroup
	wg.Add(1) // カウントを1増やす
	go func() {
		defer wg.Done() // goroutine終了時にWaitGroupカウントを1減らす
		fmt.Println("1st goroutine passed.")
		time.Sleep(1)
	}()

	wg.Add(1) // カウントを1増やす
	go func() {
		defer wg.Done() // goroutine終了時にWaitGroupカウントを1減らす
		fmt.Println("2nd goroutine passed.")
		time.Sleep(2)
	}()

	wg.Wait() // WaitGroupカウンターが0になるまで以降の処理を実行させない（待つ）
	fmt.Println("all goroutine finished.")
}

// 実行結果
// $ go run ./main.go
// 2nd goroutine passed.
// 1st goroutine passed.
// all goroutine finished.
```

WaitGroupで配列に対する操作を行う場合には注意が必要。以下の例では実行結果すべてで`3`が出力されている。

```go
func main() {
	l := []string{"1","2","3"}
	var wg sync.WaitGroup

	for _, s := range l {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Println(s)
		}()
	}

	wg.Wait()
	fmt.Println("all goroutine finished.")
}

// 実行結果
// $ go run ./main.go
// 3
// 3
// 3
// all goroutine finished.
```

根本的な原因としては、goroutineが走る前にfor-rangeが終わってしまうためである。`go func(){...}`が実行される前にfor-range処理が終わるのであれば、goroutineで参照している`s`は何なんだ？となるかもしれないが、これはヒープに残された値（slice lの最後の値）である。
これを回避するためには、goroutineの無名関数に対して以下のようにして値を渡す必要がある。

```go
func main() {
	l := []string{"1","2","3"}
	var wg sync.WaitGroup

	for _, s := range l {
		wg.Add(1)
		go func(s string) { // 外部から渡されるs（コピーされている）に対して型を対応させる
			defer wg.Done()
			fmt.Println(s)
		}(s) // goroutineの無名関数にsを渡す
	}

	wg.Wait()
	fmt.Println("all goroutine finished.")
}

// 実行結果
// $ go run ./main.go
// 3
// 1
// 2
// all goroutine finished.
```

## Channel
goroutine間でデータを受け渡しするためにchannelを用いる。 channelには書き込み/読み込みでき、goroutine間でのパイプラインとしての役割を果たす。

```go
var ch chan string // 読み書きどちらでも対応可能
var writer chan<- string // 送信（書き込み）専用
var reader <-chan string // 受信（読み込み）専用
```

送信（書き込み）/受信（読み込み）はコードで書くとなんとなく分かりやすい。

```go
func main() {
	strStream := make(chan string)
	go func() {
		strStream<- "hello world" // strStreamに文字列を送信（書き込み）する
	}()

	fmt.Println(<-strStream) // strStreamに書き込んだ文字列を受信（読み込み）する
	// result,ok := <-strStream このように第2戻り値を用意すると、channelに値が入っている場合にはtrue,そうでない場合にはfalseを返す
}

// 結果
// $ go run ./main.go
// hello world
```

ここで`fmt.Println(<-strStream)` が実行されてからmain関数が終了する理由は、channelはブロックを発生させるためである。channelは以下のように条件を満たすまでは処理をブロックする性質を持っている。
- 送信（書き込み）用のchannel（`chan<-`）
  - channelのキャパシティが空くまで書き込みを待機する
- 受信（読み込み）用のchannel（`<-chan`）
  - channelにデータが1つ以上入る(またはchannelがcloseされる)まで読み込みを待機する

`fmt.Println(<-strStream)` の `<-strStream` は受信（読み込み）用のchannelであり、これは送信（書き込み）が完了するまで待機するため、sleepなど入れずとも`hello world`が出力されるまで処理が終了しないのである。

上記のことから、channelはデータの`送信（書き込み）` -> `受信（読み込み）` の流れを経る必要がある（必ずしも受信する必要はないが、受信するためには送信を経る必要がある）。
送信（書き込み）を行わずに受信（読み込み）すると、以下のようにエラーとなる。

```go
func main() {
	strStream := make(chan string)
	fmt.Println(<-strStream)
}

// 実行結果
// $ go run ./main.go
// fatal error: all goroutines are asleep - deadlock!
// 
// goroutine 1 [chan receive]:
// main.main()
// /Users/<USER_NAME>/...../main.go:10 +0x5a
// exit status 2
```

### close
channelにおいて「もうコレ以上channelから値が送信されないこと」を表すために、channelを閉じることができる。 方法としては`close(cha)`でchannelを閉じることができる。
以下のように、goroutineを生成する際に `defer close(...)` とすることでchannelを閉じている。
基本的にはgoroutineを抜ける前に `defer close(...)` するのが良い。

```go
func main() {
	intStream := make(chan int, 4)
	go func() {
		defer close(intStream)
		defer fmt.Println("goroutine done.")
		for i:=0;i<5;i++ {
			fmt.Printf("Send: %d\n",i)
			intStream <- i
			time.Sleep(100*time.Millisecond)
		}
	}()

	for i := range intStream{
		fmt.Printf("Received %d.\n", i)
	}
}

// 実行結果
// $ go run ./main.go
// Send: 0
// Received 0.
// Send: 1
// Received 1.
// Send: 2
// Received 2.
// Send: 3
// Received 3.
// Send: 4
// Received 4.
// goroutine done.
```

### channelでnilを使用する
channelではいろいろな型を利用できるが、nilを使うとどうなるだろう？
結果は以下の通り受信も送信もエラーとなる。

```go
// 受信（読み取り）の場合
func main() {
	var ch chan interface{}
	<-ch
}

// 実行結果
// $ go run ./main.go
// fatal error: all goroutines are asleep - deadlock!
// 
// goroutine 1 [chan receive (nil chan)]:
// main.main()
// /Users/.../main.go:10 +0x29
// exit status 2


// 送信（書き込み）の場合
func main() {
	var ch chan interface{}
	ch<- struct{}{}
}

// 実行結果
// $ go run ./main.go
// fatal error: all goroutines are asleep - deadlock!
// 
// goroutine 1 [chan send (nil chan)]:
// main.main()
// /Users/yu-croco/workspace/Programming/Golang/sample/main.go:62 +0x4c
// exit status 2


// close
func main() {
	var ch chan interface{}
	close(ch)
}

// 実行結果
// $ go run ./main.go
// panic: close of nil channel
//
//
// goroutine 1 [running]:
// main.main()
// /Users/...../main.go:62 +0x2a
// exit status 2
```

nilの場合には受信（読み取り）も送信（書き込み）もcloseも、すべてがdeadlockエラーまたはpanicを引き起こす。
そのため利用の際には`make(chan ..)`を用いて確実に初期化する必要がある。

### for-range
for文の引数でchannelと使うことで、channelがcloseした際に勝手にループを終了してくれる。

```go
func main() {
	intStream := make(chan int)

	go func() {
		defer close(intStream)
		for i:=0;i<=5;i++{
			intStream<-i
		}
	}()

	// channelが閉じるとfor文も終了する
	for i := range intStream {
		fmt.Println(i)
	}
}


// 実行結果
// $ go run ./main.go
// 0
// 1
// 2
// 3
// 4
// 5
```

## Select
[Go言語による並行処理](https://www.amazon.co.jp/Go%E8%A8%80%E8%AA%9E%E3%81%AB%E3%82%88%E3%82%8B%E4%B8%A6%E8%A1%8C%E5%87%A6%E7%90%86-Katherine-Cox-Buday/dp/4873118468/ref=sr_1_1?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=14JT6C98S85Q7&dchild=1&keywords=go%E8%A8%80%E8%AA%9E%E3%81%AB%E3%82%88%E3%82%8B%E4%B8%A6%E8%A1%8C%E5%87%A6%E7%90%86&qid=1624781084&sprefix=go%E8%A8%80%E8%AA%9E%E3%81%AB%E3%82%88%E3%82%8B%2Caps%2C262&sr=8-1)いわく、
> select文はチャネルをまとめる糊

とのこと。
channelはgoroutineを取りまとめるものであり、selectはchannelを取りまとめるもの、という感じだろう。
以下のようにselectのcaseとして使用する。 `for` と `select` を組み合わせることで必要な処理が終わるまで待機することができる（割とよく出てくる組み合わせ）。

```go
func main() {
	start := time.Now()
	c := make(chan interface{})

	go func() {
		defer close(c)
		time.Sleep(1*time.Nanosecond)
	}()

	fmt.Println("Blocking on read...")
	for {
		select {
		case <-c: // cに対する読み込みができるようになったらこの処理を通る。今回の場合は`close(c)`が呼び出されることでこのロジックを通るようになる。
			fmt.Printf("Unblocked %v later\n", time.Since(start))
			return // ループを抜ける
		default:
			fmt.Println("waiting....")
		}
	}
}

// 実行結果
// $ go run ./main.go
// Blocking on read...
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// waiting....
// Unblocked 51.1µs later
```

# Channelを利用する際の注意点
## goroutineリーク
Golangはgoroutineを自動でGCしないため、実装者が適切にgoroutineを扱わないとメモリリークのように不要なgoroutineが残ってしまう。
goroutine自体は非常に軽量であるが、例えば無限に生成され続けるような実装の中でリークしている場合にはリソースを枯渇させてしまう可能性がある。

*goroutineリークの詳細についてはこちらの記事が非常に参考になりましたので、気になる方は御覧ください。
https://qiita.com/i_yudai/items/3336a503079ac5749c35

*goroutineリークを検出するためのOSSもあるようです
https://github.com/uber-go/goleak

習慣的には以下を実践するべきであるとのこと。
- goroutineの親子関係がある場合には、親から子のgoroutineを停止できるようにする
  - 習慣としては`done`という名の読み込みchannelを利用する
- goroutineを生成する関数は、goroutineを停止させる責務も負うこと

```go
func doWork(
	done <-chan interface{}, // 親から停止できるようにしておく
	strings <-chan string,
) <-chan interface{}{
	terminated:= make(chan interface{})
	go func() {
		defer fmt.Println("doWork exited.")
		defer close(terminated)
		for {
			select {
			case s := <-strings: // stringsに対して書き込みがされるとこの処理を通る。今回の場合main関数で `s <- "hello world"` が呼ばれることで通過可能となる
				fmt.Println(s)
			case <-done: // 親でdone channelが停止(closeが呼ばれる)された場合にはループを抜けられる。これによりgoroutineから抜けられる
				return
			}
		}
	}()
	fmt.Println("return terminated on doWork()")
	return terminated
}

func main() {
	done :=make(chan interface{})
	s := make(chan string)
	terminated := doWork(done,s)

	go func() {
		time.Sleep(1*time.Second)
		fmt.Println("closing done channel...")
		close(done)
	}()

	s <- "hello world"

	<-terminated // doWork()関数のgoroutineをmainに引き上げる
	fmt.Println("Done")
}
```

# 参考
- [Go言語による並行処理](https://www.amazon.co.jp/Go%E8%A8%80%E8%AA%9E%E3%81%AB%E3%82%88%E3%82%8B%E4%B8%A6%E8%A1%8C%E5%87%A6%E7%90%86-Katherine-Cox-Buday/dp/4873118468/ref=sr_1_1?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=14JT6C98S85Q7&dchild=1&keywords=go%E8%A8%80%E8%AA%9E%E3%81%AB%E3%82%88%E3%82%8B%E4%B8%A6%E8%A1%8C%E5%87%A6%E7%90%86&qid=1624781084&sprefix=go%E8%A8%80%E8%AA%9E%E3%81%AB%E3%82%88%E3%82%8B%2Caps%2C262&sr=8-1)
- [[Golang] Goroutine を支える技術](https://note.crohaco.net/2019/golang-goroutine/)
- [いまさら聞けないselectあれこれ](https://www.slideshare.net/lestrrat/select-66633666)
