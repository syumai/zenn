---
title: "インタフェースの仕様の変更"
---

本章では、インタフェースの仕様の変更について説明を行います。

従来のインタフェースは、メソッドセット (method set) を表すものでした。
ある型がインタフェースを実装すると言った時、その型はインタフェースに宣言されている全てのメソッドを持っている必要がある、と定義されていました。

例えば、下記の例では、型 `File` は `Closer` インタフェースを実装しています。

```go
type Closer interface {
  Close()
}

type File struct{}

func (f File) Close() {}
func (f File) Open() {}
```

型 `File` は `Closer` に含まれていない `Open()` メソッドも持っていますが、 `Closer` インタフェースが持つ全てのメソッドを持っていることには変わりないので、問題なく `Closer` インタフェースを実装していると言えます。

Go 1.18 において、インタフェースはこの従来の挙動を維持したまま定義が大きく変わりました。

## インタフェースの新しい定義

新しい定義は、次のような内容になっています。

> **インタフェース型は、型セット (type set) を定義する**。
> そして、ある型がインタフェースを実装すると言った時、**その型はインタフェースの型セットに含まれる**。

定義がこのように改められた理由は、概要で説明したとおり、型パラメータの型制約が、メソッドによる制約のみでは不十分だったためです。
インタフェースの定義がメソッドに依存していたのを改め、より広い概念へと拡張されています。

先ほどの `Closer` インタフェースの例で言うと、
*`Closer` の型セットには、 `Close()` メソッドを実装する全ての型が含まれる。*
と言うことが出来ます。

そして、
*`Close()` メソッドを実装する全ての型は、 `Closer` インタフェースを実装する。*
と言うことが出来ます。

より正確に理解をしたい方は、Nobishiiさんの [初めての型セット](https://speakerdeck.com/nobishino/introduction-to-type-sets) をご覧になることをおすすめします。

型制約を表現するために、インタフェースの構文に新たな要素が追加されたので、これについて説明します。

## インタフェース要素

追加されたのは *インタフェース要素* (interface element) で、これは従来のメソッドを示す *メソッド要素* (method element) と *型要素* (type element) の 2 種類に分けられます。
インタフェースは、インタフェース要素を並べたものとなります。

例えば、次のような `StringableFloat` は、インタフェース要素として、型要素の `~float32 | ~float64` と メソッド要素の `String() string` を持ちます。
`StringableFloat` インタフェースは、基底型が `float32` または `float64` で、 `String() string` メソッドを持っている型を受け付け、例に示している `StringableFloat32` は、このインタフェースを実装しています。

```go
type StringableFloat interface {
  ~float32 | ~float64 // 型要素
  String() string     // メソッド要素
}

type StringableFloat32 float32

func (f StringableFloat32) String() string {
  return fmt.Sprintf("%f", f)
}
```

続いてインタフェース要素の説明を行いますが、メソッド要素については従来のインタフェースと基本的に変わらないため省略し、型要素についての説明を行います。

## 型要素

型要素は、 任意の数の、型または基底型 (underlying type) を `|` 記号で結合したものです。
型要素は union[^1] ともいい、含んでいる型または基底型が 1 つだけであってもこのように呼ばれることがあります。
型要素には、名前付きの型 (例: int) 、型リテラル (例: []int) のいずれも含むことができます。

### 型要素の例

型要素を使ったインタフェース `IntString` の例を示します。

```go
type IntString interface {
  int | string
}
```

ここで `|` は、その記号の両側で示されている型セットの和集合を表しています。
`int` と `string` の型セットにはそれぞれ `int` 型と `string` 型しか含まれていないため、
`int | string` と言う型要素は、`int` 型と `string` 型のみからなる型セットを示します。
`IntString` インタフェースは、インタフェース要素を一つしか持っていないため、`int` 型と `string` 型の値しか受け付けません。

例えば、下記のように `int` に対して定義した `MyInt` 型を受け付けることは出来ません。

```go
type MyInt int

func PrintIntString[T IntString](v T) {
  fmt.Println(v)
}

func main() {
  i1 := 0
  PrintIntString(i1) // OK

  i2 := MyInt(0)
  PrintIntString(i2) // NG
}
```

### 基底型

`MyInt` 型を受け付けられるようにするには、基底型を型要素に含める必要があります。
基底型は、`~型名` の形式で表され、その型セットには、`~` の付けられた型に対して定義された全ての型が含まれます。
例えば、 `~int` は、 `int` に対して定義された全ての型を含む型セットを示します。

```go
type Int interface {
  ~int
}

type MyInt1 int    // 基底型が `int` なので、 `Int` インタフェースを実装している
type MyInt2 MyInt1 // 基底型が `int` なので、 `Int` インタフェースを実装している
```

これを使って、 `IntString` インタフェースを次のように修正します。

```go
type IntString interface {
  ~int | ~string
}
```

`~int` と `~string` の型セットにはそれぞれ `int` 型および `string` 型に対して定義された全ての型が含まれているため、
`~int | ~string` と言う型要素は、それらを全て含む型セットを示します。

これによって、 `IntString` インタフェースは次のような `MyInt` と `MyString` を受け付けることが出来るようになりました。

```go
type MyInt int
type MyString string

func PrintIntString[T IntString](v T) {
  fmt.Println(v)
}

func main() {
  i := MyInt(1)
  PrintIntString(i) // OK

  s := MyString("abcde")
  PrintIntString(s) // OK
}
```

### 別のインタフェースを含む型要素

型要素が含む型は、別のインタフェース型であっても構いません。
例えば、先ほどの `IntString` インタフェースを次のように分解して宣言することも出来ます。

```go
type Int interface { ~int }
type String interface { ~string }
type IntString interface {
  Int | String // ~int | ~string と同じ
}
```

型要素として並べられるインタフェースは、その型セットの一部、または全部が重複していても問題がありません。
例えば、下記の `IntString` と `Float32String` は、いずれも型セットに `~string` を含んでいますが問題がなく、結果の型セットはこれらの和集合となります。(型要素の形式で示すと、 `~int | ~float32 | ~string` と同義になります)

```go
type IntString interface { ~int | ~string }
type Float32String interface { ~float32 | ~string }
type IntFloat32String interface {
  IntString | Float32String // ~int | ~float32 | ~string と同じ
}
```

ここで示したように、別のインタフェース型をインタフェースの定義に含むことをインタフェースの *埋め込み* (embed) と言います。
従来のインタフェースの仕様に存在した埋め込みは、型要素に単一のインタフェース型のみを含むもの、として表現出来ます。

```go
type Opener interface { Open() }
type Closer interface { Close() }
type OpenCloser interface {
  Opener // 型要素に Opener インタフェースのみを含む
  Closer // 型要素に Closer インタフェースのみを含む
}
```

### インタフェースの型セット

## 型要素の制限

## インタフェースの制限

[^1]: 適切な訳語が思い当たらず、カタカナ表記するのも微妙な気がしたのでそのままにしています。ここで `union` の対象となるのは `type term` なので、 `union type term` の訳語を割り当てるのが適切かも知れません。

