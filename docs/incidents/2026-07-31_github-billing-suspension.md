# 復旧手順: GitHub課金問題によるActions停止

- 作成日: 2026-07-31
- 対象: GitHub Organization `Fourdesign-LLC`（リポジトリ `Fourdesign-LLC/danswer` ほか）
- 症状: GitHub Actions のジョブが起動しない / `Actions are disabled due to payment issues.` または `The job was not started because recent account payments have failed or your spending limit needs to be increased.`

## 追記: 復旧完了（2026-07-31）

Budgets and alerts 画面で復旧を確認。**4製品すべて 100% バッジが消え、バーがグレーに戻った。**

| 製品 | 対応前 | 対応後 | 状態 |
|---|---|---|---|
| Git LFS | **$0**（100%・停止） | **$10.00** | 復旧 |
| Codespaces | **$0**（100%・停止） | **$20.00** | 復旧 |
| Packages | $30.00 | $30.00 | 変更なし（元から正常） |
| Actions | $30.00 | $30.00 | 変更なし（元から正常） |

`Stop usage: Yes` は4製品とも維持。上限の歯止めを残したまま停止を解除できている。
`spent` は全製品 $0 のため、当面この予算額で枯渇する見込みはない。

### 未解決として残る論点

**`danswer` のワークフロー実行が0件である件は本件とは別問題。** Actions は元から予算 $30 /
消費 $0 で、課金ブロックを受けていなかった。原因は以下のいずれかで、必要になった時点で切り分ける。

- Organization の Actions ポリシーが `Disable actions`（Org **Settings** → **Policies** → **Actions**）
- fork のため Actions が未有効化
- 単にトリガー条件を満たすイベントが発生していないだけ（`danswer` は public fork のため
  Actions 自体は無料枠無制限で、そもそも課金の影響を受けない）

---

## 確定診断（2026-07-31 Budgets and alerts 画面で確認）

`Fourdesign-LLC` の **Budgets and alerts** に Organization budget が4件。実測値は以下。

| 製品 | 予算 | Stop usage | spent | 状態 |
|---|---|---|---|---|
| **Codespaces** | **$0** | Yes | $0 | **100% — 停止中** |
| Packages | $30.00 | Yes | $0 | 正常 |
| **Actions** | **$30.00** | Yes | $0 | **正常（予算ブロックなし）** |
| **Git LFS** | **$0** | Yes | $0 | **100% — 停止中** |

### 結論

**停止しているのは Codespaces と Git LFS の2製品。Actions は予算 $30 で正常。**

`予算 $0` + `Stop usage: Yes` の組み合わせが停止の直接原因。$0 予算は割り算上つねに
100% 消化済みと判定され、その製品が完全にブロックされる。

GitHub の画面上部の説明文がそのまま根拠になる:

> If no budget is set, usage for that product is unlimited.

つまり **予算 $0 は「無制限」ではなく「全面禁止」**。ここが直感に反する落とし穴。
「無料枠しか使わないから $0 にしておく」という設定が、そのまま停止スイッチとして機能する。

### 影響

- **Git LFS 停止** → LFS 管理下のファイルが `git push` / `git clone` / `git lfs pull` で取得・送信不可
- **Codespaces 停止** → Codespace の新規作成・起動が不可

### 対処

STEP 3-新 の手順で、Codespaces と Git LFS の予算を $0 から実額へ変更する（後述）。
Actions については課金起因ではないため、STEP 4（Actions ポリシー）側を確認する。

---

## 観測されている事実

| 項目 | 状態 |
|---|---|
| ワークフロー定義 | 10本すべて `state: active`（`pr-python-checks.yml`, `run-it.yml`, `docker-build-push-*` ほか） |
| ワークフロー実行履歴 | **0件**（API `list_workflow_runs` → `total_count: 0`） |
| `danswer` の可視性 | **public**（`onyx-dot-app/danswer` からの fork） |
| Org の他リポジトリ | `owlcas-exporter` `tnp-app` `CommandLineTools` `toss-nas-putter` ほか **private**（全30リポジトリ） |

### 重要: `danswer` の実行0件は課金停止の証拠にならない

**public リポジトリの Actions は GitHub-hosted standard runner で無料枠無制限**。したがって
`danswer`（public）は、予算 $0 でも未払いがあっても Actions は止まらない。

`danswer` で実行0件になる原因は課金とは別系統:

- fork では Actions が既定で無効。Actions タブに `I understand my workflows, go ahead and enable them` ボタンが出る
  （ただし本リポジトリのワークフローは `disabled_fork` ではなく `active` なので、この線は薄い）
- Organization レベルの Actions ポリシーが `Disable actions` になっている
- Org 全体の課金ロックにより public を含めて全リポジトリが停止している

