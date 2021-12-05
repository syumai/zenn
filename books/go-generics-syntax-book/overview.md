---
title: "型パラメータの使用方法概説"
---

本章では、Goのジェネリクスを実現する型パラメータの使用方法の概要を説明します。
初めから各項目の詳細を知りたい場合は、読み飛ばしてしまって問題ありません。

## 型パラメータ

*型パラメータ* (type parameters) は、型定義、および関数宣言に追加することが出来ます。
型パラメータは、通常の型と同じように型定義、および関数宣言に利用することができ、具体的な型を後から受け取るようにすることが出来ます。

例えば、下記の例のように、
`X` と言う型が存在するものとして、構造体型に対するフィールドの宣言を行ったり、
`Y` と言う型が存在するものとして、関数宣言におけるパラメータ及び戻り値の宣言を行うことが出来ます。

```go
// 型定義への型パラメータ `X` の追加
type S[X any] struct {
  Field X
}

// 関数宣言への型パラメータ `Y` の追加
func F[Y any](v Y) Y {
  return v
}
```

上記のような型パラメータを持つ型、および関数の事を、 *パラメータ化* (parametarized) された型、および関数と呼びます。

型パラメータは *型パラメータリスト* (type parameters list) として宣言され、型定義、および関数宣言は共に複数の型パラメータを持つことが出来ます。

```go
type SMulti[A any, B any] struct {
  FieldA A
  FieldB B
}

// 型パラメータの制約 any (後述します) が共通しているので、 [A, B any] のように宣言をまとめることも可能
func FMulti[A, B any](a A, b B) (A, B) {
  return a, b
}
```

**2021年11月現在の仕様では、メソッド、型エイリアスは型パラメータリストを持てない**、と言う点に注意する必要があります。
下記に、簡単な例を示します。

### メソッドの例

```go
type Slice[T any] []T

// NG: 型パラメータ `U` を持つメソッド `Convert` は宣言できない
func (s Slice[T]) Convert[U any](conv func(v T) U) Slice[U] {
  ...
}
```

### 型エイリアスの例

```go
type Map[T comparable, U any] map[T]U

// NG: 型パラメータ `U` を持つ型エイリアス `IntMap` は宣言できない
type IntMap[U any] = Map[int, U]

// OK: 型エイリアスが型パラメータを持っていないので宣言できる
type IntStringMap = Map[int, string]
```

これらの詳細については別途解説を行います。

## インスタンス化

パラメータ化された型、および関数を使うには、具体的な型を *型引数* (type arguments) として渡す必要があります。
ここで渡された型引数は、型、および関数の型パラメータを置き換えます。この置き換え作業のことを *インスタンス化* (instantiation) と言います。

インスタンス化を行うと、型引数によって置き換えられた型、および関数が新たに宣言されると考えると理解しやすいです。
下記に、インスタンス化のイメージを示します。

```go
func main() {
  // 1: 型 `S` の型パラメータ `X` に対して `int` 型を渡しインスタンス化する
  var sInt S[int]{
    Field: 123,
  }

  // 2: 型 `S` の型パラメータ `X` に対して `string` 型を渡しインスタンス化する
  var sString S[string]{
    Field: "abc",
  }

  // 3: 関数 `F` の型パラメータ `Y` に対して `float64` 型を渡しインスタンス化する
  fmt.Println(F[float64](1.23))
}

// 型定義のインスタンス化のイメージ

// 1: 型 `S` の型パラメータ `X` を `int` 型で置き換えた型の定義
type S[int] struct {
  Field int
}

// 2: 型 `S` の型パラメータ `X` を `string` 型で置き換えた型の定義
type S[string] struct {
  Field string
}

// 関数宣言のインスタンス化のイメージ

// 3: 関数 `F` の型パラメータ `Y` を `float64` 型で置き換えた関数の定義
func F[float64](v float64) float64 {
  return v
}
```

このとき、型引数として渡される型は何でもよい訳ではなく、型パラメータリストの宣言に示されている型制約を満たす必要があります。

## 型制約

冒頭で紹介した `S` 構造体の型定義において、型パラメータ `X` は、 `X any` と宣言されていました。

```go
type S[X any] struct {
  Field X
}
```

この宣言中の `any` の部分を、型パラメータ `X` に対する *型制約* (type constraints) と呼びます。

