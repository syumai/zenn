---
title: 初めてDefinitelyTypedにPRを出した話
emoji: ⌨️
type: tech
topics: ["TypeScript"]
published: false
---

# 事のあらまし

ある日、必要に駆られて[encoding-japanese](xxx)というライブラリにパッチを送る機会がありました。
こちらのライブラリは、UTF-8 <-> Shift-JIS間の文字列変換を行ってくれる非常に便利なライブラリで、この時必要だったのは[Shift-JISで扱えない文字を文字列中から削除する機能]()でした。
無事、パッチが取り込まれたものの、ここで、**実はライブラリ本体とは別でTypeScript用の型定義ファイルがメンテナンスされている**事実に突き当たりました。

実のところ、パッチを出す前からDefinitelyTyped側に型定義が存在していることを知ってはいたのですが、実際に更新されたライブラリを使用するにあたって、自分が新たに追加したオプションが型エラーになってしまうという不便さに直面することで、型定義を修正することの必要性を強く認識しました。

# DefinitelyTypedとは

DefinitelyTypedは、あらゆるJavaScript製ライブラリに対するTypeScriptの型定義を管理しているリポジトリです。
まだ、TypeScriptとFlowのどちらを使うか？という選択肢の間でJavaScriptを使う開発者たちが揺れ動いていた、TypeScript黎明期を支えた偉大なプロジェクトです。

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

2024年現在では、型付きのJavaScriptを書くための方法としてTypeScriptが一強という状態になっているため、わざわざ外部のプロジェクトで型定義を管理する必要性がほぼありません。
TypeScript以外の言語で開発されているライブラリであれば、DefinitelyTypedも選択肢に入るかもしれませんが、それよりも自分のライブラリと合わせて手書きの `.d.ts` ファイルを公開した方がずっと管理が楽です。

ライブラリ本体と型定義のメンテナンスが分かれていることによる苦悩については、Prettierのメンテナのsosukesuzukiさんがブログに書かれているので、ぜひこちらも読んでみてください。

https://sosukesuzuki.dev/posts/prettier-type-definitions/

## DefinitelyTypedのプロジェクト構成

2024年11月現在、DefinitelyTypedは、数えた限りで `types` ディレクトリ配下に**8953プロジェクト**を管理している超巨大なpnpm workspaceとなっています。

前述の各npm packageに対応するpackage.jsonに記述された `owner` キーの情報は、GitHubのCODEOWNERSファイルの自動生成に使われており、こちらは現在8962行あるようでした。
https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/.github/CODEOWNERS

各 `types` ディレクトリ配下の構造は比較的自由で、 `package.json` と `tsconfig.json` が基本的なものとして存在すればよく、追加で `.eslintrc.json` や `.npmignore` を設定ファイルとして持てるようです。
テストコードは、 `${package名}-tests.ts`というファイル名で追加するように[READMEで指示されています](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/68f519b4fee27028a73cebb98f60d78469ddea55/README.md#create-a-new-package)が、これに従っていないプロジェクトも多そうです。テストコードに対しては、型チェックのみが行われます。
残りは、プロジェクトごとに必要なディレクトリを切って `.d.ts` ファイルを配置しているようでした。

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

# DefinitelyTypedにパッチを出す流れ
