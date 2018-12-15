---
title: "[豆知識]Bounds Check Elimination"
description: "BCEについて調べました"
date: 2018-12-15T19:38:27+09:00
categories:
  - 勉強の記録
tags:
  - Go
draft: false
---

## はじめに

この記事は[Gopher道場 Advent Calendar](https://qiita.com/advent-calendar/2018/gopherdojo)15日目の記事です。  

配列やスライスに対して添字アクセスをしようとすると、通常はその添字でアクセスが可能なのか範囲のチェックが必要です。
しかし、ある状況下ではチェックを省略することができます。これをBounds Check Elimination(BCE)といいます。  
[先日のGo Conferenceで触れられており](https://speakerdeck.com/orisano/profiling-go-application)初めて知ったので調べてみました。  

以下のようなケースを考えてみましょう。f1,f2ともスライスの0~4番目の要素の合計を求めるものです。

```
package main

func main() {
	s := []int{0, 1, 2, 3, 4}

	f1(s)
	f2(s)
}

func f1(nums []int) int {
	sum := nums[0]
	sum += nums[1]
	sum += nums[2]
	sum += nums[3]
	sum += nums[4]
	return sum
}

func f2(nums []int) int {
	sum := nums[4]
	sum += nums[3]
	sum += nums[2]
	sum += nums[1]
	sum += nums[0]
	return sum
}
```

`-gcflags="-d=ssa/check_bce/debug=1"` というオプションを付けてbuildを実行してみましょう。

```
$ go build -gcflags="-d=ssa/check_bce/debug=1" main.go
# command-line-arguments
./main.go:11:13: Found IsInBounds
./main.go:12:13: Found IsInBounds
./main.go:13:13: Found IsInBounds
./main.go:14:13: Found IsInBounds
./main.go:15:13: Found IsInBounds
./main.go:20:13: Found IsInBounds
```

`IsInBounds` (もしくは `IsSliceInBounds` )が表示された行ではBounds Checkが必要です。
逆に、`f2`では先にスライス[4]にアクセスしているため、0~3については存在することが確定するため、コンパイラが安全だと判断してチェックを省略します。
コンパイラがこのような判断をできるのは、Go1.7から導入されたSSA(Single Static Assignment)のおかげです。
SSAについては私がまだ深掘りできていないため割愛します。  

Bounds Checkが省略される分、2つのコードには性能に差が出ます。  
下記のようなコードでベンチを取ってみました。

```
package main

import "testing"

func BenchmarkF1(b *testing.B) {
	nums := []int{}
	for i := 0; i < 5; i++ {
		nums = append(nums, i)
	}
	for n := 0; n < b.N; n++ {
		_ = f1(nums)
	}
}

func BenchmarkF2(b *testing.B) {
	nums := []int{}
	for i := 0; i < 5; i++ {
		nums = append(nums, i)
	}
	for n := 0; n < b.N; n++ {
		_ = f2(nums)
	}
}
```

結果は以下のようになりました。

```
BenchmarkF1-4   	2000000000	         0.99 ns/op
BenchmarkF2-4   	2000000000	         0.33 ns/op
```

## Bounds Check Eliminationが適用される他のケース

ifとlenで範囲を確認してからアクセスする際にもBCEが適用されます。コンパイラはかなり賢いようです。

```
func f3(s []int, index int) {
	if index >= 0 && index < len(s) {
		_ = s[index]        // Bounds Check Elimination!!
		_ = s[index:len(s)] // Bounds Check Elimination!!
	}
}

func f4(s []int) {
	if len(s) > 2 {
	    _, _, _ = s[0], s[1], s[2] // Bounds Check Elimination!!
	}
}
```

forで配列のループを回している際はその配列に関してBCEが適用されます。

```
func f5(s []int) {
	for i := range s {
		_ = s[i]        // Bounds Check Elimination!!
		_ = s[i:len(s)] // Bounds Check Elimination!!
		_ = s[:i+1]     // Bounds Check Elimination!!
	}
}

func f6(s []int) {
	for i := 0; i < len(s); i++ {
		_ = s[i]        // Bounds Check Elimination!!
		_ = s[i:len(s)] // Bounds Check Elimination!!
		_ = s[:i+1]     // Bounds Check Elimination!!
	}
}

```

ただし、次のように別の配列やスライスにアクセスしようとした際にはBounds Checkが行われます。

```
func f7a(s []int) []int {
	var hoge int
	s2 := make([]int, len(s))
	for i := range s {
		hoge = -s[i]
		s2[i] = hoge // Bounds Checkが行われる
	}
	return s2
}
```

`make([]int, len(s))` なのでBCEが適用されてもおかしくはないところですが、まだコンパイラがそこまで解釈できないようです。  

ちなみにこのようにループに入る前にs2にアクセスしておくとループ内でのBounds Checkを省略できる、というちょっとしたテクニックがあります。

```
func f7b(s []int) []int {
	var hoge int
	s2 := make([]int, len(s))
	s2 = s2[:len(s)] // Bounds Checkが行われる
	for i := range s {
		hoge = -s[i]
		s2[i] = hoge // Bounds Check Elimination!!
	}
	return s2
}
```

ただし、この例ではループ内でBCEが適用されているとはいえ、ベンチの性能の改善が見られませんでした。  
それどころかスライスの要素が大きくなるほど `s2=s2[:len(s)]` のコストが高くなるのか(?)、却ってベンチ結果が悪くなるという結果になりました。

```
package main

import "testing"

func BenchmarkF7A(b *testing.B) {
	nums := []int{}
	for i := 0; i < 50000; i++ {
		nums = append(nums, i)
	}
	for n := 0; n < b.N; n++ {
		_ = f7a(nums)
	}
}

func BenchmarkF7B(b *testing.B) {
	nums := []int{}
	for i := 0; i < 50000; i++ {
		nums = append(nums, i)
	}
	for n := 0; n < b.N; n++ {
		_ = f7b(nums)
	}
}
```

```
BenchmarkF7A-4   	   20000	     79592 ns/op
BenchmarkF7B-4   	   20000	     85497 ns/op
```

## 所感

forループを回したりifで事前に使える添字なのか調べておくことでBCEが適用されるので、安全なコードを書くように心がけていれば知らないうちにBCEが適用されているという感じになっています。なので、「普段からBCEを意識して効率の良いコードを書くぞ！」と構える必要はなさそうです。また、f7bのようなテクニックを使ったところでどれほど性能が改善されるかは分からないので、とりあえずテクニック使っておけばよいという話ではなく、ベンチをちゃんと取りましょう。

## ちなみに

参考にした記事には他にも例が載っているので興味を持たれた方は読んでみるとよいでしょう。  
[Bounds Check Elimination](https://go101.org/article/bounds-check-elimination.html)
