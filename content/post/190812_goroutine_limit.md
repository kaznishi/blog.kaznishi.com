---
title: "goroutineの同時実行数に制限をかける"
description: "goroutineの同時実行数に制限をかける方法を2つ紹介しています。1つはchannelによる方法。もう一つはgolang.org/x/sync/semaphoreパッケージによる方法です。"
date: 2019-08-12T23:00:00+09:00
categories:
  - 勉強の記録
tags:
  - Go
draft: false
---
Goにはgoroutineという強力な仕組みが備わっています。goroutineを使うことによって並行処理を簡単に実装することができます。
例えば次のような、実行に時間がかかる関数`doSomething`があったとします。

```
func doSomething(input string) {
	time.Sleep(1 * time.Second) // (何か時間がかかる処理)
	fmt.Println("input is " + input)
}
```

これを次のように直列に実行すればinputsの数分だけ時間がかかります。

```
func Baseline(inputs []string) {
	for _, v := range inputs {
		doSomething(v)
	}
}
```

しかし、次のようにgoroutineで並行化することで実行時間を大幅に短縮することができます。

```
func Concurrent1(inputs []string) {
	var wg sync.WaitGroup
	wg.Add(len(inputs))
	for _, v := range inputs {
		v := v
		go func() {
			doSomething(v)
			wg.Done()
		}()
	}
	wg.Wait()
}
```

ただし、`doSomething`がDBに接続するような処理であったり、Webページをスクレイピングするような処理であったとき、何も制限をかけずにdoSomethingを大量に実行するとDBやWebページが負荷に耐えきれなくなってしまいます。
このような場合にはgoroutineの実行数を制限しなければいけません。
それではdoSomethingを実行するgoroutineの同時実行数に制限をかける方法を2つ紹介していきます。

## channelによる方法

まず紹介するのはchannelを使った方法です。
バッファがない状態のchannelに対して送信を試みようとするとブロックする特性を利用します。
まず、制限をかけたい同時実行数分(limit)だけのバッファをもったchannelを用意しておきます。

```
limit := 3
slots := make(chan struct{}, limit)
```

あとはgoroutineの起動の直前にchannelに対してなんでもよい(容量ゼロのstruct{}{}がオススメ)のでバッファを埋めるための送信処理を行い、goroutine内でdoSomethingの終了時にchannelからの受信を行いバッファを空けてやります。

```
slots <- struct{}{}
go func() {
    doSomething(v)
    <-slots
}()
```

全体としてはこのようなコードになります。

```
func Concurrent2(inputs []string) {
	limit := 3
	slots := make(chan struct{}, limit)
	var wg sync.WaitGroup
	wg.Add(len(inputs))
	for _, v := range inputs {
		v := v
		slots <- struct{}{}
		go func() {
			doSomething(v)
			<-slots
			wg.Done()
		}()
	}
	wg.Wait()
}
```

## semaphoreパッケージを使った方法

次に`golang.org/x/sync/semaphore`パッケージを用いた方法を紹介します。
このパッケージではWeighted Semaphoreの実装を利用することができます。
初期化の際に使用できるリソースを指定して用意し、何か処理を実行する際にはそのリソースのうちいくつかを指定して消費させ、処理が終わった際にはそのリソースを解放します。
今回行いたい同時実行数制限においては、同時実行数分をリソースとして用意しておき、doSomethingでリソースを1つ消費するとして用います。

```
func Concurrent3(inputs []string) {
	allResource := int64(3)
	doSomethingResource := int64(1)

	sem := semaphore.NewWeighted(allResource)
	var wg sync.WaitGroup
	wg.Add(len(inputs))
	for _, v := range inputs {
		v := v
		sem.Acquire(context.Background(), doSomethingResource)
		go func() {
			doSomething(v)
			sem.Release(doSomethingResource)
			wg.Done()
		}()
	}
	wg.Wait()
}
```

## どちらの方法がよい?

channelの方法でも`golang.org/x/sync/semaphore`の方法でも同時実行数の制限はかけられます。
ただし、今回紹介した方法はキャンセル処理のことが考慮されていません。
キャンセル処理についてはContextを使って実装していくことになると思います。
そうなってくると、デフォルトでContext対応しているsemaphoreを使っておく方が便利かと思います。
せっかくパッケージがあるのだったら、それを使っておきましょうという感じですね。
同時実行数の制限だけかけて後は特に気にしないのであればどちらでもよいと思います。
