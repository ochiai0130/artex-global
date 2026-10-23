# ARTEX 検知ルール

日本語 · [English](README.en.md) · [한국어](README.ko.md)

防御・検知ガイド（[日本語](../docs/defense-ja.md) · [English](../docs/defense-en.md) · [한국어](../docs/defense-ko.md)、第 4 節）の疑似ルールを、各自の SIEM や EDR のクエリ言語へ変換して配備できる、ベンダー中立の [Sigma](https://sigmahq.io) 形式にしたものです。ここにある指標はすべて、推測ではなく、このリポジトリのソースで確認した文字列または挙動に根拠を置いています。すべてのルールは、自分が所有する、または書面による許可を得たシステムを守る**防御・検知の目的にのみ**使用してください。

## アトミックルール

- **`sigma/artex_enrich_user_agent.yml`** — ARTEX の資産エンリッチメント（`enrich/enrich.go`）が送る、インバウンドの `artex-enrich/1.0` User-Agent です。標的側で観察する補助指標です。`level: high`。
- **`sigma/artex_selfupdate_egress.yml`** — 自己更新ルーチン（`selfupdate/github.go`）が外へ送る `artex-selfupdate` User-Agent です。ホスト／フォレンジックの egress 指標です。`level: medium`。
- **`sigma/artex_guard_audit_framing.yml`** — ツール呼び出しがブロックされたとき、監査ログへ書き込まれるプラットフォームガードの制御マーカー（`guard/guard.go`）です。ホスト／フォレンジック指標です。`level: high`。
- **`sigma/destructive_command_hunting.yml`** — ARTEX ガードの組み込み拒否リスト（`db/db.go` のシード）を反映した、破壊的なシェル／DB コマンドです。ARTEX 固有のシグネチャではなく、汎用のハンティングの手がかりです。`level: medium`。
- **`sigma/artex_recording_proxy_ca.yml`** — 記録プロキシが `_ca/mitmproxy-ca-cert.pem` という配置で作る MITM CA ファイル（`traffic/traffic.go`）です。ホスト／フォレンジックのアーティファクトであり、ファイル名そのものは単独の mitmproxy と共有されるため、ハンティングの手がかりとして扱います。`level: medium`。

## 相関ルール（挙動）

静的な文字列は変更できますが、挙動は隠すのがより困難です。[`sigma/correlation/`](sigma/correlation/) にある Sigma の**相関**ルールは、防御・検知ガイド（第 4.1〜4.2 節および第 4.4 節）の挙動ベースの層を表しています。各ルールは上のアトミックルールを `id` で参照するため、参照を解決するには相関ファイル 1 つではなく `sigma/` ツリー全体を変換してください（下記を参照）。

- **`sigma/correlation/artex_enrich_scan_velocity.yml`** — 1 つの送信元が短い時間窓の中で `artex-enrich/1.0` のプローブをまとめて送る場合です（エンリッチメントは並行度 4 で、レート制限なしに動きます）。単発ルールが見落とす速度を捉えます。`event_count`、`level: high`。
- **`sigma/correlation/artex_enrich_fanout.yml`** — 1 つの送信元がエンリッチメントの User-Agent を、互いに*異なる*多数のホストへ運ぶ場合です。資産リストを機械の速度で横断するファンアウトであり、量だけではなく幅が手がかりになります。`value_count`、`level: high`。
- **`sigma/correlation/artex_guard_block_burst.yml`** — 1 台のホストでプラットフォームガードの制御マーカーが繰り返し記録される場合です。マーカーを引用した文書ではなく、稼働中の ARTEX が自分のガードに引っかかっている状態です。`event_count`、`level: high`。
- **`sigma/correlation/artex_guard_marker_then_destructive.yml`** — 1 台のホストで、時間窓の中にガードマーカーと破壊的コマンドが同時に現れる場合です（防御・検知ガイド §4.2、多段階）。ARTEX 固有のマーカーを、それだけでは汎用の破壊的コマンド信号と組み合わせることで、特異度を上げます。`temporal`、`level: high`。

閾値と時間窓は保守的な既定値です。各自のベースラインに合わせて調整してください。§4.2 の純粋な Web 多段階（列挙 → プローブ → 認証）は、そのパターンが単一の ARTEX 固有 User-Agent に還元されないため、依然として環境固有のベースルールが必要です。出発点になる汎用の挙動ベース Sigma テンプレートは [防御・検知ガイド §4.2](../docs/defense-ja.md)（[English](../docs/defense-en.md#42-siem-correlation-rules) · [한국어](../docs/defense-ko.md#42-siem-상관-규칙)）にあります。ARTEX のソースに根拠を置けないため、このテスト対象のルールツリーには入れていません。

## ネットワークルール（Suricata）

Sigma が扱うのはホストとログのテレメトリです。回線上で観察できる ARTEX の User-Agent は 2 つあり、どちらも [`suricata/`](suricata/) に [Suricata](https://suricata.io) ルールとして入っています。エンリッチメントプローバの `artex-enrich/1.0`（`enrich/enrich.go`）には存在シグネチャと高速列挙のバリアント（sid 1000001〜1000002）があり、norma SDK の WebFetch ツールが攻撃段階で送る `norma/0.4`（`github.com/Autumn-27/norma/tool/webfetch.go`）には存在シグネチャ（sid 1000003）があります。その他の worker ツール（Bash で動かす `curl` や `nmap`）は各自の User-Agent を使い、ARTEX 固有の指紋を持たないため、ネットワーク層は意図的にこの 2 つの UA に絞っています。範囲、TLS の注意、`suricata -T` と参照 pcap による検証方法は [`suricata/README.md`](suricata/README.md) を参照してください。

## ATT&CK カバレッジ

これらのルールがタグ付けするテクニックは、[MITRE ATT&CK](https://attack.mitre.org/) の Navigator レイヤー [`attack/artex_navigator_layer.json`](attack/) にまとめてあります。6 つのタクティク（偵察、コマンドアンドコントロール、実行、影響、認証情報アクセス、収集）にまたがる 8 つのテクニックで、それぞれルールの `attack.*` タグに根拠を置き、検知強度（ARTEX 固有のシグネチャか、汎用のハンティングの手がかりか）でスコアを付けています。[ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/) で開くと、どのルールがどの ARTEX の挙動を覆うかが見えます。スコアの付け方、テクニックとルールの対応、そして誠実な範囲（カバレッジは網羅ではありません）は [`attack/README.md`](attack/README.md) を参照してください。[一貫性テスト](tests/attack/run.sh) が、レイヤーとルールセットのずれを防ぎます。

## 指標リスト（機械可読）

検知ロジックではなくアトミックな指標そのものが欲しい防御者のために、[`indicators/artex_indicators.csv`](indicators/) は ARTEX が発する固有の指紋を 1 つの CSV に集めています。脅威インテリジェンスプラットフォーム、SIEM のルックアップ、ホストトリアージのチェックリストへそのまま入れられるように、エンリッチメントと自己更新の User-Agent、ガードの監査マーカー、サーバー／プロキシの既定エンドポイント、記録プロキシの CA 証明書、PostgreSQL の探索グラフのスキーマ指紋を載せ、各行には根拠となったソースファイルと、その上に組んだルール（あれば）を記録しています。同じ指標は、すぐにインポートできる [MISP](https://www.misp-project.org/) イベント（[`indicators/artex_indicators.misp.json`](indicators/)）としても提供します。MISP を運用している（またはそこから STIX へエクスポートする）防御者は、CSV の列を手で対応付ける必要がありません。ルールに裏付けられた指紋は `to_ids` とし、ホストフォレンジック用のポートとスキーマ指紋はそうしていません。汎用のハンティングの手がかり（破壊的コマンド）と、norma SDK が共有する `norma/0.4` の WebFetch User-Agent（Suricata の sid 1000003 が捉える回線上のシグネチャであり、ARTEX 固有の文字列ではありません）は、誤検知を避けるためインポート用リストから意図的に外しています。列の構成、MISP のタイプ対応、誠実な注意、そして CSV と MISP イベントの両方のドリフトを防ぐ一貫性テストは [`indicators/README.md`](indicators/README.md) を参照してください。

## ホストトリアージ

上のルールは、SIEM、ネットワークセンサー、または脅威インテリジェンスプラットフォームを持つ防御者のためのものです。もう一方の対応者、つまり SIEM がなく、疑わしいホスト 1 台のシェルの前にいる人のために、[`triage/artex_host_triage.py`](triage/) があります。ローカルの状態から「ここで ARTEX は動いたか」に答える読み取り専用スクリプトです。同じ指紋を運用し、**さらに CSV が意図的に Sigma ルールなしで載せている 3 つのホスト／DB 指標**（サーバーの待受ポート、記録プロキシのエンドポイント、PostgreSQL の探索スキーマ）も確認します。これらはログやネットワークでは観察できず、そのマシン上でしか確認できません。記録機能が子プロセスへ注入する環境変数、つまり実行中プロセスがプロキシ変数と mitmproxy の CA 信頼変数を同時に持っているか（`agent/worker.go`）も、`/proc` または `--proc-from` のダンプから検出します。どの検出も、対応する指標行と同じ但し書きを携えたトリアージの手がかりです。[`triage/README.md`](triage/README.md) を参照してください。組み込みの `--self-test` がマージゲートとして動きます（下記）。

## テスト

ルールには、Docker だけあれば再現できるテストが [`tests/`](tests/) に同梱されています。

- **Suricata**（[`tests/suricata/run.sh`](tests/suricata/run.sh)）は、まずルールファイル全体が `suricata -T --init-errors-fatal` で読み込めることを検証します（どのキャプチャもそのルールを通らなくても、パース失敗は捕捉されます）。次に scapy で決定的なキャプチャを合成し、その上で `suricata -r` を実行して、存在ルールがプローブごとに 1 回発火すること、速度ルールがレート閾値を超えると発火すること、良性の User-Agent のキャプチャではアラートが 0 であることをアサートします。バイナリのキャプチャはコミットせず、実行のたびに再生成します。
- **Sigma**（[`tests/sigma/run.sh`](tests/sigma/run.sh)）は、下記の `sigma check` と `sigma convert` の検証を実行可能なテストとして走らせます。エラー 0 であること、ツリー全体がバックエンドのクエリへコンパイルされること、各アトミック指標の文字列がそのクエリまで残ること、相関ルールは単体では変換に失敗することをアサートします。最後のアサートは、相関ルールが参照するアトミックルールに本当に依存していることの証明です。
- **Sigma のライブイベント照合**（[`tests/sigma_match/run.sh`](tests/sigma_match/run.sh)）は、上の Sigma スイートを妥当性とコンパイルから、アトミックルールと相関ルールの両方について実際の発火まで広げます。各アトミックルールについて、代表的な悪意のあるサンプルイベントが一致し、良性のイベントは一致しないことをアサートします（たとえば `.mitmproxy/` 配下の単独 CA ファイルは、`|all` が `_ca/` ディレクトリも要求する記録プロキシのルールには引っかかりません）。各相関ルールについては、陽性のタイムライン（1 つのグループが時間窓の中で閾値を満たす）が発火し、陰性のもの（閾値未満、ウィンドウ超過、グループ分割、レグ欠落）は静かなままであることをアサートします。パースはすべて pySigma が行い、テストはコンパイルされた条件ツリーと集約仕様を歩くだけです。参照ルールへどのイベントを渡すかは、同じアトミックなマッチャで決めます。「実行できない検知は主張にすぎない」を、Suricata と同じように Sigma 側へ適用します。
- **ATT&CK レイヤー**（[`tests/attack/run.sh`](tests/attack/run.sh)）は、ATT&CK カバレッジレイヤーがルールと整合していることを確認します。スコアを付けたテクニックとタクティクは、ルールセットの `attack.*` タグと正確に一致しなければならず、各テクニックは実在するルールファイルを名指ししなければなりません。レイヤーを更新せずにルールを追加する（またはその逆）と、テストは失敗します。
- **指標の根拠（source-of-truth）**（[`tests/indicators/run.sh`](tests/indicators/run.sh)）は、各ルールが固定した指標が、上流ソースが今も発している文字列であることを確認します。`enrich/enrich.go` の `artex-enrich/1.0`、`selfupdate/` の `artex-selfupdate`、`guard/guard.go` のガードマーカー、`db/db.go` の破壊的トークンが、ルールにもまだ固定されているかを見ます。他の 3 つが見落とすドリフト、つまりすべてのルールがコンパイルも発火もしているあいだに上流の再同期が User-Agent やマーカーを変えてしまう場合を捕捉します。同じテストは機械可読の [`indicators/artex_indicators.csv`](indicators/artex_indicators.csv) を読み直し、公開された各行が今もそのソースとルールに根拠を置いていることをアサートするので、防御者がインポートする成果物もずれません。最後に、読み取る上流ソースがすべて CI ワークフローの `push` と `pull_request` のパスフィルタに載っていることをアサートし、新しく固定したソースだけを触る PR がテストを飛ばして、そのドリフトをマージゲートに通さないようにします。これにより、上で述べた「推測ではなく、このリポジトリのソースで確認した文字列に根拠を置く」は約束ではなくガードになります。
- **MISP エクスポートの一貫性**（[`tests/misp/run.sh`](tests/misp/run.sh)）は、MISP イベント（[`indicators/artex_indicators.misp.json`](indicators/artex_indicators.misp.json)）が妥当な MISP 文書であることを証明します。[pymisp](https://github.com/MISP/PyMISP) で読み込まれ、そのオブジェクトモデルは実在しない属性タイプを拒否するため、成果物は MISP らしく見えるだけではなく、本当にインポートできます。また、上の CSV と行単位で同期していることもアサートします。同じ値、指標ごとに意図した MISP のタイプ／カテゴリ、そして CSV の誠実さに対応する `to_ids`／`disable_correlation` フラグ（ルールに裏付けがある = 対処可能なので `to_ids` は on、ホストフォレンジックのポート = トリアージのヒントなので `to_ids` は off で相関は無効）が一致しなければなりません。イベントは CSV と並べて手で保守します。CSV にはない整理済みのコメント、UUID、タグを持つため、CSV の行を追加・削除・タイプ変更するときは同じコミットで MISP イベントも更新し、両者が一致するまでこのテストは失敗します。
- **Sigma バックエンドの移植性**（[`tests/sigma_backends/run.sh`](tests/sigma_backends/run.sh)）は、ルールが Splunk の例 1 つを超えて変換できることを証明します。ツリー全体（アトミック + 相関）が Splunk、Elasticsearch の `eql` ターゲット、Grafana Loki でコンパイルされ、5 つのアトミックルールは Sigma の相関をサポートしないバックエンド（Elasticsearch の `lucene`、Microsoft の `kusto` バックエンド）でもコンパイルされます。下記の[検証と変換](#検証と変換)にあるバックエンドごとの対応表を、再実行できる検査で裏付けます。
- **SigmaHQ 規約のリント**（[`tests/sigma_lint/run.sh`](tests/sigma_lint/run.sh)）は、SigmaHQ のバリデータ一式（`pySigma-validators-sigmahq` プラグイン。素の `sigma check` はこれを読み込みません）を、[`tests/sigma_lint/validators.yml`](tests/sigma_lint/validators.yml) に文書化したベースラインに対して実行し、issues が 0 であることをアサートします。一式が実際に動いたこと、意図的に除外して文書化した 4 つの検査だけが残っていることも確認するので、ルールが新しい規約上の問題（大文字小文字の誤ったタイトル、タクソノミから外れたフィールド）を 1 つでも拾うとビルドは失敗します。

8 つのルールスイートと並べて、スイートではない 3 つのゲートが同じ CI ワークフローと [`tests/run-all.sh`](tests/run-all.sh) で動きます。ハーネス同期の検査（`run-all.sh`、CI、スイートディレクトリが同じスイートを同じ順序で名指ししていること）、[ホストトリアージツール](triage/) の `--self-test`（[`tests/triage-selftest.sh`](tests/triage-selftest.sh)。合成ホストを作り、すべてのトリアージ検査がそこで発火し、きれいなホストでは検出が 0 であることをアサートします）、そして防御テンプレートの整合検査（[`tests/check-defense-template.sh`](tests/check-defense-template.sh)）です。後者は、防御・検知ガイドがインラインの Sigma 相関テンプレートについて述べている事実を再実行します。`sigma check` が通り、splunk／eql／loki で変換でき、lucene と kusto は相関を拒否し、韓国語版と英語版のルール本体がバイト単位で同一であることです。そのテンプレートはルールツリーの外の散文として存在し、上流の sigma-cli に対して古くなっても、どのスイートも拾わないためです。

各スクリプトは、アサートが 1 つでも失敗すると 0 以外で終了します。[`tests/README.md`](tests/README.md) を参照してください。

## これらのルールを誠実に読む

- **静的指標は変更できます。** 運用者は別の User-Agent を設定できるため、`artex-enrich/1.0` や `artex-selfupdate` が**ないことは、安全を意味しません。** 持続するシグナルは*挙動*です。1 つの送信元が偵察 → 列挙 → プロービング → 認証／インジェクションの試行をつなぎ、応答に適応し、休まず動くことです。その層は防御・検知ガイド（第 1 節、第 2 節、第 4.1〜4.2 節）で説明しており、上の `sigma/correlation/` ルールが配備可能な相関（速度、ファンアウト、ガードブロックのバースト、ガードマーカーと破壊的コマンドの多段階）として提供します。純粋な Web の多段階は、依然として環境固有のベースルールが必要です。
- **破壊的コマンドのルールは汎用ハンティングです。** ARTEX のガード拒否リストを反映していますが、同じコマンドは正当な管理者が実行します。ヒットは手がかりとして扱い、環境に許可リストを置き、それだけで ARTEX に帰属させないでください。
- **ポート指標は Sigma ではなくホストフォレンジックです。** ARTEX サーバーの既定 `:8787` と記録プロキシ `127.0.0.1:8788`（`cmd/artex/main.go`）は、疑わしいホストで `ss`／`netstat` を使って確認するのが適切です。ノイズの多いネットワークルールとしては載せず、防御・検知ガイドに文書化し、トリアージ用に[指標 CSV](indicators/) へ載せています。[ホストトリアージスクリプト](triage/) は、まさにそのホストローカルの確認（ポート、記録プロキシのアーティファクト、ログマーカー、PostgreSQL スキーマ）を、シェルには入れるが SIEM はない対応者のために実行します。

## 検証と変換

これらのルールは [sigma-cli](https://github.com/SigmaHQ/sigma-cli)（pySigma）で検証しています。再現するには次のとおりです。

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install sigma-cli

# 構造とベストプラクティスの検証（期待: エラー 0、issues 0）
sigma check detections/sigma/

# このスタンドアロンのルールセット向けに文書化したベースラインで、
# SigmaHQ の規約一式を検査します（期待: issues 0）。
# 上の素の `sigma check` はこれらのバリデータを読み込みません。
pip install pySigma-validators-sigmahq
sigma check --validation-config detections/tests/sigma_lint/validators.yml detections/sigma/

# 対象のクエリ言語へコンパイルします。例: Splunk
sigma plugin install splunk
sigma convert -t splunk --without-pipeline detections/sigma/artex_enrich_user_agent.yml

# 相関ルールが id で参照するアトミックルールを解決できるよう、ツリー全体を変換します
sigma convert -t splunk --without-pipeline detections/sigma/
```

ベースラインは SigmaHQ の規約をすべて強制しますが、SigmaHQ モノレポのファイル配置とタクソノミを表す 4 つの検査は、スタンドアロンのルールセットには当てはまらないため除外しています。各除外とその理由は [`tests/sigma_lint/validators.yml`](tests/sigma_lint/validators.yml) に文書化され、上のリントテストが強制します。

## Sigma バックエンドの移植性

`sigma/correlation/` のルールはアトミックなベースルールを `id` で参照するため、Sigma の相関変換をサポートするバックエンドでしか変換できません。そのサポートはバックエンドごとに異なるので、`-t` の選択が重要です。下の対応は、固定した基準（`sigma-cli` 3.1.0、互換性のある最新バックエンド）に対して測定し、[`tests/sigma_backends/run.sh`](tests/sigma_backends/run.sh) が再現します。

- **ツリー全体（アトミック + 相関）を変換する:** Splunk（`-t splunk`）、Elasticsearch EQL（`-t eql`）、Grafana Loki（`-t loki`）。`detections/sigma/` を直接変換すると、相関クエリも得られます。
- **アトミックルールのみ（相関はまだ未サポート）:** Elasticsearch Lucene（`-t lucene`）、OpenSearch（`-t opensearch_lucene`）、Sentinel と Defender XDR を対象とする Microsoft の `kusto` バックエンド（`-t kusto`）。これらでは 5 つのアトミックルールを変換し、相関の時間窓は製品の中でネイティブに表現します（例: Sentinel のスケジュール分析の `summarize ... by bin(TimeGenerated, 30m)`）。ディレクトリ全体を渡すと、「Backend does not support correlation rules.」で変換が止まります。

```sh
# アトミックルールのみ。例: Microsoft Sentinel / Defender（kusto バックエンド）
sigma plugin install kusto
sigma convert -t kusto --without-pipeline \
  detections/sigma/artex_enrich_user_agent.yml \
  detections/sigma/artex_selfupdate_egress.yml \
  detections/sigma/artex_guard_audit_framing.yml \
  detections/sigma/artex_recording_proxy_ca.yml \
  detections/sigma/destructive_command_hunting.yml
```

固定したバージョンでの既知の境界: Elasticsearch の ES|QL ターゲット（`-t esql`）はガードマーカーのルールを拒否します（`String value expressions are not supported`）。そこでは残りの 3 つのアトミックルールを変換してください。IBM QRadar プラグイン（`ibm-qradar-aql`）は固定した pySigma と互換がなく `--force-install` が必要なため、テストの対象外です。環境に入っているバックエンドは `sigma list targets` で確認してください。

上の例は `--without-pipeline` を使い、ルール本体の汎用フィールド名（`cs-user-agent`、`cs-host`、`CommandLine`）をそのまま出します。製品のスキーマに合わせるには、そのフラグを外し、`-p` で処理パイプラインを適用してください（`sigma list pipelines` を参照）。ただし製品のパイプラインはフィールド名を対応付けますが、ルールの汎用 `logsource` が指定しない対象テーブルも必要になることがあります。たとえば `-p sentinel_asim` は、データに合わせて `query_table` を設定するまで「Unable to determine table name」で止まるので、配備の前にフィールドと宛先テーブルを環境へ合わせてください。

## 貢献

検知とハードニングの貢献を歓迎します。新しいルールは、すべての指標を観察可能な事実に根拠を置き、限界を `description` に述べ、SigmaHQ バリデータのベースラインをきれいに通過し（`sigma check --validation-config tests/sigma_lint/validators.yml`）、攻撃の手引きとして読める内容を避けてください。[`../CONTRIBUTING.md`](../CONTRIBUTING.md) を参照してください。
