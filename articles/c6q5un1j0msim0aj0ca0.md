---
title: "Go 1.18 で interface{} の代わりに any が使えるようになる話"
emoji: "️🌠"
type: "tech"
topics: ["Go"]
published: true
---

こちらはGo Advent Calendar 12日目の記事です。

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

Go 1.18 以上が利用可能であることが明らかな場合に `any` を使うようにしましょう。(ライブラリ等では Go 1.18 のリリースからある程度期間を置いてから使った方が良さそうです)

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

これはあくまで一例ではありますが、 `interface{}` をコードから削除する際の判断材料として使ってみてください。

### `interface{}` を一括して `any` に置き換える

一括で置き換えてしまいたいと言うことであれば、[標準ライブラリ中の `interface{}` を `any` に置き換えるIssueで行われている](https://go-review.googlesource.com/c/go/+/368254/)ように、 `gofmt` で一括置換することが出来ます。

```
gofmt -w -r 'interface{} -> any' .
```

以上、 `any` をコード中で使う方法や、置き換えていく方法について説明しました。
もしよければ参考にしてみてください。
