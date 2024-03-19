---
title: Goのテストで文字列のポインタ型の値に対する比較をする時に迷った話
emoji: 🥝
type: "tech"
topics: ["Go"]
published: true
publication_name: "basemachina"
---

# 文字列のポインタ型の値を比較するテストコード

Goのテストで、次のような文字列のポインタ型の値に対する比較とその結果を表示するコードを書きました。

このテストコードでは、以下のことを検証しています。

* 関数Fがnilを返さない
* 関数Fがnilを返さなかった場合、結果のポインタ型の値が指し示す内容が期待する文字列と一致する

```go
func F() *string {
	s := "value"
	return &s
}

func TestF(t *testing.T) {
	got := F()
	want := "value"
	if got == nil || want != *got {
		t.Errorf("want: %v, got: %v", want, got)
	}
}
```

https://go.dev/play/p/5B1kD-VEW_S

実際にテストを実行すると、問題なく成功して終了します。

しかしながら、以下のケースではどうでしょうか？

1. 関数Fからnilが返ったケース
2. 関数Fから期待しない文字列が返ったケース

それぞれのケースについて確認してみましょう。

## 1. 関数Fからnilが返ったケース

下記のように、関数Fがnilを返すように書き換えます。

```go
func F() *string {
	return nil
}

func TestF(t *testing.T) {
	got := F()
	want := "value"
	// got == nil で引っ掛かる
	if got == nil || want != *got {
		t.Errorf("want: %v, got: %v", want, got)
	}
}
```

https://go.dev/play/p/tnTMq0TEwf8

この時、テスト結果の出力内容は次のようになります。

```
=== RUN   TestF
    prog_test.go:14: want: value, got: <nil>
--- FAIL: TestF (0.00s)
FAIL
```

`got: <nil>` という表示は期待通りで、特に問題なく動作することが確認できました。

## 2. 関数Fから期待しない文字列が返ったケース

下記のように、関数Fが文字列 `"invalid value"` のポインタを返すように書き換えます。

```go
func F() *string {
	s := "invalid value"
	return &s
}

func TestF(t *testing.T) {
	got := F()
	want := "value"
	// want != *got で引っ掛かる
	if got == nil || want != *got {
		t.Errorf("want: %v, got: %v", want, got)
	}
}
```

https://go.dev/play/p/RUb_4UM7ulm

この時、テスト結果の出力内容は次のようになります。

```
=== RUN   TestF
    prog_test.go:15: want: value, got: 0xc0001161d0
--- FAIL: TestF (0.00s)
FAIL
```

`got: 0xc0001161d0` という表示は、明らかに期待通りではありません。

このテストでは、結果として具体的にどのような文字列が返って失敗したのかを確認したいので、別の方法を使って結果を出力した方がよいことがわかりました。

# 対応策

対応策として、以下のものを検討しました。

* t.Errorfのフォーマット方法を変える
* if文の条件を分ける
* 結果の出力時にpointer indirectionする
* reflect.DeepEqualを使う
* cmp.Diffを使う

それぞれについて、検討の結果を示します。

## t.Errorfのフォーマット方法を変える

今使っている `%v` で十分な情報が出せていないのではと思い、fmt packageのドキュメントを参考に別の書式を検討しました。

https://pkg.go.dev/fmt#hdr-Printing

> %v	the value in a default format
> 	when printing structs, the plus flag (%+v) adds field names
> %#v	a Go-syntax representation of the value
> %s	the uninterpreted bytes of the string or slice

ドキュメントから読み取れる通り、今回のような文字列のポインタ型の値の出力を詳細に行うというオプションはなく、うまくいかないとは思いつつも下記のコードで検証を行いました。

```go
func main() {
	s := "abcde"
	ptr := &s
	var nilPtr *string

	fmt.Printf("%v, %v\n", ptr, nilPtr)
	fmt.Printf("%+v, %+v\n", ptr, nilPtr)
	fmt.Printf("%#v, %#v\n", ptr, nilPtr)
	fmt.Printf("%s, %s\n", ptr, nilPtr)
}
```

結果としては以下のようになり、いずれも期待する出力にはなりませんでした。

```
0xc000102020, <nil>
0xc000102020, <nil>
(*string)(0xc000102020), (*string)(nil)
%!s(*string=0xc000102020), %!s(*string=<nil>)
```

