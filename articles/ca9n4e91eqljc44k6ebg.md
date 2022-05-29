---
title: "Cloudflare WorkersでGoのHTTPサーバーを動かすためのライブラリを作った"
emoji: "️⛈️"
type: "tech"
topics: ["Go", "Cloudflare Workers"]
published: false
---

Cloudflare Workersで簡単にGoのHTTPサーバーを動かすためのライブラリを作ったので、こちらの紹介をさせていただきます。(まだかなり実験的な実装です)

https://github.com/syumai/workers

## 特徴

* http.Handlerを作って、 `worker.Serve` に渡すだけでCloudflare Workers上でHTTPサーバーとして動作する
* 必要なツールは[tinygo](https://tinygo.org/)と[wrangler](https://developers.cloudflare.com/workers/wrangler/) (Cloudflare WorkersのCLI) だけ
* JavaScript側のコードを触る必要が無い
* Cloudflare R2のバインディング（一部）を提供している
  - 今後、KV等の対応も頑張る予定

## デモ

下記のURLにアクセスすると `Hello, world!` と表示され、URL末尾の `world` を `syumai` に差し替えると、 `Hello, syumai!` と表示されます。

https://hello.syumai.workers.dev/?name=world

[ソースコード](https://github.com/syumai/workers/tree/5c55a2ebf7f7a4149dcd88474c71cd0a1c1b371a/examples/hello)は、以下のような非常にシンプルな内容となっています。

```go
func main() {
	handler := http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		name := req.URL.Query().Get("name")
		if name == "" {
			name = "world"
		}
		fmt.Fprintf(w, "Hello, %s!", name)
	})
	workers.Serve(handler)
}
```

`workers.Serve(handler)` の部分が、本ライブラリの提供している機能となります。

handler変数は、通常のhttp.HandlerFuncと同じように実装しているだけです。

## 作った動機

以前Cloudflare Workersで動作する[tinygoのサンプル](https://github.com/syumai/workers-playground/tree/main/tinygo-add)を作ってみましたが、短時間で実用的なものを作るのは大変そうな感じがしました。
Worker自体HTTPリクエストを処理するための機構ですし、サクッと実用的なアプリケーションを作るには、やはりhttp.Handlerをサポートするのが一番の早道だなと思ったのでこの実装方針にしました。

あとは、WebAssemblyとJavaScriptの知識が要求される感じがあり、Goプログラマにとっては少しハードルが高いかもしれないと思ったので、なるべくJavaScript側の世界を意識しないで使えるライブラリが欲しいと思っていました。
この辺りは、Cloudflare WorkersのRustバインディングの[workers-rs](https://github.com/cloudflare/workers-rs)がうまくやっているので意識しつつ実装しています。

## 使い方

先ほどのデモで紹介した通りですが、

ただ `http.Handler` を実装して、 `workers.Serve` に渡すだけで終わりです。

```go
func main() {
	var handler http.HandlerFunc = func (w http.ResponseWriter, req *http.Request) { ... }
	workers.Serve(handler)
}
```

または、 `http.Handle` や `http.HandleFunc` を呼び出して、 `workers.Serve` に `nil` を渡して呼び出してもOKです。
この場合は、デフォルトの挙動として `http.DefaultServeMux` が使われます。

```go
func main() {
	http.HandleFunc("/hello", func (w http.ResponseWriter, req *http.Request) { ... })
	workers.Serve(nil)
}
```

実際に動かすには、WebAssemblyをロードするJavaScriptのコード等も必要になります。
それらファイルをまとめた、**tinygoでCloudflare Workersのアプリケーションを作るためのテンプレートリポジトリ**を用意しているので、簡単に始めたい方はこちらをご利用ください。

https://github.com/syumai/worker-template-tinygo

## ライブラリの実装について

実装は単純で、

* JavaScript側で受け取ったRequestオブジェクトをGoに渡す
* Go側でResponseオブジェクトを組み立ててJavaScript側に渡す

と言う形式になっています。

Cloudflare Workersは、fetch eventをハンドリングして、eventに付与されたRequestオブジェクトを処理し、Responseオブジェクトを返すと言う構造になっています。
このとき問題になるのは、上記で示した内容から推測できる通り、**JavaScriptとGo間での値の変換**です。これは基本的には頑張るしかないので、値を変換するコードを大量に書いています。

あとは、方針として、**出来る限りデータをメモリ上に持たない**ようにする形で進めているので、JavaScriptのReadableStreamとGoのio.Readerを相互に変換するような実装が含まれています。

### JavaScript側からGo側に値を渡す箇所について

JavaScript側からGo側に値を渡す箇所については、Go側からグローバルオブジェクトに `handleRequest` と言う関数を登録するようにしました。

https://github.com/syumai/workers/blob/v0.2.0/handler.go#L13-L33

JavaScript側から、この関数に対する呼び出しを行う箇所が、リクエストの処理開始地点となります。

https://github.com/syumai/workers/blob/v0.2.0/examples/simple-json-server/worker.mjs#L15

### ストリームの変換

ストリームの変換は、[deno_stdのコード](https://deno.land/std@0.139.0/streams/conversion.ts)を参考にしました。
DenoのReader / WriterはGoのio packageを参考に作られたもので、近年のWeb APIへの追従に伴って、WHATWG Streamとの相互変換が必要なシーンが増えていました。
そのため、上記リンクの `conversion.ts` と言うファイルが作られていたのですが、この実装をほとんど移植するだけで完了しました。

とは言え、syscall/jsでラップする必要があり、Promiseの扱いなどはやや面倒でした。

[Streamの変換処理の実装はこの辺りのコード](https://github.com/syumai/workers/blob/v0.2.0/stream.go)で行っています。

ところどころで、Go側からJSのPromiseをawaitするような処理が入っていたりします。
チャネルを使ってPromiseの非同期処理の結果を待ち受ける実装になっていて、streamの処理以外でも必要になったのでユーティリティ関数として切り出しています。

https://github.com/syumai/workers/blob/v0.2.0/jsutil.go#L39-L62

### ResponseWriterのReadableStreamへの変換

これが地味に迷ったポイントだったのですが、

* Goのレスポンス形式は、ResponseWriterへの書き込み処理
* JS側のレスポンス形式は、ReadableStreamからの読み込み処理

と言うことで、レスポンスの形式として**書き込みと読み込みが逆転していました**。

これで1日くらいぼんやり悩んでいたのですが、 `io.Pipe` が問題を解決することに気付いてからの実装はすぐでした。

https://twitter.com/__syumai/status/1526456929174048768?s=20&t=qoY_AS-0dOaSC4zXKhdBKw

responseWriterの実体に、io.Pipeから入手したreaderとwriterの両方を持たせ、Go側からはこのwriterへの書き込みを行い、

https://github.com/syumai/workers/blob/v0.2.0/handler.go#L45

JS側ではreaderからの読み込みを行うようにしました。

https://github.com/syumai/workers/blob/v0.2.0/response.go#L28

これで、Go側とJS側でレスポンスをストリーム処理することに成功しました。

## 小話

### tinygoじゃないとダメなのか？

手元の `wrangler dev` では通常のGoでも動作することを確認できましたが、[**圧縮後のサイズが1MB以内**](https://developers.cloudflare.com/workers/platform/limits/#worker-size)でないといけない制約があるため、バイナリサイズが大きくなる通常のGoではpublishに成功しませんでした。
実はtinygoでもギリギリで、依存ライブラリを増やすと簡単に越えます。
なるべく依存を増やさずに、質素なアプリケーションを作っていく必要はありそうです。

### tinygoで困ったことはないか？

tinygoと言う制約による悩みは結構多かったです。

例えば、`encoding/json`が動きません。これはtinygoのreflectの実装が完全でないためで、代替のライブラリを探す必要があります。(takasagoさんにツイートで代替をいくつか教えていただきました)

https://twitter.com/sago35tk/status/1526538044073193472?s=20&t=qoY_AS-0dOaSC4zXKhdBKw

下記に、easyjsonを使ったサンプルを示しますが、構造体の定義を変更する度にコード生成をする必要があるので少々手間です。
とは言え、ちょっとした用途なら全然使えそうな感じはしました。

https://github.com/syumai/workers/tree/main/examples/simple-json-server

あとは、`(*http.Client).Do` などがうまく動きませんでした(自分のやり方が悪いのか…？)。ベーシック認証をかけたプロキシサーバーを実装してみようと思ったのですが、外部にHTTPリクエストを送れなかったので断念しました。
これは制約としてかなりキツいです。

ひとまずベーシック認証が動作するだけのWorkerのサンプルは出来ましたが、あんまり実用性は無いような感じがします。

https://github.com/syumai/workers/blob/v0.2.0/examples/basic-auth-server

### 現状 Cloudflare R2 はどれくらい使えるのか？

一応、head / get / put / delete / listの全てを呼べます。ただし、オプションはまだほとんど対応していません。

どれくらい動くかについては、実際に動作するサンプルを2つ用意しているので、こちらを見ていただくのが早いです。

`r2-image-viewer` では、画像をR2から取得して返すだけの実装を行なっています。

https://github.com/syumai/workers/tree/v0.2.0/examples/r2-image-viewer

デモ: https://r2-image-viewer-tinygo.syumai.workers.dev/syumai.png

wrangler.tomlで指定したbucketName (下記) を指定する必要はありますが、Goのみで実装出来ていることが確認いただけると思います。

https://github.com/syumai/workers/blob/v0.2.0/examples/r2-image-viewer/main.go#L13-L14

[R2ObjectのBody](https://pkg.go.dev/github.com/syumai/workers@v0.2.0#R2Object)は、io.Readerを実装しているので、http.ResponseWriterにio.Copyすることが出来ます。こうした部分で、ややGoらしさを意識した実装となっています。

```go
	bucket, err := workers.NewR2Bucket(bucketName)
	if err != nil { ... }
	imgPath := strings.TrimPrefix(req.URL.Path, "/")
	imgObj, err := bucket.Get(imgPath)
	if err != nil { ... }
  io.Copy(w, imgObj.Body)
```

また、 `r2-image-server` には、画像のR2バケットへのput / get / deleteを行う実装が含まれています。

https://github.com/syumai/workers/tree/v0.2.0/examples/r2-image-server

こちらはWeb上のデモは用意していませんが、お手元で簡単に確認いただけますので、ぜひ動かしてみてください。　

putの処理は、ReadableStreamの実装ではPromiseが永遠に解決されない状況に陥ってしまい、うまく動かせなかったため、現状メモリに全データをロードする実装になっています。こちらは早く直したいですが、かなりの格闘の末に失敗したので、ちょっと期間を置いてからチャレンジしようと思っています。

## おわりに

以上、まだまだ開発中のライブラリではありますが、試しに遊んでみてください！

何か作ってみた方がいらっしゃったら、ここのコメントやTwitterで教えていただけると嬉しいです！ぜひよろしくお願いします。
