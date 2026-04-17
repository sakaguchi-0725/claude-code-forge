---
paths:
  - ".claude/rules/*.md"
---

# Rule 作成ルール

## 構造

- ALWAYS `ALWAYS` / `NEVER` / `MUST` を大文字で使う
- ALWAYS `ALWAYS` / `NEVER` / `MUST` を **行頭** に置く（文中に埋め込まない）
- ALWAYS 肯定的制約と否定的制約を別セクションに分離する（混在させない）
- ALWAYS 言語・ディレクトリ固有ルールは `paths:` フロントマターでスコープする
- ALWAYS 詳細な手順やコード例は Agent / Skill へクロスリファレンスで委譲する

## スコープ

- NEVER 手順を書く（→ skill に書く）
- NEVER ペルソナを定義する（→ agent に書く）
- NEVER 長いコード例を載せる（→ skill に書く。OK / NG 1 組までは可）
- NEVER 具体的なパターン・正規表現・ファイル名リストを列挙する（→ skill に逃がす）
- NEVER `ALWAYS` / `NEVER` / `MUST` を散文中に埋め込む（行頭以外で使う）
- NEVER 1 ファイルに複数の関心事を詰め込む（ファイルを分割する）

## 関連

- **rule** agent -- rule の生成・編集を担当（構造テンプレート・記載例の詳細）
- **planner** agent -- skill / agent / rule のどれを作るかの判断が必要なとき
