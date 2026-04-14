# 月次BS更新：貸借対照表 & KPI自動入力

毎月のルーティン。各種データソースからデータを収集し、貸借対照表スプシに書き込むGASコードを生成する。

## 前提

### 環境
- Chromeで以下にログイン済みであること：
  - SBIネット銀行（https://www.netbk.co.jp）
  - アメリカン・エキスプレス（https://www.americanexpress.com/ja-jp/）
  - Carousell（https://www.carousell.sg）
- Chrome → 表示 → デベロッパー → 「Apple EventsからのJavaScriptを許可」がONであること

### 事業構造（専門家AI分析で必ず考慮すること）

**Carousell（シンガポール）= 無在庫販売**
- 在庫を持たずに出品→売れてから仕入れる方式
- 在庫リスクなし。利益率は仕入れ原価＋送料＋手数料で決まる
- いのっちが運営

**eBay = 有在庫販売（複数アカウント体制）**
- メインアカウント：現在**販売停止中**（アカウント問題）
- 山田さんアカウント：
  - 契約：粗利の**4%**をいのっちが受け取る
  - いのっちの担当：出品、顧客対応、ラベル発行
  - 山田さんの担当：仕入れ、検品、発送
- 松本さんアカウント：**準備中**
  - 契約：営業利益を**折半**
  - いのっちの担当：仕入れ資金の提供のみ
  - 松本さんの担当：運用全般
- 田中さんアカウント：**準備中**
  - 契約：営業利益を**折半**
  - いのっちの担当：仕入れ資金、リサーチ、出品
  - 田中さんの担当：それ以外の運用
- 今後スプシ等を共有予定。アカウントが立ち上がれば売上が追加される

**分析時の注意点：**
- eBayが2月以降ゼロなのはメインアカ停止が原因。「市場の問題」ではない
- 山田さんアカは1月に310万円売上実績あり。再開すれば同水準が見込める
- 松本・田中アカが立ち上がれば、仕入れ資金提供＋利益折半で売上は増えるが利益率は変わる構造
- Carousellは無在庫なので「在庫回転率」の概念は当てはまらない。在庫67個はeBay用

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

### 10. 専門家AI分析（GASコード実行後に実施）

GASでの書き込みが完了した後、以下の7人の専門家AIの視点から今月のBS・KPIを分析・評価する。過去月のデータと比較し、トレンドや課題を指摘する。

**7人の専門家ペルソナ：**

1. **CEO（経営者視点）**
   - 資産全体の推移、純資産の増減トレンド
   - 事業全体の方向性は正しいか
   - 「このまま行くと半年後にどうなるか」のシミュレーション

2. **eBay専門家（越境EC視点）**
   - eBay売上・保留金の推移
   - 為替リスクの評価
   - 出品数 vs 成約率の分析

3. **経理・財務（会計視点）**
   - BS（貸借対照表）のバランス評価
   - 未払金（アメックス）と口座残高の比率
   - キャッシュフローの健全性
   - 在庫回転率の評価

4. **リスク管理（守りの視点）**
   - 在庫金額が資産に占める割合（過剰在庫リスク）
   - 売掛金の回収リスク
   - 為替変動が利益に与えるインパクト
   - 単一プラットフォーム依存度

5. **AI効率化担当**
   - 今月の自動化成果
   - KPIの改善ポイントとClaude活用の提案
   - 次に自動化すべき業務の優先順位

6. **投資家視点**
   - ROI（投下資本に対するリターン）
   - 成長率と利益率のバランス
   - スケールする可能性のある領域

7. **リアリスト（現実主義者）**
   - 数字だけを見た冷静な評価
   - 見落としている問題点
   - 「これだけは今月中にやれ」の1つ

**分析のフォーマット：**
各専門家ごとに以下を簡潔に出力：
- 📊 今月の評価（一言）
- 📈 良い点
- ⚠️ 懸念点
- 💡 提案（1〜2つ）

最後に全専門家の意見を踏まえた **総合評価** と **今月のアクションアイテムTOP3** をまとめる。

### 11. レポートHTML生成 & Google Drive保存

専門家分析まで完了したら、全データ・分析・アクションアイテムを含むHTMLレポートを生成する。

- ファイル名: `bs_report_YYYYMM.html`（例: bs_report_202404.html）
- 保存先（ローカル）: `/Users/tkc2525/Documents/claude_offkai_gas/`
- 保存先（Drive）: `/Users/tkc2525/Library/CloudStorage/GoogleDrive-inotake.s3@gmail.com/マイドライブ/2.eBay事業_猪川/06_業務効率化/`
- 右上に「PDF保存/印刷」ボタンを配置
- レポートに含める内容:
  - KPI概要（口座残高・総資産・負債・純資産）
  - 貸借対照表（売掛金の内訳付き）
  - 月次売上推移（年初からの全月 + 前月比）
  - 販売ルート別推移（Carousell/eBay/メルカリ/その他）
  - Carousell KPI（前月比付き）
  - 当月着地予測
  - 7人の専門家AI分析
  - アクションアイテムTOP3
- 過去のレポートと比較できるよう、Drive上に月次で蓄積していく
