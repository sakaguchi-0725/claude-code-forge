# Claude Code Forge

## 概要

everything-claude-code の設定ファイル（skill / agent / rule）の記載パターンをテンプレート化した Claude Code プラグイン。
ユーザーは「やりたいこと」を述べるだけで、適切な種別の設定ファイルが生成・リファクタされる。

## 機能一覧

| スラッシュコマンド | 用途 | planner 経由 |
|-----------------|------|-------------|
| `/forge:skill` | skill が必要と確信しているとき | × 直接 |
| `/forge:agent` | agent が必要と確信しているとき | × 直接 |
| `/forge:rule` | rule が必要と確信しているとき | × 直接 |
| `/forge:new` | 種別が不明 / 複数種別の組合せが必要なとき | ○ planner |
| `/forge:refactor` | 既存設定をテンプレ準拠に書き換える（種別検証必須） | ○ planner |

## 使い方

種別が決まっているなら直接コマンドが最短。判定が必要なときだけ `/forge:new` を使う。

```
# 種別確定 → planner をスキップ
ユーザー:「Go の N+1 回避手順を skill として共有したい」
       → /forge:skill
       → skill agent が .claude/skills/go-n-plus-one/SKILL.md を生成

ユーザー:「PR レビュー担当の agent を追加したい」
       → /forge:agent
       → agent agent が .claude/agents/pr-reviewer.md を生成

# 判定が必要 → planner 経由
ユーザー:「コミット前にシークレット混入を防ぐ仕組みが欲しい」
       → /forge:new
       → planner が rule + agent の組合せと判定し、各種別の「含める内容 / 除外する内容」を分配
       → rule agent が .claude/rules/commit-secret-check.md を生成（手順は除外）
       → agent agent が .claude/agents/secret-scanner.md を生成（コード例は除外）

# 既存ファイルの整形
ユーザー:「この SKILL.md、中身が agent っぽいんだけど整えて」
       → /forge:refactor
       → planner が種別ずれを検出しユーザーに確認
       → 承認後、agent agent が agents/ 配下に新規生成、旧ファイル削除
```

## 内部構造

```
claude-code-forge/
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   ├── planner.md    # 種別判定（opus）
│   ├── skill.md      # skill 執筆（sonnet）
│   ├── agent.md      # agent 執筆（sonnet）
│   └── rule.md       # rule 執筆（sonnet）
├── skills/
│   ├── skill/SKILL.md    # /forge:skill -- skill author 直叩き
│   ├── agent/SKILL.md    # /forge:agent -- agent author 直叩き
│   ├── rule/SKILL.md     # /forge:rule  -- rule author 直叩き
│   ├── new/SKILL.md      # /forge:new   -- planner → author（曖昧 / 組合せ用）
│   └── refactor/SKILL.md # /forge:refactor -- planner → author（種別検証必須）
└── rules/
    ├── skills.md   # .claude/skills/**/SKILL.md にパススコープで適用
    ├── agents.md   # .claude/agents/*.md にパススコープで適用
    └── rules.md    # .claude/rules/*.md にパススコープで適用
```

設計の核心は **「判定は planner・執筆は author・境界は rule」** の責務分離。種別が確定しているときは planner をスキップして author を直接呼び、judging cost をゼロにする。種別が曖昧 / 組合せが必要なときだけ planner が「採用種別 + 各種別への内容分配 + 除外項目」を出力し、author はその境界を厳守する。author が書き換え中は path-scoped な `rules/*.md` が自動適用され、テンプレート違反を二重に防ぐ。

## インストール

Claude Code セッション内で以下を実行する。

```
/plugin marketplace add sakaguchi-0725/claude-code-forge
/plugin install forge@claude-code-forge
```

インストール後は `/forge:skill` / `/forge:agent` / `/forge:rule` / `/forge:new` / `/forge:refactor` が使える。

## ローカルで試す（開発用）

リポジトリをクローンしてパス指定で起動。

```bash
git clone https://github.com/sakaguchi-0725/claude-code-forge
claude --plugin-dir ./claude-code-forge
```

## 設定テンプレートの種別

| 種別 | 役割 | 行数目安 | 書くもの |
|------|------|---------|---------|
| rule | ガードレール（常にやる / 絶対やらない） | 20〜80 行 | ALWAYS / NEVER / MUST の宣言 |
| agent | ペルソナと判断の重み付け | 100〜300 行 | 「あなたは〜です」+ CRITICAL/HIGH/MEDIUM または Phase または DO/DON'T |
| skill | 詳細手順とリファレンス知識 | 200〜600 行 | When to Activate + 手順 + PASS/FAIL コード例 |

詳細なテンプレート知識は `agents/skill.md` `agents/agent.md` `agents/rule.md` にインライン化されている（planner が参照）。
