---
title: "Go 1.18 で interface{} の代わりに any が使えるようになる話"
emoji: "️🌠"
type: "tech"
topics: ["Go"]
published: true
---

こちらは[Go Advent Calendar](https://qiita.com/advent-calendar/2021/go) 12日目の記事です。

Go 1.18 で interface{} の代わりに any が使えるようになるという話が出ていたので、これの使い方について書いてみようと思います。

any が入った詳細な経緯などについては[10日目のsg0hsmtさんの記事](https://qiita.com/sg0hsmt/items/06449d7ec8382b68d457)で解説いただいていましたので、こちらをご覧ください。 (内容がやや被っていてすみません)

## any とは何か

`any` は Go 1.18 で入る予定の、事前宣言された `interface{}` に対する型エイリアスです。
Goのプログラム全体のスコープ (ユニバースブロック) で、次のようなエイリアス宣言が行われているものと考えると理解しやすいと思います。

```go
type any = interface{}
```

Go 1.18 のドラフト版の言語仕様を読むと、事前宣言された識別子の中に `any` が追加されていることがわかります。

```
Types:
	any bool byte comparable
  ...
```

https://tip.golang.org/ref/spec#Predeclared_identifiers

## any の使い方

前述の通り、 `any` は `interface{}` の型エイリアスなので、 `interface{}` が登場しうる箇所で使用できます。

例えば、変数宣言、構造体のフィールド宣言、関数のパラメータの宣言などで型として `any` を指定できます。

```go
// 変数宣言
var v any
v = 12345
v = "abcde"

// 構造体のフィールド宣言
type S struct {
  field any
}

// 関数のパラメータの宣言
func F(v any) (result any) {
  return v
}
```

また、 Go 1.18 で追加される見込みの `型パラメータ` の `型制約` で同様に `any` を使うことが出来ます。

型制約はインタフェース型で表され、全ての型を許容する型制約 `interface{}` は頻繁に使われることになります。
`any` は、 この `interface{}` を型制約に繰り返し書かなくて済むよう、利便性のために追加されています。

```go
func F1[T any](v T) T { // `any` の箇所が型パラメータ `T` に対する型制約
  return v
}

func F2[T interface{}](v T) T { // `[T any]` と同じ意味
  return v
}
```

## 互換性について

`any` の追加は、後方互換性を保って行われています。
この変更によって、過去のコードのコンパイルが Go 1.18 で通らなくなるようなことはありません。

---

過去に `interface{}` を使って書かれたコードに対して、 `any` を使っていくことを考えます。

`any` は型エイリアスなので、`interface{}` を引数として受け付ける関数であったり、メソッドのシグニチャには影響を及ぼしません。

```go
func OldFunc(v interface{}) {
  ...
}

func main() {
  var v any
  v = 1
  OldFunc(v) // `interface{}` を受け付ける関数に `any` を渡すことは何も問題ない
}
```

また、例えば過去に `interface{}` を受け付けるパラメータを持つメソッドを宣言したインタフェースがあった場合、これを実装するのに `any` を使うことが出来ます。(逆も同様で、 `any` を受け付けるパラメータを持つメソッドを `interface{}` を受け付ける形で実装できます。)

```go
type OldInterface interface {
  M(v interface{})
}

type S struct{}

func (s S) M(v any) {} // `interface{}` ではなく `any` を使う

func main() {
  var v OldInterface = S{} // ok: S は OldInterface を実装している
  ...
}
```

上記の通り、 `any` は `interface{}` の代替として使えますし、混在していても問題がありません。

[sg0hsmtさんの記事](https://qiita.com/sg0hsmt/items/06449d7ec8382b68d457)で紹介されているように、Go 本体のソースコード内で使われている `interface{}` を全て `any` に書き換える Issue が出ています。
この変更は、あくまで Go 1.18 に対して行われるものですし、ここで説明した通り破壊的変更となる内容ではありません。
https://github.com/golang/go/issues/49884

### 過去のバージョンの Go について

当然ですが、 Go 1.17 以前には事前宣言された `any` は存在しないので、`type any = interface{}` とコード中に明示的に記述しない限り、コンパイルに失敗します。

Go 1.18 以上が利用可能であることが明らかな場合に `any` を使うようにしましょう。
ライブラリ等では Go 1.18 のリリースからある程度期間を置いてから使った方が良さそうです。

## `interface{}` を `any` に置き換える手順

`interface{}` を `any` に置き換えるにあたって、単純置換してしまっていいケースがほとんどだと思われます。

単純置換しない方法としては、`any` を型制約以外でも使えるようにする提案の Issue で、 Russ Cox が `interface{}` を少しずつ `any` に置き換えていく方法についてコメントしていたので紹介します。

https://github.com/golang/go/issues/33232#issuecomment-915333205

> 4. If we allow 'any' for ordinary non-generic usage too, then seeing interface{} in code could serve as a kind of signal that the code predates generics and has not yet been reconsidered in the post-generics world.
> Some code using interface{} should use generics. Other code should continue to use interfaces.
Rewriting it one way or another to remove the text 'interface{}' would give people a clear way to see what they'd updated and hadn't. (Of course, some code that might be better with generics must still use interface{} for backwards-compatibility reasons, but it can still be updated to confirm that the decision was considered and made.)

文脈としては、 Russ Cox が この "`any` を型制約以外で許容する提案" を進めてもよい、と判断した理由を述べている 4 項目中の 1 つとなります。

ざっくり説明すると、

* ジェネリクス導入後の世界で、 `interface{}` と書かれたコードが残っているのは、そのコードがジェネリクスを使うべきか判断されていない目印となる。
  - 過去に `interface{}` を使って書かれていたコードの中には、ジェネリクスに置き換えられるものと、そうでないものがある。
  - コードから `interface{}` を削除してジェネリクスまたは `any` を使う形に書き換えることは、そのコードに対してジェネリクスを使うべきか判断した目印となる。

といった内容となります。

では、どういった場合にジェネリクスを使った方がよい、もしくは `any` に単純置換した方がよいと言えるでしょうか？

#### ジェネリクスを使った方がよいケース

例えば、次のような関数があったとします。

```go
func First(s []interface{}) interface{} {
  return s[0]
}
```

この First 関数は、`interface{}` 型の要素を持つスライスを受け取って、その最初の要素を返します。

この関数は、ジェネリクスを使うようにした方がよいでしょうか？、それとも、 `interface{}` を `any` に単純置換した方がよいでしょうか？

このケースでは、**ジェネリクスを使った方がよい**です。

実際に使ってみるとわかりますが、実は、この First 関数は非常に使い勝手が悪く、例えば次のようなスライス `a` を直接受け取ることが出来ません。

```go
a := []int{1,2,3}
// `[]int{}` は `[]interface{}` に代入不可
first := First(a) // NG

---

// 値を `[]interface{}` に詰め替える
a := []int{1,2,3}
s := make([]interface{}, len(a))
for i, v := range a {
  s[i] = v
}
first := First(s) // OK
```

この `First` 関数が行いたかったのは、任意の要素型を持つスライスから、最初の要素を取り出すことでした。
こうしたケースでは、`interface{}` を `any` に単純置換するのではなく、ジェネリクスを使った方がよいでしょう。

```go
// ❌: 単純置換。元の関数と同じ挙動となる
func First(s []any) any {
  return s[0]
}

---

// ⭕: 任意の型のスライスを受け取って、
//      その要素型で値を返すことが出来る
func First[S ~[]Elem, Elem any](s S) Elem {
  return s[0]
}

func main() {
  a := []int{1,2,3}
  // ジェネリクス版 `First` の呼び出し
  // `a` を渡すことができ、int型の値が返る
  first := First(a)
  ...
}
```

#### `any` に単純置換した方がよいケース

次のような関数があったとします。

```go
var storage = map[string]interface{}{}

func Store(key string, value interface{}) {
  storage[key] = value
}

func Load(key string) interface{} {
  return storage[key]
}
```

この Store と Load は、任意の型の値 `value` を特定のキーに対して紐付けて保存を行います。

この関数は、ジェネリクスを使うようにした方がよいでしょうか？、それとも、 `interface{}` を `any` に単純置換した方がよいでしょうか？

このケースでは、**`any` に単純置換した方がよい**です。

`value` として渡ってくる値は、文字列かもしれないし、整数かも知れないし、何かしらの構造体かも知れません。
こうした、数え切れない種類の型の値を保存し、返す必要があるようなケースでは、型パラメータを使い関数をジェネリックにするメリットがありません。

例えば、 `Store` と `Load` をそれぞれジェネリックにしたとしても、保存先の map の要素型は `interface{}` なので、 Load の際に型アサーションが必要になってしまいます。
下記のコードの例のように、 `Store` で `int` 型の値を保存し、それを `Load` で `string` として取り出そうとすると、 型アサーションに失敗して panic します。
これでは、 `interface{}` 型の値をそのまま扱い型アサーションしているのと大差なく、ジェネリクスを使う必要性がありません。

```go
var storage = map[string]interface{}{}

func Store[T any](key string, value T) {
  storage[key] = value
}

func Load[T any](key string) T {
  v := storage[key]
  return v.(T) // `T` 型で型アサーションする
}

func main() {
  // int 型の値を保存
  Store("a", 1)
  // int 型の値を string 型で取得出来ず panic する
  v := Load[string]("a")
  ...
}
```

https://go.dev/play/p/p_f4U7cxS2P?v=gotip

最終的に、先ほどのコード中の `interface{}` を `any` に置き換えたコードを示します。

```go
var storage = map[string]any{}

func Store(key string, value any) {
  storage[key] = value
}

func Load(key string) any {
  return storage[key]
}
```

ここでは単純置換を行いましたが、もしかしたら、もっとよい設計の関数をジェネリクスを使って書くことが出来るかも知れないので、これらの関数を非推奨としてコメントを付けてもよいかも知れません。

これらはあくまで一例ではありますが、 `interface{}` をコードから削除する際の判断材料として使ってみてください。

### `interface{}` を一括して `any` に置き換える方法

一括で置き換えてしまいたいと言うことであれば、[標準ライブラリ中の `interface{}` を `any` に置き換えるIssueで行われている](https://go-review.googlesource.com/c/go/+/368254/)ように、 `gofmt` で一括置換することが出来ます。

```
gofmt -w -r 'interface{} -> any' .
```

以上、 `any` をコード中で使う方法や、置き換えていく方法について説明しました。
もしよければ参考にしてみてください。

---

## 過去のバージョンの Go についての補足 (2021/12/14 追記)

[Twitter で tenntenn さんがコメントされていました](https://twitter.com/tenntenn/status/1470228212475379712)が、 `!go118` ビルドタグを使ったファイルを各packageに配置し、そこで `type any = interface{}` を宣言する技も使えそうです。

ファイルのイメージ

```go
// +build !go118
//go:build !go118

package a // any を使いたい package の名前

// any が無い Go 1.18 以外のバージョンの Go の場合だけエイリアスを宣言する
type any = interface{}
```

ただし、このやり方では、 Go 1.19 以降など、 Go のバージョンが上がる度に記載するタグの内容を増やしつつ変更したりする必要もありやや面倒な感じもします。(初めから複数バージョン書いておいても良さそう？）

先述した通り、 `any はユニバースブロックで宣言されている` 扱いになっているので、実はパッケージブロックで宣言が重複していても問題になりません。
この方針で対応するのであれば、ビルドタグ無しで、単純に any の型エイリアス宣言をしているだけのファイルを配置することで対応完了、としてしまっても良いのかも知れません。

```go
package a // any を使いたい package の名前

// ユニバースブロックとパッケージブロックで宣言が重複しても、
// 狭いスコープ (パッケージブロック) が優先されるのでコンパイルエラーにならない
type any = interface{}
```