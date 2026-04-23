---
name: memorizer
description: >
  コンテキスト管理スキル。作業コンテキストをトピック別ファイルに保存・ロード・一覧表示する。
  `/memorizer` で初期化、`/memorizer save [topic]` で保存、
  `/memorizer load <topic...>` でロード、`/memorizer list` で一覧。
---

# Memorizer: コンテキスト管理

## 設計原則

**1トピック = 1つの具体的な問題・機能・目的**

トピックは最小単位で管理し、必要なものを組み合わせてロードする。
コードのモジュールと同じ考え方。

```
✗ auth                     ← 広すぎ
✓ auth-jwt-refresh         ← リフレッシュトークン設計
✓ auth-provider-selection  ← プロバイダー比較・選定
✓ auth-session-storage     ← セッション保存方式
```

## データ構造

```
memory/contexts/
  index.md            — 全トピックの1行サマリー（list用インデックス）
  {topic}.md          — 現在の状態サマリー（上書き、常に小さく保つ）
  {topic}/
    context-log.md    — 現在有効な制約・決定の一覧（更新）
```

### index.md のフォーマット

```markdown
| topic | updated |
|-------|---------|
| {topic} | {date} |
```

### {topic}.md のフォーマット

```markdown
---
topic: {topic}
updated: {date}
depends_on:
  - {topic-a}
  - {topic-b}
---

## 現在の状態
（1〜3行）

## 決定事項
- ...

## 次のアクション
- ...
```

`depends_on` は省略可。依存トピックのロード時にそのコンテキストも合わせて読み込まれる。

**サイズルール：** 各セクション最大5項目。完結した決定はcontext-logに移してサマリーから削除する。

### context-log.md のフォーマット

```markdown
## {date} {タイトル}
{調査・決定の内容}
→ {結果・理由・影響}
```

---

## context-log に記録する基準

**記録する（プロジェクト内側にしか存在しない情報）:**
- このプロジェクト固有の制約（予算・チーム・既存インフラ）
- 選択肢を比較してXを除外した理由
- 試して失敗したこと・その原因
- ステークホルダーの意思決定とその背景
- プロジェクト内で発見した仕様・挙動の特殊性

**記録しない（プロジェクト外側にある再検索可能な情報）:**
- 公開ドキュメントに書いてある情報（インストール方法・APIの使い方）
- 一般的なベストプラクティス
- Stack Overflow等で再現できるエラーの解決策

---

## `/memorizer`（引数なし）— 初期化

### Step 1 — index.md の確認

`memory/contexts/index.md` を Read する。

### Step 2 — 宣言

`index.md` が存在してトピックがある場合、一覧を表示する：

```
利用可能なコンテキスト:
- {topic}  ({updated})
```

`index.md` が存在しない、またはトピックがない場合は「コンテキストなし」と表示する。

---

## `/memorizer save [topic]` — 保存

### Step 1 — トピック決定

`topic` が指定されていない場合、会話の内容からトピック名を推定する（英小文字・ハイフン区切り、例: `auth-refactor`）。
既存トピックへの追記か新規作成かを判断する。

### Step 2 — サマリー上書き

現在の作業内容を要約して `memory/contexts/{topic}.md` を上書きする。
完結した決定はサマリーから削除し、context-logに移す（サマリーを小さく保つ）。

### Step 3 — index.md 更新

`memory/contexts/index.md` の該当トピック行を更新する（存在しなければ追加）。
`現在の状態` セクションの1行目を使う。

### Step 4 — context-log 更新

`memory/contexts/{topic}/context-log.md` を以下のルールで更新する：
- 今回の会話で「記録する基準」に該当する内容があれば追記する
- 解決済み・無効になったエントリは削除する
- 該当する内容がなければスキップする

完了を報告する。

---

## `/memorizer load <topic...>` — ロード

複数トピックを同時にロードできる。

```
/memorizer load ckeditor-html-embed ckeditor-license
```

### Step 1 — ファイル確認

指定された各トピックについて `memory/contexts/{topic}.md` が存在するか確認する。
存在しないトピックは「コンテキスト `{topic}` が見つかりません」と報告してスキップ。

### Step 2 — 依存関係の解決

各トピックの `depends_on` を確認し、依存トピックを再帰的にロード対象に追加する。
循環依存が検出された場合は警告して該当ループをスキップする。
ロード順は依存元 → 依存先の順（依存先を先に読む）。

### Step 3 — ロードと表示

解決済みの全トピックの `{topic}.md` を Read する。`{topic}/context-log.md` はこの時点では読まない（ユーザーが必要と判断したタイミングで読む）。
全トピックの内容をまとめて3〜5行で要約して表示する。ロードしたトピック一覧も表示する。

---

## `/memorizer list` — 一覧表示

`memory/contexts/index.md` を Read して以下の形式で表示する：

```
{topic}  ({updated})
```

`index.md` が存在しない場合は「インデックスがありません。`/memorizer save` で作成してください。」と表示する。
