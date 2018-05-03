---
title: "Goに入門する - A Tour of Go"
description: "Goの勉強を始めました。手始めにA Tour of Goに手を付けました。"
date: 2018-05-03T14:24:30+09:00
categories:
  - 勉強の記録
tags:
  - Go
draft: false
---

ゴー(Go)ルデンウィークということで、Go言語の勉強を始めました。  
仕事でバリバリ実装するという予定はまったくないのですが、最近お世話になっているツールにGo実装のものが増えてきた感があるのと、最近勉強に買っている色んな本にもGoの例が載っていたりと、色々なところで見かけるようになってきたのでそろそろやっておかないとな、という感じです。  

何から始めようかなと思ったのですが、[kakakakakkuさんの記事](http://kakakakakku.hatenablog.com/entry/2017/10/16/081755)によると[Voyageさんのインターン研修資料](https://go-talks.appspot.com/github.com/voyagegroup/talks/2017/treasure-go/intro.slide#1)がよいということでした。
しかしその研修資料内に準備として [A Tour of Go](https://go-tour-jp.appspot.com/welcome/1) やっておこう！とあり、まずはそれに従ってA Tour of Goを通してみました。

以下、A Tour of Goやりながら取った自分用のメモ

- Exported names
    - 大文字開始: エクスポートされており、パッケージ外部から参照可能
    - 小文字開始: パッケージ内のみ参照可能
- Functions
    - 関数の引数は `(x int)` のように `引数名 型` の順番で書く
    - `(x,y int)` は、xもyもintという意味の省略記法
- Multiple results
    - 関数は複数の戻り値を返せる
- Named return values
    - 戻り値の変数に名前をつけることが可能
    - 名前をつけた戻り値の変数は、returnを書かなくてもよい(naked return)が、長い関数の中で使うと可読性を損なう
- Short variable declarations
    - 関数の中では `var hoge = 値` の代わりに短い代入文 `hoge := 値` で暗黙の型宣言ができる
- Zero values
    - 変数に初期値を与えずに宣言すると、ゼロ値が入った状態になる
        - ゼロ値
            - 数値型: 0
            - bool型: false
            - string型: ""
- forの省略記法
    - 初期化と後処理ステートメントの記述は任意
    - loop条件も省略することができ、その場合は無限ループになる
- If with a short statement
    - ifの条件の前に、評価のための簡単なステートメントを記述可能
        - `if v := math.Pow(x, n); v < lim { ... }`
        - このとき `v` はifの中でのみ使用可能
- Defer
    - deferで遅延実行
    - deferが複数のときはLIFOで処理
- (ポインタと構造体は昔やったC言語を思い出せば大体OK?)
- Arrays
    - `[n]T` という宣言の仕方
        - 例 `[2]string`
- Slices
    - 配列 => 固定長
    - スライス => 可変長
        - `[]T`
    - スライスは配列への参照
    - 長さ `len` と容量 `cap`
    - `make(型, 長さ, 容量)`
    - sliceへの要素追加 `append`
- Range
    - forでのスライスやマップの反復処理
    - インデックスのみが必要な場合は `_` で捨てる
        - `for _, value := range hogeSlice { ... }`
- map
    - キーに対する要素が存在するかどうか
        - `elem, ok = m[key]`
- Methods
    - メソッド定義
        - レシーバ
            - `func (r 型) メソッド名(引数) 戻り値の型 { ... }`
        - ポインタレシーバを持つメソッドは、レシーバが指す変数を変更できる
            - レシーバ自身を更新することが多いため、変数レシーバよりもポインタレシーバの方が一般的
- interface
    - 他の言語にあるimplements、extendsは書かない
    - メソッドを実装していくことでそのinterfaceを実装したことになる
- Type assertions
    - `t := i.(T)`
    - `t, ok := i.(T)`
- Goroutines
    - `go hogeFunc` でhogeFuncを非同期処理にできる
    - channelでgoroutineの同期を取る


最後の方は明らかに疲れてきているのでメモが少なめになっている。  
先程書籍([プログラミング言語Go](https://www.amazon.co.jp/dp/4621300253))を買ってきたので、復習が必要なところは復習しておく。
goroutineとかchannelとかポインタレシーバとか。