**課金による停止が実害として効くのは private リポジトリ側**（Free プランの無料枠は private 向け 2,000分/月）。
Org の大半が private なので、業務影響はそちらに出ている。

## 前提: 3つの原因パターン

課金停止は原因が3層あり、**上から順に潰す**。1つ直しても他が残っていると復旧しない。

| # | 原因 | 症状の見分け方 |
|---|---|---|
| A | 支払い失敗・請求未払い（past due） | 画面上部に赤/黄バナー。`Payment history` に `Failed` 行がある |
| B | 予算（Budget / Spending limit）が $0 | バナーは出ないが `The job was not started because ... spending limit needs to be increased` |
| C | サーバ側の課金ロックフラグが残留 | A・Bを直しても数時間〜1日たっても復旧しない |

**注意**: パターンBは無料枠が残っていても発生する。Actions の予算が $0 だと GitHub は無料分すら実行を許可しない。

---

## STEP 0: 正しい課金スコープを開く（最重要）

個人アカウントの課金画面を見ても意味がない。ワークフローは Organization 配下で走るため、**Organization の課金設定**を開く。

### 画面遷移

1. https://github.com を開く
2. 画面右上の**アバター画像**をクリック
3. ドロップダウンから **Your organizations** をクリック
4. 一覧の `Fourdesign-LLC` の行、右側の **Settings** ボタンをクリック
5. 左サイドバー **Access** セクション内の **Billing & Licensing**（クレジットカードのアイコン）をクリック

**直リンク**: https://github.com/organizations/Fourdesign-LLC/settings/billing

### ここでプラットフォーム世代を判別する

左サイドバーのラベルを見て、以降どちらの手順を使うか決める。

| サイドバー表示 | 世代 | 以降の手順 |
|---|---|---|
| **Billing & Licensing** | 新（enhanced billing platform） | STEP 3-新 を使う |
| **Billing and plans** | 旧（legacy） | STEP 3-旧 を使う |

※ Enterprise アカウント配下の Organization の場合、支払い方法と予算は **Enterprise 側**にある。その場合は
アバター → **Your enterprises** → 該当 Enterprise → **Settings** → **Billing** に読み替える。
Organization 側の画面には「請求は Enterprise で管理されています」の旨が表示される。

---

## STEP 1: 未払い請求を精算する（パターンA）

### まず前提: 「Latest invoice」はサイドバーのメニューではない

**Latest invoice は Overview ページ本文中のセクション**であって、左サイドバーの項目ではない。
サイドバーを探しても見つからないのが正常。Overview ページ本体を上から見ること。

### 「Latest invoice」セクション自体が存在しない場合 → パターンAは除外

Overview 本文にも見当たらないなら、それは異常ではなく **請求が1件も発生していない**ことを意味する。
以下のいずれかに該当する。

| 表示 | 意味 | 次の行動 |
|---|---|---|
| `Estimated next payment` と表示 | 未払いはなく、次回請求予定額を表示中 | パターンA除外 → **STEP 3 へ** |
| セクション自体が無い / $0.00 | Free プランで課金実績なし | パターンA除外 → **STEP 3 へ** |
| 「請求は Enterprise で管理」の旨 | Enterprise 配下 | Enterprise 側の課金画面へ（STEP 0 参照） |

**これは重要な切り分け**: 請求書が存在しない = GitHub は一度も課金していない
= 「支払い失敗」ではあり得ない。**予算 $0 による強制停止（パターンB）が原因である可能性が極めて高い。**

Free プランで予算 $0 の場合、無料枠(private 2,000分/月)を超えた時点でジョブが止まり、
超過分は課金されない。だから**請求書が生成されず、未払いも発生しない**。
「請求書が無いのに Actions が止まる」のはこの状態の典型的な見え方。

→ **この場合は STEP 1 を飛ばして STEP 3（予算）へ進む。**

### 請求書がある場合の画面遷移

1. **Billing & Licensing** 画面の **Overview** タブ（開くと最初に表示される）
2. **Latest invoice** セクションを見る
3. 金額の右にある **Pay now** ボタンをクリック
4. セキュア決済ダイアログが開くので、カード情報を入力して決済を実行
5. 決済後、画面上部の `past due` / `We are having a problem billing your account` バナーが消えることを確認

### 滞留分の確認

1. 左サイドバー（Billing & Licensing 内）の **Past invoices** をクリック
2. ステータスが `Past due` / `Unpaid` の請求が他にないか確認。あれば同様に精算
3. 左サイドバー **Payment history** をクリック
4. **`Failed` 表示の行**がないか確認する。**数セント単位の失敗でもロックの原因になる**
5. `Failed` 行に **Retry** ボタンがあればクリック

---

## STEP 2: 支払い方法を「削除 → 再登録」する

有効なカードが登録済みでも、**再オーソリを走らせないとロックが解けない**。更新ではなく削除→再登録が確実。

