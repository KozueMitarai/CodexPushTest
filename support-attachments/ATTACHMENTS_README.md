# サポート添付資料の準備

このフォルダーを問い合わせに添付できるよう、次の3点構成にしています。

1. `01-environment-screen.png`
   - 今回チャットに添付した Codex Cloud の環境設定画面を端末へ保存し、このファイル名に変更してください。
   - チャットにアップロードされた画像ファイル自体はタスク VM から取得できないため、リポジトリには含めていません。
   - メールアドレスなど、問い合わせに不要な個人情報は必要に応じてマスクしてください。
2. `02-task-url-and-error.md`
   - 対象 Task の URL/ID、発生日時、画面に表示されたエラーを追記してください。
3. `03-redacted-diagnostic.log`
   - 今回取得できた診断ログです。token 値、cookie、Authorization header は含めていません。

送信直前にも、PAT、OAuth token、cookie、`Authorization` header、secret の値が含まれていないことを確認してください。
