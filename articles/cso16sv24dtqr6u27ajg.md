---
title: 初めてDefinitelyTypedにPRを出した話
emoji: ⌨️
type: tech
topics: ["TypeScript"]
published: false
---

# PRを出すことになった経緯

ある日、必要に迫られて[encoding-japanese (encoding.js)](https://github.com/polygonplanet/encoding.js)というライブラリにパッチを送る機会がありました。

https://github.com/polygonplanet/encoding.js/pull/41

こちらのライブラリは、UTF-8 <-> Shift-JIS間の文字列変換を行ってくれる非常に便利なライブラリで、この時必要だったのはShift-JISで扱えない文字を文字列中から削除する機能でした。
無事、パッチが取り込まれたものの、ここで、**実はライブラリ本体とは別でTypeScript用の型定義ファイルがメンテナンスされている**事実に突き当たりました。

実のところ、パッチを出す前からDefinitelyTyped側にencoding-japaneseの型定義が存在していることを知ってはいたのですが、実際に更新されたライブラリを使用するにあたって、自分が新たに追加したオプションが型エラーになってしまうという不便さに直面することで型定義を修正することの必要性を強く認識し、今回のコントリビュートに繋がりました。

# DefinitelyTypedとは

DefinitelyTypedは、あらゆるJavaScript製ライブラリに対するTypeScriptの型定義をメンテナンスしているリポジトリです。
まだ、TypeScriptとFlowのどちらを使うか？という選択肢の間でJavaScriptを使う開発者たちが揺れ動いていた、TypeScript黎明期を支えた偉大なプロジェクトです。

https://github.com/DefinitelyTyped/DefinitelyTyped

基本的なスタイルとしては、npmを通じて配布されるJavaScriptのライブラリに対応する型定義ファイルを `@types/${npm package名}` という名前で配布する形になっており、**ライブラリ本体のメンテナンスと独立して更新される**点が特徴となります。
有名なもので言うと、 `@types/node` や `@types/react` などは多くの方が使ったことがあるのではないでしょうか？

## DefinitelyTypedの課題

前述の通り、DefinitelyTypedのメンテナンスはライブラリ本体から独立しています。
そのため、今回自分が直面したように、**ライブラリ自体の更新に型定義の更新が追いつかない**ということが容易に発生します。

また、メンテナがライブラリ本体と分かれている場合があるという問題もあります。
DefinitelyTypedは、npmで配布されている任意のJavaScriptのライブラリに対して**第三者が型定義を追加する**ことで成長してきたプロジェクトです。
ただし、流石に、全てのライブラリの型定義を第三者がメンテナンスしているということはありません。各npm packageに対応するDefinitelyTypedリポジトリ内のpackage.jsonに、`owner`キーとしてメンテナを指定することができるようになっているため、これをライブラリ開発者が自分自身に設定しているものはメンテナが一致しています。
とは言え、ライブラリ開発者が自分からDefinitelyTypedに乗り込んでいって、自分をメンテナに指定しない限りは第三者によるメンテナンスが続きます。
（そもそも、第三者が勝手に追加した型定義ファイルに対するメンテナンス責任が元のライブラリ開発者に降り掛かってしまうとなるとおかしな話なので、独立していることそのものは仕方ないと思います）

2024年現在では、型付きのJavaScriptを書くための方法としてTypeScriptが一強という状態になっているため、わざわざ外部のプロジェクトで型定義をメンテナンスする必要性がほぼありません。
TypeScript以外の言語で開発されているライブラリであれば、DefinitelyTypedも選択肢に入るかもしれませんが、それよりも自分のライブラリと合わせて手書きの `.d.ts` ファイルを公開した方がずっと楽です。

ライブラリ本体と型定義のメンテナンスが分かれていることによる苦悩については、Prettierのメンテナの[sosukesuzuki](https://github.com/sosukesuzuki)さんがブログに書かれているので、ぜひこちらも読んでみてください。

https://sosukesuzuki.dev/posts/prettier-type-definitions/

## DefinitelyTypedのプロジェクト構成

2024年11月現在、DefinitelyTypedは、数えた限りで `types` ディレクトリ配下に**8953プロジェクト**を保持している超巨大なpnpm workspaceとなっています。

前述の各npm packageに対応するpackage.jsonに記述された `owner` キーの情報は、GitHubのCODEOWNERSファイルの自動生成に使われており、こちらは[現在8962行ある](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/b5dc32740d9b45d11cff9b025896dd333c795b39/.github/CODEOWNERS)ようでした。

`types` 配下の各ディレクトリが持つべきファイルは[READMEに示されています](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/68f519b4fee27028a73cebb98f60d78469ddea55/README.md#create-a-new-package)。基本的なものとしては `package.json` と `tsconfig.json` が存在すればよく、追加で `.eslintrc.json` や `.npmignore` を設定ファイルとして持つことができます。テストコードは、 `${npm package名}-tests.ts`というファイル名で追加します。テストコードに対しては、型チェックのみが行われます。
残りは、プロジェクトごとに必要なディレクトリを独自の判断で切りつつ `.d.ts` ファイルを配置しているようでした。

例として、encoding-japaneseでは次のような構成になっています。

```
./types/encoding-japanese
├── test
│   ├── encoding-japanese-tests.cjs.ts
│   └── encoding-japanese-tests.global.ts
├── index.d.ts
├── package.json
└── tsconfig.json
```

### (おまけ) DefinitelyTypedがpnpm workspaceになったのは結構最近という話

DefinitelyTypedがpnpm workspaceになったのは、2023年10月のことでした。

https://github.com/DefinitelyTyped/DefinitelyTyped/pull/67085

メンテナンスやコントリビュートにおけるなんらかの苦労があったためにpnpm化されたのだと思いますが、幸い（？）筆者はpnpm workspace化された後に初めて触れたので、非常に快適にコントリビュートを行うことができました。

## DefinitelyTypedのバージョニング

DefinitelyTyped配下のプロジェクトのpackage.jsonに記載するバージョンはやや特殊で、常にpatchバージョンとして `.9999` を指定します。encoding-japaneseの例としては、以下のように `2.2.9999` を指定していました。

https://github.com/DefinitelyTyped/DefinitelyTyped/blob/77472518b28d0e7001bf23715cc08c4b212c613a/types/encoding-japanese/package.json#L2-L4

ライブラリ本体と合わせる必要があるのはmajor, minorバージョンまでで、patchバージョンはDefinitelyTypedのCI/CDがnpm publishを行う際に自動で割り当てられます。
初めて使われたminorバージョンについては、patchバージョンが0でpublishされ、その後の更新でインクリメントされていきます。上記の例で言うと、 `2.2.0` が初めにpublishされ、次の変更で `2.2.1` にインクリメントされていくという流れになります。

ここで重要なのは、**DefinitelyTypedから配布されるpackageのpatchバージョンはライブラリ本体と同期していない**点です。ライブラリ本体のpatchバージョンと、`@types`で配布される型定義のpatchバージョンを合わせる形でインストールすることには特に意味がないということを理解した上で利用しましょう。

# DefinitelyTypedにパッチを出す流れ

コントリビューションの方法については[READMEに記載されています](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/README.md#how-can-i-contribute)、また、[Pull Requestのテンプレートでもチェックリストとして示されています](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/.github/PULL_REQUEST_TEMPLATE.md)。
行う作業は大まかに以下の内容となります。

1. DefinitelyTypedのリポジトリをcloneし、必要なdependenciesをインストールする
2. 型定義を修正する
3. テストコードを直して、型チェックを通過することを確認する
4. コードをフォーマットする
5. (必要があれば) バージョンを更新する
6. PRをOpenする

## 1. DefinitelyTypedのリポジトリをcloneし、必要なdependenciesをインストールする

まずは `github.com/DefinitelyTyped/DefinitelyTyped` をcloneします。
その後、必要なdependenciesをインストールするのですが、 ここで無邪気にpnpm installを実行すると、8900個以上あるWorkspaceのpnpm installが一斉に走って完了に時間がかかってしまいます。そうなってしまうと大変なので、 `pnpm install -w --filter "...{./types/npm package名}...` といった形で変更を加えるpackageのWorkspaceのみをフィルタしてインストールします。

## 2. 型定義を修正する

こちらがメインの作業です。
今回は、自分が追加した `"ignore"` と、別のPRで追加されていた `"error"` を[fallbackのオプションに追加](https://github.com/DefinitelyTyped/DefinitelyTyped/pull/69982/files?diff=unified&w=0#diff-cbac93669dac57dea9a110b7832ddf9bd4d4e960e33365d8f93a004f910e37f7)しました。

```diff ts
- fallback?: "html-entity" | "html-entity-hex";
+ fallback?: "html-entity" | "html-entity-hex" | "ignore" | "error";
```

## 3. テストコードを直して、型チェックを通過することを確認する

型定義を修正したら、続いて自分の変更が意図通りに型チェックを通過するか検証するコードをテストコードに追加します。
今回は、 `fallback` プロパティに `"ignore"` と、`"error"` を追加してもそれぞれ型エラーにならないことを検証しました。以下は、 `"ignore"` のテストの抜粋です。

```ts
const sjisArray5 = Encoding.convert("🐙", {
    to: "SJIS", // to_encoding
    from: "UTF8", // from_encoding
    type: "string",
    fallback: "ignore",
});
sjisArray5; // $ExpectType string
```

テストコードの修正ができたら、 `pnpm test ${npm package名}` で実行します。型エラーがあればテストが失敗します。
全ての修正が完了したら、最後に `pnpm run test-all` を実行して、他のpackageへの影響がないことを確認します。

## 4. コードをフォーマットする

全ての変更作業が終わったら、[dprint](https://dprint.dev/)でフォーマットを行います。 `pnpm dprint fmt -- 'types/${npm package名}/**/*.ts'` の実行で簡単にフォーマットが適用されます。

## 5. (必要があれば) バージョンを更新する

前述の通り、major / minorバージョンはライブラリ本体と合わせる必要があります。自分が型定義の変更を行う対象のライブラリ本体のmajor / minorバージョンが上がっている場合はpackage.jsonに記載のバージョンを更新します。
今回は、encoding-japanese本体のバージョンが `v2.2.0` に上がっていたので、DefinitelyTyped側のバージョンを `v2.0.9999` から `v2.2.9999` にしました。(v2.1系では型定義の更新が行われていなかったようです)

```diff json
- "version": "2.0.9999",
+ "version": "2.2.9999",
```

## 6. PRをOpenする

PRをOpenすると、CODEOWNERにレビューの通知が飛びます。あとは、変更内容に問題がないかの確認が完了するのを待ちます。
今回は、CODEOWNERのrhysdさんが非常に迅速にレビューしてくださったので、PRを出したその日のうちにマージ&リリースまで進めていただくことができました。

# おわりに

DefinitelyTypedはあまりに巨大なプロジェクトなので、自分で試みるまでは変更を加えるのが非常に難しいのではないかと思っていました。しかし、実際には、コントリビュートする手順がきちんとまとまっていますし、最近行われたpnpm workspace化もあり、ほとんど苦労することなく作業を完遂することができました。
かなり整ったレールに乗って作業でき、流石、これだけの規模で長期に渡ってメンテナンスされ続けているプロジェクトはすごい…。という感動がありました。
もし、お使いのライブラリで、ライブラリ本体とDefinitelyTypedから配布されている型定義がズレてしまっており困った時は、ぜひ気軽にコントリビュートを検討してみてください。

## 補足

本記事は、TSKaigi Kansai 2024のLTをベースにした内容となっています。
スライド版は下記リンクからご覧いただけます。

https://speakerdeck.com/syumai/definitelytypednichu-meteprwochu-sitahua
