---
title: "AIチャットのストリーミング機能を実装する上でハマったこと"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "ChatGPT", "ストリーミング"]
published: false
publication_name: "irusiru"
---

こんにちは！
イルシルでEMをしているmizukiです。
EMを名乗ってはいますが、スタートアップで人手が足りない状況なので実態としては一人前の開発タスクをこなしており、本当にEMと言えるのかどうかが不明な今日この頃です😇

今回はイルシル( https://app.irusiru.jp/ )にあるAIチャット実装にストリーミング機能を実装した際にハマったことをメモがてら書いていこうと思います。
デバッグしながらでも気づけるとは思いますが、事前に把握できていればもっと効率よく開発できるかなと思うポイントをまとめていくので、今後実装する誰かの参考になればいいなと思います。

## 実装した機能

イルシルでは「AIチャット」という機能が存在します。（※まだβ版です）

![AIチャットβ版](https://storage.googleapis.com/zenn-user-upload/9a19d9312ce0-20241229.png)

通常のチャット機能はもちろんのこと、現在はチャット完結でスライドをAI生成もできてすごく便利なので気になった方は使ってみてください！
※3スライドまでなら無料で作成できます

今回はこのAIチャット機能にストリーミング機能を実装した時のことについて書いていきたいと思います。

イルシルでは `FE ⇔ Lambda ⇔ AI（ChatGPTやClaudeのAPI）` といった形で、Lambdaを経由してフロントからAIにリクエストをしています。
基本はAIからストリーミングでデータを受け取る時にハマったことを書いていきますが、最後にLambdaの設定でもハマったことを書いていきます。

## ハマったこと

### ①JSONデータの型が結構複雑

今回はChatGPTのストリーミングを例に書いていきます。
ChatGPTをAPIで使用する際にストリーミングでデータを受け取ると、1つのchunkの中に以下の形式でデータが格納されています。

:::details 長いので閉じています、適宜開閉してご確認ください
```
data: 
{
  "choices": [
    {
      "content_filter_results": {
        "hate": {
          "filtered": false,
          "severity": "safe"
        },
        "self_harm": {
          "filtered": false,
          "severity": "safe"
        },
        "sexual": {
          "filtered": false,
          "severity": "safe"
        },
        "violence": {
          "filtered": false,
          "severity": "safe"
        }
      },
      "delta": {
        "content": "こんにちは"
      },
      "finish_reason": null,
      "index": 0,
      "logprobs": null
    }
  ],
  "created": 1735438890,
  "id": "chatcmpl-XXXXXXXXXXXXXXXXXXX",
  "model": "gpt-XXXXXXXXX",
  "object": "chat.completion.chunk",
  "system_fingerprint": "fp_5154047bf2",
  "usage": null
}
```
:::

全てのchunkに、フィルタリング結果の情報やモデル名、生成日時などなどが入っています。
そのため、一番欲しいデータであるチャットのデータは、 `data.choices[0].delta.content` という深い階層に入っています。

じゃあちょっと長くなるけどその記述で取り出せばいいのか！となるかもしれませんが、そうは問屋が卸しません。
なぜかというと、返ってくるデータが以下の形式の場合があるからです。

**1. deltaが空オブジェクトでcontentというキーが存在しない**

:::details データ例
```
data: 
{
  "choices": [
    {
      "content_filter_results": {},
      "delta": {},
      "finish_reason": "stop",
      "index": 0,
      "logprobs": null
    }
  ],
  "created": 1735438890,
  "id": "chatcmpl-XXXXXXXXXXXXXXXXXXX",
  "model": "gpt-XXXXXXXXX",
  "object": "chat.completion.chunk",
  "system_fingerprint": "fp_5154047bf2",
  "usage": null
}
```
:::

これはチャットのテキストデータを全て返し終わったタイミングで返ってきます。
`"finish_reason": "stop",` を伝えるためかなと推測されます。

**2. choicesが空配列**

:::details データ例
```
data:
{
  "choices": [],
  "created": 1735438890,
  "id": "chatcmpl-XXXXXXXXXXXXXXXXXXX",
  "model": "gpt-XXXXXXXXX",
  "object": "chat.completion.chunk",
  "system_fingerprint": "fp_5154047bf2",
  "usage": {
    "completion_tokens": 14,
    "completion_tokens_details": {
      "accepted_prediction_tokens": 0,
      "audio_tokens": 0,
      "reasoning_tokens": 0,
      "rejected_prediction_tokens": 0
    },
    "prompt_tokens": 8,
    "prompt_tokens_details": {
      "audio_tokens": 0,
      "cached_tokens": 0
    },
    "total_tokens": 22
  }
}
```
:::

これはチャットデータが返し終わった後に返ってくるデータで、usageの中に消費token数のデータが入っています。
※消費token数はリクエストパラメータの中に`stream_options: { include_usage: true }`を含める必要があるため、これを省いた場合はこのデータは返却されないかと思います（筆者は未検証のためtoken数を取得しない場合の挙動は適宜お確かめください）

**3. 大元のdataがオブジェクトではなく`[done]`という文字列**

:::details データ例
```
data: [DONE]
```
:::

これは全てのストリーミング配信が終わった最後のレスポンスで返ってきます。
このデータを受信したらAIとのストリーミング配信は完全に終了です。

上記のように、形式がわかっても中身のデータが異なるので予期せぬエラーを引き起こす可能性があります。
あり得るデータ型のパターンに対して適切なハンドリングをする必要があります。

### ②1つのchunkの文字列データの中に複数のJSONが入ってる

先ほどは1つのJSONデータを紹介しましたが、必ずしも1度のストリーミングで1つのJSONが返ってくるわけではありません。
1つのストリーミングで返ってくるchunkの中に、複数のJSONが入っている場合もあります。

:::details こちらも長くなるので閉じています。適宜開閉してください
```
data: 
{
  "choices": [
    {
      "content_filter_results": {
        "hate": {
          "filtered": false,
          "severity": "safe"
        },
        "self_harm": {
          "filtered": false,
          "severity": "safe"
        },
        "sexual": {
          "filtered": false,
          "severity": "safe"
        },
        "violence": {
          "filtered": false,
          "severity": "safe"
        }
      },
      "delta": {
        "content": "こんにちは"
      },
      "finish_reason": null,
      "index": 0,
      "logprobs": null
    }
  ],
  "created": 1735438890,
  "id": "chatcmpl-XXXXXXXXXXXXXXXXXXX",
  "model": "gpt-XXXXXXXXX",
  "object": "chat.completion.chunk",
  "system_fingerprint": "fp_5154047bf2",
  "usage": null
}


data: 
{
  "choices": [
    {
      "content_filter_results": {
        "hate": {
          "filtered": false,
          "severity": "safe"
        },
        "self_harm": {
          "filtered": false,
          "severity": "safe"
        },
        "sexual": {
          "filtered": false,
          "severity": "safe"
        },
        "violence": {
          "filtered": false,
          "severity": "safe"
        }
      },
      "delta": {
        "content": "！"
      },
      "finish_reason": null,
      "index": 0,
      "logprobs": null
    }
  ],
  "created": 1735438890,
  "id": "chatcmpl-XXXXXXXXXXXXXXXXXXX",
  "model": "gpt-XXXXXXXXX",
  "object": "chat.completion.chunk",
  "system_fingerprint": "fp_5154047bf2",
  "usage": null
}


data: 
{
  "choices": [
    {
      "content_filter_results": {
        "hate": {
          "filtered": false,
          "severity": "safe"
        },
        "self_harm": {
          "filtered": false,
          "severity": "safe"
        },
        "sexual": {
          "filtered": false,
          "severity": "safe"
        },
        "violence": {
          "filtered": false,
          "severity": "safe"
        }
      },
      "delta": {
        "content": "今日は"
      },
      "finish_reason": null,
      "index": 0,
      "logprobs": null
    }
  ],
  "created": 1735438890,
  "id": "chatcmpl-XXXXXXXXXXXXXXXXXXX",
  "model": "gpt-XXXXXXXXX",
  "object": "chat.completion.chunk",
  "system_fingerprint": "fp_5154047bf2",
  "usage": null
}
```
:::

このように `data: {~データの中身~}` が1つのchunkに複数入っていることもあります。
そのため、データを受け取ったらまずは1つのJSONに分解するという処理が必要になります。
こちらは簡単で、受け取るデータは必ず `改行2つ分` だけ空いているため、`chunk.split("\n\n")` のように書くことで1つずつのJSONに分解することができます。

### ③欠損したJSONが含まれる

一番やっかいだったのがこいつです。
先ほどまではJSONが途中で切れることなく完全なデータでしたが、ストリーミングの性質上レスポンス自体が分裂してしまう可能性もあります。

:::details こちらも長いので閉じています。適宜開閉してください
1. 1つ目のchunk
```
data: 
{
  "choices": [
    {
      "content_filter_results": {
        "hate": {
          "filtered": false,
          "severity": "safe"
        },
        "self_harm": {
          "filtered": false,
          "severity": "safe"
        },
        "sexual": {
          "filtered": false,
          "severity": "safe"
        },
        "violence": {
          "filtered": false,
          "severity": "safe"
        }
      },
      "delta": {
        "content": "こんにちは"
      },
      "finish_reason": null,
      "index": 0,
      "logprobs": null
    }
  ],
  "created": 1735
```

2. 2つ目のchunk
```
438890,
  "id": "chatcmpl-XXXXXXXXXXXXXXXXXXX",
  "model": "gpt-XXXXXXXXX",
  "object": "chat.completion.chunk",
  "system_f
```

3. 3つ目のchunk
```
ingerprint": "fp_5154047bf2",
  "usage": null
}
```
:::

データ例を見てもらうとわかるように、3つのchunkが集まって初めて1つのJSONとなるパターンです。
このような場合は受け取ったデータがちゃんとしたJSONであるかどうかを判定し、欠損している場合は次のchunkと合体させて再度判定する、みたいな処理が必要となります。

僕は以下のように `JSON.parse()` が成功するかどうかで判定をしました。

```js
let incompleteChunk = "" // これはchunkを受け取るforなどのループ処理の外側で定義しています

// === 以下はchunkを受け取るforなどのループ処理内 ====

// 前回のchunkに欠損したデータがなければそのまま処理、あれば前回と結合して処理
const targetChunkStr = !incompleteChunk
  ? chunk
  : `${incompleteChunk}${chunk}`;

try {
  const targetDataJson = JSON.parse(targetChunkStr);
  incompleteChunk = ""; // parseに成功したら一時保存していたデータは初期化

  // ~chunkの処理~
} catch {
  // 欠損したデータだった場合は一時保存して次のループへ
  incompleteChunk = targetChunkStr;
}
```

このようにすると欠損したデータの場合は次回と結合し、完成したらchunkの処理に移ることができます。

### ④Lambdaの設定

最後にLambda設定についても色々調べつつ試行錯誤したので、その内容について書いていきます。

**1. Lambdaからのレスポンスをストリーミングにする方法**

Lambdaからのレスポンスをストリーミング配信するには、「API Gatewayを挟む」か「Lambdaの呼び出しモードをRESPONSE_STREAMにする」の2通りが主なやり方でした。
弊社はAPI Gatewayを使用していないため、後者の「Lambdaの呼び出しモードをRESPONSE_STREAMにする」を採用しました。

**2. Lambdaの呼び出しモードをRESPONSE_STREAMにする方法**

Lambdaの呼び出しモードをRESPONSE_STREAMにするには、「Nodejsのランタイムを使用する」か「カスタムランタイムでストリーミング可能な設定にする」の2択でした。
さすがにカスタムランタイムを1から作るメリットも余力もなかったのでNodejsのランタイムを選択しました。
[参考AWSドキュメント](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/configuration-response-streaming.html)

ちなみに弊社では元々pythonのランタイムを選択しており、このAIチャットも元々はpythonで書かれていたので全て書き直しになりました。。。

**3. Lambdaからのレスポンスも、複数のレスポンスが1つのchunkに含まれる**

これはおそらくLambdaの仕様かと思うのですが、複数のレスポンスをLambda側でよしなにマージしてフロントへ返却している挙動がありました。

例えば、Lambdaからのレスポンスを `text` というキーを持ったJSONにした場合、フロントで受け取ると
```
{"text": "こんにちは"}{"text": "!"}{"text": "今日は"}
```
という感じで複数のJSONデータが1つのレスポンスに含まれていました。

そのため、フロント側で分割する処理が必要になります。
僕はレスポンスのJSONの間に区切り文字を入れ、その区切り文字を用いてフロントで分割しています。
（これに関してはもっといいやり方がありそうだなと思ったりしています・・・）

## 終わりに

以上が、僕がAIチャット機能でストリーミングを実装した際にハマったことでした。
もし新たにストリーミング機能を実装したり、Lambdaでストリーミング設定をする人がいれば参考にしてもらえると嬉しいです。

また、イルシルではエンジニアを募集しています！
現在サービスがとても成長しているため、この成長をさらに加速させてくれる仲間を大募集中です！
詳しくは固定コメントにある採用ページからご確認ください。
