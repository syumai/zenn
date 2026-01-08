---
title: Go 1.24でジェネリックになった型エイリアスの紹介
emoji: 🔍
type: tech
topics: ["go"]
published: true
---

従来、型エイリアスは型パラメータを持つことができませんでした。
Go 1.24のリリースによって、型エイリアスに型パラメータを持てるようになり、従来は不可能だった表現が可能となっています。
特に、型の互換性を維持したリファクタという観点で、本機能は非常に重要なものとなっています。
今回は、本機能の仕様の解説に始まり、型エイリアスの機能のモチベーションそのものから主要なユースケースについて説明します。

# 言語仕様の変更点

## 従来の仕様

従来、型パラメータを持つことができたのは、**型定義** (type definition) と **関数宣言** (function declaration) のみでした。

型定義によって導入される、型パラメータを持つ型のことを、 **ジェネリック型** (generic type) と呼びます。

型パラメータは、以下のように、 **型制約** (type constraints) を伴って宣言されます。

```go
// 型定義の例
type Map[K comparable, V any] map[K]V

// 関数宣言の例
func Min[T cmp.Ordered](a, b T) T {
  if a < b {
    return a
  }
  return b
}
```

## Go 1.24で導入された仕様

Go 1.24からは、これに加えて **エイリアス宣言** (alias declaration) も型パラメータを持つことができるようになりました。

以下のような、型パラメータを持つ型エイリアスのことを、 **ジェネリックエイリアス** (generic alias) と呼びます。

```go
// エイリアス宣言の例
type Map[K comparable, V any] = map[K]V
```

ジェネリックエイリアスの型パラメータは、複合型 (composite type) [^1] の一部や、別のジェネリックな型への型引数として使えます。

[^1]: array, struct, pointer, function, interface, slice, map, and channel https://go.dev/ref/spec#Types

```go
// 複合型の一部
type IntMap[K comparable] = map[K]int

// 別のジェネリックな型への型引数
// iter packageの `type Seq2[K, V any] func(yield func(K, V) bool)` に対するエイリアス
type IndexedSeq[V any] = iter.Seq2[int, V]
```

ジェネリックエイリアスは、使用する際に必ずインスタンス化 (instantiation) される必要があります。

```go
// NG
var m1 Map // 型引数を省略することはできない

// OK
var m2 Map[string, int]
```

また、ジェネリックエイリアスの型パラメータそのものに対するエイリアス宣言を行うことはできません。

```go
type A[P any] = P // illegal: P is a type parameter
```

# ジェネリック型と、ジェネリックエイリアスの違い

ジェネリック型と、ジェネリックエイリアスの最も大きな違いは、 **型同一性** (type identity) にあります。
型定義によって導入されたジェネリック型は、使用する際に必ずなんらかの **名前付き型** (named type) を得ます。

```go
type IntMap[K comparable] map[K]int

m1 := map[string]int{"a": 1} // m1はmap[string]int型
m2 := IntMap[string]{"b": 2} // m2はIntMap[string]型
```

一方で、ジェネリックエイリアスは、使用する際に新たな名前付き型を導入しません。(別のジェネリック型に対するエイリアスだった場合は、エイリアスの対象の新たな名前付き型が導入されます。)

```go
type IntMap[K comparable] = map[K]int

m1 := map[string]int{"a": 1} // m1はmap[string]int型
m2 := IntMap[string]{"b": 2} // m2もmap[string]int型
```

Goの言語仕様に書かれている [^2] 通り、ある名前付き型は常に他の名前付き型とは異なる型となります。

> A named type is always different from any other type.

異なる型同士では、代入や演算処理などの操作が制限されることがあります。

```go
// int型に対して名前付き型のMyIntを定義
type MyInt int

i := 1          // int型の変数iを宣言
var myInt MyInt // MyInt型の変数iを宣言
myInt = i       // int型の値はMyInt型の変数には代入できない
```

# ユースケース

型エイリアスの最も主要なユースケースは、プログラムの **リファクタリング** です。
特に、ちょっとした関数の分割にとどまらない、パッケージの移動などの大規模なリファクタリングを指します。

