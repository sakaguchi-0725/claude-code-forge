---
name: new
description: 種別が不明・複数種別の組合せが必要な要望を受け取り、planner agent に種別判定と内容分配を任せた上で、対応する author agent に順次委譲するハーネス系 skill。
---

# new

ユーザーの要望から「skill / agent / rule のどれか / 組合せか」が即断できないとき、または明らかに複数種別の連携が必要なときに、planner agent を経由して各 author に書かせるエントリ。種別が確定している場合は本 skill を使わず `/forge:skill` `/forge:agent` `/forge:rule` を使う。

## When to Activate

- ユーザーが `/forge:new` を実行したとき
- 要望から種別が即断できないとき（「コミット前にシークレット混入を防ぎたい」のように制約・手順・レビュー観点が混ざる）
- 複数種別を組み合わせた体制構築（例: セキュリティレビュー一式 = rule + agent + skill）
- ユーザー自身が「どれを作ればいいかわからない」と相談してきたとき

種別が確定しているなら直接コマンド（`/forge:skill` `/forge:agent` `/forge:rule`）を使う。本 skill は **判定が必要なとき専用** に位置付ける。

## 設計思想

- 種別判定（planner）と執筆（author）を分離する。planner は「種別 + 各種別への内容分配 + 除外項目」を出力し、author はそれを厳守する。
- planner の出力契約は「採用種別」だけでは足りない。要望の中の **どの内容を / どの種別に書き、何を書かないか** まで指示しなければ、author 側で越境（skill に rule っぽい宣言が混じる、agent に手順が混じる等）が起きる。
- 組合せ生成（rule + agent + skill）も planner が依存関係と分配を決め、author に順次渡す。
- 書き換え中は path-scoped な `rules/*.md` が自動適用され、テンプレート違反を別レイヤーで防ぐ。

## アーキテクチャ

```
ユーザー要望（種別不明 or 組合せ）
    │
    ▼
[new skill]（本 skill）
    │
    ▼
[planner agent] ── 種別判定 + 内容分配 + 除外項目 + 生成順序
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

### Step 1: 直接コマンドへの誘導判定

要望文から種別が一意に決まる兆候があれば、直接コマンドへ誘導する。

| 兆候 | 誘導先 |
|------|-------|
| 「skill を作りたい」「手順を共有したい」と種別を明示 | `/forge:skill` |
| 「〜役の agent が欲しい」「レビュアーを立てたい」 | `/forge:agent` |
| 「常に〜する / 絶対〜しない」が短い宣言で済む | `/forge:rule` |

該当する場合はユーザーに「直接コマンドの方が速い」と提案し、合意があれば誘導する。判断に迷うなら本 skill を続行する。

### Step 2: planner agent に委譲

planner agent に以下を渡す:

- ユーザー要望の全文
- 既存の `.claude/` 構造（Read / Glob で確認した結果）
- 「採用種別 + 各種別への内容分配 + 除外項目 + 生成順序」を出力するよう明示

planner の出力は次のフォーマットを期待する:

```
## 判定結果

### 採用種別
- {skill | agent | rule}（組合せの場合は複数）

### 各種別への内容分配
#### {種別 A}
- 含める内容: {bullet}
- 除外する内容: {bullet}（理由: 他種別の責務）

#### {種別 B}
- ...

### 根拠
- 制約軸: {0-2} / 判定軸: {0-2} / 知識軸: {0-2}
- 採用理由: {1-2 文}

### 生成順序
1. {種別} -- {ファイル名案} -- 担当: {author agent}

### 検討事項
- {リスクや未確定点、または「なし」}
```

planner 返答の読み取り方:

- 単一種別: 該当 author agent に「含める内容」「除外する内容」を渡してそのまま委譲
- 組合せ: `FORGE_CONFIRM_COMBO=true` ならユーザーに内訳（種別ごとの分配）を提示して承認を取る
- スコア拮抗: planner が候補を併記してきたらユーザーに選択を委ねる
- 「いずれの種別にも当てはまらない」: 削除 / CLAUDE.md などの代替形式を相談

### Step 3: author agent に順次委譲

planner の生成順序に従い、対応する author を順に呼ぶ。各 author に渡す入力には **必ず planner の「除外する内容」を含める**。

| 生成対象 | 担当 agent | 渡す入力 |
|---------|-----------|---------|
| rule | `rule` agent | 配置先 + 含める制約 + 除外項目 + スコープ |
| skill | `skill` agent | 配置先 + 含める主題 + 除外項目 + 通常型/ハーネス系希望 |
| agent | `agent` agent | 配置先 + 含める役割 + 除外項目 + バリエーション指示 |

各 author は除外項目を絶対に書かない。書きたくなった内容が除外項目に該当する場合は、本 skill に戻して別 author への分配を提案させる。

### Step 4: 結果レポート

生成したファイル一覧と、planner の分配方針に従ったかを集約してユーザーに報告する。

```
## 生成結果

### 作成したファイル
1. .claude/rules/{name}.md    -- ALWAYS/NEVER 各 n 件
2. .claude/skills/{name}/SKILL.md -- 通常型, PASS/FAIL x 件
3. .claude/agents/{name}.md   -- バリエーション X, model: sonnet

### 分配の遵守状況
- 各 author は planner の除外項目を遵守したか: {Yes / 該当ファイル名}

### 次に確認すべきこと
- {テンプレ違反がないか、関連ファイル間のクロスリファレンス漏れ}
```

## エラーハンドリング

- **planner 判定が曖昧（スコア拮抗）**: 候補を提示してユーザー選択を待つ。無断で片方を採用しない
- **planner が分配を出力しない**: 旧フォーマット（種別のみ）の出力を検出したら、planner に再要求して「内容分配 + 除外項目」を埋めさせる
- **author agent が除外項目を書いてしまった**: 該当ファイルを再委譲し、除外項目を改めて明示する（最大 1 回）
- **既存ファイルと衝突**: 上書きせず、パスの変更または `/forge:refactor` への切替を提案する
- **FORGE_MAX_COMBO 超過**: planner が 4 種別以上を提案しても上限で頭打ちし、残りはユーザーに次回要望として提示

## アンチパターン

- **種別が明確なのに本 skill を使う**: `/forge:skill` `/forge:agent` `/forge:rule` を使う。planner の往復は無駄
- **1 要望に対して機械的に 3 点セット生成**: 本当に必要なものだけ作る。planner が必要と判定したもののみ
- **planner の「除外項目」を author に渡さない**: 越境の根本原因。除外項目はファイル間境界を保つ最重要パラメータ
- **既存資産を確認せず新規作成**: planner の Phase 1 で既存 agent / skill / rule を Grep / Glob で必ず確認する

---

**Remember**: 種別判定が必要なときの本 skill、確信があるときは直接コマンド。planner には「採用種別」だけでなく「除外項目」まで出させて、author に厳守させる。
