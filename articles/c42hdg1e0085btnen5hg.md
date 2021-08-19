---
title: "GoのGenerics関連プロポーザル最新状況まとめと簡単な解説 (2021年8月版)"
emoji: "☄️"
type: "tech"
topics: ["go", "プログラミング", "言語仕様"]
published: false
---

本記事では、GoのGenerics関連のProposal一覧のまとめと、簡単な解説を行います。

# Generics関連のProposal一覧

| Proposal                                              | Status                   | Author         | GitHub Issue                                        | Proposal Document / Gerrit                                                                               |
| :---------------------------------------------------- | :----------------------- | :------------- | :-------------------------------------------------- | :------------------------------------------------------------------------------------------------------- |
| type parameters                                       | **accepted (2021/2/11)** | ianlancetaylor | [#43651](https://github.com/golang/go/issues/43651) | [Proposal](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md)     |
| type sets                                             | **accepted (2021/7/22)** | ianlancetaylor | [#45346](https://github.com/golang/go/issues/45346) | [Gerrit](https://go-review.googlesource.com/c/proposal/+/306689)                                         |
| constraints package                                   | **accepted (2021/8/19)** | ianlancetaylor | [#45458](https://github.com/golang/go/issues/45458) |                                                                                                          |
| slices package                                        | **accepted (2021/8/12)** | ianlancetaylor | [#45955](https://github.com/golang/go/issues/45955) |                                                                                                          |
| maps package                                          | 議論中 (2021/8/17現在)   | rsc            | [#47649](https://github.com/golang/go/issues/47649) |                                                                                                          |
| sync, sync/atomic: add PoolOf, MapOf, ValueOf         | 議論中 (2021/8/17現在)   | ianlancetaylor | [#47657](https://github.com/golang/go/issues/47657) |                                                                                                          |
| go/ast changes for generics                           | 議論中 (2021/8/17現在)   | findleyr       | [#47781](https://github.com/golang/go/issues/47781) | [Proposal](https://go.googlesource.com/proposal/+/master/design/47781-parameterized-go-ast.md)           |
| go/types changes for generics                         | 議論中 (2021/8/17現在)   | findleyr       | -                                                   | [Gerrit](https://go-review.googlesource.com/c/proposal/+/328610)                                         |
| go/parser: add a mode flag to disallow the new syntax | 議論中 (2021/8/17現在)   | findleyr       | [#47783](https://github.com/golang/go/issues/47783) |                                                                                                          |
| disallow type parameters as RHS of type declarations  | 議論中 (2021/8/17現在)   | findleyr       | [#45639](https://github.com/golang/go/issues/45639) |                                                                                                          |
| Generic parameterization of array sizes               | 議論中 (2021/8/17現在)   | ajwerner       | [#44253](https://github.com/golang/go/issues/44253) | [Proposal](https://go.googlesource.com/proposal/+/refs/heads/master/design/44253-generic-array-sizes.md) |
| container/heap package                                | 議論中 (2021/8/17現在)   | cespare        | [#47632](https://github.com/golang/go/issues/47632) |                                                                                                          |

# 各Proposalの紹介

type parameters / type setsについては他に優れた資料があるので、ここでは内容に踏み込まず簡単な紹介に留めます。

## type parameters

Status: accepted
Issue: https://github.com/golang/go/issues/43651

Goでジェネリックなプログラミングを行えるようにするために、型や関数が *type parameter* を受け付けることを出来るようにする提案です。
型パラメータが受け付ける型に対しての *constraints* の導入や、型推論のルールについてもこの提案に含まれています。
表にあるProposalは全てこれをベースに提案が行われているものです。

Proposal内[^1]のコード例
[^1]: https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md

```go
// Stringer is a type constraint that requires the type argument to have
// a String method and permits the generic function to call String.
// The String method should return a string representation of the value.
type Stringer interface {
	String() string
}

// Stringify calls the String method on each element of s,
// and returns the results.
func Stringify[T Stringer](s []T) (ret []string) {
	for _, v := range s {
		ret = append(ret, v.String())
	}
	return ret
}
```

概要については [tenntennさんの資料](https://docs.google.com/presentation/d/10YX-P5wChDmBRXqUDDWNThtBXvg81EdS4vhUS_iRtdk/edit#slide=id.p) を参照いただくのがよいと思います。

## type sets

Status: accepted
Issue: https://github.com/golang/go/issues/45346

前述の、type parametersのProposalがacceptされた時点で含まれていた、constraintsにおける *type list* を置き換える提案です。
type parameterのconstraintsにおけるtype listのわかりにくさを解消し、より一般的な解決法を提案したもので、2021年7月にacceptされました。

コード例

```go
type PredeclaredSignedInteger interface {
	int | int8 | int16 | int32 | int64
}

type SignedInteger interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64
}
```

2021年8月現在、[Type Parameter Proposal](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md)のドキュメントが[type sets版に書き換えられている](https://go-review.googlesource.com/c/proposal/+/306689)ところです。
また、[言語仕様の変更](https://go-review.googlesource.com/c/go/+/294469)も既にtype sets版で作業が行われています。

type setsの詳細については、Nobishiiさんの下記の記事を参照ください。

* [Go の "Type Sets" proposal を読む - Zenn](https://zenn.dev/nobishii/articles/99a2b55e2d3e50)
* [Type Sets Proposalを読む(2) - Zenn](https://zenn.dev/nobishii/articles/type_set_proposal_2)

## constraints package

Status: accepted
Issue: https://github.com/golang/go/issues/45458

constraints packageは、type parameterのconstraintsに頻繁に使われるであろう定義をまとめたpackageです。
例えば、 `constraints.Integer` は全ての整数型にマッチする制約になっており、下記のような、整数型の値のみを受け付けるジェネリックな関数を宣言する時に使えます。

```go
import "constraints"

func DoubleInteger[T constraints.Integer](i T) T {
  return i + i
}

func main() {
  DoubleInteger(int(1))   // => int(2)
  DoubleInteger(uint8(2)) // => uint8(4)
  DoubleInteger(int64(3)) // => int64(6)
}
```

constraints packageは、初めtype parameters proposalに登場しており、本Proposalはそのpackageの内容を精緻化するものです。
無事acceptされたので、type parameterのリリースと同時にGoに実装されると思われます。

### constraints packageに定義されている型の一覧

```go
package constraints

// 符号付き整数型の制約
type Signed interface { ... }

// 符号なし整数型の制約
type Unsigned interface { ... }

// 整数型の制約
type Integer interface { ... }

// 浮動小数点数型の制約
type Float interface { ... }

// 複素数型の制約
type Complex interface { ... }

// 順序付けが可能な型の制約 (次の演算子をサポートする型 < <= >= >)
type Ordered interface { ... }

// スライス型の制約
type Slice[Elem any] interface { ~[]Elem }

// マップ型の制約
type Map[Key comparable, Val any] interface { ~map[Key]Val }

// チャネル型の制約
type Chan[Elem any] interface { ~chan Elem }
```

### 使われ方の想定

constraints packageは、他のジェネリックな機能を提供する標準packageで頻繁に使われそうです。
後述する slices / maps でも `Slice`, `Map` や　 `Ordered` の制約が使われていたりします。
単純に、Sliceだけを受け付ける関数を書きたい！といった通常の使い方ももちろん出来ます。

### 補足

他のGoが提供するconstraintsには、 `any` と `comparable` があります。
anyは、全ての型を受け付ける制約で、comparableは `比較可能` な型 (==, !=, <, <=, >, >= をサポートする型) を受け付ける制約です。
これは事前宣言された識別子に紐付く型として、Goに組み込まれます (packageとしての提供ではありません)。

Orderedとcomparableはかなり紛らわしいですが、[その違いは、comparableがinterface型を含む (ただし `==` で比較するとpanicする) こと](https://github.com/golang/go/issues/45458#issuecomment-828784686)らしいです。

## slices package

Status: accepted
Issue: https://github.com/golang/go/issues/45955

ジェネリックなslice操作を行うためのpackageを導入する提案です。
このpackageによって、これまで大変だった次のような操作が簡単に出来るようになります。

* slice同士の比較 (for文を使って全要素を比較する必要があった)
* sliceの一部を取り除いたり、sliceの途中に要素を挿入したりする操作 (これまで[SliceTrick](https://github.com/golang/go/wiki/SliceTricks)を駆使する必要があった)

### slices packageで宣言されている関数の一覧

```go
package slices

import "constraints" // See #45458

// 2つのsliceの長さが同じで、含まれる要素とその順番が等しいかどうかを返す
func Equal[T comparable](s1, s2 []T) bool

func EqualFunc[T1, T2 any](s1 []T1, s2 []T2, eq func(T1, T2) bool) bool

// 2つのsliceを比較する
// 結果は `0 if s1==s2, -1 if s1 < s2, and +1 if s1 > s2` の数値で得られる
func Compare[T constraints.Ordered](s1, s2 []T) int

func CompareFunc[T any](s1, s2 []T, cmp func(T, T) int) int

// vのs内でのindexを返す
func Index[T comparable](s []T, v T) int

func IndexFunc[T any](s []T, f func(T) bool) int

// vがsに含まれているかどうかを返す
func Contains[T comparable](s []T, v T) bool

// vをsのi番目に挿入し、変更されたsliceを返す
func Insert[S constraints.Slice[T], T any](s S, i int, v ...T) S

// s[i:j]をsから除去して、変更されたsliceを返す
func Delete[S constraints.Slice[T], T any](s S, i, j int) S

// sを複製したsliceを返す
func Clone[S constraints.Slice[T], T any](s S) S

// 等しい要素を取り除いたsliceを返す。(Unixのuniq commandのようなイメージ)
func Compact[S constraints.Slice[T], T comparable](s S) S

func CompactFunc[S constraints.Slice[T], T any](s S, cmp func(T, T) bool) S

// 容量をn増やしたsliceを返す
func Grow[S constraints.Slice[T], T any](s S, n int) S

// sliceの使われていない容量を取り除いたsliceを返す
func Clip[S constraints.Slice[T], T any](s S) S
```

### 補足

* Map, Filter, Reduceはここには含まれていません。
  - どこか、より包括的な *streams API* の一部になるとよいだろう、と[rscがコメント](https://github.com/golang/go/issues/45955#issuecomment-884406307)しています。

---

(2021/8/19時点でacceptedなのはここまで)

## maps package

Status: 議論中
Issue: https://github.com/golang/go/issues/47649


