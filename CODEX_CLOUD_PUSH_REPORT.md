# Codex Cloud から GitHub へ push できない問題の調査報告

調査日: 2026-08-30 (UTC)

## 結論

タスク VM のシェルから実行した通常の `git push` は **成功しませんでした**。直接の失敗理由は、GitHub の書き込み認証情報がシェル環境に渡されていないことです。さらに、環境作成時点でローカルリポジトリに `origin` 自体が設定されていませんでした。

一方、その後ユーザーが Codex Cloud 画面右上の **「PR を作成」ボタン**を利用したところ、Pull Request の作成と GitHub 上での merge には成功しました。したがって「Codex Cloud 全体に書き込み権限がない」のではなく、少なくとも今回の環境では、**シェルの `git push` 経路は認証されていないが、Codex Cloud の PR 作成経路（サーバー側の GitHub Connector）は正常に利用できる**と訂正します。ドキュメントのようなテキスト変更は、PR 作成ボタンを使う運用で連携可能です。

添付画像では `KozueMitarai/CodexPushTest` が環境に関連付けられており、「エージェントのインターネットアクセス」は「有効: 無制限」です。しかし、UI 上の関連付けと、タスク VM 内の Git remote・GitHub 書き込み資格情報は一致していません。今回の症状は、公開 issue で報告されている Codex Cloud の「`work` ブランチだけがあり remote がない」という不具合と非常によく一致します。

## 実測結果

初期状態で以下を確認しました。

| 確認項目 | 結果 | 判断 |
|---|---|---|
| `git status --short --branch` | `## work` | タスク固有のローカル `work` ブランチ |
| `git remote -v` | 出力なし | `origin` が未設定 |
| `.git/config` | `[core]` の設定のみ | remote 情報がチェックアウト時に注入されていない |
| `gh auth status` | GitHub にログインしていない | `gh` の認証なし |
| 認証関連の環境変数名 | `GH_TOKEN` / `GITHUB_TOKEN` なし | CLI が利用できるトークンなし |
| `git ls-remote https://github.com/KozueMitarai/CodexPushTest.git` | 成功、調査時の `main` と HEAD は `d2fa103...` | GitHub への通信と公開リポジトリの読み取りは正常 |
| 画面右上の「PR を作成」→ GitHub で merge | 成功（ユーザー確認） | サーバー側 Connector の PR 作成経路は正常 |

診断のため `origin` を公開 HTTPS URL として追加し、リモートを書き換えない `--dry-run` で push を試しました。

```console
$ GIT_TERMINAL_PROMPT=0 git push --dry-run origin HEAD:refs/heads/codex/push-diagnostic-20260830
fatal: could not read Username for 'https://github.com': terminal prompts disabled
```

終了コードは `128` でした。したがって、今回の失敗はネットワーク遮断や GitHub リポジトリの不存在ではなく、**書き込み時に必要な GitHub 認証がないこと**まで切り分けられます。`--dry-run` のため、この試験では GitHub 側にブランチや変更を作成していません。

## 原因候補（可能性順）

