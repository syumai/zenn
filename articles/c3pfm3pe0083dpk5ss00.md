---
title: "Goのdefer文と名前付き結果パラメータを組み合わせた時の挙動を理解する"
emoji: "🌊"
type: "tech"
topics: ["go", "プログラミング", "言語仕様"]
published: true
---
軽い気持ちでTwitterに投下したクイズに思った以上に反響があったので、解説記事を書きました。

# クイズについて

[投下したクイズ](https://twitter.com/__syumai/status/1415309048686149639?s=20)は次のような内容です。

```go
func main() {
  println(f())
}

func f() (a, b int) {
  defer func() {
    a += 1
    b += 1
  }()
  return 2, 3
}
```

この時の出力結果を問う問題で、選択肢は下記の4つです。

1. `0 0`
2. `1 1`
3. `2 3`
4. `3 4`

## 問題のポイント

このコードの挙動を知るためのポイントは下記の3つです。

1. 関数は何を返しているのか
1. return 文は何をしているのか
1. defer 文で遅延された関数の実行タイミング

まず、関数宣言の構文のおさらいを行って、その後解説に進みます。

# おさらい

下記に、今回の内容に関係する部分のEBNFを抜粋[^1]したものを示します。

```
FunctionDecl (関数宣言) = "func" FunctionName Signature [ FunctionBody ] .
Signature (シグニチャ) = Parameters [ Result ] .
Result (結果) = Parameters | Type .
Parameters (パラメータ) = "(" [ ParameterList [ "," ] ] ")" .
```

[^1]: 引用元: [Function declarations](https://golang.org/ref/spec#Function_declarations), [Function types](https://golang.org/ref/spec#Function_types)

雑な図で恐縮ですが、関数宣言のコードと照らし合わせると下記のような構成になっています（厳密なものではありません）

![](https://storage.googleapis.com/zenn-user-upload/17c73ca68cc574f7cef21962.png)

今回の記事で重要なのは下記の項目です。

* **結果パラメータ (result parameters)**
  - 関数 / メソッドを呼び出した結果として呼び出し元に返されるパラメータのこと。上記図中での `(b int)` に対応する。
    - 関数のパラメータ (上記図中での `(a int)`) とは区別される。
* **名前付き結果パラメータ (named result parameters)**
  - 結果パラメータに対して、名前を付与したもの。
    - 結果パラメータは名前無しで `(b int)` ではなく `int` とだけ書くことが出来る[^2]。

本記事中での呼称も上記の通りとなります。

[^2]: 結果パラメータについては、どちらかと言うと名前無しの方が使用頻度は高い。

# ポイントの解説

先ほど挙げた、

1. 関数は何を返しているのか
1. return 文は何をしているのか
1. defer 文で遅延された関数の実行タイミング

について解説を行います。

## 1. 関数は何を返しているのか

関数は、**関数の実行終了時点で結果パラメータに設定されている値を返します**。
これは、Goの言語仕様の [`Calls`](https://golang.org/ref/spec#Calls)[^3] に記載されています。

[^3]: `Calls` は、`式` の一つ。関数またはメソッドを呼び出した時に得られる値について定義されている

> The return parameters of the function are passed by value back to the caller when the function returns.

ざっくり訳すと、 *関数が返るとき、関数の戻りパラメータが呼び出し元に値で渡される* と書かれています。(仕様のこの箇所でのみ `return parameters` と書かれていますが、これは `result parameters` の事を指していると考えて問題無いでしょう。)

次に、`関数の実行終了時点で結果パラメータに設定されている値を返す`と言った時の、**結果パラメータに設定されている値**とは何かについて考えます。

下記のような関数宣言があったとします。
```go
func F(a int) (b int) {
  ...
}
```

この時、名前付き結果パラメータの `b` は次のように使うことが出来ます。
```go
func F(a int) (b int) {
  b = 10
  return
}
```

このように、名前付き結果パラメータは、関数内で通常のローカル変数と同じように扱うことが出来ます[^4]。
ここから、結果パラメータに設定されている値とは、**関数実行終了時点で、名前付き結果パラメータの変数に設定されている値**のことを指すと考えられます。
上記の例で言うと、関数実行終了時点で変数 `b` に設定されている値は `10` なので、関数呼び出しの結果は `10` となります。

[^4]: 仕様では、 *The result parameters act as ordinary local variables and the function may assign values to them as necessary.* と書かれています。

## 2. return 文は何をしているのか

return 文にはいくつか機能がありますが、重要な機能として**結果パラメータへの値の設定**[^5]があります。

[^5]: 仕様では、 *A "return" statement that specifies results sets the result parameters ...* と書かれています。

例を見てみましょう。

### 例1) 単一の値の返却

```go
func F() (i int) {
  return 100
}
```

この時、 `return 100` は、名前付き結果パラメータの変数 `i` に対して `100` を設定します。
関数は、実行終了時点で結果パラメータの変数に設定されている値を返すので、 `100` を返します。

### 例2) 複数の値の返却

```go
func F() (i int, err error) {
  e := fmt.Errorf("error")
  return 200, e
}
```

この時、 `return 200, e` は、名前付き結果パラメータの変数 `i` に対して `200` を、 `err` に対して `e`を設定します。
関数は、実行終了時点で結果パラメータの変数に設定されている値を返すので、 `200` , `e` の2つを返します。

上記の例から、**return 文は結果パラメータの変数への代入を行う機能を持っている**と考えると理解しやすいと思います。

return 文は、関数を終了させる機能も持つので、if 文などで分岐して複数のreturn 文を書いた場合は、実行されたreturn 文によって、結果パラメータの変数へ代入される値が変わります。

:::details 名前無し結果パラメータとreturn 文を一緒に使った場合についての補足
実は、名前の無い通常の結果パラメータに対しても、同等のルールが適用されていると考えられます。
これは、[DQNEOさんが Gophers Slack に書いていた内容](https://gophers.slack.com/archives/C0178L0C5EU/p1626268859229200)ですが、結果パラメータに名前が無かった場合、 **return 文は名前の無い見えない変数への代入を行う** と言う考え方があります。
例として、

```go
func f() (int) {
   return 1
}
```

は、

```go
func f() (_r0 int) {
   return 1
}
```

と言った形に内部的に変換され、return 文は結果パラメータの変数 `_r0` に値を設定すると言う考え方です。

実際に、このような実装になっているGo言語処理系も存在しているかも知れません。
:::

## 3. defer 文で遅延された関数の実行タイミング

defer 文で遅延された関数は、**return 文が実行された後、関数が呼び出し元に値を返す前**に実行されます[^6]。
したがって、defer 文によって遅延された関数は **return 文が結果パラメータの変数に設定した値を使うことが出来ます**。

[^6]: 仕様には、 *if the surrounding function returns through an explicit return statement, deferred functions are executed after any result parameters are set by that return statement but before the function returns to its caller.* と書かれています。

コード例を挙げます。

```go
func F() (i int) {
  defer func() {
    fmt.Println(i * 2)
  }()
  return 100
}
```

このような場合、`defer func(){ ... }()` の呼び出し[^7]は、return 文が結果パラメータの変数 `i` に値を設定した後に行われます。
そのため、出力結果は `100 * 2` で `200` となります。

[^7]: 厳密には、 `defer 文によって遅延された関数の呼び出し` ですが、わかりやすさのために表記を簡略化しています。

さらに、もう一つ特筆すべき事項があります。
ここまで説明した内容に、下記のようなものがありました。

* 結果パラメータは、関数内では通常のローカル変数と同じように扱うことができる
* 関数は、関数の実行終了時点で結果パラメータに設定されている値を返す

これらと、defer 文によって遅延された関数の実行タイミングが *return 文が実行された後、関数が呼び出し元に値を返す前* となる仕様を組み合わせると、 **defer 文は、結果パラメータに設定された値を変更して、関数の返す値を操作することが出来る** と言えます。

```go
func F() (i int) {
  defer func() {
    i = i + 200
  }()
  return 100
}
```

このようなコードを書いた時、`defer func(){ ... }()` の呼び出し時点では、 `i` に `100` が設定されているので、 `i = i + 200` は、 `i` に `300` を設定します。
この結果、関数が返す値は `300` となります。

ここまでで、今回の問題を解くために必要な性質は全て説明できました！

# 問題の解説

では、問題を振り返ってみましょう。

```go
func main() {
  println(f())
}

func f() (a, b int) {
  defer func() {
    a += 1
    b += 1
  }()
  return 2, 3
}
```

ここまで説明した性質から、このコードは次のように実行されるとわかります。

1. `return 2, 3` が `a, b` のそれぞれに値を設定する
   - `a` は `2`、 `b` は `3` となる
2. `defer func(){...}()` が呼び出され、 `a, b` のそれぞれに `1` ずつ足す
   - `a` は `3`、 `b` は `4` となる
3. 関数の実行が終了し、結果パラメータに設定された値を返す
   - `a, b` に設定された `3, 4` を返す
   
よって、答えは `4` 番の `3 4` でした！

# まとめ

最後に、今回説明したポイントについてまとめます。

1. **関数は何を返しているのか**
   - 関数実行終了時点で、結果パラメータに設定されている値を返す
1. **return 文は何をしているのか**
   - 結果パラメータに値を設定している
1. **defer 文で遅延された関数の実行タイミング**
   - return 文が実行された後、関数が呼び出し元に値を返す前に実行される
     - 結果パラメータの値を操作することも出来る

以上、defer 文と名前付き結果パラメータの組み合わせの挙動に迷った場合は、ぜひ本記事を思い出してみてください。

# 補足

## 今回の性質を使ったパターン

### エラーのキャプチャリング

今回説明した性質を使うと、 **関数が返すエラーをキャプチャする** ことが出来ます。

次のような関数があったとします。

```go
// a が b より小さいことを確認する。そうでなければエラーを返す　
func Less(a, b int) (err error) {
  if a > b {
    return fmt.Errorf("a must not be greater than b") // return 文 A
  }
  if a == b {
    return fmt.Errorf("a and b must not be equal") // return 文 B
  }
  return nil
}
```

この時、次のようなdefer 文を書けば、`return 文 A`と`return 文 B`のどちらでエラーが起きたが記録することが出来ます。

```go
func Less(a, b int) (err error) {
  defer func() {
    // a が bより小さくなかった場合、ログにエラーが記録される
    log.Printf("error happened in Less: %v", err)
  }()
  if a > b {
    return fmt.Errorf("a must not be greater than b")
  }
  if a == b {
    return fmt.Errorf("a and b must not be equal")
  }
  return nil
}
```

実用的な例を挙げられず恐縮ですが、このような使い方があると覚えておくと便利かもしれません。

### panicに渡された値をrecoverで取得して結果パラメータに設定する

これはtenntennさんがTwitterに書かれていたもので、非常に実用的です。興味のある方は下記Tweetから資料をご覧いただければと思います。

https://twitter.com/tenntenn/status/1416941261328588801?s=20
