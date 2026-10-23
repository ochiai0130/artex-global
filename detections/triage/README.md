# ARTEX ホストトリアージ

日本語 · [English](README.en.md) · [한국어](README.ko.md)

**疑わしいホスト 1 台の上で**実行し、ローカルの状態から「ここで ARTEX は動いたか」に答える読み取り専用のトリアージ補助です。このディレクトリの残りは、SIEM（[Sigma](../sigma/)）、ネットワークセンサー（[Suricata](../suricata/)）、または脅威インテリジェンスプラットフォーム（[指標](../indicators/)）を運用する防御者のためのものです。このスクリプトはもう一方の対応者、つまりホストのシェルの前にいて SIEM がなく、マシン上にあるものから迅速で根拠のある答えが必要な人のためのものです。自分が所有する、または書面による許可を得たホストにのみ使用してください。

このディレクトリの他の資料が載せているのと同じ指紋を運用し、**さらに[指標リスト](../indicators/artex_indicators.csv)が意図的に Sigma ルールなしで載せている 3 つのホスト／DB 指標**も確認します。それらはログやネットワークでは観察できず、ホスト自身の上でしか確認できないためです（`server-listen-port`、`recording-proxy-endpoint`、`postgres-exploration-schema`）。

## 何を確認するか

どの確認も、このリポジトリのソースで検証した文字列またはパスに根拠を置き、どの検出も対応する Sigma ルールまたは指標行と同じ誠実な但し書きを携えます。

- **待受ポート** — `:8787`（管理 UI）と `127.0.0.1:8788`（記録プロキシ）。[`cmd/artex/main.go`](../../cmd/artex/main.go) の `--addr`／`--proxy` フラグの既定値です。稼働中のホストでは `ss`／`netstat`／`lsof` の出力をパースし、`--ports-from` で渡したファイルから読むこともできます。
- **記録プロキシのアーティファクト** — 記録機能が初回起動時に書き込む MITM CA、`<data-dir>/traffic/_ca/mitmproxy-ca-cert.pem`、および隣の `_index/index.sqlite` と `_blobs/`（[`traffic/traffic.go`](../../traffic/traffic.go)。データディレクトリの既定はバイナリの隣の `data/`）です。この CA は、中間者としてトラフィックを記録する機能のトラストアンカーです（ATT&CK T1557）。
- **ログマーカー** — 指定したログファイルの中の、エンリッチメントプローバの UA `artex-enrich/1.0`（[`enrich/enrich.go`](../../enrich/enrich.go)）、自己更新の egress UA `artex-selfupdate`（[`selfupdate/github.go`](../../selfupdate/github.go)）、プラットフォームガードの監査マーカー（[`guard/guard.go`](../../guard/guard.go)。非 ASCII のフレーミングを含めて原文のまま残し、grep が一致するようにしています）です。`.gz`／`.bz2`／`.xz` に圧縮されたローテート済みログも展開して走査するので、ホストのログ履歴も対象になります。標準ライブラリにコーデックがない形式（`.zst`／`.lz4`）は、静かにきれいだと扱うのではなく**スキップ**として報告します。先に展開するか、手で `grep` してください。
- **PostgreSQL の探索スキーマ** — ARTEX ストア（[`db/schema.sql`](../../db/schema.sql)）の二重グラフのテーブル（`exploration_nodes`／`_edges`／`_anchors` と `assets`／`companies`／`activity`、および `agent_prompts` のシード）です。`psql` があれば DSN に対して実行し、なければ手で実行できる読み取り専用クエリをそのまま出力します。
- **プロセスの環境変数注入** — 環境にプロキシ変数（`HTTP_PROXY`／`HTTPS_PROXY`／`ALL_PROXY`）と、`mitmproxy-ca-cert.pem` を指すツールチェーンの CA 信頼変数（`SSL_CERT_FILE`／`CURL_CA_BUNDLE`／`REQUESTS_CA_BUNDLE`／`GIT_SSL_CAINFO`／`NODE_EXTRA_CA_CERTS`）を**同時に**持つ実行中プロセスです。ARTEX は起動するすべての worker ツールへ、まさにこれらを注入します（[`agent/worker.go`](../../agent/worker.go) の `proxyEnv`。[`agent/proxyenv_test.go`](../../agent/proxyenv_test.go) がアサートしています）。変数の**名前はソースにハードコード**されており（変更できるのは値だけです）、運用者がバイナリ名を変えたりポートを変えたりしてもこの手がかりは残ります。待受ポート単体より強いシグナルです。稼働中の Linux ホストでは `/proc` から読み、キャプチャしたダンプは `--proc-from` で読みます。プロキシと mitmproxy CA が同時にある場合は high、mitmproxy CA だけ、または ARTEX 既定のプロキシエンドポイント（`127.0.0.1:8788`）だけは medium として報告します。mitmproxy CA のない企業プロキシは意図的に検出しません。

