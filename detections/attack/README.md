# ARTEX ATT&CK カバレッジ

日本語 · [English](README.en.md) · [한국어](README.ko.md)

このリポジトリの検知ルールを、それらがタグ付けする ATT&CK（Enterprise）のテクニックへ対応付けた [MITRE ATT&CK](https://attack.mitre.org/) Navigator レイヤーです。[Sigma ルール](../sigma/) の `attack.*` タグから手で作っており、どのテクニックも、指標がこのリポジトリのソースで確認した文字列または挙動であるルールに根拠を置いています。[一貫性テスト](../tests/attack/run.sh) が、レイヤーとルールが互いにずれないように保ちます。このレイヤーは、自分が所有する、または書面による許可を得たシステムを守る**防御・検知の目的にのみ**使用してください。

- **`artex_navigator_layer.json`** — ATT&CK Navigator v4.5 形式のレイヤーです。

## スコアの意味

ここでのカバレッジは「このリポジトリが、このテクニックをタグ付けする検知を提供している」という意味であり、「このテクニックが完全に覆われている」という意味ではありません。スコアは検知強度について意図的に誠実です。

- **100 — ARTEX 固有のシグネチャまたは挙動。** ARTEX に固有の静的指標（`artex-enrich/1.0`／`artex-selfupdate` の User-Agent、ガードの監査マーカー）、またはその上に組んだ挙動ルール（エンリッチメントの速度／ファンアウト、ガードブロックのバースト）です。
- **50–65 — 汎用のハンティングの手がかり。** ARTEX ガードの拒否リストを反映した破壊的コマンドのハンティングです。同じコマンドは正当な管理者が実行するため、良性の活動でも発火します。ヒットは手がかりとして扱い、帰属の根拠にしないでください。65 は、相関ルールがコマンドを ARTEX のガードマーカーと組にして特異度を上げている場合を示します。

## 対象テクニック

6 つのタクティクにまたがる 8 つのテクニックです。それぞれ、それをタグ付けするルールに対応します。

- **偵察（Reconnaissance）— T1595 (Active Scanning)、T1592 (Gather Victim Host Information)。**
  [`sigma/artex_enrich_user_agent.yml`](../sigma/artex_enrich_user_agent.yml)、
  [`sigma/correlation/artex_enrich_scan_velocity.yml`](../sigma/correlation/artex_enrich_scan_velocity.yml)、
  [`sigma/correlation/artex_enrich_fanout.yml`](../sigma/correlation/artex_enrich_fanout.yml)、および
  [Suricata ルール](../suricata/artex.rules)（sid 1000001 / 1000002）。
- **コマンドアンドコントロール（Command and Control）— T1105 (Ingress Tool Transfer)。**
  [`sigma/artex_selfupdate_egress.yml`](../sigma/artex_selfupdate_egress.yml)。
- **実行（Execution）— T1059 (Command and Scripting Interpreter)。**
  [`sigma/artex_guard_audit_framing.yml`](../sigma/artex_guard_audit_framing.yml)、
  [`sigma/correlation/artex_guard_block_burst.yml`](../sigma/correlation/artex_guard_block_burst.yml)、
  [`sigma/correlation/artex_guard_marker_then_destructive.yml`](../sigma/correlation/artex_guard_marker_then_destructive.yml)。
- **影響（Impact）— T1485 (Data Destruction)、T1561.002 (Disk Wipe: Disk Structure Wipe)、T1489 (Service Stop)。**
  [`sigma/destructive_command_hunting.yml`](../sigma/destructive_command_hunting.yml)。T1485 は
  [`sigma/correlation/artex_guard_marker_then_destructive.yml`](../sigma/correlation/artex_guard_marker_then_destructive.yml)
  でも補強されています。
- **認証情報アクセス／収集（Credential Access / Collection）— T1557 (Adversary-in-the-Middle)。**
  [`sigma/artex_recording_proxy_ca.yml`](../sigma/artex_recording_proxy_ca.yml)。worker ツールのトラフィックを復号して記録するために、ARTEX の組み込みトラフィック記録機能（`traffic/traffic.go`）がインストールする MITM ルート CA のアーティファクトです。ホスト／フォレンジックのハンティングの手がかりです。

## 使い方

1. [ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/) を開きます。
2. **Open Existing Layer → Upload from local** を選び、`artex_navigator_layer.json` を選択します（またはこのリポジトリの raw ファイル URL を指定します）。
3. スコアを付けたテクニックが検知強度に応じた色で表示され、それぞれにルールファイルと、その根拠となった防御・検知ガイドの節を記したコメントが付きます。

## 範囲と誠実さ

- **カバレッジは網羅ではありません。** ここでスコアが付いたテクニックは、ルールがそれをタグ付けしているという意味であり、そのテクニックのあらゆる変種を検知するという意味ではありません。回線上で見える ARTEX 固有の User-Agent は 2 つだけです。偵察段階のエンリッチメントプローバ（`artex-enrich/1.0`）と、攻撃段階の norma SDK の WebFetch ツール（`norma/0.4`）です。残りの攻撃トラフィックはツール既定の指紋に従います。持続する検知は挙動です（防御・検知ガイド [日本語](../../docs/defense-ja.md) · [English](../../docs/defense-en.md) · [한국어](../../docs/defense-ko.md) の第 1〜2 節および第 4.1〜4.2 節を参照）。純粋な Web の多段階は、依然として環境固有のベースルールが必要です。
- **静的指標は変更できます。** 運用者は別の User-Agent を設定できるため、タグ付けされた指標がないことは安全を意味しません。ルールファイルにも同じ但し書きがあります。

## 検証と貢献

[一貫性テスト](../tests/attack/run.sh) を実行してください。必要なのは Docker だけで、レイヤーがスコアを付けたテクニックとタクティクがルールの `attack.*` タグと正確に一致すること、そしてすべてのテクニックが実在するルールファイルに根拠を置くことをアサートします。

```sh
detections/tests/attack/run.sh
```

ルールを追加したりタグを付け替えたりしたら、このレイヤーも合わせて更新してください。ルールのテクニックがレイヤーにない、またはレイヤーのテクニックがルールにないと、テストは失敗します。[`../README.md`](../README.md) と [`../../CONTRIBUTING.md`](../../CONTRIBUTING.md) を参照してください。