### 画面遷移

1. **Billing & Licensing** 画面の左サイドバー → **Payment information** をクリック
2. **Payment method** セクションの現在のカードを確認
3. カード横の **Remove**（またはケバブメニュー `…` → **Remove**）をクリックして削除
4. **Add payment method** をクリック
5. カード情報を入力 → **Save payment method**
6. GitHub が $1 程度のオーソリを実行する（後日返金される）。これが通ればロック解除条件を満たす

### 日本発行カードでの詰まりどころ

- **JCB は非対応**。Visa / Mastercard / American Express、または PayPal を使う
- 銀行・カード会社側の**海外決済ブロック**や**3Dセキュア未設定**で弾かれることが多い。カード会社アプリで海外利用を一時的に許可する
- 物理カードが通らない場合、バーチャルカード（Wise、Revolut 等）で登録すると通る事例が多い
- 請求先住所（Billing address）が**カード登録住所と一致**していること。ローマ字表記で入力する

### 請求先メールアドレスの確認

未払い通知が誰にも届いていないと再発する。

1. **Billing & Licensing** → **Additional billing details** をクリック
2. **Email recipients** セクションで **Add** をクリックし、経理・管理者のアドレスを追加
3. Primary contact が有効なアドレスになっているか確認（違えば **Edit** で修正）

---

## STEP 3-新: 予算（Budgets and alerts）を $0 から上げる（パターンB）

**サイドバーが「Billing & Licensing」の場合はこちら。**
**「Latest invoice が無い」ケースの本命はここ。**

### 先に無料枠の消費状況を確認する

1. **Billing & Licensing** → 左サイドバー **Usage** をクリック
2. 期間を今月（`This month`）にする
3. 製品フィルタで **Actions** を選択
4. **Included（無料枠）の 2,000分を使い切っていないか**を確認する

使い切っていれば、予算 $0 で停止しているという診断が確定する。
（Free プランの無料枠 2,000分/月は **private リポジトリのみ**が対象。public リポジトリの消費は
ここにカウントされない）

### 既存予算を修正する場合（本件の実作業はここ）

対象は **Codespaces** と **Git LFS** の2件（いずれも予算 $0 で停止中）。

1. **Billing & Licensing** → 左サイドバー **Budgets and alerts** をクリック
2. `Codespaces` の行の右端のメニューアイコン **`…`** → **Edit** をクリック
3. **Budget** の金額欄を **$0 → 必要額**（例: `20`）に変更
4. **Stop usage when budget limit is reached** は**チェックしたまま残す**
   - 外すと上限の歯止めが無くなる。金額を入れたうえでチェックを残すのが安全
5. **Save**（または **Update budget**）をクリック
6. `Git LFS` の行でも同様に `…` → **Edit** → 金額を **$0 → `10`** 程度に → **Save**
7. 一覧に戻り、対象行の **100% バッジとオレンジのバーが消えている**ことを確認する（反映は概ね即時）

#### 予算そのものを削除する選択肢（非推奨）

`…` → **Delete** でも復旧する。画面の説明文どおり、予算未設定の製品は無制限になる。
ただし暴走課金を止める手段が無くなるため、**金額を入れて残すほうを推奨**する。

#### 注意: 予算 $0 は「無制限」ではなく「全面禁止」

`$0 spent / $0 budget` は 100% 消化済みと判定される。`Stop usage: Yes` と組み合わさると、
その製品は無料枠の有無に関係なく完全にブロックされる。
「無料枠しか使わないから $0 にしておく」という設定が停止スイッチとして機能してしまう。

### 予算が1件も無い / 新規作成する場合

1. **Budgets and alerts** 画面で **New budget** をクリック
2. **Budget type** で **Product-level budget** を選択
3. Product のプルダウンで **Actions** を選択
4. **Budget scope** で対象（Organization `Fourdesign-LLC`、または特定リポジトリ）を選択
5. **Budget** に金額を入力
6. **Receive budget threshold alerts** にチェック（75% / 90% / 100% でメール通知）
7. **Alert Recipients** に通知先を指定
8. **Create budget** をクリック

同様に **Packages**、**Codespaces** の予算も $0 になっていないか確認する（同じ課金ロックに巻き込まれる）。

## STEP 3-旧: Spending limits を $0 から上げる

**サイドバーが「Billing and plans」の場合はこちら。**

1. **Billing and plans** 画面を下にスクロールし **Spending limits** セクションへ
2. **Actions & Packages** の項目を探す
3. **Update limit**（または **Change spending limit**）をクリック
4. **Limit spending** のラジオボタンを選び、金額を **$0 以外**に入力
   （もしくは **Unlimited spending** を選択）
5. **Update limit** をクリックして保存

---

## STEP 4: Actions 側の権限設定が落ちていないか確認

