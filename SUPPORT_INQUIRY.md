# OpenAI Support 問い合わせ文面

以下をコピーし、`[ ]` の箇所を記入して送信してください。

```text
件名: Codex Cloud のシェルでは origin/認証がなく git push できない（PR作成ボタンは成功）

OpenAI Support ご担当者様

Codex Cloud (https://chatgpt.com/codex/cloud) で GitHub repository
KozueMitarai/CodexPushTest を環境に関連付けています。

発生日時: 2026-08-30 [時刻] UTC
ChatGPT プラン: [Plus/Pro/Business/Enterprise 等]
環境名: CodexPushTest
Task URL / ID: [記入]
再現頻度: [毎回 / N回中N回]

タスク VM のシェルでの結果:
- git status --short --branch: ## work
- 初期状態の git remote -v: 出力なし
- gh auth status: GitHub hosts に未ログイン
- 公開 HTTPS URL への git ls-remote: 成功
- origin を手動追加後の git push --dry-run:
  fatal: could not read Username for 'https://github.com': terminal prompts disabled

一方、Codex Cloud 画面右上の「PR を作成」ボタンからは Pull Request を作成でき、
GitHub 上での merge にも成功しました。Agent internet access は unrestricted です。

質問:
1. PR 作成経路とは異なり、タスク VM のシェルに remote と GitHub write credential が
   provisioning されない動作は想定仕様でしょうか。
2. シェルから git push できることが想定されている場合、当該 task の provisioning
   ログをご確認いただけますか。
3. ユーザー側で追加すべき安全な設定があればご案内ください。

類似事例として openai/codex#40086, #22660, #12498 が見つかりました。

添付:
1. 01-environment-screen.png
2. 02-task-url-and-error.md
3. 03-redacted-diagnostic.log

PAT、OAuth token、cookie 等の秘密情報は添付していません。
よろしくお願いいたします。
```
