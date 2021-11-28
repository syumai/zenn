---
title: "Go 1.18 で使えるようになるジェネリクス関連の文法まとめ"
emoji: "️🌄"
type: "tech"
topics: ["Go"]
published: false
---

Go 1.18 では、いわゆる Generics を実現するための、型パラメータの機能が追加される見込みです。

これに伴って行われる言語仕様や、標準ライブラリなどに対する変更を簡単にまとめました。

基本的には文法に絞った内容とし、型推論などの詳細には触れません。

---

# Overview

## 型パラメータ

まず、基本的な *型パラメータ* (type parameters) の利用方法について紹介します。

型パラメータは、型定義、および関数宣言に追加することが出来ます。
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

パラメータ化された型、および関数を使うには、具体的な型を *型引数* (type arguments) として渡す必要があります。
ここで渡された型引数は、型、および関数の型パラメータを置き換えます。この置き換え作業のことを *インスタンス化* (instantiation) と言います。

インスタンス化を行うと、型引数によって置き換えられた型、および関数が新たに宣言されると考えると理解しやすいです。
下記に、置き換えのイメージを示します。

```go
func main() {
  //
  var sInt S[int]{
    Field: 1,
  }
}


// 型定義A: 型 `S` の `X` を `int` 型で置き換える
type S[int] struct {
  Field int
}

// 型定義B: 型 `S` の `X` を `string` 型で置き換える
type S[string] struct {
  Field string
}
```

として


```go
func main() {
  // パラメタ化された型の使用

  // * `int` 型でインスタンス化している
  var sInt = S[int] {
    Field: 1, // int
  }

  // * `[]string` 型でインスタンス化している
  var sStrings = S[[]string] {
    Field: []string{ // []string
      "A",
      "B",
    },
  }
}
```

例えば、下記の例のように、`S` 構造体のフィールドの型を、 `int` や `[]string` 型などの任意の型として扱うことが出来ます。

## 型制約

先ほど紹介した、型パラメータの宣言に注目してみましょう。
`S` 構造体の型定義において、型パラメータ `X` は、 `[X any]` と宣言されていました。
ここに書かれている `any` の部分は、型パラメータ `X` に対する *型制約* (type constraints) です。

型制約は、型パラメータがどういった型を型引数として受け付けることが出来るかを示す制約で、これは *インタフェース* (interface) を使って表現されます。
例えば、次のような `Stringer` インタフェースは、 `String() string` メソッドを持つ型のみを受け付ける制約として使用できます。

```
type Stringer interface {
  String() string
}
```

この `Stringer` インタフェースを制約として持つ `PrintStringer` 関数について考えてみましょう。
この関数は、 `Stringer` インタフェースを満たす型のみを受け付け、 `String() string` メソッドを呼び出した結果をPrintします。

```
func PrintStringer[T Stringer] (v T) {
  return fmt.Println(v.String())
}

type StringableInt int

func (i *StringableInt) String() string {
  return strconv.Itoa(i, 10)
}

func main() {
  i1 := IntStringer(1)
  i2 := 10 // 型無し定数 `10` のデフォルト型の `int` 型
  s := "abcde" // 型無し定数 `"abcde"` のデフォルト型の `string` 型
  PrintStringer[StringableInt](i1) // OK
  PrintStringer[int](i2) // build error (`int` 型は `Stringer` インタフェースを満たしていない)
  PrintStringer[string](s) // build error (`string` 型は `Stringer` インタフェースを満たしていない)
}
```

上記の例のように、 `PrintStringer` 関数の宣言時点では具体的にどの型を受け付けるか決まっていませんが、型制約によって型パラメータ `T` 型は `String() string` メソッドを持つことが保証されるので、`T` 型の変数 `v` に対して `String()` メソッドを呼び出すことが出来ています。

最初の例で使用していた `[X any]` の `any` も、 `Stringer` 同様にインタフェースです。
`any` は、 `interface{}` に対するエイリアスで、このインタフェースはメソッドを1つも要求しないので、全ての型を受け付ける制約として使用することが出来ます。

## インタフェースの仕様の拡張

また、メソッドによって決定出来ない種類の制約を実現するために、インタフェースの仕様が拡張されました。
例えば、 `+` 演算子を使った処理が可能な型に対しての制約を行いたいケースを考えてみましょう。
この時、許可したい型は、文字列型、整数型、浮動小数点数型、複素数型となります。
もし、これらの型が特別な `Add()` メソッドを持っていたとすればインタフェースで表現可能でしたが、実際にはそのようなことは無いため、従来のインタフェースではこの要求を満たすことが出来ません。

インタフェースは、これまで Go で使われていたものと同一のもので、今回 Generics の機能を実現するために一部拡張が行われています。(詳細は後述します)



---

# 型パラメータリスト

型パラメータリストは、それを持つ型、または関数が、どういった型引数を受け付けることが出来るかを示す宣言です。
型パラメータリスト内で宣言された **型パラメータ** は、型定義、または関数宣言内で、あたかもその名前の型が存在するかのように扱うことが出来ます。
型定義、または関数宣言内でどのように型パラメータを使うことが出来るかは後述します。

型エイリアス宣言、メソッド宣言に対しては型パラメータリストを追加することが出来ません。(後述しますが、型エイリアス宣言に対しては、Go 1.19 以降のどこかで追加される見込みです)

型定義に対する型パラメータリスト追加の例

```go
type A[T any] struct {
}
```


---

# 型パラメータの制約

---

# インタフェース

---

# any, comparable

事前宣言された識別子として、新たに any と comparable が追加されます。
いずれも、型パラメータの制約として使うために追加された、 **インタフェース型の値** を示します。

## any

any は、 `interface{}` に対するエイリアスです。

`interface{}` は、全ての型を型セットに含むインタフェースなので、 `any` も同様に任意の型を受け付ける制約として使うことが出来ます。

```go
func PrintAny[T any](v T) {

}
```

```go
```

```go
var v any
v = 1        // OK
v = "string" // OK
```

---

# 型宣言

---

# 関数宣言

---

# メソッド宣言

---

# 型パラメータに対する操作

---

# 型引数

---

# まだ出来ないこと

## 型エイリアス

## 

---

# メソッド

---

# 参考文献