課金を直しても、Actions 自体が Disabled に落ちていると動かない。**Organization → リポジトリ**の順で確認する。

### Organization レベル

1. `Fourdesign-LLC` の **Settings**
2. 左サイドバー **Code, planning, and automation** → **Actions** → **General**
3. **Policies** の項目が **Allow all actions and reusable workflows** になっているか確認
4. 変更したら **Save**

**直リンク**: https://github.com/organizations/Fourdesign-LLC/settings/actions

### リポジトリレベル

1. https://github.com/Fourdesign-LLC/danswer を開く
2. 上部タブの **Settings** をクリック
3. 左サイドバー **Actions** → **General** をクリック
4. **Actions permissions** で **Allow all actions and reusable workflows** を選択
5. **Save** をクリック

**直リンク**: https://github.com/Fourdesign-LLC/danswer/settings/actions

---

## STEP 5: 復旧確認

このリポジトリは実行履歴が0件なので「再実行」できるジョブがない。手動で1本走らせて確認する。

1. https://github.com/Fourdesign-LLC/danswer の **Actions** タブを開く
2. 「Actions が無効です」等の警告バナーが消えていることを確認
3. 左の一覧から任意のワークフローを選ぶ
4. **Run workflow** ボタンがあればクリック → ブランチを選んで実行
   （`workflow_dispatch` 未定義で Run workflow が出ない場合は、対象ブランチに空コミットを push して PR 系ワークフローを起こす）
5. ジョブが `Queued` → `In progress` へ進めば復旧

反映は**即時〜数時間**かかる。すぐに緑にならなくても、STEP 1〜4 を終えた直後に諦めない。

---

## STEP 6: それでも復旧しない場合（パターンC）

支払いも予算も正常なのに止まったままなら、**GitHub 側のアカウントロックフラグが残留**している。これは自力では解除できないケースがある。

### 先に試す再評価トリガー

課金状態の再評価を強制的に走らせる小技。順に実施する。

1. **Budgets and alerts** で Actions の予算を **$1 に設定 → Save**
2. 数分待つ
3. 同じ予算を **$0 に戻す → Save**
4. **Payment information** でカードを **Remove → Add payment method** で入れ直す
5. 1時間ほど待って STEP 5 を再試行

（この「予算の上げ下げ＋カード再登録」の組み合わせで、サポートを介さず解除された報告が複数ある）

### アカウントロックの解除

1. https://github.com/settings/billing を開く
2. **Unlock account** の表示があればクリックし、指示に従う

### GitHub サポートへの問い合わせ

1. https://support.github.com/ を開く
2. **Contact support** をクリック
3. GitHub アカウントでサインイン
4. **Account or organization** で `Fourdesign-LLC` を選択
5. カテゴリで **Billing** を選択
6. 本文に以下を英語で記載する
   - 症状（`Actions are disabled due to payment issues.`）
   - 未払い請求は精算済みであること（決済日時）
   - 支払い方法を再登録済みであること
   - 予算が $0 でないこと
   - リクエスト: `Please clear the billing lock flag on the organization.`
7. **Send request** をクリック

**回答まで数日〜2週間かかる**ため、待つ前提でなく STEP 6 の再評価トリガーを先に試すこと。

---

## 復旧までの業務継続

Actions が止まっている間の代替。

- **イメージビルド**: `docker-build-push-*` 系が止まるため、コンテナイメージは**ローカルまたは別のCI環境でビルドして push** する
- **Lint / テスト**: `.pre-commit-config.yaml` が整備済みなので、ローカルで `pre-commit run --all-files` を回して品質チェックを代替する
- **タグ運用**: `tag-nightly.yml`（Nightly Tag Push）が止まるため、必要なタグは手動で打つ

## 再発防止

1. **予算を $0 にしない**（本件の直接原因）。「無料枠しか使わないから $0」は誤り。
   `$0` + `Stop usage: Yes` はその製品の全面禁止として機能する。使わない製品なら予算を
   **削除**する（未設定＝無制限）か、少額を入れておく。$0 で残すのが最悪の選択
2. **全4製品（Actions / Packages / Codespaces / Git LFS）の予算を定期点検する**。
   今回 Actions と Packages は $30 で正常だったが、Codespaces と Git LFS だけ $0 のまま
   放置され停止していた。1製品だけ直しても他が残る
3. **予算アラートの有効化**（STEP 3-新）。75% / 90% / 100% で事前にメール通知が来る。
   画面右上の **Included usage alerts** も **On** にしておく
4. **請求先メールの複数登録**（STEP 2 末尾）。通知が1人にしか届かないと退職・見落としで再発する
5. **カード有効期限の管理**。期限切れでの決済失敗が最も多い原因。カード更新時に GitHub の登録更新をタスク化する