型制約は、型パラメータがどういった型を型引数として受け付けることが出来るかを示す制約で、 *インタフェース* (interface) を使って表現されます。
例えば、次のような `Stringer` インタフェースは、 `String() string` メソッドを持つ型のみを型引数として受け付ける制約として使用できます。

```go
type Stringer interface {
  String() string
}
```

この `Stringer` インタフェースを制約とした型パラメータを持つ `PrintStringer` 関数について考えてみましょう。
この関数は、 `Stringer` インタフェースを満たす型のみを型引数として受け付け、受け取った引数 `v` の `String() string` メソッドを呼び出した結果をPrintします。

```go
// `Stringer` インタフェースを制約とした型パラメータ `T` を持つ関数
func PrintStringer[T Stringer] (v T) {
  return fmt.Println(v.String())
}

// `String() string` メソッドを実装する整数型 `StringableInt` を定義する
type StringableInt int

func (i *StringableInt) String() string {
  return strconv.Itoa(i, 10)
}

func main() {
  // PrintStringer に型引数を渡して実行する
  var (
    i1  StringableInt = 1
    i2  int           = 10
    s   string        = "abcde"
  )
  // OK
  PrintStringer[StringableInt](i1)
  // NG: `int` 型は `Stringer` インタフェースを満たしていない
  PrintStringer[int](i2)
  // NG: `string` 型は `Stringer` インタフェースを満たしていない
  PrintStringer[string](s)          
}
```

上記の例のように、 `PrintStringer` 関数の宣言時点では具体的にどの型を受け付けるか決まっていませんが、型制約によって型パラメータ `T` に渡される型が `String() string` メソッドを持つことが保証されるので、引数 `v` に対して `String()` メソッドを呼び出すことが出来ています。

冒頭の例で使用していた `X any` の `any` も、 `Stringer` 同様にインタフェースです。
`any` は、 `interface{}` に対するエイリアスで、このインタフェースはメソッドを1つも要求しないので、全ての型を受け付ける制約として使用することが出来ます。

## インタフェースの仕様の拡張

ここまでは従来のインタフェースの仕様に基づいて型制約の解説を行いましたが、実は、メソッドによって決定出来ない種類の制約を実現するために、インタフェースの仕様が拡張されています。

例えば、 `+` 演算子を使った処理が可能な型に対しての制約を宣言したいケースを考えてみましょう。
この時、許可したい型は、文字列型、数値型 (整数型、浮動小数点数型、複素数型) となります。
仮に、これらの型が共通した `Add()` メソッドを持っていたとすればインタフェースで表現可能でしたが、実際にはそうではないため、従来のインタフェースではこの要求を満たすことが出来ません。

この問題に対する解決策として、新たにインタフェースは、 *メソッド要素* (method element) だけではなく、 *型要素* (type element) も含むことが出来るようになりました。

型要素を含んだインタフェースを使って、先ほどの

> `+` 演算子を使った処理が可能な型 

を表現すると、次の `Addable` interface (Type Parameters Proposalより引用) のようになります。
[*1より引用: https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#type-sets-of-embedded-constraints]

```go
type Addable interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
		~float32 | ~float64 | ~complex64 | ~complex128 |
		~string
}
```

型要素を使ったインタフェースの宣言方法については、別途解説を行います。

## 型推論

最後に、型推論について簡単に紹介を行います。
先ほど紹介したインスタンス化の例に、下記のようなコードがありました。

```go
func main() {
  // 1
  var sInt S[int]{
    Field: 123,
  }

  // 2
  var sString S[string]{
    Field: "abc",
  }

  // 3
  fmt.Println(F[float64](1.23))
}
```

実は、3の例については型引数を推論させることが可能です。
型推論を利用すると、下記のようになります。

```go
func main() {
  ...
  fmt.Println(F(1.23)) // 型なし定数 `1.23` のデフォルト型 `float64` を型引数として推論する
}
```

`3` の例で使用した、関数の引数から型引数を推論する形式の型推論を、 *関数引数型推論* (function type inference) と呼びます。一般的に使われる型推論はこちらの形式となります。`1`, `2` の例については、型推論を行うことは出来ません。

関数引数型推論とは別に、型制約の内容から型引数を推論する *制約型推論* (constraint type inference) と言う形式も存在しますが、本書では解説を行いません。

---

次の章からは、個々のトピックについてより詳細に解説を行います。