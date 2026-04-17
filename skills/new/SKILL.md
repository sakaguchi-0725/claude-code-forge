---
name: new
description: 新しい設定（skill / agent / rule）を追加したいとき。要望を解析して適切な種別を判定し、対応する author agent に生成を委譲する。
---

# new

ユーザーの「〜したい」という要望から skill / agent / rule のどれを作るべきか判断し、該当する author agent を呼び出して生成するハーネス系 skill。

## When to Activate

- ユーザーが `/forge:new` を実行したとき
- 「新しい skill / agent / rule を追加したい」と明示したとき
- 「コミット前チェックが欲しい」「PR レビュー担当がほしい」「N+1 回避の手順を共有したい」など、設定で解決できる要望を述べたとき
- 複数種別を組み合わせた体制構築（例: セキュリティレビュー一式）を求められたとき

## 設計思想

種別判定（planner）と執筆（author agents）を分離する。分離により:

- planner は種別変更を柔軟に提案できる（執筆者は種別固定になる）
- 各 author agent は自分の種別の責務だけに集中できる
- 組合せ生成（rule + agent + skill セット）も planner が依存関係を決め、author に順次渡せる
- 生成時は path-scoped な `rules/*.md` が自動適用され、テンプレート違反を別レイヤーで防ぐ

## アーキテクチャ

```
ユーザー要望
    │
    ▼
[new skill]（本 skill）
    │
    ▼
[planner agent] -- 種別判定 + 生成順序決定
    │
    ├── rule  ──▶ [rule agent]  ──▶ .claude/rules/{name}.md
    ├── agent ──▶ [agent agent] ──▶ .claude/agents/{name}.md
    └── skill ──▶ [skill agent] ──▶ .claude/skills/{name}/SKILL.md
                                          │
                                          ▼
                          path-scoped [rules/*.md] が自動適用
```

## 配置先

| 種別 | デフォルトパス |
|------|---------------|
| skill | `.claude/skills/<kebab-name>/SKILL.md` |
| agent | `.claude/agents/<kebab-name>.md` |
| rule | `.claude/rules/<kebab-name>.md` |

プロジェクト固有の配置規約がある場合は planner の判定後にユーザーへ確認する。

## 設定

| 環境変数 | デフォルト | 説明 |
|---------|----------|------|
| FORGE_MAX_COMBO | 3 | 1 回の要望で生成を許す最大種別数（rule + agent + skill） |
| FORGE_CONFIRM_COMBO | true | 組合せ生成時にユーザー確認を挟むか |

## ワークフロー

### Step 1: 要望の受け取り

ユーザーの要望をそのまま受け取る。明確化質問はこの skill では最小限に留め、判断の詳細は planner agent に委譲する。

```bash
# ユーザー発話例
# 「コミット前にシークレット混入をチェックする仕組みを入れたい」
# 「Go の N+1 回避手順を skill として共有したい」
# 「PR レビュー担当の agent を追加したい」
```

### Step 2: planner agent に委譲

planner agent に要望を渡して種別判定を行わせる。

```bash
# 入力: ユーザー要望の全文 + 既存の .claude/ ディレクトリ構造
# 出力:
#   - 種別（単一 or 組合せ）
#   - 各ファイルの命名案
#   - 生成順序（rule → skill → agent の依存順）
#   - 根拠（3 軸スコア）
```

planner 返答の読み取り方:

- 単一種別: 該当 author agent へそのまま委譲
- 組合せ: `FORGE_CONFIRM_COMBO=true` ならユーザーに内訳を提示して承認を取る
- スコア拮抗: planner が候補を併記してきたらユーザーに選択を委ねる

### Step 3: author agent に順次委譲

planner の生成順序に従い、対応する author agent を順に呼び出す。

| 生成対象 | 担当 agent | 入力 |
|---------|-----------|------|
| rule | `rule` agent | 配置先パス + 制約内容 + スコープ |
| skill | `skill` agent | 配置先パス + 主題 + 通常型/ハーネス系の希望 |
| agent | `agent` agent | 配置先パス + バリエーション指示（レビュー/プランナー/リゾルバー） |

各 author は書き換え中に path-scoped な `rules/*.md`（例: `rules/skills.md` は `.claude/skills/**/SKILL.md` 編集時に発動）が自動適用されるため、テンプレート違反を二重に防げる。

### Step 4: 結果レポート

生成したファイル一覧を集約してユーザーに報告する。

```
## 生成結果

### 作成したファイル
1. .claude/rules/{name}.md    -- ALWAYS/NEVER 各 n 件
2. .claude/skills/{name}/SKILL.md -- 通常型, PASS/FAIL x 件
3. .claude/agents/{name}.md   -- バリエーション X, model: sonnet

### 次に確認すべきこと
- {テンプレート違反がないか、関連ファイルとのクロスリファレンス漏れ}
```

## エラーハンドリング

- **planner 判定が曖昧（スコア拮抗）**: 候補を提示してユーザー選択を待つ。無断で片方を採用しない
- **author agent 生成失敗**: 失敗したファイルを明示。既に生成済みのファイルは残す（MVP: 手動で掃除）。最大リトライは 1 回
- **既存ファイルと衝突**: 上書きせず、パスの変更または `/forge:refactor` への切替を提案する
- **FORGE_MAX_COMBO 超過**: planner が 4 種別以上を提案しても `FORGE_MAX_COMBO` で頭打ちし、残りはユーザーに次回要望として提示

## アンチパターン

- **1 要望に対して機械的に 3 点セット生成**: 本当に必要なものだけ作る。planner が必要と判定したもののみ
- **planner をスキップして author を直接呼ぶ**: 種別判定の一貫性が失われる。必ず planner を経由する
- **生成中に path-scoped rule を無視**: 自動適用されるので手動で再確認する必要はない。二重に制約をかけない
- **既存資産を確認せず新規作成**: planner の Phase 1 で既存 agent / skill / rule を Grep / Glob で必ず確認する

---

**Remember**: 判定は planner に、執筆は author に、境界は rule に委ねる。本 skill は繋ぎ役に徹する。