https://go.dev/play/p/biDhAjIHU7D

## if文の条件を分ける

このパターンは、一番素朴で、かつ明らかにうまくいきます。

```go
func F() *string {
	s := "invalid value"
	return &s
}

func TestF(t *testing.T) {
	got := F()
	want := "value"
	if got == nil {
		t.Errorf("want: %v, got: %v", nil, got)
	} else if want != *got {
		t.Errorf("want: %v, got: %v", want, *got)
	}
}
```

https://go.dev/play/p/52so8G3Mqqv

出力結果は以下の通りです。

```
=== RUN   TestF
    prog_test.go:17: want: value, got: invalid value
--- FAIL: TestF (0.00s)
FAIL
```

しかしながら、if ~ elseの分岐をこの目的で書くとなると、コードの冗長性が増えてしまうのでやや気になるところです。

## 結果の出力時にpointer indirectionする

このパターンは、明らかにうまくいきません。

関数Fが文字列のポインタ型の値を返す時はうまく動きますが、nilを返した時はpanicします。

```go
func F() *string {
	return nil
}

func TestF(t *testing.T) {
	got := F()
	want := "value"
	if got == nil || want != *got {
		// *gotでpanic!
		t.Errorf("want: %v, got: %v", want, *got)
	}
}
```

https://go.dev/play/p/jzW4S3Kq2rd

出力結果は以下の通りです。

```
=== RUN   TestF
--- FAIL: TestF (0.00s)
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x504c67]
```

## reflect.DeepEqualを使う

reflect.DeepEqualは、ポインタ型の変数の指し示す内容も含めた比較を行ってくれます。

https://pkg.go.dev/reflect#DeepEqual

このパターンは、if文の条件を単純化するのには使えますが、結局結果の出力の問題は解決できません。

```go
func F() *string {
	return nil
}

func TestF(t *testing.T) {
	got := F()
	wantVal := "value"
	want := &wantVal
	if !reflect.DeepEqual(want, got) {
		// *gotでpanic
		t.Errorf("want: %v, got: %v", want, *got)
	}
}
```

https://go.dev/play/p/MtQdzaTZ_OO

## cmp.Diffを使う

最後に試したのがこのパターンです。
今回のテスト対象のプロジェクトでは、 [go-cmp](https://pkg.go.dev/github.com/google/go-cmp)を導入していたので、これを使います。
冒頭の2パターンのそれぞれについて、go-cmpを使った場合の結果を見てみましょう。

### 1. 関数Fからnilが返ったケース

```go
func F() *string {
	return nil
}

func TestF(t *testing.T) {
	got := F()
	wantVal := "value"
	want := &wantVal
	if diff := cmp.Diff(want, got); diff != "" {
		t.Errorf("(-want +got):\n%s", diff)
	}
}
```

https://go.dev/play/p/wVDLpnK0v_o

出力結果は以下の通りで、nilが返ってきたことが明確にわかる形式となっています。

```
=== RUN   TestF
    prog_test.go:18: (-want +got):
          (*string)(
        - 	&"value",
        + 	nil,
          )
--- FAIL: TestF (0.00s)
FAIL
```

### 2. 関数Fから期待しない文字列が返ったケース

```go
func F() *string {
	s := "invalid value"
	return &s
}

func TestF(t *testing.T) {
	got := F()
	wantVal := "value"
	want := &wantVal
	if diff := cmp.Diff(want, got); diff != "" {
		t.Errorf("(-want +got):\n%s", diff)
	}
}
```

https://go.dev/play/p/pc-JLGHXgKc

出力結果は以下の通りで、nilの場合と同様非常に見やすい形式です。

```
=== RUN   TestF
    prog_test.go:19: (-want +got):
          &string(
        - 	"value",
        + 	"invalid value",
          )
--- FAIL: TestF (0.00s)
FAIL
```

# 今回の結論

結論としては、cmp.Diffを使うのが最も楽で、かつ出力結果も見やすいのでこちらを使うことにしました。
個人的には、cmp.Diffは複雑なフィールド構成の構造体の比較に使うことが多かったのですが、今回、単純なポインタ型の値の比較で使っても便利ということがわかったので、使い方の幅を広げてみようと思います。
外部のライブラリに依存せずに、今回の目的を単純な形で達成できる方法がもしあれば、コメントなどでぜひ教えてください！