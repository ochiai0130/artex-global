# `/btw` の検証記録

日本語 · [한국어](VALIDATION.ko.md) · [中文](VALIDATION.zh.md)

日付: 2026-09-10。ブランチ: `codex/btw-side-question`。基準コミット（baseline）: `8dae851b9b622f2ff2631f332fde9719d0b16fba`。

> この文章は、中国語の原文を韓国語にローカライズした文書を、さらに日本語へ訳したものです。上流との差分を取りやすいように、中国語の原文（`VALIDATION.zh.md`）はそのまま保存してあります。

独立した PostgreSQL のテスト DB とデータディレクトリを使いました。実際のモデルの資格情報は独立したテスト環境にだけ注入し、コードやこの記録には書いておらず、製品の既定モデルも変えていません。Go 1.26.3、norma v0.3.6、Next.js 16.2.9。

実際のモデルの会話、返却オブジェクト、エンジニアリングの断言、Qwen の原文の審査テキストは [validation-2026-09-10.json](validation-2026-09-10.json) に保存してあり、その中に API の資格情報はありません。

## エンジニアリング検査

下の項目はすべて通過しており、括弧の中は根拠のテストです。

- 構造化メッセージとツール引数の深いコピー: 通過（根拠 `TestCheckpointDeepCopyAndBoundaries`）。
- 要約と圧縮のリクエストが上書きしないこと、完結した応答と終了状態の発行、生成途中の半分の応答の除外: 通過（根拠 `TestCheckpointDeepCopyAndBoundaries`、`TestSnapshotExcludesPartialStreamAndSelectsPoolMember`）。
- 実際のモデルプールの構成員の身元: 通過（根拠 `TestSnapshotExcludesPartialStreamAndSelectsPoolMember`）。
- ツールの組合わせ、20 組の再生、予算の削減と超過エラー: 通過（根拠 `TestBuildRequestCompactionToolPairingAndBudget`）。
- メインとサイド質問の並列実行、双方向のキャンセル隔離: 通過（根拠: ブロッキング方式の Provider、`TestMainSideConcurrencyAndIndependentCancellation`）。
- ツール実行がないこと、ストリーミングと非ストリーミング、失敗時点の既存の使用量: 通過（根拠 `TestServiceNoToolsAndUsageOnFailure`）。
- 実際の norma ChatAgent とローカルの Read ツール、メインの transcript とアクティビティの隔離: 通過（根拠 `TestSideActualChatCheckpointToolResultAndTranscriptIsolation` のストリーミングと非ストリーミングの下位ケース）。
- 永続化、ページ分け、冪等性、再起動後の部分応答の保存: 通過（根拠 `TestSideHistoryIdempotencyPagingAndRecovery`）。
- クリアと遅れた書き込みの競合、親リソースの削除、バージョン比較: 通過（根拠 `TestSideClearLateWritersAndDeletedParent`）。
- MainAgent と Worker のアーカイブと復元（v1、v2、v3）: 通過（根拠 `TestSideTaskArchiveVersions`）。
- 3 つの親インターフェース、認証、リソースの帰属、Worker の論理削除: 通過（根拠 `TestSideHTTPGlobalLimitTaskWorkerAndDeletion`、`TestSideCheckpointPersistsBeforeAdmissionAndRestart`）。
- 忙しいメインセッションでもサイド質問ができること、独立した SSE の再接続と切断、キャンセル、クリア: 通過（根拠 `TestSideHTTPBusyIsolationClearAndReconnect`）。
- 親セッションあたり 1 つ、全体で 4 つの同時実行: 通過（根拠: 2 つの `TestSideHTTP…` のケース）。
- 提出前のスナップショット保存、再起動後の続けての質問、古いセッションがスナップショットを偽造できないこと: 通過（根拠 `TestSideCheckpointPersistsBeforeAdmissionAndRestart`）。
- キャッシュにある設定が削除されるか、モデルが変わると、続行を拒否すること: 通過（根拠 `TestSideRejectsDeletedOrChangedCachedProfile`）。
- アーカイブの前にキャンセルし、最終応答と使用量が保存されるのを待つこと: 通過（根拠 `TestSideTaskDrainPersistsBeforeArchive`）。
- ストリーミングの消費者が早くキャンセルしても、使用量を 1 回だけ記録し、サイド質問に帰属させること: 通過（根拠 `TestSideUsageRecordedOnceOnConsumerCancellation`）。
- 再起動で自動復元された Worker と deadline の実行コンテキストが、引き続き新しいスナップショットを発行すること: 通過（根拠 `TestSideRestoredWorkerRuntimePublishesNewCheckpoint`）。
- 関連パッケージの race 検査: 通過（根拠: 下のコマンド）。
- TypeScript とプロダクションビルド: 通過（根拠 `npx tsc --noEmit`、`npm run build`）。
- 新しく追加したフロントエンドモジュールの Biome 検査: 通過（根拠 `biome check`、新しいモジュール 3 つ）。

