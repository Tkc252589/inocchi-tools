# 月次BS更新：貸借対照表 & KPI自動入力

毎月のルーティン。各種データソースからデータを収集し、貸借対照表スプシに書き込むGASコードを生成する。

## 前提
- Chromeで以下にログイン済みであること：
  - SBIネット銀行（https://www.netbk.co.jp）
  - アメリカン・エキスプレス（https://www.americanexpress.com/ja-jp/）
  - Carousell（https://www.carousell.sg）
- Chrome → 表示 → デベロッパー → 「Apple EventsからのJavaScriptを許可」がONであること

## 書き込み先スプシ
- 貸借対照表: https://docs.google.com/spreadsheets/d/118D1t-XMxr6fVttUgEKyjgQzQPEzJ6rlpGxpFYybGlg/edit?gid=1804692462#gid=1804692462
- スプシID: 118D1t-XMxr6fVttUgEKyjgQzQPEzJ6rlpGxpFYybGlg

## セル位置
- C4: 口座残高（SBI）
- C5: 在庫金額
- C7: 売掛金（eBay保留 + Carousell当月売上×レート）
- E5: 未払金（アメックス）
- C30: Carousell出品数
- C32: Carousell平均単価（SGD）
- C34: Carousell粗利率（%）
- C35: Carousell月商（万円）
- C36: Carousell成約件数
- C39: 国内 仕入原価
- C40: 国内 回転率
- C41: 国内 粗利

## 手順

### 1. せどり管理ツールからデータ取得
スプシURL: https://docs.google.com/spreadsheets/d/1ZJ3t06_69LpmzSFisCkk-vvQBSY45qp1EAoX942GJIc/

- TOPタブ（gid=377376737）をCSVエクスポートで取得
  - 在庫数: H11相当
  - 在庫金額: I11相当
- 販売タブ（gid=419093705）をCSVエクスポートで取得
  - AB列（27列目）が日付、I列（8列目）が販売ルート
  - 当月 + Carousell → N列（13列目）のSGD売上を合計
  - 当月 + メルカリ/その他 → 売上(11列目)、原価(28列目)、粗利(34列目)を集計
  - 件数、平均単価、月商、粗利率を計算

### 2. 為替レート取得
- exchangerate-api.com から SGD/JPY と USD/JPY を取得
- curl -s "https://api.exchangerate-api.com/v4/latest/SGD" でSGD→JPY
- curl -s "https://api.exchangerate-api.com/v4/latest/USD" でUSD→JPY

### 3. eBay保留金額
- 固定値: $14,438.12 USD
- ※変更がある場合はユーザーに確認する

### 4. SBIネット銀行の残高取得
ChromeのAppleScript経由でログイン済みページからテキスト抽出：
- 全ウィンドウ・全タブから「SBI」「住信」を含むタブを探す
- document.body.innerTextから「円普通」の後の金額を抽出
- キーワード: 「残高」「円」

### 5. アメックス未払金取得
ChromeのAppleScript経由：
- 全ウィンドウ・全タブから「American Express」「アメリカン」を含むタブを探す
- なければ https://www.americanexpress.com/ja-jp/account/login を開く
- 「新規ご利用金額」の後の「￥」付き金額を抽出

### 6. Carousell出品数取得
ChromeのAppleScript経由：
- https://www.carousell.sg/manage-listings?source=me_page を開く
- 「Active」の前の数字を抽出（例: 「4190 Active」→ 4190）

### 7. 売掛金の計算
売掛金 = eBay保留金額 × USD/JPYレート + Carousell当月売上SGD × SGD/JPYレート

### 8. GASコード生成
上記すべてのデータを使って、以下のGASコードを生成しクリップボードにコピー：

```javascript
function updateBS() {
  var ss = SpreadsheetApp.openById('118D1t-XMxr6fVttUgEKyjgQzQPEzJ6rlpGxpFYybGlg');
  var sheet = ss.getSheetByName('貸借対照表') || ss.getSheets()[0];
  // 各セルに値をsetValue
}
```

### 9. 報告
収集したデータのサマリーを報告：
- 貸借対照表の各項目
- KPIの各項目
- 前月との比較（可能であれば）
- 為替レート
