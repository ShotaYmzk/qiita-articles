---
title: 【完全無料】 2026年最新版：GASでNotionの就活DBをLINEに毎朝自動通知するBotを作ってみた 【サーバー不要】
tags:
  - GoogleAppsScript
  - GAS
  - 初心者
  - LINEmessagingAPI
  - NotionAPI
private: false
updated_at: '2026-05-12T18:47:40+09:00'
id: 7bade20ffe18d36e4fd8
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---


## はじめに

就活のタスク管理はNotionでやってるのに、Notionを開くのを忘れてES締め切りを危うく見逃しかけた。

「どうせ毎朝LINEは絶対見る」という発想で、**GAS + Notion API + LINE Messaging API**を使って自動通知Botを作ったところ、思ったより簡単に動いたのでまとめます。

Notionの仕様の変更やGASの変更があって他のQiita記事ではできないことがありましたのでこの記事を参考にしていただけたら幸いです。

- ✅ サーバー不要
- ✅ デプロイ不要  
- ✅ 完全無料
- ✅ APIを叩いたことがあれば誰でも再現できる

### 完成イメージ

毎朝7時にこんな通知がLINEに届く。

>📋 就活スケジュール通知
>5月11日(月)
>─────────────────
>🔴 今日 5月11日(月)
>　- ES〆切｜NRI　[進行中]
>🟡 明日 5月12日(火)
>　- ES〆切｜Accenture　[未着手]
>🟢 明後日 5月13日(水)
>　- インターン〆切｜日鉄ソリューションズ　[進行中]

---

## 使用技術・費用

| 役割 | サービス | 費用 |
|---|---|---|
| スケジュール管理 | Notion | 無料 |
| 自動実行・API連携 | Google Apps Script | 無料 |
| 通知先 | LINE Messaging API | 無料（月1000通まで） |

---

## システム構成
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4429374/4d4ef569-2aa6-416a-859b-2a5e087f69be.png)

GASが中継役となりNotionのDBを読んでLINEに送信するだけのシンプルな構成。

---

## 事前準備

### 1. Notion APIキーの取得
![スクリーンショット 2026-05-11 12.03.09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4429374/e2257ac4-25cb-4d35-85f0-685fa9e3b55f.png)

1. https://www.notion.so/profile/integrations/internal 内部コネクトにアクセス
2. ①「＋ 新規コネクトを作成する」→ コネクト名を入力（例：`就活通知`）
②画像は適当で大丈夫です
③読み取りたいDBがあるワークスペースを選択してください→保存

3.コネクトが作成されましたと表示されるので、管理に進むとインストールのアクセストークンが発行されているのでこれをコピーしておいてください

次に、NotionのDBページを開いて右上「…」→「接続」→アクティブな接続で先ほどのコネクト名が表示されているか確認してください
![IMG_2173.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4429374/a2028f67-9e94-44ec-8627-cdb5405ea362.png)


> ⚠️ **これを忘れるとAPIがDBを読めない。** インテグレーション作成だけでは不十分。
### 2. NotionのDB IDを確認

