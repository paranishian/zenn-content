---
title: "リングフィットアドベンチャーの記録で友達と競える仕組みを作った"
emoji: "🏋️‍♀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['GAS', 'JavaScript', 'Slack', '自動化', 'Bot']
published: true
---

# 🐣 はじめに

おうちで気軽にフィットネスができる「リングフィットアドベンチャー」。
一人でがんばるのもいいけどみんなで競い合ったほうがもっと楽しいし継続できるよね！ってことで、そんな仕組みを作りました。

具体的には

- 運動結果のSlack通知（「今日もちゃんと運動して偉い！」）
- 運動結果データのログ保存（いつ・だれが・どれくらい運動したか）
- ログの集計・可視化・通知（「8月のカロリー部門１位は○さんでした！」）

を自動化しています。

運動結果のスクショをTwitterに投稿するだけで参加できます。
この仕組みを作ってから、今では10人くらいでわちゃわちゃ楽しくやってます。
また後述しますが、すべて無料枠で運用しています。

## 主な機能

1. Twitterの投稿を検知してSlackに通知します。
![](https://storage.googleapis.com/zenn-user-upload/70c011418329c5ad2b391da4.png)

2. 毎週月曜日に進捗をお知らせします。
![](https://storage.googleapis.com/zenn-user-upload/4b85b60ba9eab28482d3ce68.png)

3. 月初に前月のサマリーを投稿します。（テキストだけ人力🤫）
![](https://storage.googleapis.com/zenn-user-upload/d632ef4bde868b008522f0e9.png)

# 🎯 技術的なポイント

![](https://storage.googleapis.com/zenn-user-upload/612ac8536809490dc3791424.png)

使っているサービスは以下のとおりです。
- pipedream
- Google Apps Script(GAS)

すべて無料枠で運用しています。


## pipedream
https://pipedream.com
pipedreamはいろんなAPIをつなげてワークフローをサクッと作れちゃう、いわゆるiPaaSです。Zapierより安いし（無料枠でたいてい事足りる）連携してるAPIも多いのでよく使っています。オススメです。

今回は、
- Twitter投稿の監視
- Slackに投稿
- 投稿されたスクショをGoogle Driveに保存

というワークフローを作っています。
![](https://storage.googleapis.com/zenn-user-upload/1ee2ff9419211b81f9def73b.png)

## Google Apps Script(GAS)
GASでは運動ログの保存とSlack通知を実装しています。

### 運動ログの保存

運動ログの保存は

- Google Driveに保存されている画像を取得
- 画像にOCRを実行
- OCR結果からデータを抽出してシートに出力

という流れで実装しています。
なんとOCRが無料で使えるのです。Google様様ですね。

OCRを利用するために[Drive API(V2)](https://developers.google.com/drive/api/v2/reference)を使っています。コードを抜粋します。
```js
function appendRowFromImage() {
  // フォルダーに入っているファイル一覧を取得
  const result = Drive.Children.list(folderId, { q: "mimeType = 'image/jpeg'" })

  // なにも入ってなければ終了
  if (result.items.length === 0) return;

  const ss = SpreadsheetApp.openById('##SHEET_ID##');
  const sheet = ss.getSheetByName('db');

  const resource = { title: "test" };
  const option = {
    "ocr": true,
    "ocrLanguage": "ja",
  };

  result.items.forEach(item => {
    const fileId = item.id;
    const file = Drive.Files.get(fileId);
    const date = Utilities.formatDate(new Date(file.createdDate), "JST", "yyyy-MM-dd HH:mm:ss");

    // 画像をコピーしてOCRを実行する
    const image = Drive.Files.copy(resource, fileId, option)
    // OCRのテキストを取得する
    const ocrText = DocumentApp.openById(image.id).getBody().getText();
    // コピー画像はもう不要なので削除
    Drive.Files.remove(image.id)
    // OCRのテキストから必要情報を抽出
    const { user, hour, minite, second, kcal, km } = extractData(ocrText);
    // シートに追加
    sheet.appendRow([date, user, hour, minite, second, kcal, km])
    // スクショ画像ももう不要なので削除
    Drive.Files.remove(fileId);
  })

}
```

こんな感じでログが保存されていきます 👇
![](https://storage.googleapis.com/zenn-user-upload/b84b6e24db40fea5760fc30a.png)

### ログの集計・可視化
シートにログがどんどん溜まっていくので、あとはスプシ芸を駆使して料理していくだけです。
`QUERY`, `ARRAY_FORMULA`, `VLOOKUP`あたりでゴニョゴニョして、前月、当月のデータを抽出し、チャートを作成します。
（カロリーの他、プレイ時間・走行距離・プレイ回数でもチャートを作っています）

![](https://storage.googleapis.com/zenn-user-upload/e386253f3b496b97cad861bf.png)

あとは、

- チャートを画像化してGoogle Driveに保存
- ダウンロードURLを取得
- Slackに投稿

という処理をGASで実装しています。コードを抜粋します。

```js
function postProgressToSlackChannel() {
  const charts = getCharts("current_month");
  const attachments = []
  charts.forEach((chart, i) => {
    const imageUrl = getImageUrl(chart);
    const title = chart.getOptions().get("title");
    attachments.push({
      fields: [
        {
          title: title
        }
      ],
      image_url: imageUrl
    })
  });

  const channel = "###CHANNEL_ID###";
  const token = PropertiesService.getScriptProperties().getProperty("SLACK_BOT_TOKEN");
  const apiResponse = callWebApi(token, "chat.postMessage", {
    channel: channel,
    text: "ハロハロー！！今月の進捗はコチラ！！",
    attachments: attachments
  });
}

function getCharts(sheetName) {
  const ss = SpreadsheetApp.openById("##SHEEET_ID##");
  const sheet = ss.getSheetByName(sheetName);
  return sheet.getCharts();
}

function getImageUrl(chart) {
  const today = Utilities.formatDate(new Date(), 'Asia/Tokyo', 'YYYY-MM-dd');

  const graphImg = chart.getBlob();
  const chartId = chart.getChartId();
  const folderId = '###FOLDER_ID###';
  const folder = DriveApp.getFolderById(folderId);
  const file = folder.createFile(graphImg);
  file.setName(`${today}_${chartId}`);

  // Slackに投稿する画像なので公開設定する
  file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW)
  const imageUrl = file.getDownloadUrl();
  return imageUrl;
}
```

# 💡 工夫したところ

## Twitter連携は共有アカウントで

写真投稿には共有用のTwitterアカウントを使っています。

各々がTwitterにシェアした写真をなにかしらの方法（特定のハッシュタグ等）で拾ってくる手もありますが、

- 投稿時にSwitchで追加で文字を打つのがとにかくめんどくさい
- 自分のアカウントで毎回投稿してTLを汚すのつらい

というペインがあります。

共有用のTwitterアカウントを連携しておけば、

1. ゲーム終了後に表示される「本日の運動結果」をスクショする
1. ギャラリーページから該当のスクショを選択してTwitterに投稿する

ただこれだけで作業は完了です。
実質、ギャラリーから決定ボタン連打するだけです。
これなら怠惰な人でも継続できます。きっと。

## ボディビルダー掛け声

ランダムでボディビルダーへの掛け声を叫んでくれます。100種類くらいあります。
ガチャ要素があると楽しいですよね。
![](https://storage.googleapis.com/zenn-user-upload/4480603a2fd74eb37065c6ab.png)

# 🤝 今後やっていきたいこと

## OCR改善
OCRの結果がたまーにミスってるときがあるので改善していきます。
Cloudinary等の画像加工サービスを使って余分な部分（アイコン等）を除去してからOCRを実行すればうまくいきそう。やっていき！ 💪
![](https://storage.googleapis.com/zenn-user-upload/060c0a6a0eaad25c3b7835c1.png)


# 🍣 おわりに

在宅ワークで最近ぜんぜん運動していないそこのあなた！
運動不足による疾病リスクは深刻です。
楽しく運動して一緒に健康的な生活を送りましょう！

ぼくが参加しているSlackコミュニティを貼っておきます。興味があればどうぞ。
https://community.camp-fire.jp/projects/view/280040

追伸：
Zenn初記事でしたがとても反響をいただけて嬉しかったです。
これからも個人開発や自動化についてゆるりと発信していこうと思います。
https://twitter.com/paranishian/status/1436089917491212294