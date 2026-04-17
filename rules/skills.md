---
paths:
  - ".claude/skills/**/SKILL.md"
---

# Skill 作成ルール

## 構造

- ALWAYS `When to Activate` セクションでトリガー条件を明示する
- ALWAYS 手順には実行可能な bash コマンドを含める
- ALWAYS コード例は PASS / FAIL ラベル付きで対比させる
- ALWAYS 末尾を `**Remember**:` の哲学行で締める

## スコープ

- NEVER ペルソナを定義する（→ agent に書く）
- NEVER ALWAYS / NEVER で制約を宣言する（→ rule に書く）
- NEVER 重要度の判定基準（CRITICAL / HIGH / MEDIUM）を書く（→ agent に書く）
- NEVER 薄い内容で終わらせる（skill は厚く書いてこそ価値がある）

## 関連

- **skill** agent -- skill の生成・編集を担当（構造テンプレート・ハーネス系パターンの詳細）
- **planner** agent -- skill / agent / rule のどれを作るかの判断が必要なとき