> ⚠️ URLには**ページID**と**DB ID**の2種類が存在する。ページ内に埋め込まれたDBのIDを取得する必要があるため、URLからコピーしたIDが間違っている場合は404エラーになる。詳しくは[後述](#db-idが間違っていた)。

### 3. LINE Messaging APIの設定

1. https://manager.line.biz/ でLINE公式アカウントを作成（無料）
2. LINE Developersコンソールhttps://developers.line.biz/console/ でMessaging APIチャンネルを有効化
3. Messaging API設定画面の
一番下にある「チャンネルアクセストークン（長期）」を発行してコピー
4. チャネル基本設定の一番下にある「あなたのユーザーID」（`Uxxxxxxxx`形式）をコピー
5. 作成した公式アカウントを**自分のLINEで友達追加**
![スクリーンショット 2026-05-11 12.29.34.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4429374/4a4f7043-7bf0-43f2-be2e-1e55bbef08ea.png)
> ⚠️ **LINE Notifyは2025年3月末でサービス終了済み。** ネットで古い記事を参考にしている場合は要注意。代替はLINE Messaging API（個人へのPush送信は月1000通まで無料）。

### 3. NotionのDB IDを確認

> ⚠️ URLには**ページID**と**DB ID**の2種類が存在する。ページ内に埋め込まれたDBのIDを取得する必要があるため、URLからコピーしたIDが間違っている場合は404エラーになる。詳しくは[後述](#db-idが間違っていた)。

---

## 実装

### NotionのDB構成（最低限）

| プロパティ名 | 型 |
|---|---|
| 名前 | タイトル |
| 日付 | 日付 |
| ステータス | ステータス |

コードはこの3プロパティを前提にしている。既存のDBに合わせてプロパティ名を変更すればOK。
私は以下のような形のDBを使っています
https://www.notion.so/marketplace/templates/jobhunting-dashboard?cr=pro%253Afilmiana
このテンプレートを使わせていただいています
![スクリーンショット 2026-05-11 12.31.45.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4429374/189474cd-84f3-40cc-b3f7-aa4fcb4d16fa.png)

### GASのコード全文

https://script.google.com で新しいプロジェクトを作成し、以下を貼り付ける。

```javascript
const NOTION_TOKEN = "ここにNotionのアクセストークン";
const NOTION_DB_ID = "ここにDB ID（32文字）";
const LINE_TOKEN   = "ここにLINEのチャンネルアクセストークン";
const LINE_USER_ID = "ここにあなたのユーザーID（Uxxxxxxxx）";

// デバッグ用：日付を指定して動作確認できる。本番時はnullにする
const DEBUG_DATE = null; // 例："2026-05-15"

function sendDailyNotification() {
  const today = DEBUG_DATE ? new Date(DEBUG_DATE + "T00:00:00+09:00") : new Date();
  const fmt   = d => Utilities.formatDate(d, "Asia/Tokyo", "yyyy-MM-dd");
  const fmtJP = d => Utilities.formatDate(d, "Asia/Tokyo", "M月d日(E)");

  const tomorrow = new Date(today); tomorrow.setDate(today.getDate() + 1);
  const dayAfter = new Date(today); dayAfter.setDate(today.getDate() + 2);
  const todayStr    = fmt(today);
  const tomorrowStr = fmt(tomorrow);
  const dayAfterStr = fmt(dayAfter);

  // Notion APIで今日〜3日後のタスクを取得（完了除く）
  const payload = {
    filter: {
      and: [
        { property: "日付", date: { on_or_after: todayStr } },
        { property: "日付", date: { on_or_before: fmt(new Date(today.getTime() + 3 * 86400000)) } },
        { property: "ステータス", status: { does_not_equal: "完了" } }
      ]
    },
    sorts: [{ property: "日付", direction: "ascending" }]
  };

  const res = UrlFetchApp.fetch(
    `https://api.notion.com/v1/databases/${NOTION_DB_ID}/query`,
    {
      method: "post",
      contentType: "application/json",
      headers: {
        "Authorization": `Bearer ${NOTION_TOKEN}`,
        "Notion-Version": "2022-06-28"
      },
      payload: JSON.stringify(payload)
    }
  );

  const pages = JSON.parse(res.getContentText()).results;

  if (pages.length === 0) {
    sendLine(`📋 就活スケジュール通知\n${fmtJP(today)}\n─────────────────\n✅ 今後3日間の締め切りはありません！`);
    return;
  }

  // 日付ごとにグループ化
  const groups = {};
  pages.forEach(p => {
    const dateStr = p.properties["日付"]?.date?.start?.slice(0, 10);
    const name    = p.properties["名前"]?.title?.[0]?.plain_text || "（名称不明）";
    const status  = p.properties["ステータス"]?.status?.name || "";
    if (!dateStr) return;
    if (!groups[dateStr]) groups[dateStr] = [];
    groups[dateStr].push({ name, status });
  });

  // ラベルの定義
  const labelMap = {
    [todayStr]:    `🔴 今日 ${fmtJP(today)}`,
    [tomorrowStr]: `🟡 明日 ${fmtJP(tomorrow)}`,
    [dayAfterStr]: `🟢 明後日 ${fmtJP(dayAfter)}`,
  };

  // メッセージ組み立て
  let msg = `📋 就活スケジュール通知\n${fmtJP(today)}\n`;
  msg += "─────────────────\n";

  Object.keys(groups).sort().forEach((d, i) => {
    if (i > 0) msg += "\n";
    const label = labelMap[d] || `📅 ${d}`;
    msg += `${label}\n`;
    groups[d].forEach(t => {
      msg += `　• ${t.name}　[${t.status}]\n`;
    });
  });

  sendLine(msg);
}

