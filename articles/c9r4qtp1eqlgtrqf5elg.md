---
title: "SUNMI端末アプリを開発したい方向け情報まとめ"
emoji: "🧾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["sunmi", "android", "reactnative"]
published: true
---

先日、安かったと言う理由だけで、つい[SUNMI V1s](https://www.sunmi.com/ja/v1s/)を購入してしまいました。
SUNMI V1sは、レシート印刷機能内蔵のオシャレなAndroid端末です。通常は、QR決済に対応したレジなどに置いてありますが、最近は中古品が大量に流通しているようです。
また、家電量販店でも同シリーズの端末を購入することが出来るそうです。

これを有効活用するには自分オリジナルのコンテンツを印刷出来るアプリを作るしかない！と思ったので作ってみました。

https://twitter.com/__syumai/status/1520923861206061056

アプリを作る過程で知った情報についてまとめることで、似た境遇の方が助かるのではと思ったので書いておきます。

本記事の内容は、基本的に[Printing Service - Sunmi Developer Docs](https://docs.sunmi.com/en/general-function-modules/printing-service/)を参考にしています。

## 大前提

SUNMI端末のプリンタを自由に使うには、**Androidアプリを作る**ことが絶対に必要になります。
最終的に、何かしらの手段でアプリを作って自分の端末にインストールしましょう。

## プリンタとの接続方法

接続方法には3種類(+おまけ1種類)あります。

* AIDL (Android Interface Definition Language)
* Bluetooth (Bluetooth device name: InnerPrinter)
* JS bridge
* (printerlibrary)

### AIDL

SUNMI端末内のプリンタとの基本的な接続方法です。
AIDLそのものについては、Androidのドキュメントを参照ください。

https://developer.android.com/guide/components/aidl?hl=ja

SUNMIのサイトでは、各端末シリーズ向けのプリンタとのインタフェースがAIDLファイルとして記述、配布されています。
P series / V seriesのAIDLファイルは下記URLにてZIPで配布されています。(GitHubでもなく直接ZIPと言うのが味があります)

https://ota.cdn.sunmi.com/DOC/resource/re_cn/AIDL%E6%96%87%E4%BB%B6/P1&P14g&V1s-1.zip

これを組み込んだAndroidアプリを作れば、簡単にプリンタと通信できると言う仕組みになっているようです。
ただし、端末によってインタフェースの差異があり、これを吸収することは出来ません。(V1s向けに作ったアプリが他のシリーズの端末向けで動くかはわからない)

### Bluetooth

Bluetoothの仮想デバイスです。
実際にはプリンタは端末に直接接続されていますが、アプリから扱いやすくするためにBluetooth端末としても認識可能になっています。
この接続方法を使う場合は、通常の手段でAndroidアプリからBluetooth端末に接続しに行けばよいと思われます。
端末名は、固定で `InnerPrinter` となっています。

余談ですが、これを使えばWebBluetooth経由でブラウザから直接プリンタに繋げるのではと思ったのですが、仮想デバイスであるためか、端末の検索で引っ掛からずに失敗しました（誰か成功したら教えてください）

### JS bridge

AndroidのWeb View内のJavaScriptから呼び出す方式です。
ぶっちゃけあまり調べていないのですが、アプリ自体がほぼWeb Viewで書かれているケースで便利そうです。
この場合でも、結局Androidアプリの作成は必要になります。

### printerlibrary

最近出来たらしいです。Developer DocsのPDFに記載があります。
Mavenのリポジトリで配布されています。

https://mvnrepository.com/artifact/com.sunmi/printerlibrary

SUNMIの公式サンプルアプリではこのライブラリを使っています。

https://github.com/shangmisunmi/SunmiPrinterDemo/blob/ef11dfa087249327a44383c0e8d9a2c3ceb6bac8/README.md#library

printerlibraryを使うと、AIDLと違い端末ごとの差異を気にしなくていいといったようなことが書かれています。
実際の通信方法として何を使っているのかはわかりませんが、AIDLか仮想Bluetoothデバイスを使ってるんだろうなと思います。

今は、これを使うのが一番良さそうな気配があります。

## アプリの作成方法

2つの方法を示します。

* Android Studioで作る
* React Nativeで作る

### Android Studioで作る

正直、筆者がここ最近Androidアプリを作っていないため、全く詳しくない方法です。
ですが、上記のprinterlibraryを使った[公式サンプルアプリ](https://github.com/shangmisunmi/SunmiPrinterDemo)を元にコードをいじれば簡単に出来るのではと思います。
printerlibrary経由のプリンタに対する呼び出しは、AIDLを使った呼び出しと基本的に共通したインタフェースになっています。（下記React Native用ライブラリも同様です）

### React Nativeで作る

AIDLをラップしたReact Native向けライブラリを作っている方がいらっしゃるので、これを使わせていただきます。

https://github.com/januslo/react-native-sunmi-inner-printer

ただ、残念ながらこのライブラリが非常に古く、そのままだとビルドに失敗してしまったので自分用にforkしたものを用意しました。(正確には、forkのforkのforkのforkです…。)

https://github.com/syumai/react-native-sunmi-v2-printer

これを使って実際に作ったアプリを下記に示すので、使い方の参考にしてみてください。

https://github.com/syumai/sunmi-namecard-printer

## 印刷機能を呼び出す

SUNMI端末の内蔵プリンタの印刷機能を呼び出すには、2つの方法があります。

* Character Printing機能を使う
* ESC/POS instructionを使う

(これらを使う前に、 `printerInit()` の呼び出しを行って、プリンタの状態 (文字サイズの設定など) をリセットしておきましょう)

### Character Printing機能を使う

最も一般的な印刷機能の呼び出し方です。
事前に用意されたメソッドの呼び出し形式で簡単に印刷処理を行うことが出来ます。

[Developer DocsのPDF](https://file.cdn.sunmi.com/SUNMIDOCS/%E5%95%86%E7%B1%B3%E5%86%85%E7%BD%AE%E6%89%93%E5%8D%B0%E6%9C%BA%E5%BC%80%E5%8F%91%E8%80%85%E6%96%87%E6%A1%A3EN-0224.pdf) の 1.2.6 で説明されています。

例えば、下記のようなメソッドが用意されています。

```java
void setAlignment(int alignment, ICallback callback)
void setFontSize(float fontsize, ICallback callback)
void printText(String text, ICallback callback)
void printTextWithFont(String text, String typeface, float fontsize, ICallback callback)
void printOriginalText (String text, ICallback callback)
```

* setAlignmentで左、真ん中、右揃えを変え、
* setFontSizeで文字を大きくし、
* printTextで文字を印刷する

といった形で直感的に使うことが出来ます。

レシートでよくある表組み (商品名、価格、個数を表形式で配置) を印刷するための、 `printColumnsText` なども用意されています。

同様の呼び出し方で、QRコード、バーコードも簡単に印刷することが出来ます。

```java
void printBarCode(String data, int symbology, int height, int width, int textPosition, ICallback callback)
void printQRCode(String data, int modulesize, int errorlevel, ICallback callback)
void print2DCode(String data, int symbology, int modulesize, int errorlevel, ICallback callback)
```

別途QRコードの画像を用意したりする必要が無く便利です。

### ESC/POS instructionを使う

ESC/POSは、一般的なレシート印刷のためのプロトコルで、SUNMI端末の内蔵プリンタは `sendRAWData` メソッド経由でこれを受け付ける事が出来ます。
メソッドの名前の通り、ESC/POS命令のバイト列を生で書き込む必要があるため、入力は非常に大変です。

命令セットについて記載したWordファイルがSUNMIから配布されているので、これを参照しながら入力する形になります。

https://ota.cdn.sunmi.com/DOC/resource/re_en/ESC-POS%20Command%20set.docx

例えば、文字を太くするには次のように命令を入力します。

```java
// Instruction: make fonts bold {0x1B, 0x45, 0x1}
woyouService.sendRAWData(newbyte[]{0x1B,0x45,0x1}, callback);
// Instruction: cancel bold {0x1B, 0x45, 0x0}
woyouService.sendRAWData(newbyte[]{0x1B,0x45,0x0}, callback);
```

ESC/POS instructionは、Character Printing機能と一緒に使うことが出来ます。
Character Printing機能で不足する表現を、sendRAWDataをラップした別のメソッドなどで実現するといった使い道が考えられます。

全てをESC/POS instructionで表現したいとなると、別の形式のデータをESC/POSに変換することを考える必要があります。
この目的に最も近いツールが `receiptline` です。

#### receiptlineについて

receiptlineは、OFSC (Open Foodservice Systems Consortium)の開発したレシート記述言語および、そのツールチェインです。

https://github.com/receiptline/receiptline

ツールはJavaScriptで記述されていて、ブラウザ上やNode.jsで、receiptlineで記述されたレシートをESC/POSなどの命令に変換することが出来ます。

**receiptlineの例**

左がreceiptlineで、右がそのレンダリング結果(SVG)です。

![](https://pbs.twimg.com/media/FRw5n--aIAAMjm9?format=jpg&name=large)

receiptline designer と言う公式のエディタで簡単に出力結果を確認することが出来ます。

https://www.ofsc.or.jp/receiptline_/designer/index.html

これを使って、自由にSUNMI端末用のレシートを記述出来るのではと思いましたが、結論として、うまくいきませんでした。

#### receiptlineがうまく使えなかった理由

これは、単に自分の知識不足でしたが、**ESC/POSには数多くの方言があります**。
[Wikipedia](https://ja.wikipedia.org/wiki/ESC/P#%E3%83%90%E3%83%AA%E3%82%A8%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3)にも、いくつかのバリエーションについての言及があります。

下記ツイートで、receiptlineの提供するESC/POS方言の `escpos` モードと `impact` モードでの出力結果を使って印刷した画像を示していますが、いずれも期待通りの表示になっていないことがわかります。

https://twitter.com/__syumai/status/1521154895919087616

期待通りの印刷を可能にするには、SUNMIの提供するWordファイルを参照しつつ、receiptline本体をいじって、**SUNMI用のモードを追加する必要がありそう**でした。

そこまでのやる気が湧かなかったため、今回はそこまでは実装しませんでした。

ただ、receiptlineを受け取ってSUNMI端末のプリンタで印刷するReact Nativeアプリ自体は実装できたので、SUNMIモードの実装が出来たらいつかここに入れたいです。

https://github.com/syumai/sunmi-receiptline-printer

## 画像を印刷する

画像の印刷には `printBitmap` および、 `printBitmapCustom` メソッドが使うことが出来ます。

```java
void printBitmap(Bitmap bitmap, ICallback callback)
void printBitmapCustom(Bitmap bitmap, int type, ICallbackcallback)
```

ただし、当然ですがSUNMI端末のプリンタは白と黒の2色しか印刷できないため、printBitmapを使うと、画像が白黒2値で表示されてしまい、何が写っているのか全くわからなくなってしまいます。

`printBitmapCustom` を使うと、白黒2値のモードと別に、グレースケールモードが選択出来るそうなので、もしかしたらこちらでうまく印刷出来るかもしれません（まだ試していないです）

他には、自前で[ディザリング](https://ja.wikipedia.org/wiki/%E3%83%87%E3%82%A3%E3%82%B6)を行って、画像を白と黒の2色に減色しておくと言う方法があります。
ディザリングの有名なアルゴリズムにFloyd-Steinbergと言うものがあり、これのJavaScript実装がありました。

https://github.com/noopkat/floyd-steinberg

これをWrapした簡単なCLIツールを用意しておいたので、手元でサクッと白黒画像を作りたいと言う方はこちらをぜひ使ってみてください。（自分のアプリではこれを使いました）

https://github.com/syumai/ditherer

## 最後に

以上、SUNMI端末用のアプリを作る過程でわかった情報などについて書いてみました。

いいアプリが出来たら、SUNMIのストアにpublishも出来るそうなのでやってみても良さそうです。
自分もいつかやってみたいと思っています。

何かアプリが出来たら、気になるのでぜひここのコメント欄やTwitter ([@__syumai](https://twitter.com/__syumai))で教えてください！
