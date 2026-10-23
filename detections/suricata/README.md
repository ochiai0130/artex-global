# ARTEX 検知ルール（Suricata / ネットワーク）

日本語 · [English](README.en.md) · [한국어](README.ko.md)

[Sigma ルール](../sigma/) のネットワーク層の対になるものです。これらの [Suricata](https://suricata.io) シグネチャは、回線上で観察できる 2 つの ARTEX のアーティファクトを覆い、どの指標も推測ではなく、このリポジトリのソースで確認した文字列または挙動に根拠を置いています。ホスト・ログ・SIEM 層は [`../sigma/`](../sigma/) にあり、全体像は防御・検知ガイド（[日本語](../../docs/defense-ja.md) · [English](../../docs/defense-en.md) · [한국어](../../docs/defense-ko.md)）が説明します。すべてのルールは、自分が所有する、または書面による許可を得たシステムを守る**防御・検知の目的にのみ**使用してください。

## ルール — [`artex.rules`](artex.rules)

- **sid 1000001** — `ARTEX enrichment prober User-Agent`。User-Agent が `artex-enrich/` で始まるインバウンド HTTP `GET` です。資産エンリッチメントのプローバ（`enrich/enrich.go:233`）です。単発リクエストの存在指標です。`classtype: attempted-recon`。
- **sid 1000002** — `ARTEX enrichment prober high-rate enumeration`。同じ User-Agent が `detection_filter` のレートである**送信元あたり 300 秒に 30 リクエスト**を超える場合です。単発ルールが見落とす、機械の速度です。Sigma 相関 `artex_enrich_scan_velocity` に対応します。`classtype: attempted-recon`。
- **sid 1000003** — `ARTEX worker WebFetch User-Agent`。User-Agent が `norma/` で始まるインバウンド HTTP リクエストです。norma SDK の WebFetch ツール（`github.com/Autumn-27/norma/tool/webfetch.go`）です。この UA は norma のすべてのバージョン（v0.1.0〜v0.4.3、検証済み）でハードコードされており、リクエストヘッダを変更しない記録プロキシ（`traffic/traffic.go`）を通って標的へ届きます。エンリッチメントプローバと異なり、**攻撃段階**（能動的な脆弱性プロービング）で発火します。`classtype: attempted-recon`。

## 範囲と誠実さ — デプロイ前にお読みください

- **回線上で観察できる ARTEX の User-Agent は 2 つです。** エンリッチメントプローバは偵察中に `artex-enrich/1.0`（`enrich/enrich.go:233`）を送り、norma SDK の WebFetch ツールは攻撃段階に `norma/0.4`（`github.com/Autumn-27/norma/tool/webfetch.go`）を送ります。記録プロキシ（`traffic/traffic.go`）はリクエストヘッダを変更しないため、両方の UA が回線上で標的に届きます。その他の worker ツール（`curl` や `nmap` のような Bash サブプロセス）は各自の User-Agent を使います。それらは汎用のスキャナシグネチャと、[`../sigma/`](../sigma/) 配下の挙動ベースの SIEM ルールで検知してください。
- **User-Agent が見えるのは平文だけです。** トラフィックが平文 HTTP であるか、TLS を終端するプロキシ／WAF で検査される場所に現れます。エンドツーエンドの TLS はこれを暗号化するため、実際に HTTP リクエストバッファが見える場所へ配備してください。
- **静的な User-Agent は運用者が変更できる**ため、ないことは安全を**意味しません**。持続するシグナルは挙動、つまり速度と幅です。sid 1000002（および Sigma の相関層）が速度を鍵にする理由、そして純粋な Web の多段階検知が環境固有のベースルールを必要とする理由はここにあります。
- **意図的に省いたもの。** 自己更新の User-Agent `artex-selfupdate` は GitHub へ HTTPS で向かうため、ネットワークでは観察できません（TLS の SNI だけでは、アラートにするにはありふれすぎます）。監査の制御マーカーは運用者側のログのアーティファクトであり、標的へ向かうトラフィックではありません。[`../sigma/artex_guard_audit_framing.yml`](../sigma/artex_guard_audit_framing.yml) で検知してください。サーバーポート `:8787` と記録プロキシ `127.0.0.1:8788`（`cmd/artex/main.go`）はホストフォレンジック（`ss`／`netstat`）であり、ネットワークシグネチャではありません。
- **指標の 1 つは、このリポジトリ自身のソースではなく、固定した依存関係に根拠を置きます。** 冒頭では、どの指標もこのリポジトリのソースで確認したと述べました。sid 1000003 がその唯一の例外です。その `norma/` User-Agent は norma SDK（`github.com/Autumn-27/norma/tool/webfetch.go`）にハードコードされており、ARTEX はこれをこのツリーへ vendoring せず、`go.mod` で `v0.4.3` に固定した依存関係として使います。したがってこの文字列は、ここにあるファイルではなく、その固定した上流の依存関係（v0.1.0〜v0.4.3 にわたってハードコード、検証済み）に根拠を置きます。指標ソースのテストが ARTEX 自身の発出側は読み直しても、これだけは読み直さない理由はここにあります。[指標リスト](../indicators/README.md) も同じ但し書きを述べています。上の「このリポジトリのソースで確認した」という主張が、言わずに例外を残さないように、ここにも記します。

## 検証とテスト

Suricata 8 で検証しています。読み込みテストはトラフィックを必要とせず、常に実行します。

```sh
# 構文とエンジンの読み込みテスト（期待: "Configuration provided was successfully loaded"）
docker run --rm -v "$PWD/detections/suricata":/r -w /r jasonish/suricata:latest \
  suricata -T -S artex.rules -l /tmp --init-errors-fatal
```

`--init-errors-fatal` は、パースはできても初期化に失敗するルールもハードエラーにするため、静かに捨てられたシグネチャのまま読み込みテストが通ることはありません。

ルールが実際に発火することを確認するには、再現可能な回帰テストが [`../tests/suricata/`](../tests/suricata/) にあります。まず同じ読み込み検査を実行し、次に scapy で決定的なキャプチャを合成し、その上で `suricata -r` を実行してアラート数をアサートします。必要なのは Docker だけです。

```sh
detections/tests/suricata/run.sh
```

sid 1000001 がプローブごとにちょうど 1 回発火すること（35 フローのキャプチャで 35 回）、sid 1000002 が 300 秒に 30 のレートを超えること（Suricata 8.0.7 で **5** 件のアラート、フロー 31〜35）、同じキャプチャを良性のブラウザ User-Agent で流すとアラートが **0** であることをアサートします。シグネチャが特異であることの確認です。[`../tests/README.md`](../tests/README.md) を参照してください。代わりに自分のトラフィックで確認するには、ローカルサーバーに対するループバックの `curl -A 'artex-enrich/1.0'` をキャプチャし、アラートを読んでください。

```sh
suricata -r enrich.pcap -S artex.rules -l out && \
  grep -c '"signature_id":1000001' out/eve.json    # 存在: プローブごとに 1 回
```

## 貢献

検知の貢献を歓迎します。新しいルールは、すべての指標を観察可能な事実に根拠を置き、限界をコメントに述べ、`suricata -T` をきれいに通過し、攻撃の手引きとして読める内容を避けてください。[`../../CONTRIBUTING.md`](../../CONTRIBUTING.md)、および [`../sigma/`](../sigma/) ／ [`../README.md`](../README.md) の Sigma 層を参照してください。