捨ててもよいデータベースに `ARTEX_PG_DSN` を設定すると、自動化検査を再現できます（運用 DB を指さないでください）。

```sh
go test -race ./agent ./db ./server ./sidequestion ./llmrec ./llmpool \
  -run 'Test(Side|Checkpoint|Snapshot|BuildRequest|Service|MainSide|CaptureRun|TaskArchive|CompleteForwards|StopIntent|CancelIntent)' -count=1
cd web
npx tsc --noEmit
npx biome check src/lib/side-questions.ts src/hooks/use-side-questions.ts src/components/side-question-workspace.tsx
npm run build
```

Go の回帰テスト全体が、すべて通過（green）だったわけではありません。`server` パッケージの既存テスト 2 つが、一時ディレクトリの整理段階で失敗し、どちらも `TempDir RemoveAll … directory not empty` を報告します。

- `TestInheritedActivityDetailAndRelationDeletion`
- `TestTaskMetadataPatchReturnsRenameAndPin`

上の、修正していない基準コミットからソースを取得し、同じ隔離環境で `server` パッケージを回し直しても、この 2 つの整理失敗は同じように再現します。基準コミットの実行では、`TestCoreTaskLifecyclePG` の対象ノード数の断言失敗も別に現れましたが、最終修正版の `server` 回帰では、その断言失敗はありませんでした。ほかのパッケージは通過し、今回のサイド質問に関係するケースと race 検査も通過しました。基準コミットの問題を、今回の検収の通過としては表示せず、隠れた問題を覆うために既存の断言を変えることもしませんでした。

Next.js のビルドは、複数の lockfile と workspace ルートの推論に関する既存の警告を出力しますが、ビルドは完了し、すべてのページが正常に生成されます。

## ブラウザ検査

Codex の組み込みブラウザで、独立したローカルの Go サービスと Next.js の開発サーバーへ接続しました。デスクトップと 390 × 844 の狭い画面で、次の手動と自動の操作を行い、スクリーンショットとブラウザのログを確認しました。

- 一般のチャットが実行されているあいだに `/btw` を入れると、本文の内容とサイド質問が同時に表示され、デスクトップのサイドバーも正常です。
- 続けて追加の質問をしました。サイド質問を停止しても、すでに生成された部分は残り、本文の流れは続きます。
- パネルを閉じてもリクエストは続き、再び開くと完結した応答を復旧します。ページを再読み込みしたあと、内容なしで `/btw` だけを入れると、記録を復旧します。
- 狭い画面の Drawer で、入力、ボタン、記録、閉じる操作が正常で、横方向の溢れはありません。
- クリアは確認のポップアップを出し、クリアしたあとは記録が消えますが、メインの transcript とスナップショットはそのまま残ります。
- タスクの MainAgent と 2 つの Worker にそれぞれ質問して切り替えたとき、エージェントのタブと記録は混ざりませんでした。
- ブロッキング方式のローカルモデルのフィクスチャで Worker を動かし続けた状態で、Worker のメイン入力欄から `/btw` を提出しました。サイド質問を停止したあとも、Worker はリアルタイムの実行状態と自分の一時停止ボタンをそのまま表示し、サイド質問は部分応答を保存しました。
- ブラウザのエラーと警告のログは空です。

