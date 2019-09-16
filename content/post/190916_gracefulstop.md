---
title: "シグナルハンドリングとContextを使って安全にGoのプログラムを終了させる"
description: "Goプログラムを途中終了させる際の安全な実装方法について書きました。"
date: 2019-09-16T19:28:00+09:00
categories:
  - 勉強の記録
tags:
  - Go
draft: false
---

こんな処理があるとします。
時間のかかる処理doSomethingをループで何回も回す処理です。

```
func main() {
	fmt.Println("func main started.")

	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		doSomethingLoop()
		wg.Done()
	}()

	wg.Wait()
	fmt.Println("func main finished.")
}

func doSomethingLoop() {
	for {
		doSomething()
	}
}

func doSomething() {
	fmt.Println("doSomething started.")
	time.Sleep(3 * time.Second) // 何か重い処理
	fmt.Println("doSomething finished.")
}
```

この時間のかかるプログラムを途中で止めたいとき、どうするでしょうか？
killコマンド(デフォルトでSIGTERM)、Ctrl+CによるSIGINT送信するなりといった手段でプログラムを終了させるかと思います。
また、GKEなどでコンテナのアプリケーションを稼働させている場合には、突然SIGTERMでコンテナが終了させられることもあります。

いつ止まっても大丈夫な処理であればよいのですが、途中で止まったら困る処理をしていることもあるでしょう。
そんなときに、シグナルハンドリングとContextを使って、プログラムに終了シグナルが届いたときの挙動をコントロールすることが大事です。

例のプログラムのdoSomethingLoopを、ループの途中でプログラムが終了しないようにコントロールするようプログラムを書き換えてみます。

まず、シグナルハンドリングを追加します。
SIGINTやSIGTERMが渡されたときにプログラムがすぐ終了しないようにします。
まずは第一段階として、シグナルを受け取ったらプログラムが終了せずにデバッグメッセージを出すように書き換えてみましょう。

```
func main() {
	fmt.Println("func main started.")

	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		doSomethingLoop()
		wg.Done()
	}()

	// ここから追加
	sigCh := make(chan os.Signal)
	signal.Notify(sigCh, os.Interrupt, syscall.SIGTERM)

	go func() {
		<-sigCh
		fmt.Println("signal received.")
	}()
	// ここまで追加

	wg.Wait()
	fmt.Println("func main finished.")
}
```

プログラムが走った状態でCtrl+CでSININTを送ると、このように出力されます。

```
signal received.
```

このままではプログラムが終了しないので(上記のプログラムはkill -9で止めます)、あとはデバッグメッセージを表示する代わりに、「ループを抜ける処理を発火する」ということをすればよいです。
ループを抜ける発火処理はcontextパッケージにより実現できます。

準備として、cancelできるContextを用意します。

```
func main() {
	fmt.Println("func main started.")

	ctx, cancel := context.WithCancel(context.Background())
	...
    
	...
	go func() {
		<-sigCh
		fmt.Println("signal received.")
		cancel()
	}()
	...
}
```

cancelを実行することで、チャネルctx.doneがクローズされます
ctxをdoSomethingLoopに渡し、doSomethingLoop側でctx.doneのクローズを検知して、ループの切れ目でdoSomethingLoopを抜けるように処理を書いておきます。selectを使えばチャネルのクローズを検知できます。

```
func doSomethingLoop(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		default:
			doSomething()
		}
	}
}
```

全体のプログラムとしてはこのようになります。

```
func main() {
	fmt.Println("func main started.")

	ctx, cancel := context.WithCancel(context.Background())
	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		doSomethingLoop(ctx)
		wg.Done()
	}()

	sigCh := make(chan os.Signal)
	signal.Notify(sigCh, os.Interrupt, syscall.SIGTERM)

	go func() {
		<-sigCh
		fmt.Println("signal received.")
		cancel()
	}()

	wg.Wait()
	fmt.Println("func main finished.")
}

func doSomethingLoop(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		default:
			doSomething()
		}
	}
}

func doSomething() {
	fmt.Println("doSomething started.")
	time.Sleep(3 * time.Second) // 何か重い処理
	fmt.Println("doSomething finished.")
}

```

