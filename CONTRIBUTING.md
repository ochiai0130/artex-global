# 貢献ガイド (Contributing)

日本語 · [English](CONTRIBUTING.en.md) · [한국어](CONTRIBUTING.ko.md)

ARTEX グローバル版（`artex-global`）に関心をお寄せいただき、ありがとうございます。この文書は、貢献を始める前に知っておくべき範囲・方針・手順を日本語でまとめたものです。貢献を送る前に、必ず[利用範囲と法的責任](#利用範囲と法的責任)と[ローカライズ方針](#ローカライズ方針)を先にお読みください。

- バグを報告する、または機能を提案するときは → [Issue テンプレート](https://github.com/jiwoochris/artex-global/issues/new/choose)を使ってください。
- 翻訳・ローカライズの誤りを見つけたら → 「翻訳・ローカライズの誤り」Issue テンプレートを使ってください。
- セキュリティ上の脆弱性を見つけたら → **公開 Issue にはせず**、[SECURITY.md](SECURITY.md)の手順に従ってください。
- すべての参加者は、[行動規範(CODE_OF_CONDUCT.md)](CODE_OF_CONDUCT.md)を守らなければなりません。

---

## 利用範囲と法的責任

ARTEX は、LLM マルチエージェントが**自律的に**ペネトレーションテストを行う、攻撃側のセキュリティツールです。貢献者にも、ユーザーと同じ範囲の制限が適用されます。

- コードを検証するときは、**自分が所有している対象、または書面で明示的な許可を得ている対象**、もしくは**ローカルに隔離された環境**（例: Docker で起動した OWASP Juice Shop や DVWA のように、意図的に脆弱であり、自分が所有している対象）に対してだけ、ツールを実行してください。
- 許可された範囲の外にある実システム・本番システム・リモートシステムに対してスキャン・探査・エクスプロイトを行うコード、またはそのような利用を助長する変更は、受け付けません。
- 大韓民国では、無許可で他人の情報通信網に侵入したり、障害を引き起こしたりする行為は「情報通信網の利用促進及び情報保護等に関する法律」に違反します。収集されたり露出したりする個人情報には「個人情報保護法」が適用されます。詳しい告知は[README](README.md#️-最初にお読みください--許可された利用と法的告知)にあります。

貢献したコードや文書がどう使われるかの法的責任は、それを実行するユーザー本人が負います。このリポジトリは「現状のまま」（AS IS）で提供されます。

---

## ローカライズ方針

このリポジトリがあるのは、オリジナルの [Autumn-27/ARTEX](https://github.com/Autumn-27/ARTEX)（現在アクセス不可、2026-10 確認）の**判断性能をそのまま保ちながら、ユーザー向けの出力だけを設定した言語に変える**ためです。この方針から外れる翻訳の貢献は性能を下げることがあるため、受け付けません。

- **エージェントの内部推論プロンプト（行動指示の本文）は翻訳しないでください。** 原文（中国語）でベンチマークされた挙動を保つ必要があります。この本文は `agent/promptcatalog.go` と DB のシード（`agent_prompts`）にあります。翻訳すると、エージェントの判断がドリフトします。
- **ユーザー向けの出力だけを、設定した言語に強制します。** 検出結果（`report_finding`）、事実の要約（`record_fact`）、最終レポート、チャット応答がこれに当たります。この強制は、`agent/prompt.go` の `langDirective()` というコード固定の末尾として、各ロールの system prompt の終わりに付けられます。出力言語を変えるには、この関数を修正してください。
- **コマンド・ペイロード・コード・URL・ログの原文は翻訳しません。** 分析に必要な原文なので、そのまま残します。
- **オリジナルの中国語は残します。** 文書は `README.zh.md` に、UI 文字列は `web/messages/zh.json` に原文をそのまま残し、上流リポジトリの変更と突き合わせやすくします。韓国語の翻訳は `web/messages/ko.json` に書きます。
- UI 文字列を新しく翻訳するときは、ハードコードせず、メッセージファイルのキーとして追加してください。
- **出力言語の強制は、ハードキャップではなく、プロンプトによる誘導です。** `langDirective()` は出力言語を設定した言語にするよう**指示する**だけで、言語を固定するわけではありません。そのため、言語の忠実度はモデルの能力・ロール・文脈によって変わります。ローカライズの変更を検証するときは、能力の高いモデル（フロンティアクラス）を使ってください。安価で小さいモデルでは、レポートや要約が原文（中国語）に戻ることがあります。翻訳が正しく適用されたかどうかは、安価なモデルの出力だけから判断しないでください。OpenAI 系のモデルを使うときの `max_tokens` 設定の落とし穴は、[README の「モデル選択と出力言語」](README.md#モデル選択と出力言語)にまとめてあります。
- **上流の変更に追いつく手順は、メンテナー向けの文書にあります。** オリジナルの ARTEX が更新されたときに、残す資産と翻訳対象を切り分けて反映し、翻訳の対称性とドリフトを検査するランブックは、[MAINTAINING.md](MAINTAINING.md)にまとめてあります。

---

## 開発環境

このプロジェクトは、**Go バックエンド**（単一バイナリにフロントエンドを埋め込む）と **Next.js フロントエンド**で構成されます。

主な機能の設計意図は、`docs/` の設計文書にまとめてあります。脆弱性とトラフィック証拠を結びつける機能（レポートエージェントの自動バインディング、`report_finding` の `traffic_refs` など）を扱うときは、先に[脆弱性の複数トラフィック証拠に関する設計文書](docs/finding-traffic-evidence-ja.md)を読んでください。原文（中国語）は、同じフォルダの `finding-traffic-evidence-zh.md` に保存してあります。

### 要求バージョン

- Go 1.26 以上（`go.mod` 基準）
- Node.js 22 以上（リリースワークフロー基準）
- Docker と Docker Compose（ローカルでの実行と検証用）

### バックエンド (Go)

ローカルに Go が入っていれば、リポジトリのルートで次を実行します。

```bash
go build ./...
go vet ./agent/
go test ./agent/
```

ローカルに Go がなければ、Docker で同じ検証ができます。モジュールとビルドのキャッシュを named volume に置いておくと、再実行が速くなります。

```bash
docker run --rm -v "$PWD":/src -w /src \
  -v artexko-gomod:/go/pkg/mod -v artexko-gocache:/root/.cache/go-build \
  golang:1.26 sh -c 'go build ./... && go vet ./agent/ && go test ./agent/'
```

#### DB 統合テスト（PostgreSQL が必要）

上の `go test ./agent/` は、PostgreSQL につながらないと動かない **DB 統合テストを静かにスキップします。** `agent`・`config`・`db`・`evidence`・`llmrec`・`server` の 6 パッケージには、実際のデータベースがないと動かないテストが入っています。環境変数 `ARTEX_PG_DSN` もなく、設定ファイルにも `database` の項目がなければ、それらのテストは `--- SKIP` で飛ばされ、パッケージは `ok` で終わります。そのため、この 6 パッケージを直したあと DSN なしで確認すると、**ローカルでは通過（ok）するのに、PR の `go-db` ジョブは失敗する**ことがあります。

これらのテストをローカルで動かすには、PostgreSQL を起動して `ARTEX_PG_DSN` を渡します。以下は、CI と同じ `postgres:16-alpine` を隔離ネットワークで起動して実行する例で、上と同じ named volume を再利用します。

```bash
# 1) 隔離ネットワークと空の postgres を起動します（CI と同じイメージ・アカウント）。
docker network create artexko-db 2>/dev/null || true
docker run -d --name artexko-pg --network artexko-db \
  -e POSTGRES_USER=artex -e POSTGRES_PASSWORD=artex -e POSTGRES_DB=artex \
  postgres:16-alpine
until docker exec artexko-pg pg_isready -U artex -d artex >/dev/null 2>&1; do sleep 1; done

# 2) DSN を渡して DB 統合パッケージを実行します（DSN の host はコンテナ名です）。
#    直したパッケージだけ実行するには、./agent/ の部分を config・db・evidence・llmrec・server に置き換えます。
docker run --rm --network artexko-db -v "$PWD":/src -w /src \
  -v artexko-gomod:/go/pkg/mod -v artexko-gocache:/root/.cache/go-build \
  -e ARTEX_PG_DSN='postgres://artex:artex@artexko-pg:5432/artex?sslmode=disable' \
  golang:1.26 sh -c 'go test ./agent/ -count=1'

# 3) 片付けます。
docker rm -f artexko-pg && docker network rm artexko-db
```

CI の `go-db` ジョブは、この 6 パッケージを**それぞれ専用の postgres で隔離し**、マージ前に必ず実行します（`.github/workflows/ci.yml`）。DB 統合パッケージを変更した場合は、PR を出す前に、上の方法でそのパッケージを直接確認することをおすすめします。

### フロントエンド (web)

```bash
cd web
npm ci
npm run dev          # 開発サーバー
npm run build        # 本番ビルド
npm run build:static # 静的エクスポートのビルド（マージゲート · TypeScript の型検査を含む）
npm run check        # Biome のリント・フォーマット検査（情報用 · 既存の負債があるため、まだマージゲートではない）
npm run check:fix    # 自動修正
```

ローカルに Node（要求バージョン 20+）がないとき、またはインストール済みのネイティブ依存がプラットフォームと合わないときは、Docker で同じ検証ができます。`node_modules` を named volume に置くと、ホストのプラットフォーム向けバイナリ（例: `@parcel/watcher` や Biome）と混ざらず、再実行も速くなります。このリポジトリの web 検証は、CI と同じ方法で行います。

```bash
docker run --rm -v "$PWD":/src -w /src/web \
  -v artexko-web-nm:/src/web/node_modules \
  node:22 bash -c 'npm ci && npm run build:static'
```

コミット前のフォーマットとリントは Biome で管理します。`lint-staged` が、ステージされたファイルに対して `biome check --write` を自動で実行します。

### 全体の起動（Docker Compose）

```bash
cp .env.example .env     # POSTGRES_PASSWORD を設定
docker compose up -d     # artex + postgres を起動 → http://localhost:8787
```

---

## 貢献の手順

1. まず **Issue を開きます。** 大きな変更は、作業を始める前に Issue で方向を合わせておくのがよいです。小さな修正（誤字・リンク・明らかなバグ）は、すぐに PR を送ってもかまいません。
2. リポジトリを **フォーク** し、トピックブランチを作ります。ブランチ名は `feat/...`、`fix/...`、`docs/...`、`i18n/...` のように、変更の性質を先頭に置きます。
3. 変更を書き、**その範囲の検証を自分で実行します。** Go の変更であれば、上の `build`・`vet`・`test` を通します。web の変更であれば、`npm run build:static`（マージゲートであり、TypeScript の型検査も同時に行います）を通します。`npm run check`（Biome）は、上流から持ち込まれた既存のリント負債が残っているため、まだマージゲートではなく、`web.yml` では情報用の段階としてだけ実行されます。そのため、全体を通す必要はありません。代わりに、**自分の変更が新しいエラーを増やしていないか**だけを確認すれば十分です（コミットするとき、`lint-staged` がステージしたファイルだけに `biome check --write` を自動で適用します）。文書（`.md`）を変えた場合は、`python3 -I scripts/check-doc-links.py` で、リポジトリ内のリンク・画像の参照と、文書アンカー（`#見出し`）のリンクが壊れていないかを確認します。アンカーは GitHub と同じ規則で見出しから slug を作って照合するため、見出しの文字を変えても、その見出しを指していたアンカーリンクを一緒に直さないと、ここで失敗します（CI の `docs` ワークフローが、同じ検査をマージゲートとして強制します）。この検査は、リポジトリルートの [`.pre-commit-config.yaml`](.pre-commit-config.yaml) に `docs` フックとしても入っているので、`pre-commit install` をしておけば、コミット時に自動で実行されます（Python の標準ライブラリだけを使い、ネットワークにも接続しないため、Docker なしで終わります）。
4. **PR を開きます。** タイトルと説明は [PR テンプレート](.github/PULL_REQUEST_TEMPLATE.md) に従い、何をなぜ変えたか、どのように検証したかを書きます。UI を変えた場合は、スクリーンショットを添付します。
5. ユーザーに見える変更（機能・ローカライズ・文書・検知ルールなど）であれば、[変更履歴 (CHANGELOG.md)](CHANGELOG.md) の `[Unreleased]` の節に 1 行足します。内部のリファクタリングや、テストだけの変更は省いてもかまいません。

### コミットメッセージ

これまでのコミット履歴の慣例に従います。形式は `type(scope): 説明` で、説明は日本語で書きます。

- `type`: `feat` · `fix` · `docs` · `chore` · `refactor` · `test` · `i18n` など
- `scope`: 変わった領域（`agent` · `web` · `server` など）。省略できます

例です。

```
feat(agent): ユーザー向け出力を韓国語に強制 (langDirective)
docs: 韓国語 README を作成し、原文は README.zh.md として保存
i18n(web): ダッシュボードのナビゲーションラベルを韓国語に翻訳
```

---

## 検知ルール・検知テストへの貢献

このリポジトリは、ARTEX のような自律型 AI 攻撃を**防御・検知**するためのルールを [`detections/`](detections/) に一緒に置いています。デプロイできる [Sigma](https://sigmahq.io) ルール（[`detections/sigma/`](detections/sigma/)）、ネットワーク向けの [Suricata](https://suricata.io) ルール（[`detections/suricata/`](detections/suricata/)）、[MITRE ATT&CK](https://attack.mitre.org/) のカバレッジレイヤー（[`detections/attack/`](detections/attack/)）、そしてこれらのルールが実際に発火することを再現可能な形で証明するテスト（[`detections/tests/`](detections/tests/)）で構成されます。検知ルールを新しく送るとき、または直すときは、以下の契約を守ってください。8 つのテストスイートがこの契約のかなりの部分を機械的に強制するため、ルールだけを変えてテストやレイヤーを更新しないと、テストは失敗します。

- **すべての指標を、観測できる事実に基づくようにします。** ルールが使う文字列・User-Agent・行動の閾値は、このリポジトリのソースで実際に確認できるものでなければならず、推測で作ってはいけません。根拠となるソースファイルをルールの中に示してください（例: `artex-enrich/1.0` の指標は `enrich/enrich.go` で確認できます）。指標一致テスト（[`detections/tests/indicators/`](detections/tests/indicators/)）が、各指標が上流のソースとルールの両方にまだあるかを検査します。そのため、上流の再同期でソースの文字列が変わると、ルールを一緒に直さない限りテストは失敗します。機械可読の指標一覧（[`detections/indicators/artex_indicators.csv`](detections/indicators/artex_indicators.csv)）を変えたら、その指標をそのまま載せた MISP イベント（[`detections/indicators/artex_indicators.misp.json`](detections/indicators/artex_indicators.misp.json)）も一緒に更新します。MISP のエクスポートテスト（[`detections/tests/misp/`](detections/tests/misp/)）が、2 つのファイルが行単位で一致すること、そしてそのイベントが pymisp で読み込める妥当な MISP 文書であることを強制します。
- **限界を正直に書きます。** Sigma ルールは `description` に、Suricata ルールはコメントに、そのルールが捉えられない場合と誤検知の可能性を書きます。ARTEX 固有のシグネチャではなく、一般的なハンティングの手掛かり（例: 破壊コマンド）であればそのように明示し、1 回の一致だけで攻撃者を ARTEX だと断定しないようにします。
- **静的検証を通します。** Sigma ルールは、SigmaHQ の検証基準を指摘 0 件で通過しなければなりません（`sigma check --validation-config detections/tests/sigma_lint/validators.yml`）。既定の `sigma check` は pySigma の中核検証器だけを実行するため、タイトルの表記、フィールドやログソースの分類、参照リンクといった SigmaHQ の慣例は、この基準でしかふるい落とされません。4 つの例外は、単独のルールセットには合わない SigmaHQ モノレポの慣例であり、その理由は [`detections/tests/sigma_lint/validators.yml`](detections/tests/sigma_lint/validators.yml) に書いてあります。Suricata ルールは `suricata -T` で問題なくロードされなければなりません。
- **再現可能なテストを一緒に送ります。** ルールが発火するか（または構造が妥当か）を、[`detections/tests/`](detections/tests/) 以下のテストで証明します。入力はバイナリをリポジトリに入れず、毎回決定的に生成します。エンジンのバージョンに依存しない性質（発火すること、誤検知がないこと）は正確にアサートし、バージョンによって揺れる数値は下限でアサートし、基準値は別に記録します。Sigma の相関ルールを追加または修正すると、バックエンド移植性テスト（[`detections/tests/sigma_backends/`](detections/tests/sigma_backends/)）が、そのルールが複数のバックエンドで変換されるかを確認します。[`detections/README.md`](detections/README.md) のバックエンド対応の説明と食い違わないように保ってください。
- **ATT&CK レイヤーを一緒に更新します。** ルールに `attack.*` タグを追加または変更したら、[`detections/attack/artex_navigator_layer.json`](detections/attack/artex_navigator_layer.json) のテクニックとスコアも合わせて更新します。整合テストがルール↔レイヤーの双方向一致を強制するため、レイヤーにないルールのタグや、ルールにないレイヤーのテクニックがあると失敗します。
- **攻撃の案内として読める内容は入れません。** このリポジトリの検知資料は、防御・検知の姿勢だけを保ちます。エクスプロイトの実行方法や検知回避の手法のように、攻撃を助ける記述は受け付けません。

8 つのテストスイートは Docker さえあればそのまま実行でき、生成物をリポジトリにコミットしません。各スクリプトは、アサートが 1 つでも失敗すると 0 以外のコードで終わるため、CI や pre-commit フックにそのまま入れられます。

```bash
detections/tests/sigma/run.sh           # Sigma: sigma check + バックエンド変換 + 指標の保持
detections/tests/sigma_match/run.sh     # Sigma: アトミックルールが悪意のあるサンプルで発火し、正常サンプルでは沈黙する
detections/tests/sigma_lint/run.sh      # Sigma: SigmaHQ 慣例の全検証器 + 文書化された基準
detections/tests/sigma_backends/run.sh  # Sigma の移植性: 相関ルールが複数のバックエンドで変換されるか
detections/tests/suricata/run.sh        # Suricata: pcap を合成 → suricata -r → アラート数をアサート
detections/tests/attack/run.sh          # ATT&CK: レイヤー ↔ ルールの双方向の整合
detections/tests/indicators/run.sh      # 指標: ルールが固定した指標 ↔ 上流ソースの双方向一致
detections/tests/misp/run.sh            # MISP: 指標 CSV ↔ MISP イベントの同期 + pymisp の妥当性
```

8 つを一度に実行するには、[`detections/tests/run-all.sh`](detections/tests/run-all.sh) を使ってください。CI と同じ順序で 8 つを順に実行し、前のスイートが失敗しても残りを最後まで走らせたあと、スイートごとの PASS/FAIL の要約を出力します。1 つでも失敗すれば 0 以外のコードで終わります。このランナーを pre-commit フックとしてそのまま掛ける設定例が、リポジトリルートの [`.pre-commit-config.yaml`](.pre-commit-config.yaml) にあります。`pip install pre-commit && pre-commit install` で導入すると、検知ルール、またはそのルールが固定した上流ソースが変わるコミットのときだけ（CI と同じ範囲で）ランナーが動き、ルールとテストの不一致をプッシュ前に捉えます。同じ設定ファイルには、文書内部のリンク・画像・アンカーを検査する `docs` フック（上の貢献手順 3 の `check-doc-links.py`）も一緒に入っています。

この 8 つのテストは、リポジトリの CI（[`.github/workflows/detections.yml`](.github/workflows/detections.yml)）が、`detections/` 以下が変わったプッシュと PR のたびに実行します。指標一致テストは、その指標が指す上流のソースファイル（`enrich/`・`selfupdate/`・`guard/`・`db/`・`cmd/artex/main.go`）が変わるときにも動き、上流の再同期が User-Agent・マーカー・既定のポートを変えてルールが静かに古くなる場合も一緒に捉えます。したがって、ルールだけを変えてテストやレイヤーを更新していない変更、SigmaHQ の慣例を破ったルール、またはソースと食い違うルールは、マージ前の CI で赤く表示されます。

ルールの索引と各ルールの根拠・限界は [`detections/README.md`](detections/README.md) に、テストのアサート項目と実行方法は [`detections/tests/README.md`](detections/tests/README.md) にまとめてあります。

---

## ライセンス

このプロジェクトは **GNU Affero General Public License v3.0（AGPL-3.0）** で配布されます。貢献を提出すると、その貢献も **AGPL-3.0 で公開されることに同意した**ものとみなされます。特に、このプロジェクトを改変してネットワーク経由で（例: オンラインサービスとして）ユーザーに提供する場合は、そのユーザーに、対応する完全なソースコードを公開しなければなりません。条項の全文は [LICENSE](LICENSE) ファイルにあります。
