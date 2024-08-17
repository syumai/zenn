---
title: Go 1.23のイテレータについて知っておくべきこと
emoji: 🌌
type: "tech"
topics: ["Go"]
published: true
---

# はじめに

2024年8月13日、[Go 1.23がリリース](https://go.dev/blog/go1.23)され、ついにイテレータが利用可能となりました。
この記事では、Goのイテレータについて、

* どうやって使うのか
* どこまで知っておく必要があるのか

を理解することをゴールとします。

# 基本的な知識

基本的な知識としては、以下の内容を知っていれば問題ないです。

* for文のrangeループの仕様が変わった
  - 関数を対象にrangeループを回せるようになる
  - rangeループの対象にできる種類の関数を**イテレータ**と呼ぶ
* イテレータには3種類ある

## for文のrangeループの仕様が変わった

Go 1.22までは、for文によるrangeループの対象にできたのは、配列, slice, 文字列, map, channel, 整数だけでした。
Go 1.23で、ここに関数（ただし、特定の形式に限る）が加わりました。

ここで、rangeループの対象にできる形式の関数を**イテレータ関数**[^1] (以下、イテレータ) と呼びます。

[^1]: イテレータは単なる俗称ではなく、仕様中でも[iterator](https://go.dev/ref/spec#For_range)と記載されている箇所があります。

## イテレータには3種類ある

イテレータには値の返し方が異なる、3つの種類があります。

1. イテレーションごとに値を返さない
2. イテレーションごとに1つの値を返す (channel, 整数に対するrangeループと同じ)
3. イテレーションごとに2つの値を返す (slice, mapなどに対するrangeループと同じ)

それぞれ、以下の形式の関数となります。

```go
// 1
func(func() bool)
// 2
func(func(V) bool)
// 3
func(func(K, V) bool)
```

2, 3のケースで、 `V` と `K` には任意の型を使うことができ、rangeループではその型の値をそれぞれ受け取ります。

```go
// 1
var f1 func(func() bool)
for range f1 {} // 値が何も返らないので、 x := range f1 の形式では書けない
// 2
var f2 func(func(int) bool)
for x := range f2 {} // xはint型
// 3
var f3 func(func(string, int) bool)
for x, y := range f3 {} // xはstring型、yはint型
```

2, 3と同じ形式の関数の型が標準ライブラリの[iter package](https://pkg.go.dev/iter)に定義されているので、1のパターン以外は基本的にはこれらを使うことになります。

```go
package iter

// 2: func(func(V) bool)
type Seq[V any] func(yield func(V) bool)
// 3: func(func(K, V) bool)
type Seq2[K, V any] func(yield func(K, V) bool)
```

実際のrangeループの例として、文字列のsliceから `iter.Seq[string]` と `iter.Seq2[int, string]` を生成する `slices.Values` と `slices.All` の使用例を紹介します。

```go
s := []string{"a","b","c"}

seq := slices.Values(s) // iter.Seq[string] 型
for v := range seq {
	fmt.Println(v) // a, b, c が順番に表示される
}

seq2 := slices.All(s) // iter.Seq2[int, string] 型
for i, v := range seq2 {
	fmt.Println(i, v) // 0 a, 1 b, 2 c が順番に表示される
}
```

https://go.dev/play/p/rC21b4kARS7

# Goのイテレータとは何なのか

筆者の理解では、Goのイテレータは **「任意のデータ構造を隠蔽してデータ列として扱うための、一般的な形式」** です。

これについて少し掘り下げて説明します。

## データ構造の隠蔽

Goのイテレータはただの関数なので、どんなデータ構造も隠蔽できます。
すなわち、**任意のデータ構造を対象にrangeループが使えるようにできます**。
ただし、イテレータはインタフェースではないので、任意のデータ構造をそのままの型（構造体など）で使うことはできません。rangeループで使うためには、**対象のデータ構造を隠蔽した関数を生成する**必要があります。

値の集合や、値の列を表すデータ構造には色々ありますが、Go 1.23以降の世界では、それらを**イテレータに変換する関数やメソッドが各ライブラリから提供されるようになっていく**はずです。

例として、Russ Coxの作った[omap](https://github.com/rsc/omap) packageでは、Ordered Mapのデータ構造に`All() iter.Seq2[K, V]`[メソッドが実装](https://github.com/rsc/omap/blob/40dad5c0c0fb2f7b45318fa732252edfd450083b/omap.go#L250-L255)されており、簡単にrangeループが使えるようになっています。

## イテレータはsliceよりも一般的なデータ列の形式

Go 1.22までの世界では、sliceがデータ列を表現する最も柔軟なデータ形式でした。しかしながら、データ構造によっては、sliceの利用が効率的ではないケースや、そもそも表現できないケースが存在します。

sliceがイテレータと異なる点は、データ列の長さ分のメモリが最低でも必要な点と、データ列に終わりがある点です。例えば、循環リストに対してrangeループを回したい時、このデータ列には終わりがないので、sliceに変換することができません。また、全要素を集めてsliceにするとメモリを膨大に使ってしまうものの、元のデータ構造をそのまま使って一つずつ要素を取得していく分には効率的なケースもあるでしょう。

以上の通り、**イテレータはsliceよりも多くのケースに対応できる、より一般的なデータ列の形式**と言えます。そして、Go 1.23以降では、データ列の操作は基本的にイテレータを対象に行う形が想定されます。使用ライブラリなどとの兼ね合いで、一部、データ列をsliceとして扱う必要のある操作もあるでしょう。そういったケースでは、**slices packageを使って、sliceとイテレータを相互変換して使う**形になります。同様に、maps packageを使うことでmapとの相互変換も可能です。

一つ、注意しておくべき点として、イテレータにはイテレーションごとに値を返さない形式があります。この種類のイテレータをデータ列と呼ぶべきかどうかは怪しいです。（他にもっと適切な表現があるかも知れませんが）Goのイテレータは、全体としては「データ列の一般的な形式」と言うより、「繰り返し処理の継続と終了」を一般化したものとして捉えた方がよいでしょう。ただ、基本的な理解としては、イテレータのほとんどのユースケースに対応するであろう「データ列の一般的な形式」として捉えてしまってよいと筆者は考えています。

# イテレータの使い方

イテレータには、主に2つの使い方があります。

1. rangeループで使う
2. イテレータを受け取る関数に渡す

1 については前述の通りで、sliceやmapのrangeループと特に使い方が変わりません。
2 については、具体的な例をいくつか紹介します。

## データ列の変換

このケースでは、イテレータを受け取った関数が、データ列に含まれる各要素を別の要素に変換を行います。

簡単な例として、string型のデータ列を受け取って、全て大文字に変換して返すものを紹介します。(他の言語で言う `map` の操作をイメージするとわかりやすいです)

```go
func ToUpper(seq iter.Seq[string]) iter.Seq[string] {
	/* 実装は省略 */
}

func main() {
	s := []string{"a", "b", "c"}

	for _, v := range s {
		fmt.Println(v) // a, b, c
	}

	for v := range ToUpper(slices.Values(s)) {
		fmt.Println(v) // A, B, C
	}
}
```

https://go.dev/play/p/EiA7MJOyntv

上記の例において、ToUpper関数は`iter.Seq[string]`を同じ`iter.Seq[string]`に変換する形でラップしています。ToUpper関数によって得られるイテレータは、ラップしたイテレータの要素を、一気にまとめてではなく、一つずつ順番に取り出して処理するため効率的です。
また、string型の値を要素とするイテレータとして表現されていれば、どんなデータ構造に対しても適用可能となっています。

この例は、データ列を効率的に扱えると言う点で、`io.Reader`とそのラッパーに似ています。

```go
func ToUpperReader(r io.Reader) io.Reader {
	/* 実装は省略 */
}

func main() {
	s := "abc"

	sr := strings.NewReader(s)
	io.Copy(os.Stdout, sr) // abc

	ur := ToUpperReader(strings.NewReader(s))
	io.Copy(os.Stdout, ur) // ABC
}
```

https://go.dev/play/p/7ZUVAlMlYLh

しかしながら、イテレータが任意の型の値をデータ列に含む事が出来るのに対し、`io.Reader`はバイト列のストリームでしかない点で大きく異なります。(リンク先の`ToUpperReader`のサンプルコードでは、読み取ったバイト列に対して`bytes.ToUpper`を呼ぶ実装としていますが、本来はbyteではなくruneに対して操作を行うべきで、安全な文字列操作とは言えません)

## データ列の集約

このケースでは、イテレータを受け取った関数は、列挙された値を一つの値に集約します。(他の言語で言う `reduce` や `fold` の操作がイメージに近いです)

例えば、`slices.Collect`は`iter.Seq[V]`のイテレータを`V[]`のsliceに集約します。

```go
s1 := []string{"a","b","c"}
seq := slices.Values(s1) // iter.Seq[string]
s2 := slices.Collect(seq) // []string{"a","b","c"}
```

https://go.dev/play/p/yK6TkvJwlab

同様に、`maps.Collect`は`iter.Seq[K,V]`のイテレータを`map[K]V`のmapに集約します。

```go
m1 := map[string]int{
	"a": 1,
	"b": 2,
	"c": 3,
}
seq := maps.All(m1) // iter.Seq2[string, int]
m2 := maps.Collect(seq) // map[string, int]
```

https://go.dev/play/p/giDvNj5f53z

下記の例のように、`iter.Seq[int]`のデータ列の値を足し合わせる`SumInt`のような関数も書けます。

```go
func SumInt(seq iter.Seq[int]) int {
	var result int
	for i := range seq {
		result += i
	}
	return result
}

func main() {
	ints := []int{1, 2, 3}
	sum := SumInt(slices.Values(ints))
	fmt.Println(sum) // 6
}
```

https://go.dev/play/p/A1sPvW9ofoB

## その他の利用ケース

ここで紹介した以外にも、

* データ列の要素を条件式によって抽出する関数 (filter)
* データ列の連結

など、様々な利用ケースが考えられます。

[go-functional v2 betaのit package](https://pkg.go.dev/github.com/BooleanCat/go-functional/v2@v2.0.0-beta.6/it)に、Map, Fold, Filter, Chainなどの実装があったので、興味のある方はこちらをぜひご覧ください。

# slice / mapとイテレータの相互変換

Go 1.23のリリースに合わせて、slices / maps packageにイテレータを扱うための関数がいくつか追加されました。
[追加された関数の一覧がGo 1.23のリリースノートに記載されている](https://go.dev/doc/go1.23#iterators)ので、詳しく知りたい方はこちらをご覧ください。

追加された関数の中で、最もよく使うと考えられるのは、slice / mapとイテレータを相互変換して使う関数です。

## sliceとイテレータを相互変換する関数

sliceをイテレータに変換するのに使う関数は以下の通りです。

* [slices.All()](https://pkg.go.dev/slices#All)
  - `[]V`のsliceを、`iter.Seq2[int, V]`の形式で、index、値の組のイテレータに変換します。
* [slices.Values()](https://pkg.go.dev/slices#Values)
  - `[]V`のsliceを、`iter.Seq[V]`の形式で、値のみのイテレータに変換します。

イテレータをsliceに変換するのに使う関数は以下の通りです。

* [slices.Collect()](https://pkg.go.dev/slices@go1.23.0#Collect)
  - `iter.Seq[V]`のイテレータを、`[]V`のsliceに変換します。
* [slices.Sorted()](https://pkg.go.dev/slices@go1.23.0#Sorted)
  - `iter.Seq[V cmp.Ordered]`のイテレータを、ソートされた`[]V`のsliceに変換します。
  - ソート方法を指定できる[`slices.SortedFunc()`](https://pkg.go.dev/slices@go1.23.0#SortedFunc)も追加されました。

## mapとイテレータを相互変換する関数

mapをイテレータに変換するのに使う関数は以下の通りです。

* [maps.All()](https://pkg.go.dev/maps#All)
  - `map[K]V`のmapを、`iter.Seq2[K, V]`の形式で、キー、値の組のイテレータに変換します。
* [maps.Keys()](https://pkg.go.dev/maps#Keys)
  - `map[K]V`のmapを、`iter.Seq[K]`の形式で、キーのみのイテレータに変換します。
* [maps.Values()](https://pkg.go.dev/maps#Values)
  - `map[K]V`のmapを、`iter.Seq[V]`の形式で、値のみのイテレータに変換します。

イテレータをmapに変換するのに使う関数は以下の通りです。

* [maps.Collect()](https://pkg.go.dev/maps@go1.23.0#Collect)
  - `iter.Seq2[K, V]`のイテレータを、`map[K]V`のmapに変換します。

# どこまで知っておく必要があるのか

基本的には、本記事で紹介した、

* イテレータの種類
* イテレータの使い方
* slice / mapとイテレータの相互変換

について知っていれば問題ないと思います。

イテレータの実装方法について知るのももちろん有益ですが、標準ライブラリだけでもイテレータとslice / mapの相互変換はできますし、サードパーティーのライブラリについてはライブラリ作者がイテレータへの変換機構を用意してくれることを期待できるでしょう。
イテレータに関連する複雑な操作を行いたいようなケースでも、先ほど紹介した[go-functional](https://github.com/BooleanCat/go-functional)などでまかなえるものが多いと思います。

まだ、イテレータは登場したばかりですが、徐々にGoの周辺ライブラリがイテレータに対応していくと思われますので、ライブラリが揃うのに合わせて徐々に移行していくので十分でしょう。

自分でイテレータを実装してみたいという方は下記の資料をご覧いただくのがよいと思います。

* [tenntennさんのGo Conference 2024の登壇資料](https://tenn.in/gocon24)
* [tenntenn Conference 2024のハンズオンのアーカイブ、資料](https://www.youtube.com/live/03Lo84Ugnyk?si=mosvTWMgcB4yqINE&t=687)
* [Panaさんの記事](https://zenn.dev/kkkxxx/articles/d9505540581b5d)
* [iter packageのドキュメント](https://pkg.go.dev/iter)
* [range over func Proposal](https://go.dev/issue/61405)

# まとめ

Go 1.23で追加されたイテレータは、その実装方法を詳しく知らなくても、種類や、使い方についての基本的な知識さえ持っていれば十分に活用できます。
移行についてはエコシステムが充実してきてからで問題ないので急ぐ必要はありませんが、イテレータを使うためのライブラリの実装も出てきているので、興味のある方はぜひ試してみてください。

何か内容に問題があったり、質問などあれば、記事のコメントや、[記事のリポジトリのIssue](https://github.com/syumai/zenn/issues)、[X](https://x.com/__syumai)などでぜひご連絡ください。
