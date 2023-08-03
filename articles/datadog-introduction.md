---
title: "Datadogでアプリケーションのログを監視できるようにする"
emoji: "🐶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['datadog']
published: false
publication_name: "spacemarket"
---

こんにちは。
[株式会社スペースマーケット](https://www.spacemarket.com/)でフロントエンドエンジニアをしているmizukiです。

## はじめに

エンジニアとして仕事をしていると、コードを書くことが多くてDatadogのようなツール系のキャッチアップがどうしても後回しになってませんでしょうか？？
何を隠そう、私自身がそうで、Datadogに限らずこういうったツール系には少し苦手意識を持っていました・・・

スペースマーケットでは技術負債の解消やメンバーの成長を目的とした「チャプター活動」というものをおこなっておりまして、そのチャプター活動の中でこういったツール系をみんなが使えるようになろうといった目的のもと、Datadogについての勉強会を行いました。
そこで学んだDatadogでのログの見方をまとめていこうと思います！

## そもそもDatadogとは？？

Datadogとは、一言でいうと「アプリケーションの運用監視システム」です。
アプリケーションにアクセスされたログや、どのくらいサーバーに負荷がかかっているかなど、アプリケーションを運用していく上で必要になる情報を収集・監視できるサービスです。

## いろいろな指標の確認方法

Datadogをうまく駆使すれば色々なデータやログを確認できるようになるのですが、その中でも
・平均応答時間
・平均応答時間（GoogleBot）
・リクエスト数
という3つの指標の確認方法を紹介していこうと思います。

### ①平均応答時間

あるURLにアクセスがあった際の、平均応答時間を見る方法です。

まずは左のメニューにある「Logs」から「Search」をクリックします。

![](/images/datadog-introduction/menu_logs.png)

次に検索条件を指定します。
ここではstatusを200のみに絞ったり、監視したいページのURLを絞ったりします。
（※ここではテストとして `/hoge` というURLにしてますが、適宜置き換えてください）

ここでは平均応答時間を確認したいので、「Visualize as」で `Timeseries` 選択し、「Show」で `http.request_time_ms` を選択します。

![](/images/datadog-introduction/search_response.png)

そうすると下にグラフが表示されるので、ここから確認できます！

例）
![](/images/datadog-introduction/graph_response.png)


### ②平均応答時間（GoogleBot）

ここでは①の派生系として、GoogleBotからのアクセスを監視する方法を紹介します。
と言っても簡単なのですが、①で説明した検索条件の中で、userEgentをGoogleBotのものに絞って検索するだけです。

![](/images/datadog-introduction/search_response_googlebot.png)

```
@http.useragent:(*Googlebot* OR *AdsBot-Google* OR *Mediapartners-Google* OR *APIs-Google* OR *FeedFetcher-Google* OR *Google-Read-Aloud* OR *DuplexWeb-Google* OR *googleweblight* OR *Storebot-Google*)
```

SEO観点からGoogleBotの監視をしたい場合でも、こちらから監視ができます。

### ③リクエスト数

次にリクエスト数を監視する方法を紹介します。

検索条件は①と同じですが、「Show」の部分を `all logs` に設定します。
その右にある「roll up every」では監視したい期間を選択します。（今回は1日で見ていきます）

![](/images/datadog-introduction/search_request.png)

そうすると下のグラフから1日あたりのリクエスト数が確認できます。

例）
![](/images/datadog-introduction/graph_request.png)

## おわりに

今回はDatadogのログの見方の中でも初歩の初歩となる内容を解説しました。
Datadogはこれ以外にもたくさん機能があり、ログを確認する以外にもさまざまな活用方法があります。
エンジニアとしてアプリケーションをスケールさせて改善してくために、もっと使いこなせるようになりたいところです。

## 最後に宣伝

スペースマーケットでは一緒に働く仲間を募集しています！
カジュアルに話を聞きたいだけという方でも大歓迎ですので、ちょっとでも興味があれば以下からご応募お待ちしております！

**▼SRE/インフラエンジニア**
https://www.wantedly.com/projects/1113570

**▼バックエンドエンジニア**
https://www.wantedly.com/projects/1113544

**▼Androidエンジニア（iOSも大歓迎です！）**
https://www.wantedly.com/projects/1061116

**▼エンジニア採用ページ（迷ったらこちらからどうぞ！）**
https://spacemarket.co.jp/recruit/engineer/

