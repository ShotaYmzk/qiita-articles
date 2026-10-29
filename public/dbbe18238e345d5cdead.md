---
title: 【後編】GASでLINEグループに就活通知を送る方法＋Webhookでコマンド応答Bot化する2026年最新版
tags:
  - GoogleAppsScript
  - GAS
  - 初心者
  - LINEmessagingAPI
  - Notion
private: false
updated_at: '2026-05-12T18:44:30+09:00'
id: dbbe18238e345d5cdead
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---


## はじめに

[前編](https://qiita.com/ShotaYmzk/items/7bade20ffe18d36e4fd8)では、GAS + Notion API + LINE Messaging APIを使って**個人のLINEに毎朝就活スケジュールを自動通知**するBotを作った。

本記事（後編）では以下の2点を追加実装する。

- ✅ **グループIDの取得**とグループへのPush送信
- ✅ **GASをWebhookとして動かし**、LINEからコマンドを送ると返信するBot化

前編のコードをベースに追記する形で進めるので、先に前編を読んでおくことを推奨する。

---

## Push と Reply の使い分け（重要）

グループ通知を実装する前に、LINE Messaging APIの送信方式の違いを把握しておく。

| | Push送信 | Reply送信 |
|--|------|-------|
| 用途 | 自発的な送信（定時通知など） | 受信メッセージへの返信 |
| 送信先指定 | User ID / Group ID | replyToken（有効期限あり） |
| 無料枠 | **月1000通まで** | **無制限** |

**設計方針：定時通知はPush、コマンド応答はReply** で使い分けることで無料枠の消費を最小限に抑えられる。

---

## グループIDの取得

グループへのPush送信には**グループID**（`C`始まりの文字列）が必要。取得にはWebhookを使う。

### 手順の概要

1. GASをウェブアプリとしてデプロイしてWebhook URLを発行する
2. LINE DevelopersコンソールでそのURLをWebhook URLとして登録する
3. BotをLINEグループに招待する
4. グループで「グループID」と送信 → Botがグループ IDを返信する

順番に実装していく。

---

## GASのコード全文

前編のコードに以下を**追記**する。

```javascript
const NOTION_TOKEN = "ここにNotionのアクセストークン";
const NOTION_DB_ID = "ここにDB ID（32文字）";
const LINE_TOKEN   = "ここにLINEのチャンネルアクセストークン";
const LINE_USER_ID = "ここにあなたのユーザーID（Uxxxxxxxx）";
const LINE_GROUP_ID = ""; // グループID取得後にここに入れる

// ─── 前編のコード（sendDailyNotification, sendLine）はここに ───

// ===== 後編追加分 =====

// Webhookエンドポイント（LINEからのPOSTを受け取る）
function doPost(e) {
  try {
    const events = JSON.parse(e.postData.contents).events;
    events.forEach(handleEvent);
  } catch (err) {
    console.log("doPost error: " + err);
  }
  // LINEは200レスポンスが返らないと再送してくるので必須
  return ContentService
    .createTextOutput(JSON.stringify({ status: "ok" }))
    .setMimeType(ContentService.MimeType.JSON);
}

function handleEvent(event) {
  if (event.type !== "message" || event.message.type !== "text") return;

  const text    = event.message.text.trim();
  const source  = event.source;
  const token   = event.replyToken;

  // グループIDを返す（グループに招待後に使う）
  if (text === "グループID") {
    const id = source.groupId || "（グループ外からの送信）";
    replyLine(token, `🆔 グループID:\n${id}`);
    return;
  }

  // 今日の予定を返す
  if (text === "今日の予定") {
    const schedule = getTodaySchedule();
    replyLine(token, schedule);
    return;
  }

  // 励ましメッセージ
  if (text.includes("就活終われ")) {
    replyLine(token, "もう少し！頑張れ！🔥");
    return;
  }
}

// Reply送信（受信メッセージへの返信。無料枠を消費しない）
function replyLine(replyToken, message) {
  UrlFetchApp.fetch("https://api.line.me/v2/bot/message/reply", {
    method: "post",
    contentType: "application/json",
    headers: { "Authorization": `Bearer ${LINE_TOKEN}` },
    payload: JSON.stringify({
      replyToken,
      messages: [{ type: "text", text: message }]
    })
  });
}

// Push送信をグループ宛にする（グループID設定後に使う）
function sendDailyNotificationToGroup() {
  // LINE_GROUP_IDが設定されていればグループへ、なければ個人へ
  const target = LINE_GROUP_ID || LINE_USER_ID;
  sendLineToTarget(target);
}

function sendLineToTarget(to) {
  // 前編のsendDailyNotificationの中身をそのままtoに向けるだけ
  // （前編コードのsendLineをsendLineToTargetに置き換える）
  UrlFetchApp.fetch("https://api.line.me/v2/bot/message/push", {
    method: "post",
    contentType: "application/json",
    headers: { "Authorization": `Bearer ${LINE_TOKEN}` },
    payload: JSON.stringify({
      to,
      messages: [{ type: "text", text: buildMessage() }]
    })
  });
}

// 今日のスケジュールをNotionから取得してテキストに変換する
function getTodaySchedule() {
  const today = new Date();
  const fmt   = d => Utilities.formatDate(d, "Asia/Tokyo", "yyyy-MM-dd");
  const fmtJP = d => Utilities.formatDate(d, "Asia/Tokyo", "M月d日(E)");
  const todayStr = fmt(today);

  const payload = {
    filter: {
      and: [
        { property: "日付", date: { equals: todayStr } },
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
    return `📋 ${fmtJP(today)}\n─────────────────\n✅ 今日の締め切りはありません！`;
  }

  let msg = `📋 ${fmtJP(today)} の予定\n─────────────────\n`;
  pages.forEach(p => {
    const name   = p.properties["名前"]?.title?.[0]?.plain_text || "（名称不明）";
    const status = p.properties["ステータス"]?.status?.name || "";
    msg += `　• ${name}　[${status}]\n`;
  });
  return msg;
}
```

---

## GASをWebhookとして設定する

### 1. ウェブアプリとしてデプロイ

![スクリーンショット 2026-05-11 14.57.36.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4429374/b66515e2-3ecb-4a74-ac91-35b790a13d22.png)

GASのウェブアプリデプロイは**コードを変更するたびに「新しいバージョン」として更新**しないと反映されない点に注意。

1. GAS画面右上「デプロイ」→「新しいデプロイ」
2. 種類：「ウェブアプリ」
3. 実行ユーザー：「自分」
4. アクセスできるユーザー：「全員」
5. 「デプロイ」→ 表示されたURLをコピー

:::note warn
**コードを修正したら必ず「デプロイを管理→ 右上の✏️マークから編集 → 新しいバージョンで更新」が必要。**
保存しただけでは古いコードが動き続ける。これを知らずに30分溶かすことになる。
![スクリーンショット 2026-05-11 15.01.46.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4429374/8eab5121-6c8f-43a7-9fb8-de6abb6cd3a0.png)

:::

### 2. LINE DevelopersでWebhook URLを登録

1. LINE Developersコンソール → Messaging API設定
2. 「Webhook URL」に先ほどのGASのURLを貼り付け
3. 「検証」ボタンを押して「成功」が出ることを確認
4. 「Webhookの利用」をオンにする

![スクリーンショット 2026-05-11 15.16.30.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4429374/0e19d74a-49bd-4ce5-bc6a-be8cd6d2d578.png)

### 3. BotをLINEグループに招待

通知を送りたいグループを開いて、作成したLINE公式アカウント（Bot）を招待する。

---

## グループIDの取得

Botを招待したグループで **「グループID」** とメッセージを送信する。

BotがグループIDを返信してくれるので、それをコピーして `LINE_GROUP_ID` に貼り付ける。

```javascript
const LINE_GROUP_ID = "Cxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"; // Botが返信したIDを入れる
```

あとはGASのトリガーを `sendDailyNotificationToGroup` に変更（または追加）すれば、グループへの定時通知が完成する。
```javascript
logger.log,console.log
```
よりもわかりやすい形で表示されるので初心者でも使えると思います

---

## 沼ったポイント

### `doPost` をGASエディタから手動実行するとエラーになる

`doPost(e)` の引数 `e` はLINEからHTTPリクエストが来て初めて渡される。GASエディタから直接実行すると `e` が `undefined` になってエラーになる。

**動作確認はLINEからメッセージを実際に送って確認する。**

### レスポンスを返さないとLINEが再送を繰り返す

`doPost` からHTTPレスポンスを返さないと、LINEサーバーがエラーと判断して同じリクエストを再送する。`ContentService.createTextOutput` でのレスポンス返却が必須。

### ログの確認方法

`console.log` を使って、GAS画面左メニュー「実行数」→ 対象行をクリックして展開するとログが見られる。`Logger.log` でも可。

と他のqiitaに書いてあったが、自分のGASでは見れませんでした

---

## まとめ

| 項目 | 内容 |
|---|---|
| 追加開発時間 | 約1〜2時間 |
| 追加費用 | 完全無料 |
| 追加スキル | Webhookの基本的な概念 |

前後編合わせると、**毎朝グループに通知 ＋ コマンドで今日の予定を即呼び出し**という実用的なBotが完成する。

就活が終わっても、研究室のゼミ管理・バイトのシフト共有など同じ仕組みで使い回しがきく。
