# Claude Code Forge

## 概要

everything-claude-code の設定ファイル（skill / agent / rule）の記載パターンをテンプレート化した Claude Code プラグイン。
ユーザーは「やりたいこと」を述べるだけで、適切な種別の設定ファイルが生成・リファクタされる。

## 機能一覧

| スラッシュコマンド | 暗黙トリガ | 用途 |
|-----------------|-----------|------|
| `/forge:new` | 「〇〇したい」「〇〇が欲しい」 | 新しい skill / agent / rule を追加する |
| `/forge:refactor` | 「この設定を整えたい」 | 既存の設定ファイルをテンプレート準拠に書き換える |

## 使い方

要望をそのまま述べるだけで良い。内部で planner agent が種別を判定し、適切な author agent に生成を委譲する。

```
ユーザー:「コミット前にシークレット混入を防ぐ仕組みが欲しい」
       → planner が rule + agent の組合せと判定
       → rule agent が .claude/rules/commit-secret-check.md を生成
       → agent agent が .claude/agents/secret-scanner.md を生成

ユーザー:「この SKILL.md、中身が agent っぽいんだけど整えて」
       → /forge:refactor が起動
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
│   ├── new/SKILL.md      # 新規追加のエントリ（planner → author）
│   └── refactor/SKILL.md # 既存書き換えのエントリ（planner → author）
└── rules/
    ├── skills.md   # .claude/skills/**/SKILL.md にパススコープで適用
    ├── agents.md   # .claude/agents/*.md にパススコープで適用
    └── rules.md    # .claude/rules/*.md にパススコープで適用
```

設計の核心は **「判定は planner・執筆は author・境界は rule」** の責務分離。author が書き換え中は path-scoped な `rules/*.md` が自動適用され、テンプレート違反を二重に防ぐ。

## インストール

Claude Code セッション内で以下を実行する。

```
/plugin marketplace add sakaguchi-0725/claude-code-forge
/plugin install forge@claude-code-forge
```

インストール後は `/forge:new` / `/forge:refactor` が使える。

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
