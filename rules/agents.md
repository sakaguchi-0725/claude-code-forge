---
paths:
  - ".claude/agents/*.md"
---

# Agent 作成ルール

## 構造

- ALWAYS 冒頭を「あなたは〜です」でペルソナ定義から始める
- ALWAYS レビュー系は CRITICAL → HIGH → MEDIUM → LOW の重要度順で構造化する
- ALWAYS 出力フォーマットを明示する
- ALWAYS `tools` フロントマターは最小権限で指定する
- ALWAYS 末尾を `**Remember**:` の哲学行で締める

## スコープ

- NEVER 詳細なコード例を載せる（→ skill に書く）
- NEVER 制約・規約を定義する（→ rule に書く）
- NEVER 手順書のように書く（→ skill に書く）
- NEVER tools に不要なものを含める（最小権限の原則）

## 関連

- **agent** agent -- agent の生成・編集を担当（3 バリエーション・model 選択基準の詳細）
- **planner** agent -- skill / agent / rule のどれを作るかの判断が必要なとき