ヒットは**トリアージの手がかりであり、帰属ではありません。** どの検出もないことは**健全であることの証明ではありません。** 運用者はバイナリ名を変え、データディレクトリを移し、ポートを変更できます。

## プラットフォーム対応

このスクリプトは純粋な Python 3（標準ライブラリのみ）なので、Python 3 が動くところならどこでも動きます。Linux（CI のセルフテスト）と macOS で確認済みです。OS 固有の確認は 2 つあり、どちらも失敗せずきれいに縮退します。

- **稼働中のポート走査** — `ss`、次に `netstat`、次に `lsof` を試し、出力を出した最初のものを使います。Linux では `ss`／`netstat` です。macOS／BSD では `ss` がなく、`netstat` は Linux の `-ltnp` フラグを受け付けないため（空の出力で終了します）、`lsof -nP -iTCP -sTCP:LISTEN` へ落ち、同じ方法でパースします。稼働中の走査の代わりに保存した一覧を読むには `--ports-from` を渡してください。
- **稼働中のプロセス環境の走査** — `/proc` を読むため、Linux でのみ動きます。`/proc` がないホスト（macOS／BSD）では、きれいではなく**スキップ**として報告します。Linux ホストでダンプを取り、`--proc-from` で渡してください（使い方を参照）。

残りの確認（記録プロキシのアーティファクト、ログマーカー、PostgreSQL スキーマ）はファイルシステム、ログファイル、（DSN を渡した場合は）`psql` を読むため、OS に依存しません。

## 使い方

```sh
# ホストを端から端まで確認します
detections/triage/artex_host_triage.py \
    --data-dir /opt/artex/data \
    --log /var/log/syslog --log-dir /var/log/artex \
    --pg-dsn "$ARTEX_PG_DSN"

# 機械可読の検出を出し、何かが発火したら 0 以外で終了します
detections/triage/artex_host_triage.py --data-dir /opt/artex/data --json --exit-code

# オフライン／フォレンジックイメージ: キャプチャしたプロセス環境のダンプを読みます
#   ホストでダンプを作るには:
#   for p in /proc/[0-9]*; do echo "# $p"; tr '\0' '\n' < "$p/environ"; echo; done > proc_env_dump.txt
detections/triage/artex_host_triage.py --proc-from proc_env_dump.txt

# 再現可能なフィクスチャテスト（ホストの状態には触れません）
detections/triage/artex_host_triage.py --self-test
```

このスクリプトは純粋な Python 3 標準ライブラリです。インストールは不要で、ネットワークも使わず、`--self-test` 自身の一時ディレクトリ以外には何も書きません。ホストの状態（開いているポート、データディレクトリ、ログファイル、そして DSN を渡したときだけデータベース）を読み、見つかったものを出力します。終了コードは既定で `0` です（ゲートではなくトリアージです）。`--exit-code` を付けると、指標が 1 つでも発火したとき `1` になります。

## 誠実さをどう保つか

`--self-test` は合成ホストを作ります。仕込んだ CA、インデックス、ブロブストアがあるデータディレクトリ、各マーカーを含むログ、ポート一覧、キャプチャしたプロセス環境のダンプです。すべての確認がそこで発火することをアサートし、次にきれいなホスト、良性のログ、企業プロキシのプロセスでは検出が **0** であること（誤検知がないこと）をアサートします。[`detections` の CI ワークフロー](../../.github/workflows/detections.yml) に組み込まれ、[`detections/tests/run-all.sh`](../tests/run-all.sh) が再実行するので、確認を壊す変更や、grep するソース文字列から指標がずれる変更はマージゲートで失敗します。実行できない検知は主張にすぎません。