function sendLine(message) {
  UrlFetchApp.fetch("https://api.line.me/v2/bot/message/push", {
    method: "post",
    contentType: "application/json",
    headers: { "Authorization": `Bearer ${LINE_TOKEN}` },
    payload: JSON.stringify({
      to: LINE_USER_ID,
      messages: [{ type: "text", text: message }]
    })
  });
}
```

---

## コードの解説

### Notion APIのフィルタリング

`/v1/databases/{id}/query` エンドポイントに対して、`and` 配列で複数条件を組み合わせる。

```javascript
filter: {
  and: [
    { property: "日付", date: { on_or_after: todayStr } },
    { property: "日付", date: { on_or_before: 3日後 } },
    { property: "ステータス", status: { does_not_equal: "完了" } }
  ]
}
```

Notion APIのプロパティ型（`date`、`status`、`title`など）によってクエリの書き方が変わる点に注意。型の一覧は[公式ドキュメント](https://developers.notion.com/reference/property-value-object)を参照。

### 日付ごとのグループ化

APIのレスポンスはページ（行）の配列で返ってくる。日付をキーにしたオブジェクトに詰め直すことで、同じ日のタスクをまとめて表示できる。

```javascript
const groups = {};
pages.forEach(p => {
  const dateStr = p.properties["日付"]?.date?.start?.slice(0, 10);
  if (!groups[dateStr]) groups[dateStr] = [];
  groups[dateStr].push({ name, status });
});
```

### デバッグ用の日付差し替え

`DEBUG_DATE` を文字列で指定すると、その日付のデータで動作確認できる。本番運用時は `null` に戻すだけでよい。

タイムゾーンに `+09:00` を明示しているのは、GASのサーバーがUTCで動いているため。`new Date("2026-05-15")` だとJSTで5月15日9時として解釈されてしまうのを防ぐため。

---

## 毎朝7時の自動実行設定

**デプロイは不要。** GASのトリガー機能を使う。

1. GAS画面左メニュー「⏰ トリガー」→「＋ トリガーを追加」
2. 以下のように設定して保存

| 項目 | 設定値 |
|---|---|
| 実行する関数 | `sendDailyNotification` |
| イベントのソース | 時間主導型 |
| 時間ベースのトリガーの種類 | 日付ベースのタイマー |
| 時刻 | 午前7時〜8時 |

---

## ハマったポイント

### DB IDが間違っていた

NotionのURLに含まれるIDには2種類ある。

- **ページID**：就活タスク管理ページ自体のID
- **DB ID**：ページ内に埋め込まれているデータベースのID

URLからコピーしたIDがページIDだったため、最初は404エラーが発生した。  
Notion APIで正しいDB IDを確認するには、![スクリーンショット 2026-05-11 12.45.41.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4429374/cf0ee856-bec2-4da3-adda-2441cd3ab59c.png)
`⇱フルページで開く⇲` でDBをフルページで開いた後に、`︙` からリンクをコピーするとhttps://www.notion.so/XXXXXXXXXXXXXXXXXXXXXXXXXXXX?v=YYYYYYYYYYYYYYYYYYYYYYYYYY&source=copy_link のようなURLがコピーされるのでXXXXXXに当たるところだDB IDになる。



### LINE Notifyはサービス終了済み

調べると古い記事ではLINE Notifyを使った実装が多く紹介されているが、**2025年3月末でサービス終了**している。代替はLINE Messaging APIのPush送信（月1000通まで無料）。

---

## まとめ

| 項目 | 内容 |
|---|---|
| 開発時間 | 約1〜2時間 |
| 費用 | 完全無料 |
| 必要スキル | JavaScriptの基礎・APIの基本的な概念 |

GASはサーバー不要でHTTP通信が書けるため、Notion APIやLINE Messaging APIとの相性がいい。Notionをタスク管理に使っている人は、同じ仕組みで締め切り通知・習慣トラッキングなど様々な用途に応用できる。


## 応用：LINEグループへの通知

本記事では個人のLINEへのPush送信を扱ったが、**就活仲間とグループを作ってグループ全体に通知する**こともできる。

グループへの送信にはUser IDとは別に**グループID**（`C`始まり）が必要になり、取得手順が少し変わる。また、定時通知にはPush送信、コマンド応答にはReply送信と使い分けることで**無料枠を節約**できる。

この辺りの実装は別記事にまとめているので、グループ通知に挑戦したい人はこちらを参照してほしい。

→ [【GAS × LINE】就活通知をグループに送る方法とPush/Reply使い分け解説]()
