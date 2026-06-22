# いとつむぎ — 開発コンテキスト（Claude Code 引き継ぎ用）

このファイルは `index.html`（単一ファイル PWA）の現状の設計・仕様をまとめたもの。
実装は Claude Code で行う。

---

## 1. プロダクト概要
一日のかけらをアイコンで簡易記録する日記 PWA。Android Chrome で「ホーム画面に追加」して使う。
- 完全無料・広告なし・**外部送信なし**（データは端末内のみ）
- 1日に何度でも記録、複数アイコン選択、過去日にも記録可
- 記録パレット（アイコン＋ラベル）はユーザーが編集可能

## 2. 技術スタック / 制約（変更しない前提）
- **単一 `index.html`**（HTML/CSS/JS 全部入り）＋ `manifest.webmanifest` ＋ `sw.js` ＋ アイコン PNG。
- 保存は **IndexedDB**。`navigator.storage.persist()` で消えにくくする。
- **ローカルのみ**（Drive 等は使わない）。ただし保存処理は `Store` オブジェクトに集約し、将来差し替え可能にしてある。**この抽象化は維持すること。**
- Android の Web では `showSaveFilePicker` 等が使えないため、バックアップは **JSON 書き出し／読み込み（手動）** で行う設計。これは前提として動かさない。
- PWA 機能（インストール・永続化・SW）は **HTTPS か localhost が必須**。`file://` 不可。
- 依存ライブラリなし（バニラ JS）。この方針を維持。

> **なぜ単一 `index.html`：** ビルド工程ゼロ・依存ゼロで「腐らない」ことが目的。(1) 静的ファイルを置くだけで配信できる、(2) フレームワーク陳腐化の影響を受けず長寿命、(3) トランスパイル無しで debug が素直（行番号がソースと一致）、(4) 全体を一望でき Claude Code にも丸ごと渡せる。**次の分割はバンドラではなく、まずビルド不要のネイティブ ES モジュール**（`<script type="module">` ＋ `import` で `store.js` 等へ）。minify やコンポーネント化が要るようになって初めて Vite 等を検討する。

## 3. デザイン言語
- コンセプト：「一日の小さな瞬間が連なる**糸**」。記録は縦の糸（タイムライン）に点として連なる。
- 配色は `index.html` 冒頭の CSS 変数。`:root`（ライト）と `[data-theme="dark"]`（ダーク）。`--accent` はモス green `#5c7c6e`。
- 日付見出しのみ**明朝**（`--serif`）、本文は**ゴシック**（`--sans`）。
- ライト/ダークは `theme = auto|light|dark`、`auto` は `prefers-color-scheme` 追従。
- モーダルは下からせり上がる sheet（`.scrim` > `.sheet`）。`.sheet-head` が `position:sticky` で固定、`.sheet-body` にコンテンツ。
- トースト、`prefers-reduced-motion` 対応済み。新規 UI も同じ部品・トークンを使う。
- XSS 対策：ユーザー入力の表示には必ず `escapeHtml()` を使う。

## 4. データモデルと Store API

```js
// entry: { id, ts(number ms), icons:[ {emo, lbl, genre} ], note }
// palette item: { id, emo, lbl, genre }   // genre: "mood" | "health" | "life"
// meta.genres: [ {key, label}, ... ]      // 既定 = mood/health/life の3件
// meta.schema: 2                          // マイグレーション管理
// meta.weekStart: 0|1                     // 0=日曜, 1=月曜
// meta.theme: "auto"|"light"|"dark"
// meta.reminderDays: number
// meta.lastBackup: number(ms)
```

Store のメソッド（このインターフェースは維持）:
`init / addEntry / putEntry / deleteEntry / entriesInRange(startMs,endMs) / allEntries / clearAll / getMeta(k,fb) / setMeta(k,v)`

**マイグレーション**（起動時に1回、`meta.schema` で冪等化）:
- v1 entry（`emo/lbl` があり `icons` 無し）→ `icons:[{emo,lbl,genre:null}]` に変換。
- v1 palette item（`genre` 無し）→ `genre:"life"` を暫定付与。
- 完了後 `meta.schema = 2` を保存。

**書き出し JSON（v2）**：`version:2`、`palette`、`genres`、`entries`（icons[]）、`settings`（theme / reminderDays / weekStart）。UI 状態（アコーディオン開閉など）は含めない。
**読み込み**：v1・v2 両対応。マージ（id 重複スキップ）／全置換は確認ダイアログで選択。

---

## 5. 画面構成

### 下部タブバー
`［カレンダー｜ホーム｜絞り込み］`。設定は右上 ⚙ ボタン。

### ホーム（紡いだ糸）
- 全記録を新しい順に表示。日付が変わる位置に日付ディバイダ。
- 初期10件、「もっと見る」で +10 ずつ。
- 絞り込み中は「N件中M件を表示中」を表示。

### カレンダー
- 月表示・月送り ‹ ›。タイトルをタップすると年月ジャンプシート。
- 週始まりは `meta.weekStart`（0=日曜, 1=月曜）に従う。
- 各セルにその日の最後の記録のアイコン（最大3個小さく）。
- 日をタップ → 日ビュー（その日の記録一覧）。日ビューには絞り込みが適用される。カレンダーセル自体は絞り込みを適用しない（全記録を表示）。

### 記録フォーム
- ジャンルごとのアコーディオン（`<details>`）でアイコンを選択・トグル。複数選択可。
- 下部に「記録」ボタン。押すと選択アイコン全部で1件作成。
- 「日時・メモ」アコーディオンで任意日時・メモを指定可能。

### 絞り込みシート
- ジャンルチェック＋個別アイコン選択（`.chip`）。OR 判定。
- バッジはタブバーの🔍アイコン右上のドット（`::after` 疑似要素）。

### 設定シート
- テーマ・週の始まり・記録パレット（ジャンルアコーディオン、↑↓並べ替え・削除・追加）。
- データ（書き出し・読み込み・バックアップ催促間隔・永続化依頼）。
- 全データリセット。

### 編集シート
- アイコン選択（ジャンルアコーディオン。選択済みジャンルは自動展開）・時刻・メモ。
- 保存・この記録を削除。

## 6. アイコン選択 UI（`.chip`）
記録フォーム・絞り込みシート・編集シートの3箇所で共通の `.chip` クラスを使用。
```css
.chip { display:flex; flex-direction:column; align-items:center; ... }
.chip.sel { background:var(--accent-soft); border-color:var(--accent); }
```

## 7. ジャンル設計
ジャンルはデータ駆動（`meta.genres`）。現在は固定3ジャンルで運用:
- **気分 (mood)**：楽しい😄／穏やか🙂／普通😐／眠い😴／ややつらい😟／つらい😣
- **体調 (health)**：頭痛🤕／腹痛😖／生理🩸／服薬💊／疲労😮‍💨／眠気😪
- **生活 (life)**：制作✍️／遊び🎮／休息☕／お出かけ🚶／運動🏃／買い物🛒

ジャンルの増減 UI（CRUD）は将来対応。データ側は `icons[]` に genre をスナップショットで持つため、後から変更しても過去記録は壊れない。

## 8. 運用メモ
- アプリ更新時は `sw.js` の `CACHE = "itotsumugi-vN"` の版番号を上げる（旧キャッシュ破棄）。
- アイコン差し替えは `icon-*.png` を同名で。
- 配信は HTTPS（自サイト FTP か GitHub Pages）。
- 著作権：発想（アイコン日記）は保護対象外。素材・コードは自作のものを使う（他アプリの素材/コード/固有デザインの流用はしない）。