2024年9月17日にGo Blogに書かれた[型エイリアスについての記事](https://go.dev/blog/alias-names)でも、この点が強調されていました。なお、同記事において、型エイリアスの導入から、Go 1.24でジェネリックエイリアスが導入されるまでの経緯の詳細について解説が行われていたので、以下の内容はこちらの記事に基づきます。

## 型エイリアス導入の背景

あるパッケージ `p` の分割について考えましょう。
パッケージ `p` には、以下の関数 `F`、定数 `C`、型 `T` が存在します。

```go
package p

func F() T { return 1 }
const C = 1
type T int
```

このパッケージ `p` を使うコードは例えば以下のようになっているでしょう。

```go
package main

import "github.com/syumai/example/p"

func main() {
  if p.F() == p.C {
    println("p.F() == p.C")
  }
  var v p.T = 1
  if p.F() == v {
    println("p.F() == v")
  }
  ...
}
```

ここで、パッケージ分割を行い、関数 `F` を `p1` に、 定数 `C` を `p2` に、型 `T` を `p3` に移動したとします。

```go
package p1

import "github.com/syumai/example/p3"

func F() p3.T { return 1 }
```

```go
package p2

const C = 1
```

```go
package p3

type T int
```

このとき、後方互換性を維持し、パッケージ `p` を使用しているコードが引き続き利用可能な状態に保つにはどうするとよいでしょうか？

関数と定数については簡単です。
関数は、移管先のパッケージの関数を呼ぶ形でラップすればOKです。
定数は、移管先のパッケージの定数を単に参照すればOKです。

```go
// OK
func F() p3.T { return p1.F() }
// OK
const C = p2.C
```

型についても、パッケージ `p` 側に `type T p3.T` として改めて定義すればよいように見えますが、実際には `p.T` と `p3.T` は異なる型となってしまいます。
これは、`type T p3.T` の型定義により新たな名前付き型が導入されており、「ある名前付き型は常に他の名前付き型とは異なる」ためです。

```go
// p.Tとp3.Tは別の型
type T p3.T
```

すると、パッケージ `p` を利用するコードで `p3.T` を期待するコードのビルドに失敗してしまいます。

```go
package main

import "github.com/syumai/example/p"

func main() {
  ...
  var v p.T = 1
  if p.F() == v { // build error (Fの返すp3.Tとp.Tは異なる型)
    println("p.F() == v")
  }
  ...
}
```

Go 1.9で導入された型エイリアスを使うと、新たなパッケージ `p` で新たな型が導入されず、 `p.T` と `p3.T` を同じ型として扱うことができ、後方互換性が保たれます。

```go
// p.Tという名前付き型は導入されない
type T = p3.T
```

## ジェネリック型のリファクタリング

実は、この単純な型エイリアスのユースケースがジェネリック型については利用できませんでした。

問題は、**型エイリアスが型パラメータを持てない** という点です。

先ほどの例で、パッケージ `p` の型 `T` がジェネリック型だったとします。

```go
type T[P any] struct { Field P }
```

型Tをパッケージ `p3` に移動したとき、パッケージ `p` 側で `p3.T` を参照する際に問題が発生します。

```go
// NG
type T = p3.T
```

まず、ジェネリック型は、使用する際に型引数を指定して、インスタンス化する必要があります。
上記のようなエイリアス宣言では、 `p3.T` の型パラメータ `P` に渡す型引数が定まりません。

また、エイリアス宣言で型パラメータ宣言を省略したときに、自動的にエイリアス指定先の型の型パラメータ全てを引き継ぐといった仕様もありません。

```go
type T = p3.T[/* ここに何の型が渡るか不定 */]
```

Go 1.24で型エイリアスに型パラメータを持てるようになったことで、初めてジェネリック型のパッケージを跨いだ移動が後方互換性を保った形で可能となったのです。

```go
type T[P any] = p3.T[P]
```

## その他のユースケース

### 型パラメータ追加時の後方互換性維持

ジェネリックエイリアスは、ジェネリック型に型パラメータが増えた際の後方互換性維持に使うことができます。

例えば、以下のように `string` 型の値をキーとして任意の `V` 型の値を保持する `Cache` 型について考えます。

```go
type Cache[V any] struct{
  m sync.Map
}

func (c *Cache[V]) Put(key string, v V) { /* */ }
func (c *Cache[V]) Get(key string) V { /* */ }
```

このCache型は、以下のようにインスタンス化して使います。

```go
type User struct {}
var userCache Cache[User]
```

ここで、この `Cache` 型のキーを任意の型に差し替え可能にしようとすると、後方互換性を崩してしまいます。

```go
type Cache[K comparable, V any] struct{/* */}

var userCache Cache[User] // 型引数の数が合わない
```

今回の場合は、別の名前の型 `CacheKV` に実装を移し、 `Cache` はキー型を `string` に固定した型エイリアスとするのがよいでしょう。　

```go
type CacheKV[K comparable, V any] struct{
  m sync.Map
}

func (c *Cache[_, V]) Put(key K, v V) { /* */ }
func (c *Cache[K, V]) Get(key K) V { /* */ }

// 後方互換性を保ったまま、新しい型に移行できている
type Cache[V any] = CacheKV[string, V]
```

### 型パラメータを事前に指定したジェネリック型の公開

あるジェネリック型と同じ型を使いつつ、型パラメータを指定しないでよい形で型を公開できます。
上記の後方互換性維持以外にパッと思いつかなかったですが、ライブラリだとユースケースがあるかもしれないです。

```go
type (
  intMap[K comparable] map[K]int
  StringIntMap = intMap[string]
  IntIntMap    = intMap[int]
)
```

### 複合型の型名の省略

関数型のような名前の長い名前の型が繰り返し現れるときに、その名前を省略したいが、新たな名前付き型の導入をしたくはない場合に使うことができます。

```go
type Proxy[In, Out any] = func(ctx context.Context, in In) (Out, error)

// generic aliasあり
func registerProxy1[In, Out any](p Proxy[In, Out]) {}

// generic aliasなし
func registerProxy2[In, Out any](p func(ctx context.Context, in In) (Out, error)) {}
```

## 導入に時間がかかった理由

Go 1.18でのジェネリクス導入時に合わせてリリースするべきかの議論が行われていましたが、十分にジェネリクスが使われるようになり、その知見が溜まってから実装した方がよいというGo teamの判断によって初期スコープから外されていました。
https://github.com/golang/go/issues/46477#issuecomment-852701491

Go 1.18のリリースからほぼ3年が経過し、十分に使われるようになったため、このタイミングでの実装になったのではないでしょうか。

# 参考文献

* Go言語仕様 https://go.dev/ref/spec
