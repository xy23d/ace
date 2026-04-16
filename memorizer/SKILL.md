---
name: memorizer
description: >
  セッション管理スキル。会話のサマリーをファイルに保存・切り替え・一覧表示する。
  `/memorizer` で初期化、`/memorizer switch <id>` で切り替え、
  `/memorizer save` で手動保存、`/memorizer list` で一覧表示。
---

# Memorizer: セッション管理

## データ構造

- `memory/conversations/{id}.md` — セッションの最新サマリー（上書き）。通常はこちらを参照する
- `memory/conversations/{id}/log.md` — 時系列の詳細ログ（追記のみ）。過去の経緯をすべて確認したい場合のみ読み込む
- `memory/current_session.md` — 現在のセッションIDを記録するファイル
- `memory/mapping.md` — 全セッションのインデックス（ID・日付・トピック・サマリー）

## セッションIDの仕様

- 英小文字・数字を使った **8文字** のランダム文字列（例：`f3k9m2p8`）
- 生成後に `memory/conversations/` に同名ファイルが存在しないか確認し、衝突する場合は再生成する

---

## `/memorizer`（引数なし）— 初期化

以下を順番に実行する。

### Step 1 — 既存cronの削除

CronList で実行中のサマリー保存ジョブを確認し、すべて CronDelete で削除する。

### Step 2 — セッションID生成

新しいセッションIDを生成して `memory/conversations/` に衝突がないことを確認する。

### Step 3 — current_session.md の書き込み

以下の内容で `memory/current_session.md` を上書きする：

```
{id}
```

### Step 4 — cronの起動

CronCreate で以下のジョブを作成する：
- cron: `*/30 * * * *`
- prompt: `会話のサマリーをmemory/conversations/{id}.mdに保存して、memory/conversations/{id}/log.mdに今回分を追記して、memory/mapping.mdにそのIDがなければ追記して, ただし、前回の保存以降にやりとりがない場合はスキップする`
- recurring: true

### Step 5 — 宣言

「このセッションのID: `{id}`」と宣言する。

なお、コード等のアウトプットがある場合のみ `outputs/{id}/` に出力する。

---

## `/memorizer save` — 手動保存

### Step 1 — 現在IDの取得

`memory/current_session.md` を Read して現在のセッションIDを取得する。

### Step 2 — サマリー上書き

会話の内容を要約して `memory/conversations/{id}.md` に**上書き**する。

### Step 3 — ログ追記

`memory/conversations/{id}/log.md` に以下の形式で**追記**する：

```
## {datetime}

{今回の保存範囲のやりとりの詳細サマリー}
```

### Step 4 — mapping更新

`memory/mapping.md` にそのIDが存在しなければ1行追加する。

フォーマット：`| {id} | {date} | {topic} | {summary} |`

完了を報告する。

---

## `/memorizer switch <id>` — セッション切り替え

### Step 1 — ファイル確認

`memory/conversations/{id}.md` が存在するか確認する。
存在しない場合は「セッション `{id}` が見つかりません」と伝えて終了。

### Step 2 — 現在セッションの保存

`/memorizer save` の手順と同じ内容で、現在のセッションを保存する。

### Step 3 — current_session.md の更新

`memory/current_session.md` を指定された `{id}` で上書きする。

### Step 4 — cronの更新

CronList で現在のジョブを CronDelete で削除し、新しいIDで CronCreate する（初期化のStep 4と同じ要領）。

### Step 5 — ロードと宣言

`memory/conversations/{id}.md` を Read して内容を把握する。

「このセッションのIDを `{id}` に切り替えました」と宣言し、ロードしたセッションの概要を3〜5行で表示する。

---

## `/memorizer list` — 一覧表示

`memory/mapping.md` を Read して、以下の形式で1セッション1行で表示する：

```
`{id}`  {date}  {topic}
  {summary}
```

---

## `/memorizer current` — 現在のセッションID表示

`memory/current_session.md` を Read して現在のセッションIDを表示する。

出力形式：
```
現在のセッションID: `{id}`
```
