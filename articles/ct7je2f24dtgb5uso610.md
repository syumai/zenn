---
title: import / exportの記法だけではない、CommonJS modulesとES modulesの違い
emoji: 👾
type: tech
topics: ["javascript", "nodejs"]
published: true
---

本記事は[syumai Advent Calendar 2024](https://adventar.org/calendars/11019) 4日目の記事です。
内容としては、主にWeb Developer Conference 2024の休憩中に[@NozomuIkuta](https://github.com/NozomuIkuta)さんと行った雑談を記事化したものです。

内容に何か問題があれば、本記事のコメント欄や、[X](https://x.com/__syumai)などでご連絡ください。

# require(esm)の登場

2024年、ついにNode.jsのCommonJS modulesから、ES modulesを利用できるようになりました。

使い方は簡単で、これまでCommonJS modulesから別のCommonJS modulesを利用するために使っていた `require` をそのままES modulesに対して使います。

```js
// ES modules側 (counter.mjs)
let count = 0;
export const currentCount = () => count;
export const increment = () => ++count;
export const decrement = () => --count;
```

```js
// CommonJS modules側 (index.js)
const {
  currentCount,
  increment,
  decrement,
} = require("./counter.mjs");
console.log(currentCount());
console.log(increment());
console.log(increment());
console.log(decrement());
```

```console
$ node --version
v23.3.0
$ node index.js
0
1
2
1
(node:20621) ExperimentalWarning: ...
```

[実際のコード](https://github.com/syumai/til/tree/fe42132781e103528a23aa4fab42b84845bf230b/js/requireesm/counter)を動かしてみるとわかりますが、拍子抜けするほどあっさり動作します。

この機能は、まず[4月にリリースされたv22.0.0](https://nodejs.org/en/blog/release/v22.0.0)でフラグ付きで導入され、[10月にリリースされたv23.0.0](https://nodejs.org/en/blog/release/v23.0.0)でフラグが外れ、デフォルトで有効になりました。そして、[11月にリリースされたv22.12.0 LTS](https://nodejs.org/en/blog/release/v22.12.0)にて、LTSバージョンでも初めてフラグ無しでの利用が可能となりました。

一応、まだ試験的な機能としての位置付けではあるので、今のところは利用時に警告が表示されます(v23.3.0 / v22.12.0で確認済み)。

# CommonJS modulesとES modulesの違い

CommonJS modulesからES modulesを利用できるようになったという話ですが、これらの違いは一体何でしょうか？
(実際には他にも色々ありますが、) 本記事で解説したい内容に絞って言うと、端的に、ECMAScriptの仕様上の概念として、Node.jsにおけるCommonJS modulesのコードは[**Script**](https://262.ecma-international.org/14.0/index.html#sec-scripts)に、ES modulesのコードは[**Module**](https://262.ecma-international.org/14.0/index.html#sec-modules)に対応するという違いがあります。

Node.jsは、 `.cjs` 拡張子のファイル、またはpackage.jsonに `type: "module"` が指定されていないときの `.js` ファイルをCommonJS modules (Script) として、 `.mjs` 拡張子のファイル、またはpackage.jsonに `type: "module"` が指定されているときの `.js` ファイルをES modules (Module) として扱います^[詳しい判定ロジック: https://nodejs.org/api/packages.html#determining-module-system]。

ScriptとModuleは、そもそも**文法が異なります**。
例えば、import / exportの構文は、Moduleにしか存在しません。そのため、Scriptで使用するとSyntaxErrorになります。

```js
// hello.cjs
export const message = "Hello, World!";
```

```
$ node hello.cjs
export const message = "Hello, World!";
^^^^^^

SyntaxError: Unexpected token 'export'
```

そして、その逆も存在します。そう、**Scriptでは有効で、Moduleでは無効となる構文**があります。

## Moduleは古いJavaScriptの構文を受け付けない

Moduleは、[ECMAScript 2015 (ECMA-262, 6th edition, June 2015)](https://262.ecma-international.org/6.0/index.html) で導入された、比較的新しい機能です。
(経緯は追っていないので想像ですが、)恐らくこの新しい機能上で、既にレガシーとみなされている言語機能をサポートする理由が無かったために、いくつかの構文が無効となっています。仕様としては、Moduleのコードは常にStrict modeのコードとして扱われる^["Module code is always strict mode code." https://262.ecma-international.org/14.0/index.html#sec-strict-mode-code]と定められています。

Strict modeで無効となる構文で代表的なものとしては、[with文](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/with)が挙げられるでしょう。with文のblock内では、渡されたObjectのプロパティをブロック内であたかもグローバルに存在するかのように扱うことができます。以下のサンプルコードのように、ブロック内でObjectのプロパティに対して行われた変更は、外側のスコープでも観測できます。

```js
// with.cjs
var obj = { a: 1, b: 2 };

with(obj) {
  console.log(a); // 1
  console.log(b); // 2
  b = 3
}

console.log(obj.b); // 3
```

これは一見便利に見えてしまうかもしれませんが、多くの混乱を招いてきたために非推奨となっています。MDNのwith文の解説の冒頭部分でも次のように書かれています。
> with 文の使用は推奨されません。混乱を招くバグや互換性問題の原因となる可能性があり、最適化ができなくなり、[厳格モード](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Strict_mode)では禁止されているからです。

以下のように、 `"use strict"` のディレクティブを付与したScriptのコードではwith文は動作しません。

```js
// strictwith.cjs
"use strict";

var obj = { a: 1, b: 2 };

with(obj) {
  // ...
```

```console
$ node strictwith.cjs
with (obj) {
^^^^

SyntaxError: Strict mode code may not include a with statement
```

同様に、Moduleのコードでも動作しません。

```console
$  node with.mjs
with (obj) {
^^^^

SyntaxError: Strict mode code may not include a with statement
```

Strict modeはECMAScript 5で追加された、Moduleと比べると古い機能です。そのため、**ModuleにはStrict modeには存在しない追加の制約も存在します**。例えば、 `await` はStrict modeでは識別子として有効ですが、Module内では無効^["await is reserved only inside async functions and modules." https://262.ecma-international.org/14.0/index.html#sec-keywords-and-reserved-words]です。

## ScriptからModuleを読み込んだ時、そしてその逆の場合の振る舞い

ここまでで、Scriptでは有効となるがModuleでは無効な構文が存在し、その逆も存在することを示しました。それでは、**互いに無効な構文のコードを含んだScript / Moduleを読み込んだら**どうなるでしょうか？

以下のような2つのファイルを用意し、両方CommonJS modules / ES modulesから読み込んでみます。

```js
// cjs.cjs
with ({}) {}
module.exports = {
  message: "Hello, CJS!",
};
```

```js
// esm.mjs
export const message = "Hello, ESM!";
```

繰り返しにはなりますが、Moduleではwith文が無効で、Scriptではexportの構文が無効です。

### Scriptからの読み込み

まず、Scriptから上記2ファイルを読み込んでみます。

```js
// index.cjs
const cjs = require("./cjs.cjs");
const esm = require("./esm.mjs");
console.log(cjs.message);
console.log(esm.message);
```

```
$ node index.cjs
Hello, CJS!
Hello, ESM!
```

結果、問題なく実行ができました。

### Moduleからの読み込み

続いて、Moduleから読み込んでみます。

```js
// index.mjs
import { default as cjs } from "./cjs.cjs";
import * as esm from "./esm.mjs";
console.log(cjs.message);
console.log(esm.message);
```

```
$ node index.mjs
Hello, CJS!
Hello, ESM!
```

こちらもうまくいきました。

以上の結果から、Scriptから読み込んだModuleはModuleとして、Moduleから読み込んだScriptはScriptとして解釈され、互いの構文が影響し合うことで無効になってしまうことはないことがわかりました。

:::message
ここでは解説しませんが、Node.js側の制約として、Top-level awaitを含んだES modulesをCommonJS modulesから `require` で読み込むことができないというルールは存在します。
:::

## (おまけ) Moduleから非Strict modeに脱出する方法

Moduleのコードは常にStrict modeとして実行されるということでしたが、それでは、Module配下では決して非Strict modeなコードを実行できないのでしょうか？
答えはNoで、限定的な状況下ではありますが、非Strict modeなコードを動かすことができます。ここで使うのは、 indirect eval そして `Function` constructorです。

まず、evalに渡されたコードはScriptとして解釈されます^[11-a: "Let script be ParseText(StringToCodePoints(x), Script)." https://262.ecma-international.org/14.0/index.html#sec-performeval]。Scriptとして解釈されるということは、「常にStrict modeとして扱われる」というModuleの制約から逃れられるように見えますが、実はevalは、呼び出し元がStrict modeかどうかという性質を引き継ぎます^[PerformEval abstract operationの `strictCaller` parameter がこれに該当します https://262.ecma-international.org/14.0/index.html#sec-performeval]。そのため、例えばeval内でwith文を使うと構文エラーになってしまいます。

```js
// exit-from-strict-1.mjs
globalThis.obj = {
  message: "Hello, World!",
};

// これは動かない
eval(`
  with(obj) {
    console.log(message);
  }
`);
```

ここで、[indirect eval](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/eval#%E7%9B%B4%E6%8E%A5%E7%9A%84%E3%81%BE%E3%81%9F%E3%81%AF%E9%96%93%E6%8E%A5%E7%9A%84%E3%81%AA_eval)を活用できます。indirect evalは、一度変数に格納したり、式として評価するなどした結果として、**間接的に得たeval関数**に対する呼び出しを指します。indirect evalは、evalが呼び出されたスコープを無視してグローバルスコープで動作するという点と、Strict mode配下で呼び出されたかどうかの文脈を無視するという特徴があります。この性質によって、Module配下でありながらwith文を使うことができます。以下のサンプルコードでは、[カンマ演算子](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Comma_operator)を使った式により間接的に得られたeval関数を使用しています。

```js
// exit-from-strict-2.mjs
globalThis.obj = {
  message: "Hello, World!",
};

// これは動く
(0, eval)(`
  with(obj) {
    console.log(message);
  }
`);
```

また、直接の関数宣言では同様に動きませんが、`Function` constructorを経由すると、非Strict modeのコードを動かすことができます。仕様は発見できていない^[恐らくCreateDynamicFunctionの辺りだとは思います https://262.ecma-international.org/14.0/index.html#sec-createdynamicfunction]のですが、どうやら `Function` constructorに渡されたコードもScriptとして扱われているようです (Moduleかつ、非Strict modeはありえないため)。

```js
// exit-from-strict-3.mjs
globalThis.obj = {
  message: "Hello, World!",
};

/*
function f() {
  // ここはModuleなので構文エラーになる
  with(obj) {
    console.log(message);
  }
}
f();
*/

// これは動く
new Function(`
  with(obj) {
    console.log(message);
  }
`)();
```

# まとめ

* Node.jsにおけるCommonJS modulesはECMAScriptのScriptに、ES modulesはModuleに対応する
* Script、Moduleは互いに無効となる構文が存在する
  - Moduleは古い構文を受け付けない
  - Scriptはimport / exportの構文を受け付けない
* Node.jsでCommonJS modulesとES modulesを互いに利用し合った時に、Script / Moduleの構文に起因する問題は発生しない
* (おまけ) Module内でも、限定的な状況下で非Strict modeのコードを動かすことは可能

## 感想

 `require(esm)` を使った時に、Script / Moduleの違いによる落とし穴にはまることは基本的に無さそうだったので、積極的に使っていい機能なのではないかと思いました。
Top-level awaitには色々と問題がありそうですが、これについてはきっと誰かが記事を書いてくれるはず…。

https://x.com/__syumai/status/1846722970208948315

また、メモ程度ではありますが、本記事にて使用したサンプルコードを一応GitHubに上げているので、手元で試してみたい方はこちらをご利用ください。

https://github.com/syumai/til/tree/eaefd616913f8a8893a565e254e9964ee9dd797e/js/requireesm

# おすすめ資料

uhyoさんの素晴らしいスライドです。ECMAScriptの仕様の面から、require(esm)で起こっていることについて解説してくださっています。

https://speakerdeck.com/uhyo/require-esm-toecmascriptshi-yang

hiroppyさんの記事です。require(esm)の使い方について詳しく書いてあります。

https://hiroppy.me/blog/nodejs-new-module-algorithm/