1. **Codex Cloud がシェルへ remote と認証情報を引き渡さない仕様、またはその処理の不具合**
   - UI では対象 repository が選択済みなのに、実環境には remote がありません。
   - 公開 issue [#40086](https://github.com/openai/codex/issues/40086)、[#22660](https://github.com/openai/codex/issues/22660)、[#12498](https://github.com/openai/codex/issues/12498) と同型です。
2. **ChatGPT Codex Connector の GitHub App installation が対象 owner/repository にない、または権限が古い**
   - GitHub アカウントを「接続済み」にしただけでは、対象 owner に GitHub App が install されていない場合があります。同様の混乱は [#38146](https://github.com/openai/codex/issues/38146) で報告されています。
3. **接続している GitHub アカウントの取り違え**
   - ブラウザでログイン中の GitHub アカウント、ChatGPT に接続したアカウント、対象 repository の owner が異なると、読み取りはできても書き込みが失敗し得ます。[#24735](https://github.com/openai/codex/issues/24735) には cloud connector とローカル `gh` の identity が別である事例があります。
4. **Repository/App の権限不足**
   - Connector が `KozueMitarai/CodexPushTest` への Contents write / Pull requests write 相当の権限を持たない、または「Selected repositories」に対象が追加されていない可能性があります。
5. **ブランチ保護・ruleset**
   - `main` への直接 push のみが拒否される場合の候補です。ただし今回はこちらに到達する前に認証で失敗し、診断先も新規 `codex/...` ブランチなので、主因である可能性は低いです。
6. **ネットワーク障害**
   - unrestricted 設定でも HTTPS が遮断された報告 [#20928](https://github.com/openai/codex/issues/20928) はあります。しかし今回は同じ VM から GitHub の `ls-remote` が成功したため、少なくとも調査時点の主因ではありません。

## 同様の公開事例

| Issue | 状況 | 今回との関係 |
|---|---|---|
| [openai/codex#40086](https://github.com/openai/codex/issues/40086) | Cloud task が detached checkout、remote なし、`work` のみ | ほぼ一致 |
| [openai/codex#22660](https://github.com/openai/codex/issues/22660) | linked repository なのに remote/auth が継承されない | ほぼ一致 |
| [openai/codex#12498](https://github.com/openai/codex/issues/12498) | `origin` が消え `work` のみになる | ほぼ一致 |
| [openai/codex#21771](https://github.com/openai/codex/issues/21771) | commit が UI/GitHub に反映されず push 不可 | 関連症状 |
| [openai/codex#38146](https://github.com/openai/codex/issues/38146) | Connector が別アカウントにだけ install され書き込み失敗 | 設定原因の候補 |
| [openai/codex#20928](https://github.com/openai/codex/issues/20928) | unrestricted でも proxy が HTTPS を拒否 | 今回は読み取り成功のため可能性低 |

issue は状況報告であり、OpenAI による原因確定や修正完了を意味しません。調査時点で上記は open でした。

## 推奨する復旧手順

1. **作業内容を退避**する（diff/patch のダウンロード等）。環境削除や再接続より先に行います。
2. ChatGPT の GitHub 接続をいったん解除し、目的の GitHub アカウント `KozueMitarai` で再接続します。
3. GitHub の **Settings → Applications → Installed GitHub Apps** で ChatGPT Codex Connector を開きます。
4. Connector が `KozueMitarai` に install され、Repository access に `CodexPushTest` が含まれることを確認します。必要なら一度 uninstall/reinstall し、対象 repository を明示的に選びます。
5. Codex Cloud 側で環境キャッシュをリセットするか、新しい環境を作成し直します。古い環境の `.git/config` 欠落がキャッシュされている可能性があります。
6. 新しい読み取り専用診断タスクで次を実行させます。

   ```bash
   git status --short --branch
   git remote -v
   git config --get remote.origin.url
   git ls-remote origin HEAD
   ```

7. `origin` が最初から存在することを確認後、空の診断 commit または小さな変更を **新規ブランチ**へ push し、PR 作成を試します。`main` へ直接 push しないことで ruleset の影響を分離できます。
8. シェルからの直接 push が必須でなければ、動作確認済みの画面右上の **「PR を作成」**を利用し、GitHub 上でレビュー・merge します。
9. シェルからの直接 push が必要で、remote が再び空または credential error になる場合、ユーザー側で PAT を環境変数に保存して回避しないでください。秘密漏えいと権限過多を避けるため、OpenAI Support にシェルの Git 認証が想定仕様かを問い合わせます。

## サポートへの連絡方法

OpenAI の公式案内 [How can I contact support?](https://help.openai.com/en/articles/6614161-how-can-i-contact-support) に従い、[help.openai.com](https://help.openai.com/) 右下のチャットバブルから問い合わせます。最初に bot が切り分けを行うため、「Codex Cloud」「GitHub connection / push」「technical issue」を選び、必要なら担当者への引き継ぎを依頼します。

送信前に以下を揃えると調査が速くなります。今回用に、そのままコピーできる本文を [`SUPPORT_INQUIRY.md`](SUPPORT_INQUIRY.md)、添付資料一式を [`support-attachments/`](support-attachments/) に用意しました。

- ChatGPT アカウントのメールアドレスと契約プラン（パスワードは送らない）
- 発生日時と timezone（例: 2026-08-30 UTC）
- Codex environment 名、repository 名、失敗した task URL / task ID
- 添付画像、エラー全文、上記診断コマンドの結果
- 再現頻度、初回発生日、別環境・キャッシュリセット後も再現するか
- GitHub App installation の対象 owner/repository が分かるスクリーンショット
- **PAT、OAuth token、cookie、環境変数の値は絶対に添付しない**

### 問い合わせ文面（コピー用）

```text
件名: Codex Cloud で linked GitHub repository の origin/認証が引き渡されず push できない

OpenAI Support ご担当者様

Codex Cloud (https://chatgpt.com/codex/cloud) で GitHub repository
KozueMitarai/CodexPushTest を環境に関連付けていますが、毎回 push に失敗します。

発生日時: 2026-08-30 [時刻] UTC
ChatGPT プラン: [Plus/Pro/Business/Enterprise 等]
環境名: CodexPushTest
Task URL / ID: [記入]
再現頻度: [毎回 / N回中N回]

タスク VM での結果:
- git status --short --branch: ## work
- git remote -v: 出力なし
- gh auth status: GitHub hosts に未ログイン
- 公開 HTTPS URLへの git ls-remote: 成功
- origin を手動追加後の git push --dry-run:
  fatal: could not read Username for 'https://github.com': terminal prompts disabled

UI では repository が選択され、Agent internet access は unrestricted です。
GitHub 側では Codex Connector の installation と対象 repository access を
[確認済み / 再接続済み] ですが、[新規環境でも] 再現します。

なお、画面右上の「PR を作成」からの PR 作成と GitHub 上の merge は成功しました。
Codex Cloud が checkout を作成する際、シェル用 Git remote と write credential を
task に provisioning しない動作が想定仕様か、サーバーログとあわせてご確認いただけますか。
類似事例として openai/codex#40086, #22660, #12498 が見つかりました。

添付（support-attachments/ATTACHMENTS_README.md の手順で準備）:
1. 01-environment-screen.png（ユーザーが保存）
2. 02-task-url-and-error.md（Task URL を追記）
3. 03-redacted-diagnostic.log（作成済み）

よろしくお願いいたします。
```

## 調査上の制約

このシェル環境には GitHub 書き込み credential が存在しないため、コマンドラインからの push は成功確認できませんでした。一方、公開 repository の読み取り成功、remote 欠落、dry-run の credential error は再現済みで、画面右上の PR 作成と GitHub 上の merge はユーザー側で成功確認済みです。この差から、ネットワーク障害や repository 全体の権限不足ではなく、**シェルの Git 経路とサーバー側 PR 作成経路で認証の扱いが異なる**と判断できます。

## ローカル環境での作業（2026-08-31 追記）

Cloud 環境とは別に、Windows のローカル環境へ本リポジトリをクローンし、ドキュメントの修正と Git コミットを行う運用を追加しました。上記の調査結果は 2026-08-30 の Cloud 環境についての記録であり、ローカル環境の状態とは区別してください。

- **クローン:** HTTPS URL からの `git clone` に成功しました。`origin` は自動設定され、取得時の `main` は `227a9c8` でした。
- **修正・コミット:** ローカルの作業ツリーでファイルを編集し、`git add` と `git commit` で変更を記録できます。本追記もこの手順でコミットします。
- **プッシュ:** ローカルから `git push origin main` を実行する方針です。ただし、現在は GitHub の書き込み認証が未完了のため、成功は未確認です。認証を無人で確認した `git -c credential.interactive=false push --dry-run origin HEAD:refs/heads/main` は `fatal: unable to get password from user` で失敗しました。

作業手順は以下のとおりです。push には対象リポジトリへの書き込み権限を持つ GitHub 認証が必要です。

```powershell
git clone https://github.com/KozueMitarai/CodexPushTest.git
cd CodexPushTest
# ドキュメントを修正
git diff --check
git add CODEX_CLOUD_PUSH_REPORT.md
git commit -m "docs: record local clone and push workflow"
git push origin main
```

この端末では Git Credential Manager が設定されています。認証は端末の正規のログイン手順で行い、トークンやパスワードをドキュメント・チャット・Git の remote URL に記載しないでください。認証後もブランチ保護などで拒否される場合は、作業ブランチへの push と PR による反映を検討します。ローカルでの成功だけでは、Cloud のシェル認証問題が解消したことにはなりません。

### ローカルからの push 再確認（2026-08-31）

ユーザーが前項のコミット `14f4149` を GitHub へ push した後、エージェントからも、承認されたネットワークアクセス付きの通常の `git push --dry-run origin main` が成功しました。

一方、`-c credential.interactive=false` を付けた確認は引き続き失敗しました。この環境では対話認証を禁止した試験の失敗だけを理由に、通常の push も不可能と判断すべきではありません。今後は通常の Git Credential Manager の認証手順を許可して `git push origin main` を実行します。認証画面でユーザー操作が必要になる場合があるため、毎回完全に無人で実行できることまでは保証しません。

ローカルへのクローン、修正、コミットをエージェントが実施し、通常の認証手順を使って push する運用とします。本追記のコミットをその実際の push 確認に使用します。Cloud 環境についての過去の調査結果は変更しません。
