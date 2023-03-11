---
title: "dayjsでn分単位の切り捨て/切り上げ処理を実装する"
emoji: "🕒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['javascript', 'typescript', 'dayjs']
published: false
---

こんにちは！
スペースマーケットでフロントエンドエンジニアをしているmizukiです！

弊社が運営する[スペースマーケット](https://www.spacemarket.com/)では掲載されているスペースを15分刻みで予約することが可能です。
その際、APIから返ってきたスペースの予約可能な時間を15分刻みに整形する必要があったのですが、dayjsには「15分刻みで切り上げ/切り捨てする」という処理は用意されていなかったため、自作して実装をしました。
この記事ではその方法について書いていきたいと思います！

## やりたいこと

:::message
背景とか前提とかはいいから実装見せてくれ！！という方は次の「実装方法」へスキップしてください🙏
:::

冒頭でも軽く触れましたが、今回やりたいことは「**dayjsで15分刻みで切り上げ/切り捨てするという処理を自作すること**」です。

APIから返ってきたデータが、以下のように、3/30の10:12〜16:23が予約不可であるというデータだったとします。

```js
// 予約可能な時間帯
availableTime: [
  { 
    startTime: '2023-03-30 00:00:00',
    endTime: '2023-03-30 10:12:00',
  },
  { 
    startTime: '2023-03-30 16:23:00',
    endTime: '2023-03-31 00:00:00',
  },
]
```

予約自体は15分刻みで予約可否を決めているため、上記のデータの場合、10:12や16:23という半端な時刻は切り上げ/切り捨てをする必要があります。
具体的には、以下のようにstartTimeは切り捨て、endTimeは切り上げ処理を入れてあげるとよさそうです。

```diff js
// 予約可能な時間帯
availableTime: [
  { 
    startTime: '2023-03-30 00:00:00',
-   endTime: '2023-03-30 10:12:00',
+   endTime: '2023-03-30 10:00:00', // 半端な時刻は予約不可にするため切り捨て
  },
  { 
-   startTime: '2023-03-30 16:23:00',
+   startTime: '2023-03-30 16:30:00', // 半端な時刻は予約不可にするため切り上げ
    endTime: '2023-03-31 00:00:00',
  },
]
```

この処理を実現する方法について書いていきます。

## 実装方法

実装したコードは以下の通りです。

:::message
ここまでの話は「15分単位」で話してきましたが、実装としてはunitとamountをpropsで受け取るため、時間の単位や数字を自由に指定することができます。
:::

### 切り上げ処理

```ts:helpers/dayjs/ceil/index.ts
import inst, { PluginFunc, UnitType } from 'dayjs'

declare module 'dayjs' {
  interface Dayjs {
    ceil(unit: UnitType, amount: number): inst.Dayjs
  }
}
export const ceil: PluginFunc = (option, dayjsClass) => {
  dayjsClass.prototype.ceil = function (unit, amount) {
    if (this.get(unit) % amount === 0) {
      return this.startOf(unit)
    }
    return this.add(amount - (this.get(unit) % amount), unit).startOf(unit)
  }
}
```

### 切り捨て処理

```ts:helpers/dayjs/floor/index.ts
import inst, { PluginFunc, UnitType } from 'dayjs'

declare module 'dayjs' {
  interface Dayjs {
    floor(unit: UnitType, amount: number): inst.Dayjs
  }
}
export const floor: PluginFunc = (option, dayjsClass) => {
  dayjsClass.prototype.floor = function (unit, amount) {
    return this.subtract(this.get(unit) % amount, unit).startOf(unit)
  }
}
```

### 使う側のコンポーネントの処理

```ts:components/hoge.ts
import dayjs from 'dayjs'
import { ceil, floor } from '../helpers/dayjs'

dayjs.extend(ceil)
dayjs.extend(floor)

const availableTime =  [
  { 
    startTime: '2023-03-30 00:00:00',
    endTime: '2023-03-30 10:12:00',
  },
  { 
    startTime: '2023-03-30 16:23:00',
    endTime: '2023-03-31 00:00:00',
  },
]

const formattedTime = availableTime.map(({startTime, endTime}) => {
  return {
    startTime: dayjs(startTime).floor('minute', 15).format(),
    endTime: dayjs(endTime).ceil('minute', 15).format(),
  }
})

console.log(formattedTime) 
// ↓log結果
// [
//   { 
//     startTime: '2023-03-30 00:00:00',
//     endTime: '2023-03-30 10:00:00', 
//   },
//   { 
//     startTime: '2023-03-30 16:30:00',
//     endTime: '2023-03-31 00:00:00',
//   },
// ]
```

## 解説

先ほど書いたコードの解説をしていきます。

### 切り上げ/切り捨て処理

大枠の方針としては、dayjsのライブラリに「ceil」と「floor」という2つの関数を追加して定義します。

まずは上半分で記述している型定義部分です。
ここは似た記述なので、ceilを例に説明します。

```ts
declare module 'dayjs' {
  interface Dayjs {
    ceil(unit: UnitType, amount: number): inst.Dayjs
  }
}
```

この部分ではdayjsに対して「ceil」という関数の型定義を追加しています。
引数でunit（時間の単位）とamount（時間の数字）を受け取り、dayjsのインスタンスを返却する関数です。

次に下半分の、処理の中身を見ていきます。
まずはceilです。

```ts
export const ceil: PluginFunc = (option, dayjsClass) => {
  dayjsClass.prototype.ceil = function (unit, amount) {
    // this.get(unit)は、指定された単位の部分の数値を取得しています。
    // 10:23に対して.get('minute')とすると23という数値を取得できます。
    if (this.get(unit) % amount === 0) {
      return this.startOf(unit)
    }
    return this.add(amount - (this.get(unit) % amount), unit).startOf(unit)
  }
}
```

ここではdayjsClassの中のprototypeに「ceil」という関数を追加しています。
処理の中身としては、
amountで割った余りが0になる場合は何も処理しないのでそのまま返却します。
例えば15分単位で処理したいときの00,15,30,45の場合は何もする必要がないので、そのまま返却しています。

そうでない場合は、amountからamountで割った余りを引くことで、切り上げに必要な数を求め、それを現在の時刻に追加しています。
例えば15分単位で処理したい場合、10:23は10:30に切り上げをしたいので、23という分数を15で割った余りである"8"を15から引くことで、最終的に足し合わせたい"7"という数字を導き出しています。

最後にstartOf(unit)と書いているのは、指定された単位の初めまでリセットをしています。
言葉にするとわかりにくいですが、例としては12:32:48のように秒数まで指定があった場合、単位の分で切り上げをしただけだと12:45:48となってしまうため、"minute"などの単位でリセットをし、12:45:00という時刻を作っています。

最後にfloorです。

```ts
export const floor: PluginFunc = (option, dayjsClass) => {
  dayjsClass.prototype.floor = function (unit, amount) {
    // subtractはシンプルに引き算をしています
    return this.subtract(this.get(unit) % amount, unit).startOf(unit)
  }
}
```

ここはとてもシンプルで、amountで割った余りを引くだけです。
12:43を15分単位で切り捨てしたい場合、43を15で割った余りである13を引き、12:30という時刻にしています。
15分単位で処理したい場合の00,15,30,45は余りが0になり、0を引くだけのためceilのようにifで分岐する必要もありません。

### 使う側の実装

あとは上記で定義した内容を使用するだけです。

```ts
import dayjs from 'dayjs'
import { ceil, floor } from '../helpers/dayjs'

dayjs.extend(ceil)
dayjs.extend(floor)
```

まずはこのように、dayjsのライブラリに対して先ほど定義したceilとfloorをextendして使える状態にします。
これでceilとfloorを使用できる状態になったので、

```ts
dayjs(startTime).floor('minute', 15)
dayjs(endTime).ceil('minute', 15)
```

このようにdayjsのオブジェクトに対してceil()やfloorを使用します。
単位と数字の指定をお忘れなく。

これで実装は完了です。

## 応用

ここまでは自分が実際に必要だったのが「15分単位」だったので、15分単位での実装を前提に話をしてしまいましたが、20分単位でも3時間単位でも実装することが可能です。
以下にその例を軽く解説します。

### 20分単位で切り上げ

記述としては、

```ts
dayjs('2023-03-30 12:32:00').ceil('minute', 20)
```

のようになります。
処理としては、単位となるminuteの32をamountの20で割った余りである"12"を、amountの20から引いた"8"を現在時刻に加えた「12:40」が返却されます。
（一気に書いてしまいましたが、都度ceilの記述内容と照らし合わせながら読んでください🙇‍♂️）

20分単位なので、12:40への切り上げができていそうですね。

### 3時間単位で切り捨て

記述としては、

```ts
dayjs('2023-03-30 13:20:00').floor('hour', 3)
```

のようになります。
処理としては、単位となるhourの13をamountの3で割った余りである1を引き、12:20:00をstartOf("hour")でリセットするため、最終的には「2023-03-30 12:00:00」という時刻が返却されます。

これも3時間単位での切り捨てができていそうですね。

## さいごに

今回はdayjsでn分単位で切り上げ/切り捨てする方法について書いていきました。
元々dayjsというライブラリ単体でもかなり便利だなとは思うのですが、今回の処理を書いたことでより一層dayjsが使いやすくなったなと感じています！

最後に宣伝です。
スペースマーケットでは一緒に働く仲間を募集しています！
カジュアルに話を聞きたいだけという方でも大歓迎ですので、ぜひ以下からご応募お待ちしております！

**▼SRE/インフラエンジニア**
https://www.wantedly.com/projects/1113570

**▼バックエンドエンジニア**
https://www.wantedly.com/projects/1113544

**▼Androidエンジニア（iOSも大歓迎です！）**
https://www.wantedly.com/projects/1061116

**▼エンジニア採用ページ（迷ったらこちらからどうぞ！）**
https://spacemarket.co.jp/recruit/engineer/

