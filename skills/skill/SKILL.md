---
name: skill
description: 種別が「skill」と確定している要望を受け取り、planner 判定をスキップして skill author agent を直接呼び出すハーネス系 skill。
---

# skill

ユーザーが「skill を追加したい」と種別を確信しているときに、種別判定を省略して skill author agent を直叩きする最短経路。`/forge:new` の planner 経由フローを通すと判定プロセスが冗長になるケースを救う。

## When to Activate

- ユーザーが `/forge:skill` を実行したとき
- 「新しい skill を追加したい」「この手順を skill として共有したい」と種別を明示したとき
- 詳細手順・PASS/FAIL コード例・リファレンス知識をまとめたい要望で、ユーザー自身が「これは skill だ」と判断しているとき

種別が不明・組合せかもしれない要望は `/forge:new`（planner 経由）に誘導する。

## 設計思想

- 種別が確定しているなら planner を通さない。判定の往復は精度に貢献せず、時間とコンテキストを消費するだけになる。
- ただし「テンプレート違反の防止」は維持する。書き換え中は path-scoped な `rules/skills.md` が自動適用されるので、author 側のテンプレ準拠は担保される。
- ユーザーが種別を誤認している可能性は author 側で検出する。skill author の責務外（ペルソナ定義・ALWAYS/NEVER 宣言・重要度判定）が要望に強く含まれていたら、author が警告して `/forge:new` への切替または分割を提案する。

## アーキテクチャ

```
ユーザー要望（種別 = skill 確定）
    │
    ▼
[skill skill]（本 skill）
    │
    ▼
[skill author agent] ──▶ .claude/skills/{name}/SKILL.md
                              │
                              ▼
              path-scoped [rules/skills.md] が自動適用
```

planner agent は呼ばない。

## 配置先

| 対象 | デフォルトパス |
|------|---------------|
| skill | `.claude/skills/<kebab-name>/SKILL.md` |

プロジェクト固有の配置規約があれば author 委譲前にユーザーへ確認する。

## 設定

| 環境変数 | デフォルト | 説明 |
|---------|----------|------|
| FORGE_SKILL_HARNESS_HINT | auto | `auto` / `normal` / `harness` -- ハーネス系か通常型かを author に提示する初期ヒント |

## ワークフロー

### Step 1: 主題と配置先の確認

ユーザー発話から「主題」「配置先（特殊なら）」「ハーネス系か通常型か」を抽出する。曖昧なら 1 問だけ確認する。

```bash
# 入力例
# 「Go の N+1 回避手順を skill として共有したい」
#   → 主題: Go の N+1 回避
#   → 配置先: .claude/skills/go-n-plus-one/SKILL.md（デフォルト）
#   → 通常型（PASS/FAIL コード例で説明できる）
```

### Step 2: 越境チェック（軽量）

要望文に skill author の責務外が強く含まれていないか、軽く確認する。

| 兆候 | 対処 |
|------|------|
| 「〜のレビュー担当が欲しい」「〜役を作りたい」（ペルソナ要素強） | `/forge:agent` か `/forge:new` を提案 |
| 「〜を絶対やらない」「常に〜する」を主目的とする要望 | `/forge:rule` か `/forge:new` を提案 |
| 「手順 + ガードレール + レビュー観点」のように 3 種別の要素が混ざる | `/forge:new`（planner 経由）を提案 |

該当しなければそのまま Step 3 へ進む。該当した場合はユーザーに切替を提案し、承認を得てから委譲する。

### Step 3: skill author に委譲

`skill` agent に以下を渡す:

- 主題（ユーザー発話の要点）
- 配置先パス
- 通常型 / ハーネス系の希望（`FORGE_SKILL_HARNESS_HINT` を初期値として参考に）
- 既存衝突確認の指示（配置先パスを Read してエラー有無で判定）

### Step 4: 結果レポート

author の出力をそのままユーザーに提示する。最低限の集約のみ行う。

```
## 生成結果
- パス: {path}
- 型: {通常型 | ハーネス系パターン A/B/C/D}
- PASS/FAIL 組数: {n}
- 次のアクション: {テンプレ違反の最終確認 / 関連 rule のクロスリンク追加など}
```

## エラーハンドリング

- **既存ファイルと衝突**: 上書きせず、`/forge:refactor` を提案する
- **author の出力がテンプレ違反を含む**: path-scoped rule で防げているはずだが、もし通り抜けたらユーザーに報告し、author を 1 回再委譲する（最大 1 回）
- **要望が skill の責務外を多く含む**: Step 2 で `/forge:new` か別コマンドへ誘導する。無理に skill として書かせない

## アンチパターン

- **越境チェックを省略して即委譲**: 種別誤認をそのまま固定化する。Step 2 は短くても必ず通す
- **planner を念のため呼ぶ**: 本 skill の存在意義を消す。planner が必要なら `/forge:new` を使う
- **配置先を勝手に決める**: kebab 命名・ディレクトリ規約はユーザーに確認する
- **ハーネス系と通常型の判定を author に丸投げしすぎる**: 主題から判定可能なら本 skill 側でヒントを渡す

---

**Remember**: 種別が決まっているなら最短経路で書く。planner は迷ったときの調停役、確信があるときは通さない。