制御できるフィクスチャは、同時実行のタイミングを精密に検証するために使ったものであり、実際のモデルの出力速度には依存しません。デバッグ中の 2 回の Worker 実行検査では、有効な同時実行の区間が作れませんでした（タスクがすでに終わっていたか、応答が先に終わった）。フィクスチャを直して再実行し、通過しました。この最初の操作は、有効な通過には数えません。

## 実際のモデルとの会話

まず `grok-4.6` を探査しました。OpenAI 互換インターフェースは `http://127.0.0.1:12580/tingly/openai` です。探査の結果は HTTP 200 とともに、モデル名 `grok-4.6` と `READY` を返し、2.82 秒かかりました。第 1 候補が使えたので、Tingly の `glm` や Zhipu の `glm-5.3` の予備チェーンはオンにせず、この 2 つの予備サービスは今回検証していません。

- メインセッションが実行されているあいだに、資産、目標、標識を質問: `redhaze.top`、最初のページの読み取りと目標の要約、`BTW-REAL-0910` を返し、サイド質問は完了（16.97 秒）。
- メインセッションが最初のページの読み取りを終えたあと、ツールの根拠を質問: WebFetch 200、curl の 301 → 302 → 200 のリダイレクト、ページのタイトルを正確に引用（7.24 秒）。
- サイド質問が Bash でテストファイルを作るよう要求: 実行を拒否し、対象ファイルは作られなかった（7.74 秒）。
- 完了後のサイド質問がメインのコンテキストを変えないこと: メイン transcript の SHA-256 とメインのアクティビティ記録はそのまま一致し、サイド質問のツール実行回数は 0。
- Go サービスを実際に停止して再起動したあと、続けて質問: 以前のサイド質問の記録 3 件を保存し、永続化したスナップショットからすぐに資産、標識、タイトルを答え、メインエージェントは回し直さなかった。
- 新しいセッションで Grok の非ストリーミング設定を使用: 資産と `ATOMIC-0910` を正確に答え、使用量を返して保存した（input 11734、output 138、cache_read 11520）。

資産の事例のメインセッションは、WebFetch と Bash/curl で公開された最初のページを読みました。ランディングページは `https://id.redhaze.top/home`、ページが返したタイトルは「红幕科技 RedHaze Group · 全球综合集团门户」でした（モデルが返した原文なので、そのまま引用します）。Bash は応答をローカルのテストファイルに一時的に保存しただけで、リモートへの書き込みは実行していません。この事実は、「サイド質問がツールを実行しなかった」という点とは別に確認しました。

メイン transcript の検証値: `e7e61f135a4a120954b539f357e8c4205d7d5cd7460dcaf3dc0fd066463e1d00`。

**使用量の限界:** Tingly の Grok ストリーミング応答は usage を返しませんでした。別に `stream_options.include_usage=true` を直接送って検証したとき、HTTP 200、データフレーム 12 個、usage フレーム 0 個でした。したがって、ストリーミングテストでの 0 は、エンドポイントが使用量を提供しないという意味であり、課金がなかったという意味に解釈してはなりません。非ストリーミングの使用量と、フィクスチャの失敗およびキャンセルの使用量は、いずれも正しく保存されました。

## Qwen の審査

審査モデルは `qwen-flash`、OpenAI 互換インターフェースは `https://dashscope.aliyuncs.com/compatible-mode/v1`、HTTP 200 です。前の 3 つの実際のサイド質問の会話、メインセッションのツール根拠、エンジニアリングの断言を渡し、`verdict: accept` と `concerns: []` が返りました。応答が資産、標識、ページ読み取りの証拠と一致し、サイド質問のツール拒否が制約に合う、と判断しました。審査の使用量: prompt 6625、completion 312、total 6937。

今回の Qwen 審査の範囲には、あとから追加したサービスの再起動と非ストリーミングのテストは入りません。Qwen の「書き込みなし」という一般化は広すぎます。メインセッションの curl は実際にローカルの応答の一時ファイルを作っており、これは前にはっきり記録しました。同時性、ツール実行 0 回、transcript の隔離はエンジニアリングの断言で判断し、モデルの審査は応答の品質評価を補助するだけです。
