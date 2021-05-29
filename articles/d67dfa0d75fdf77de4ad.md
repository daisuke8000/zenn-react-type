---
title: "駆け出しキラー、CORSを突破して幸せになる"
emoji: "🍓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['React','TypeScript']
published: false
---


## 未知（CORS）との遭遇

Reactでなんかアプリ作れるかなとこねくり回してるうち、TwitterのAPIを叩いたらこんなんでた。
(xxxxxxxxxxは自分のTwitterID)

```bash
Access to XMLHttpRequest at 'https://api.twitter.com/1.1/statuses/user_timeline.json?count=10&screen_name=@xxxxxxxxxx' from origin 'http://localhost:3000' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

「おぉ、CORSってやつか。」

普段は外部API叩くときはサーバーサイドからやってることもあって出くわしたことなかった。
そういえば、クライアント側から外部API叩くのはJsonPlaceHolderくらいだった気がする。。

まぁ、いい機会なのでちょっと調べてみる。
と思ったらなんかすごい良い感じでまとまってる記事があったのでそっちを貼る。
https://qiita.com/att55/items/2154a8aad8bf1409db2b

## Cross-Origin Resource Sharing(オリジン間リソース共有)がなんで発生するのか

そもそもCORSっていうのはブラウザに関するポリシーだということだ。

そして、これは先程の記事に書いてあるけど、兎にも角にもオリジンを理解しなきゃいけない。
オリジンっていうのは

「Protocol」
「Host」
「Port番号」

この３つすべてが一意のものを指すらしい。

つまり、今回の俺の環境では
「Protocol」 -> http://
「Host」 -> localhost
「Port番号」 -> :3000

-> `https://www.localhost:3000`
だったわけだ。

今回ブラウザからTwiiterへのリクエストをおこなったわけだが、リクエストするにあたり、つまりこういう流れだったわけだ。

1. リクエスト投げる（http//localhost:3000っていうオリジンがリクエストヘッダに含まれる）
2. レスポンスが返ってくる（レスポンスヘッダにはサーバーが許可するオリジンが含まれてる）
3. リクエスト時のオリジンとレスポンス時のオリジンが一致していない
4. ぜんぜん違うオリジンじゃねーかテメー

なるほど。ごめんなさい。

詳しくは前述のリンクなりW3C見てね（実際は単純リクエストとかプリフライトリクエストとかあるから。今回のは単純リクエスト）
事前にcurlでリクエスト投げたらなんなく取得できたから何かしらの仕組みなんだろうなとは思ったけど、そういうルールなんだな。

## それじゃ、解決しようか。

王道というかベストプラクティス的なことで言うなら多分、サーバーサイド側で処理してしまうのが正解かと。俺もそうしたいけど、今はサーバーサイドのAPIはまだ書いてないので、とりあえず以下のどっちかで検討した。
なお、今回はAxiosを使用する。

1. Chromeの拡張機能orSafariでクロスオリジンの無効化
2. ダミーAPI

テストなら１でも良かったが、ブラウザの設定をいじるのはあまり気が進まなかったので２で行うこととした。
ていうか、公式にあった。
[Create React App](https://create-react-app.dev/docs/proxying-api-requests-in-development/#configuring-the-proxy-manually)

srcディレクトリの配下にsetupProxy.jsを作成して、サンプルを参考に以下の内容を記述する。

```js
const proxy = require('http-proxy-middleware')
module.exports = function(app) {
    const headers  = {
        "Content-Type": "application/json",
        Authorization: process.env.REACT_APP_TWITTER_API_BEARERTOKEN,
    }
    //proxyの第一引数はドメイン以下の部分
    //第二引数のtarget部はドメイン
    app.use(proxy(process.env.REACT_APP_TWITTER_BASE_CONTEXT, { target: process.env.REACT_APP_BASE_URL,changeOrigin: true,secure: false,headers: headers}));
};
```

API投げるとこはとりあえずこんな感じで書いとく。

```ts
const onClickFetchData = () => {
    axios.get<Array<TweetType>>(process.env.REACT_APP_TWITTER_BASE_URL + 
        "?count=1&screen_name=@xxxxxxxxx" , {
            headers: {
                "Content-Type": "application/json",
                Authorization: "Bearer " + process.env.REACT_APP_TWITTER_API_BEARERTOKEN,
            }
        })
        .then((res) => {
            console.log(res.data);
            setTweets(res.data);
        })
};
```


