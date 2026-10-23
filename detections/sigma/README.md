# ARTEX 検知ルール（Sigma / ホスト・ログ・SIEM）

日本語 · [English](README.en.md) · [한국어](README.ko.md)

ARTEX 検知セットの、ホスト・ログ・SIEM 層です。これらの [Sigma](https://sigmahq.io) ルールは、防御・検知ガイド（[日本語](../../docs/defense-ja.md) · [English](../../docs/defense-en.md) · [한국어](../../docs/defense-ko.md)、第 4 節）の疑似ルールを、各自の SIEM や EDR のクエリ言語へ変換するベンダー中立の形式にしています。どの指標も、推測ではなく、このリポジトリのソースで確認した文字列または挙動に根拠を置いています。ネットワーク層は [`../suricata/`](../suricata/) にあり、ATT&CK カバレッジレイヤー、指標 CSV／MISP エクスポート、ホストトリアージスクリプトを含む検知セット全体は [`../README.md`](../README.md) が索引しています。すべてのルールは、自分が所有する、または書面による許可を得たシステムを守る**防御・検知の目的にのみ**使用してください。

## アトミックルール

ルール 1 つが、観察可能な事実 1 つに対応します。個別に変換しても、ツリー全体の一部として変換しても構いません。

- **[`artex_enrich_user_agent.yml`](artex_enrich_user_agent.yml)** — *ARTEX Asset Enrichment Probe User-Agent*。資産エンリッチメント（`enrich/enrich.go`）からのインバウンド `artex-enrich/1.0` User-Agent です。標的側の補助指標です。`level: high`。
- **[`artex_selfupdate_egress.yml`](artex_selfupdate_egress.yml)** — *ARTEX Self-Update Egress User-Agent*。自己更新ルーチン（`selfupdate/github.go`）からのアウトバウンド `artex-selfupdate` User-Agent です。ホスト／フォレンジックの egress 指標です。`level: medium`。
- **[`artex_guard_audit_framing.yml`](artex_guard_audit_framing.yml)** — *ARTEX Platform Guard Audit-Log Framing*。ツール呼び出しがブロックされたとき監査ログへ書き込まれるプラットフォームガードの制御マーカー（`guard/guard.go`）です。ホスト／フォレンジック指標です。`level: high`。
- **[`artex_recording_proxy_ca.yml`](artex_recording_proxy_ca.yml)** — *ARTEX Recording-Proxy MITM CA Certificate Artifact*。記録プロキシが `_ca/mitmproxy-ca-cert.pem` という配置で作る MITM CA ファイル（`traffic/traffic.go`）です。ホスト／フォレンジックのアーティファクトであり、ファイル名そのものは単独の mitmproxy と共有されるため、ハンティングの手がかりです。`level: medium`。
- **[`destructive_command_hunting.yml`](destructive_command_hunting.yml)** — *Destructive Command Execution (ARTEX Guard-List Hunting)*。ARTEX ガードの組み込み拒否リスト（`db/db.go` のシード）を反映した破壊的なシェル／DB コマンドです。汎用のハンティングの手がかりであり、ARTEX のシグネチャでは**ありません**。`level: medium`。

## 相関ルール（挙動）— [`correlation/`](correlation/)

静的な文字列は変更できますが、挙動は隠すのがより困難です。これらの Sigma **相関**ルールは、防御・検知ガイド（第 4.1〜4.2 節および第 4.4 節）の挙動ベースの層を表しています。各ルールは上のアトミックルールを `id` で参照するため、**相関ファイル 1 つではなく `sigma/` ツリー全体を変換してください**。そうしないと参照は解決しません（[Sigma テスト](../tests/sigma/) がまさにこの依存をアサートします）。

- **[`correlation/artex_enrich_scan_velocity.yml`](correlation/artex_enrich_scan_velocity.yml)** — *Enrichment Scan Velocity*。1 つの送信元が短い時間窓の中で `artex-enrich/1.0` のプローブをまとめて送る場合です（エンリッチメントは並行度 4 で、レート制限なしに動きます）。単発ルールが見落とす速度です。`event_count`、`level: high`。
- **[`correlation/artex_enrich_fanout.yml`](correlation/artex_enrich_fanout.yml)** — *Enrichment Fan-Out*。1 つの送信元がエンリッチメントの User-Agent を、互いに*異なる*多数のホストへ運ぶ場合です。資産リストを機械の速度で横断する幅であり、手がかりはリクエスト量ではなく異なるホストの数です。`value_count`、`level: high`。
- **[`correlation/artex_guard_block_burst.yml`](correlation/artex_guard_block_burst.yml)** — *Guard-Block Burst*。1 台のホストでプラットフォームガードの制御マーカーが繰り返される場合です。マーカーを引用した文書ではなく、実際に稼働している ARTEX が自分のガードに引っかかっている状態です。`event_count`、`level: high`。
- **[`correlation/artex_guard_marker_then_destructive.yml`](correlation/artex_guard_marker_then_destructive.yml)** — *Guard Marker With Destructive Command*。ガードマーカーと破壊的コマンドが、1 台のホストで時間窓の中に同時に現れる場合です（防御・検知ガイド §4.2、多段階）。ARTEX 固有のマーカーを、それだけでは汎用の破壊的コマンド信号と組み合わせることで特異度が上がります。`temporal`、`level: high`。

閾値と時間窓は保守的な既定値です。各自のベースラインに合わせて調整してください。純粋な Web の多段階（列挙 → プローブ → 認証）は、そのパターンが単一の ARTEX 固有 User-Agent に還元されないため、依然として環境固有のベースルールが必要です。出発点になる汎用の挙動ベースのテンプレートは [防御・検知ガイド §4.2](../../docs/defense-ja.md)（[English](../../docs/defense-en.md) · [한국어](../../docs/defense-ko.md)）にあり、ARTEX のソースに根拠を置けないため、このテスト対象のツリーからは外しています。

## 範囲と誠実さ — デプロイ前にお読みください

- **静的指標は変更できます。** 運用者は別の User-Agent を設定したり、CA ファイルを消したりできるため、アトミック指標が**ないことは安全を意味しません。** 持続するシグナルは、`correlation/` ルールが鍵にする挙動です。1 つの送信元が偵察 → 列挙 → プロービング → 認証／インジェクションの試行をつなぎ、応答に適応し、休まず動くことです。
- **破壊的コマンドのルールは汎用ハンティングです。** ARTEX のガード拒否リストを反映していますが、同じコマンドは正当な管理者が実行します。ヒットは手がかりとして扱い、環境に許可リストを置き、それだけで ARTEX に帰属させないでください。
- **ポートとスキーマはホストフォレンジックであり、Sigma ではありません。** サーバーの既定 `:8787` と記録プロキシ `127.0.0.1:8788`（`cmd/artex/main.go`）、および PostgreSQL の探索グラフのスキーマは、疑わしいホスト上で確認するのが適切です。ノイズの多いルールではなく、[指標 CSV](../indicators/) と[ホストトリアージスクリプト](../triage/) で提供しています。
- **`logsource` とフィールド名は汎用です。** ルールは汎用の `category`／`product` ログソースとフィールド名（`cs-user-agent`、`CommandLine`、`TargetFilename`）を使います。変換時にパイプライン（`-p`）で製品のスキーマへ対応付けてください。下記のバックエンドに関する説明を参照してください。

## 検証と変換

[sigma-cli](https://github.com/SigmaHQ/sigma-cli)（pySigma）で検証しています。リポジトリのルートから実行します。

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install sigma-cli

# 構造とベストプラクティスの検証（期待: エラー 0、issues 0）
sigma check detections/sigma/

# このルールセットの文書化したベースラインで、SigmaHQ の規約一式を検査します（期待: issues 0）
pip install pySigma-validators-sigmahq
sigma check --validation-config detections/tests/sigma_lint/validators.yml detections/sigma/

# 相関ルールが id で参照するアトミックルールを解決できるよう、ツリー全体をコンパイルします
sigma plugin install splunk
sigma convert -t splunk --without-pipeline detections/sigma/
```

バックエンドによって相関のサポートが異なるため、`-t` の選択が重要です。Splunk、Elasticsearch EQL、Grafana Loki はツリー全体を変換し、Elasticsearch Lucene、OpenSearch、Microsoft の `kusto` バックエンドは 5 つのアトミックルールだけを変換します（時間窓は製品の中でネイティブに表現します）。測定したバックエンドごとの対応表と、`--without-pipeline`／`-p` のフィールド対応の説明は [`../README.md`](../README.md) にあり、[`../tests/sigma_backends/`](../tests/sigma_backends/) が再現します。

## テスト

[`../tests/`](../tests/) 配下の再現可能な 4 つのスイートがこれらのルールを覆い、それぞれ Docker だけで動きます。[`sigma/`](../tests/sigma/)（検証、ツリー全体のコンパイル、相関ルールが単体では変換に失敗すること）、[`sigma_match/`](../tests/sigma_match/)（ルールが悪意のあるサンプルで実際に発火し、良性では静かであること）、[`sigma_backends/`](../tests/sigma_backends/)（5 つのバックエンドにわたる移植性）、[`sigma_lint/`](../tests/sigma_lint/)（SigmaHQ バリデータのベースライン一式、issues 0）です。[`../tests/README.md`](../tests/README.md) を参照してください。

## 貢献

検知の貢献を歓迎します。新しいルールは、すべての指標を観察可能な事実に根拠を置き、限界を `description` に述べ、SigmaHQ バリデータのベースラインをきれいに通過し（`sigma check --validation-config ../tests/sigma_lint/validators.yml .`）、攻撃の手引きとして読める内容を避けてください。[`../../CONTRIBUTING.md`](../../CONTRIBUTING.md)、および [`../suricata/`](../suricata/) ／ [`../README.md`](../README.md) のネットワーク層を参照してください。
