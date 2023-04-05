---
title: "【React】アクセシビリティに配慮したカレンダーUIを実装する"
emoji: "📅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "アクセシビリティ"]
published: false
publication_name: "spacemarket"
---

こんにちは。
[スペースマーケット](https://www.spacemarket.com/)でフロントエンドエンジニアをしているmizukiです。

スペースマーケットでは少し前に予約ページをNext.jsへと移行しました。
予約ページなので以下のような日時を選択するカレンダーのUIも実装したのですが、その際に少しアクセシビリティにも配慮しながら実装をしたので、今回はその内容について書いていきます。

![](/images/accessible-calendar/spm_calendar.png)

## 完成品

デモアプリを作成したのでこちらからもご確認いただけます。

:::message
この記事ではカレンダーを表示して日付をクリックできるところまでを実装しています。
日付を選択したり表示月を切り替えたりといった実装はしておりませんのでご了承ください。
:::

- github
  - https://github.com/mizuki0201/accessible-calendar
- デモアプリ
  - https://accessible-calendar.vercel.app/

## この記事で書くこと

アクセシビリティに配慮した点は以下2つです。

1. セマンティックなマークアップ
2. キー操作によるフォーカス遷移

それぞれ実装方法と共に解説してきます。

## 実装方法

:::message
ここからコードを紹介する時には、必要なコードの部分のみ抽出して書いていますのでご了承ください。
コード全体を見たい場合はリポジトリのコードor最後の完成版のコードを見ていただけると幸いです。
:::

### セマンティックなマークアップ

まずはマークアップをよりセマンティックに行います。

**①HTMLタグを適切にマークアップする**

最低限のカレンダーを作るには、divで囲われた中に各日付のセルをbuttonで実装すればよいかと思います。
ただ、それだとブラウザには「ボタンの羅列」という情報しか伝わりません。
そのため、HTMLタグを使って「表形式である」ことを伝えていきます。

構造としては以下の通りです。

```jsx:CalendarCells.tsx
<table>
  <thead>
    <tr>
      {weekDays.map((weekDay) => (
        <th>{weekDay}</th>
      ))}
    </tr>
  </thead>

  <tbody>
    {calendars.map((week, index) => (
      <tr>
        {week.map((day) => (
          <td>
            <button>
              {day}
            </button>
          </td>
        ))}
      </tr>
    ))}
  </tbody>
</table>
```

tableタグを使用して表形式の構造でマークアップします。
1つ1つの日付セルは、日付を選択する際にclickしたりフォーカス遷移をHTML標準機能を使用したいのでbuttonタグで実装しています。

基本的にはHTMLタグで実装することが望ましいですが、どうしてもtableタグを使えない場合などは、以下のようなroleを付与することでdivタグのような意味を持たないタグであっても同じようにセマンティックな実装ができます。
・[grid](https://developer.mozilla.org/ja/docs/Web/Accessibility/ARIA/Roles/grid_role)
・[rowgroup](https://developer.mozilla.org/ja/docs/Web/Accessibility/ARIA/Roles/rowgroup_role)
・[row](https://developer.mozilla.org/ja/docs/Web/Accessibility/ARIA/Roles/row_role)
・[columnheader](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/columnheader_role)
・[gridcell](https://developer.mozilla.org/ja/docs/Web/Accessibility/ARIA/Roles/gridcell_role)

**②ラベルを用いて読み上げにも対応できるようにする**

何かしらの理由でVoiceOverなどの読み上げ機能を使用する可能性もあるので、1つ1つのセルにラベルを付与します。
そうすることで、そのセルがどの日付なのかをより正確に伝えることができます。

```diff jsx:CalendarCells.tsx
{calendars.map((week, index) => (
  <tr>
    {week.map(({year, month, day}) => (
      <td>
-       <button>
+       <button aria-label={`${year}年${month}月${day}日`}>
          {day}
        </button>
      </td>
    ))}
  </tr>
))}
```

これで読み上げをする際に、「2023年4月10日」のように読み上げてくれるためよりわかりやすくなりました。

### キー操作によるフォーカス遷移

次にフォーカス遷移の実装をしていきます。

このままだとbuttonが全てフォーカス可能な要素であるので、tab切り替えで、
カレンダーの前の要素→前の月ボタン→次の月ボタン→カレンダーのセル1つずつ→カレンダーの後の要素
といった順にフォーカスが移動していくことになります。

これでも問題ないですが、30個以上あるセルをtabだけで操作するのはちょっと面倒なので、以下の方針でキー操作できるようにします。
①日付のセル全てを1まとまりとしてフォーカス遷移をさせる
②カレンダーの中では上下左右の矢印でフォーカス遷移をさせる

**①日付のセル全てを1まとまりとしてフォーカス遷移をさせる**

フォーカス遷移を制御するには、[tabIndex](https://developer.mozilla.org/ja/docs/Web/HTML/Global_attributes/tabindex)を使用します。
ドキュメントにもある通り、「0ならフォーカス可能、-1ならフォーカス不可」というルールで実装していきます。

フォーカスがカレンダーの中にあるかどうかを表すstateを定義し、フォーカスがカレンダーの外にあれば全てのセルを`tabIndex: 0`としてフォーカス可能に、カレンダーの中にあれば全てのセルを`tabIndex: -1`としてフォーカス不可にします。
※カレンダーの中から外へtab切り替えで遷移する処理は、②でonKeyDownの説明をする時に合わせて書くのでここでは書いていません。

また、フォーカスがカレンダー内にある状態でカレンダーの外をクリックしてしまうとstateが更新されずバグに繋がってしまうため、react-useにある`useClickAway`を使用してstateを更新する処理も合わせて書いています。

```jsx:CalendarCells.tsx
export const CaledarCell = () => {
  const containerRef = useRef(null);
  const [isFocusInCalendar, setIsFocusInCalendar] = useState(false);
  useClickAway(containerRef, () => setIsFocusInCalendar(false));

  return (
    <table
      ref={containerRef}
      onFocus={() => setIsFocusInCalendar(true)}
    >
      {/* 曜日のCellは省略 */}
      <tbody>
        {calendars.map((week, index) => (
          <tr>
            {week.map((day) => (
              <td>
                <button tabIndex={isFocusInCalendar ? -1 : 0}>
                  {day}
                </button>
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```
<br/>
**②カレンダーの中では上下左右の矢印でフォーカス遷移をさせる**

最後にカレンダー内でのキー操作（onKeyDown）について記載します。
基本的には、以下の4つの方針で実装します。
①tabが押されたらカレンダーの外へフォーカスが出るので、stateを更新する
②上下左右の矢印が押されたら日付をずらす
③エンターが押されたらonClickを発火させる
④それ以外のキーは何も反応しない

```jsx:CalendarCells.tsx
export const CaledarCell = () => {
  const onKeyDown = (e) => {
    // ①tabが押されたらカレンダーの外へフォーカスが出るので、stateを更新する
    if (e.key === "Tab") {
      // フォーカス移動の前にstate更新を待つ
      setTimeout(() => setIsFocusInCalendar(false), 10);
      return;
    }

    e.preventDefault();
    // ④それ以外のキーは何も反応しない
    if (
      e.key !== "ArrowLeft" &&
      e.key !== "ArrowRight" &&
      e.key !== "ArrowUp" &&
      e.key !== "ArrowDown" &&
      e.key !== "Enter"
    ) {
      return;
    }

    const focusedDate = dayjs((e.target).dataset.date);

    // セル内のフォーカス移動
    const onChangeFocus = (diffDay) => {
      const movedDate = focusedDate.add(diffDay, "day");
      const movedElement =
        containerRef.current?.querySelector(
          `[data-date="${movedDate.format("YYYY-M-D")}"]`
        );
      movedElement?.focus();
    };

    // キー操作
    switch (e.key) {
      // ②上下左右の矢印が押されたら日付をずらす
      case "ArrowLeft": {
        onChangeFocus(-1); // 1日前
        break;
      }
      case "ArrowRight": {
        onChangeFocus(1); // 1日後
        break;
      }
      case "ArrowUp": {
        onChangeFocus(-7); // 7日前
        break;
      }
      case "ArrowDown": {
        onChangeFocus(7); // 7日後
        break;
      }
      // ③エンターが押されたらonClickを発火させる
      case "Enter": {
        alert(`${focusedDate.format("YYYY年M月D日")}をクリックしました`);
        break;
      }
    }
  };

  return (
    <table>
      {/* 曜日のCellは省略 */}
      <tbody>
        {calendars.map((week, index) => (
          <tr>
            {week.map((day) => (
              <td>
                <button
                  data-date={`${year}-${month}-${day}`}
                  onClick={() =>
                    alert(`${year}年${month}月${day}日をクリックしました`)
                  }
                  onKeyDown={onKeyDown}
                >
                  {day}
                </button>
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

### まとめ
最後に完成版のコードも貼っておきます。

::::details CalendarCells.tsx全体
```tsx
import { FC, useRef, useState } from "react";
import { chakra } from "@chakra-ui/react";
import dayjs from "dayjs";
import { today } from "../data";
import { useClickAway } from "react-use";

const weekDays = [
  { keyText: "sunday", weekDay: "日" },
  { keyText: "monday", weekDay: "月" },
  { keyText: "tuesday", weekDay: "火" },
  { keyText: "wednesday", weekDay: "水" },
  { keyText: "thursday", weekDay: "木" },
  { keyText: "friday", weekDay: "金" },
  { keyText: "saturday", weekDay: "土" },
];

type Date = {
  year: string;
  month: string;
  day: string;
};

export const CalendarCells: FC<{
  calendarData: Array<Date>;
}> = ({ calendarData }) => {
  const containerRef = useRef<HTMLTableElement>(null);
  const [isFocusInCalendar, setIsFocusInCalendar] = useState(false);

  useClickAway(containerRef, () => setIsFocusInCalendar(false));

  // 1週間ごとの2次元配列に変換する
  const calendars = calendarData.reduce(
    (prev, current) => {
      if (prev[prev.length - 1].length < 7) {
        prev[prev.length - 1].push(current);
      } else {
        prev.push([current]);
      }
      return prev;
    },
    [[]] as Array<Array<Date>>
  );

  const onKeyDown = (e: React.KeyboardEvent<HTMLButtonElement>) => {
    if (e.key === "Tab") {
      // フォーカス移動の前にstate更新を待つ
      setTimeout(() => setIsFocusInCalendar(false), 10);
      return;
    }

    e.preventDefault();
    if (
      e.key !== "ArrowLeft" &&
      e.key !== "ArrowRight" &&
      e.key !== "ArrowUp" &&
      e.key !== "ArrowDown" &&
      e.key !== "Enter"
    ) {
      return;
    }

    const focusedDate = dayjs((e.target as HTMLElement).dataset.date as string);

    // セル内のフォーカス移動
    const onChangeFocus = (diffDay: number) => {
      const movedDate = focusedDate.add(diffDay, "day");
      const movedElement =
        containerRef.current?.querySelector<HTMLButtonElement>(
          `[data-date="${movedDate.format("YYYY-M-D")}"]`
        );
      movedElement?.focus();
    };

    // キー操作
    switch (e.key) {
      case "ArrowLeft": {
        onChangeFocus(-1); // 1日前
        break;
      }
      case "ArrowRight": {
        onChangeFocus(1); // 1日後
        break;
      }
      case "ArrowUp": {
        onChangeFocus(-7); // 7日前
        break;
      }
      case "ArrowDown": {
        onChangeFocus(7); // 7日後
        break;
      }
      case "Enter": {
        alert(`${focusedDate.format("YYYY年M月D日")}をクリックしました`);
        break;
      }
    }
  };

  return (
    <chakra.table
      width="100%"
      role="grid"
      ref={containerRef}
      onFocus={() => setIsFocusInCalendar(true)}
    >
      <chakra.thead>
        {/* 曜日のCell */}
        <chakra.tr
          role="row"
          display="grid"
          gridTemplateColumns="repeat(7, 1fr)"
        >
          {weekDays.map(({ keyText, weekDay }) => (
            <WeekDayCell key={keyText}>{weekDay}</WeekDayCell>
          ))}
        </chakra.tr>
      </chakra.thead>

      {/* 日付のCell */}
      <chakra.tbody minH="275px">
        {calendars.map((week, index) => (
          <chakra.tr key={`${index + 1}週目`} display="flex">
            {week.map(({ year, month, day }) => {
              const date = dayjs(`${year}-${month}-${day}`);
              const isToday = date.isSame(today, "day");

              return (
                <DateCell key={`${year}年${month}月${day}日`}>
                  <chakra.button
                    tabIndex={isFocusInCalendar ? -1 : 0}
                    aria-label={`${year}年${month}月${day}日`}
                    {...(isToday && {
                      "aria-current": "date",
                      fontWeight: "bold",
                    })}
                    data-date={`${year}-${month}-${day}`}
                    onClick={() =>
                      alert(`${year}年${month}月${day}日をクリックしました`)
                    }
                    onKeyDown={onKeyDown}
                  >
                    {day}
                  </chakra.button>
                </DateCell>
              );
            })}
          </chakra.tr>
        ))}
      </chakra.tbody>
    </chakra.table>
  );
};

// 曜日のセルのstyle
const WeekDayCell = chakra("th", {
  baseStyle: {
    fontSize: "14px",
    fontWeight: "bold",
    height: "38px",
    display: "flex",
    justifyContent: "center",
    alignItems: "center",
    _first: { color: "red" },
    _last: { color: "blue" },
  },
});

// 日付のセルのstyle
const DateCell = chakra("td", {
  baseStyle: {
    height: "56px",
    width: "100%",
    margin: "0 -1px -1px 0",
    fontSize: "14px",
    border: "1px solid #EAEAEA",
    _first: { color: "red" },
    _last: { color: "blue" },
    "& > button": {
      width: "100%",
      height: "100%",
      display: "flex",
      justifyContent: "center",
      alignItems: "flex-start",
      pt: "10px",
    },
  },
});

```
::::

これでアクセシビリティに配慮した実装ができました。

今回はUIのマークアップのみで日付の選択などの実装はしていないことと、アクセシビリティに配慮する実装方法はまだ他にも色々あると思うので、他の実装方法を知っている方がいればぜひコメントで教えていただけますと幸いです！

## おわりに

最後に宣伝です。
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