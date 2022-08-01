---
marp: true
theme: gaia
backgroundColor: '#fff'
---


<style>
  section {
    font-family: "Helvetica Neue", Arial, "Hiragino Kaku Gothic ProN",
      "Hiragino Sans", Meiryo, sans-serif;
    padding: 50px;
  }
</style>

<style scoped>
  section {
    display: flex;
    flex-direction: column;
    justify-content: center;
  }
</style>

# Cloudflare WorkersでGoのHTTPサーバーを動かすライブラリを作った話

### syumai

#### DP Engineering Monday (2022/8/1)

---

![bg right:30% 80%](https://avatars.githubusercontent.com/u/6882878?v=4)

# 自己紹介

## syumai

* Go Documentation 輪読会 / ECMAScript 仕様輪読会 主催
* 株式会社ベースマキナ所属
* GoでGraphQLサーバー (gqlgen) や TypeScriptでフロントエンドを書いています

Twitter: [@__syumai](https://twitter.com/__syumai)
Website: https://syum.ai

---

## 話すこと

* Cloudflare Workers とは？
* syumai/workers の紹介
* 実装テクニックの紹介
  - syscall/jsの処理など
* 現状の課題 
* example集の紹介

---

## Cloudflare Workers とは？

* Service Worker APIベースのJavaScriptのWorker
  - WebAssemblyも実行可能
* Cloudflare経由のリクエストに割り込んで処理を行いレスポンスを返せる
* 裏側にトラフィックを流すことなく、Workerが単独でレスポンスを組み立てて返してもOK

https://blog.cloudflare.com/ja-jp/cloudflare-workers-unleashed-ja-jp/#worker

---

## Worker の例

Requestを受け取って、Responseを返す関数を書く形式

```js
addEventListener("fetch", (event) => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  return new Response("Hello world");
}
```

https://developers.cloudflare.com/workers/ より引用

---

## Cloudflare Workers の機能

* KV
  - 分散 key / value store
  - 結果整合 (複数edgeからの同一キーへの書き込みは後勝ち)
* Durable Objects
  - edge間でのデータ同期に使えるObject
  - 強整合
* Cache API
  - ブラウザのキャッシュAPIのようなもの
  - edgeのキャッシュを操作出来る

---

## Cloudflare Workers の用途の広がり

最近発表された新機能 (まだ使えないものもあります)

* R2
  - S3 互換のオブジェクトストレージ
* D1
  - SQLiteベースの分散データベース
* Pub/Sub
  - MQTT互換のメッセージング基盤

=> Cloudflare上でフルスタックアプリケーションを作れる環境に近付いている

---

<!-- _class: lead -->

# syumai/workers の紹介

---

## workers

https://github.com/syumai/workers

* http.Handlerを作って、 `workers.Serve` に渡すだけでCloudflare Workers上でHTTPサーバーとして動作する
* 必要なツールは[tinygo](https://tinygo.org/)と[wrangler](https://developers.cloudflare.com/workers/wrangler/) (Cloudflare WorkersのCLI) だけ
  - tinygoを使ってWebAssemblyを生成して実行する
* JavaScript側のコードを触る必要が無い
* Cloudflareの機能のバインディングを提供 (R2, KV)

---

## workersを使ったコードのサンプル

普通に `http.HandlerFunc` を作るだけ

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

<small>
https://hello.syumai.workers.dev/?name=syumai (=> Hello, syumai と表示) 
</small>

---

## 作ったモチベーション

* WebAssemblyとJavaScriptの知識が無くてもGoでWorkerを書けるようにしたかった
  - 最終的に、templateのリポジトリをコピーして、Goのコードとwrangler.tomlを編集するだけで済む形になった
  - https://github.com/syumai/worker-template-tinygo
    - entrypointとしてJSのコードを含んでいるが、修正は不要
* 実用的なWorkerを短時間で実装できるようなライブラリが欲しかった
  - http.Handlerのサポートで実現

---

## workers packageを使わなかった場合の例

* GoとJSの2ファイルでそれぞれ実装が必要

https://github.com/syumai/workers-playground/tree/main/tinygo-add

---


* Go側は、処理のexportが必要
  - export commentによる関数exportはtinygoの機能

```go
package main

//export add
func add(a, b int) int {
	return a + b
}

func main() {}
```


---

* JS側は、Goインスタンスの初期化、Wasmのロード、Workerの処理の実装が必要

```js
const go = new Go();

const load = WebAssembly.instantiate(mod, go.importObject).then((instance) => { ... });

export default {
  async fetch(req) {
    const instance = await load;
    const url = new URL(req.url);
    const a = url.searchParams.get("a");
    const b = url.searchParams.get("b");
    if (a === null || b === null) {
      return new Response("invalid", { status: 400 })
    }
    const result = `${a} + ${b} = ${instance.exports.add(a, b)}`;
    return new Response(result);
  },
};
```

https://github.com/syumai/workers-playground/tree/main/tinygo-add

---

<!-- _class: lead -->

# 実装テクニックの紹介

---

## syumai/workersの基本的な実装形式

Cloudflare Workersは、基本的にRequestオブジェクトを処理し、Responseオブジェクトを返すと言う構造になっているため、下記が実装できればOK

* JavaScript側で受け取ったRequestオブジェクトをGoに渡す
* Go側でResponseオブジェクトを組み立ててJavaScript側に渡す

これをGoの標準ライブラリの[syscall/js](https://pkg.go.dev/syscall/js)を使って実装した

---

## 紹介するもの

* 基本的な値の変換
* バイト列の変換
* Promiseの待ち受け
* ストリームの変換

---

## JSからGoへの基本的な値の変換

* Goのコード上で、JavaScript側の値は、全て `js.Value` 型で扱われる
* js.Valueのメソッドを使って値を変換、操作出来る
  - Goの基本的な型の値への変換 => Int()、String()など
  - ObjectやArrayなど、複雑なObjectの操作 => Index()、Set()、Get()
  - JavaScript側のメソッド、関数の呼び出し => Call()、Invoke()

---

## GoからJSへの基本的な値の変換

* 基本的に、ValueOfのルールに従って自動的に行われるため、実はあまり考えなくていい
  - JavaScript側のnumber型は浮動小数点数型なので精度に注意

```
| Go                     | JavaScript             |
| ---------------------- | ---------------------- |
| js.Value               | [its value]            |
| js.Func                | function               |
| nil                    | null                   |
| bool                   | boolean                |
| integers and floats    | number                 |
| string                 | string                 |
| []interface{}          | new array              |
| map[string]interface{} | new object             |
```

---

## バイト列の変換

* syscall/js の CopyBytesToGo / CopyBytesToJS 関数を使う
* Uint8Array と []byte で相互にデータのコピーが可能

例

```go
func (kv *kvNamespace) PutReader(key string, value io.Reader, opts *KVNamespacePutOptions) error {
	b, _ := io.ReadAll(value)
	ua := newUint8Array(len(b))
	js.CopyBytesToJS(ua, b)
	p := kv.instance.Call("put", key, ua.Get("buffer"), opts.toJS())
	return awaitPromise(p)
}
```

---

### GoからJS側に値をコピーする時のテクニック

* コピー先のUint8ArrayがJavaScript側に無い時は、Go側から作る必要がある
* (Value).New を使ってclassのインスタンスを生成できるのでこれを使う

```go
var uint8ArrayClass = global.Get("Uint8Array")

func newUint8Array(size int) js.Value {
	return uint8ArrayClass.New(size)
}
```
https://github.com/syumai/workers/blob/v0.3.0/jsutil.go

---

## ストリームの変換


* JavaScript側のコードがReadableStreamを返した時、これをio.Readerとして扱えるようにしたかった
  - 逆もしかり
* [deno_stdのコード](https://deno.land/std@0.139.0/streams/conversion.ts)を参考に実装した
  - https://github.com/syumai/workers/blob/v0.3.0/stream.go
* 後から気付いたが、Goの標準ライブラリにも同様の実装があったのでこれを使っても良かったかもしれない
  - [net/http/roudtrip_js.go](https://github.com/golang/go/blob/go1.18.4/src/net/http/roundtrip_js.go#L209)

---

## Promiseの待ち受け

* Goから呼んだJavaScript側の関数がPromiseを返す時、Go側のコードがブロックされない
* Promiseを待ち受けるための処理はsyscall/jsでは提供されていない
* 自前でchannelを使って書く必要がある

---

* thenとcatchを呼んでchannelに送信、select文で待ち受けreturn

```go
func awaitPromise(promiseVal js.Value) (js.Value, error) {
	resultCh := make(chan js.Value)
	errCh := make(chan error)
	var then, catch js.Func
	then = js.FuncOf(func(_ js.Value, args []js.Value) any {
		defer then.Release()
		result := args[0]
		resultCh <- result
		return js.Undefined()
	})
	catch = js.FuncOf(func(_ js.Value, args []js.Value) any {
		defer catch.Release()
		result := args[0]
		errCh <- fmt.Errorf("failed on promise: %s", result.Call("toString").String())
		return js.Undefined()
	})
	promiseVal.Call("then", then).Call("catch", catch)
	select {
	case result := <-resultCh:
		return result, nil
	case err := <-errCh:
		return js.Value{}, err
	}
}
```

---

<!-- _class: lead -->

# 現状の課題

---

## ファイルサイズ制限

* [**圧縮後のサイズが1MB以内**](https://developers.cloudflare.com/workers/platform/limits/#worker-size)でないといけない制約があるため、バイナリサイズが大きくなる通常のGoではpublishに失敗した
* 実はtinygoでもギリギリで、依存ライブラリを増やすと簡単に越える

---

## tinygoでencoding/jsonが動かない

* tinygoのreflectの実装が完全でないため、encoding/jsonが動かない
* easyjsonなどの別のJSON encoder / decoderを使う必要がある

---

## tinygoでnet/httpのHTTP Clientが動かない

* tinygoを使った場合、Worker上でhttp.Getなどが動かないため、プロキシ用途で使うことが出来ない
* 一応回避策を見つけて、ベーシック認証プロキシを作ることに成功した
  - https://github.com/syumai/workers/tree/v0.3.0/examples/basic-auth-proxy
  - tinygo 0.24.0で動かなくなってしまったが、takasagoさんの修正で0.25.0でまた動くようになるはず
    - https://github.com/tinygo-org/tinygo/pull/3036

---

## example集

* [JSON APIサーバー](https://github.com/syumai/workers/tree/v0.3.0/examples/simple-json-server)
* [R2を使った画像アップロード / 配信サーバー](https://github.com/syumai/workers/tree/v0.3.0/examples/r2-image-server)
* [KVを使ったアクセスカウンター](https://github.com/syumai/workers/tree/v0.3.0/examples/kv-counter)

ぜひ、templateリポジトリからWorkerを作ってpublishしてみてください

https://github.com/syumai/worker-template-tinygo

---

## 発表内容について

Zennの方にも記事を投稿しているので、興味があればぜひご覧ください

* Cloudflare Workersで簡単にGoのHTTPサーバーを動かすためのライブラリを作った
  - https://zenn.dev/syumai/articles/ca9n4e91eqljc44k6ebg

---

<!-- _class: lead -->

## ご清聴ありがとうございました！