これを実行して途中でCtrl+Cでプログラムを終了してみましょう。

```
$ go run main.go
func main started.
doSomething started.
doSomething finished.
doSomething started.
^Csignal received.
doSomething finished.
func main finished.
```

---

さて、今まで見てきた例は単純にdoSomethingがループされたプログラムだったのですが、前回の記事のようにgoroutineによって並行化された状態の場合に、cancel以後に新しくdoSomethingが立ち上がらないように、そして実行中のdoSomethingがすべて実行終了してからmainを終了させたい場合はどのようにすればよいでしょうか？

まずはこのように組んでみました。

```
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	inputs := []string{}
	for i := 1; i <= 20; i++ {
		inputs = append(inputs, strconv.Itoa(i))
	}

	eg := errgroup.Group{}
	eg.Go(func() error {
		return doSomethingConcurrent(ctx, inputs)
	})

	sigCh := make(chan os.Signal)
	signal.Notify(sigCh, os.Interrupt, syscall.SIGTERM)

	go func() {
		<-sigCh
		fmt.Println("signal received.")
		cancel()
	}()

	if err := eg.Wait(); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}

	fmt.Println("func main completed.")
}

func doSomethingConcurrent(ctx context.Context, inputs []string) error {
	allResource := int64(3)
	doSomethingResource := int64(1)

	sem := semaphore.NewWeighted(allResource)
	var wg sync.WaitGroup
	for _, v := range inputs {
		v := v
		if err := sem.Acquire(ctx, doSomethingResource); err != nil {
			return err
		}
		go func() {
			wg.Add(1)
			doSomething(v)
			sem.Release(doSomethingResource)
			wg.Done()
		}()
	}
	wg.Wait()
	return nil
}

func doSomething(input string) error {
	fmt.Println("doSomething start: " + input)
	time.Sleep(3 * time.Second)
	fmt.Println("doSomething end: " + input)
	return nil
}

```

```
$ go run main.go
doSomething start: 3
doSomething start: 2
doSomething start: 1
doSomething end: 1
doSomething end: 3
doSomething end: 2

...

doSomething start: 7
doSomething end: 4
doSomething start: 8
doSomething end: 6
doSomething start: 9
^Csignal received.
context canceled
exit status 1
```

doSomethingConcurrentを抜けられてはいるのですが、途中でぶつ切りになってしまったgoroutineがいるようです(7,8,9のdoSomethingがendしていない)。
これではいけません。

doSomethingConcurrentに改良を加えたのが下記のプログラムです。

```
func doSomethingConcurrent(ctx context.Context, inputs []string) error {
	allResource := int64(3)
	doSomethingResource := int64(1)

	sem := semaphore.NewWeighted(allResource)
	var wg sync.WaitGroup
	for _, v := range inputs {
		v := v
		if err := sem.Acquire(ctx, doSomethingResource); err != nil {
			wg.Wait() // <-- ポイント
			return err
		}
		go func() {
			wg.Add(1)
			doSomething(v)
			sem.Release(doSomethingResource)
			wg.Done()
		}()
	}
	wg.Wait()
	return nil
}

```

cancelによるctx.doneのクローズを検知してsem.Acquireがエラーを返した際に、wg.Wait()を挟むことによって既に走っているgoroutineの終了を待つようにしたという感じです。このプログラムを走らせて、途中終了させてみましょう。

```
$ go run main.go
doSomething start: 3
doSomething start: 2
doSomething start: 1

...

doSomething start: 7
doSomething start: 8
doSomething start: 9
^Csignal received.
doSomething end: 9
doSomething end: 7
doSomething end: 8
context canceled
exit status 1
```

ちゃんとgoroutineがすべて終了してからmainが終了できているのが分かります。

### まとめ

contextのキャンセルを適切に行って安全なプログラムを書いていきましょう。
