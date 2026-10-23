# ARTEX 検知ルールのテスト

日本語 · [English](README.en.md) · [한국어](README.ko.md)

> このディレクトリは、[`../`](../) の検知ルールが実際に発火するかを、再現できる形で証明する回帰テストです。バイナリのキャプチャはリポジトリに置かず、パケットキャプチャを毎回決定的に生成したうえで [Suricata](https://suricata.io) に直接通し、アラート数を確認します。すべてのテストは、自分が所有するシステム、または書面による許可を得たシステムを守る **防御・検知の目的に限って** 使ってください。韓国語の全文は **[README.ko.md](README.ko.md)** を参照してください。

[`../`](../) にあるルールが実際に発火すること、そしてそれと同じくらい大切なこととして、良性のトラフィックでは沈黙することを、再現できる形で証明する回帰テストです。実行できない検知ルールは主張にすぎません。これらのテストは、ルールファイルと防御ガイドに書かれた主張を、レビュアーがソースから再実行できるものに変えます。

バイナリのパケットキャプチャはコミットしません。キャプチャは **実行のたびに決定的に合成** され、終わったあと削除されます。そのためテストは、不透明なフィクスチャではなく読めるソースとして配布され、リポジトリを肥大させません。

## すべてのスイートを一度に実行する — [`run-all.sh`](run-all.sh)

[`run-all.sh`](run-all.sh) は、下の 8 つのスイートを CI と同じ順序で、1 つのコマンドにまとめて実行します。8 つの `run.sh` を手で呼び出さなくて済みます。前のスイートが失敗しても、各スイートは最後まで走ります。スクリプトは最後にスイートごとの PASS/FAIL を 1 行でまとめ、どれか失敗していれば 0 以外で終了します。

スイートの前に、ハーネスの自己点検（[`check-harness-sync.sh`](check-harness-sync.sh)）を実行します。このスイート一覧、[CI](../../.github/workflows/detections.yml) のスイートごとのステップ、ディスク上のスイートディレクトリの 3 つが、互いに違うスイートや違う順序を指していると、実行は失敗します。これは 8 つのスイート自身には見えない、唯一の隙間です。3 つのうち 1 つにしか配線されていないスイート（`run-all.sh` の項目がない新しい CI ステップや、どちらにも追加されていないディレクトリ）は、スイートごとのテストはすべて通りながら、ローカルで green の `run-all.sh` が、green な CI を意味しなくなっていることに気づきにくくなります。この点検は 9 つ目のスイートではなくゲートです。下の要約には出ないので、検知スイートは 8 つのままです。

ゲートはもう 2 つ、同じやり方で動きます。スイートのループより前に実行し、要約には出さず、スイートとしては数えません。[ホストトリアージツール](../triage/) の `--self-test`（[`triage-selftest.sh`](triage-selftest.sh)）と、防御テンプレートの整合性チェック（[`check-defense-template.sh`](check-defense-template.sh)）です。後者は、防御ガイドがインラインの Sigma 相関テンプレートについて述べている事実（`sigma check` が通ること、splunk / eql / loki で変換できること、lucene と kusto は相関を拒むこと、韓国語版と英語版のルール本文がバイト単位で同一であること）を、もう一度実行します。そのテンプレートは [`../sigma/`](../sigma/) のルールツリーの外にある本文なので、上流の sigma-cli が進んで主張が古くなっても、ほかのどのスイートも捕まえられません。

```sh
detections/tests/run-all.sh
```

想定される出力（抜粋）:

```
===== detection suites summary =====
  PASS  sigma
  PASS  sigma_match
  PASS  sigma_lint
  PASS  sigma_backends
  PASS  suricata
  PASS  attack
  PASS  indicators
  PASS  misp
RESULT: PASS
```

どれか失敗すると 0 以外で終了するので、pre-commit フックにそのまま入れられます。すぐに使える例が、リポジトリ直下の [`.pre-commit-config.yaml`](../../.pre-commit-config.yaml) にあります。`pip install pre-commit && pre-commit install` で入れると、検知ルール、またはルールがピン留めしている上流のソースファイルに触れるコミットでランナーが発火します。範囲は CI と同じです。個々のスイートが受け取るイメージとバージョンの上書き（`PYTHON_IMAGE`、`SIGMA_CLI_VERSION`、`SIGMAHQ_VALIDATORS_VERSION`、`SURICATA_IMAGE`）はランナーに引き継がれるので、どれかを export すれば、すべてのスイートに一度に適用されます。

## Suricata — [`suricata/`](suricata/)

[`suricata/run.sh`](suricata/run.sh) は、[`../suricata/artex.rules`](../suricata/artex.rules) のネットワークルールを端から端まで動かし、5 つの性質を断言します。

- **妥当** — ルールファイル全体が `suricata -T --init-errors-fatal` で読み込まれます。下のどのキャプチャもそのルールを通さなくても、解析や初期化に失敗したルールはここで捕まります。素の `suricata -r` はそのようなルールを飛ばしても終了コード 0 のままなので、この読み込み検査は、Sigma スイートの `sigma check` による妥当性の断言に相当する、Suricata 側の仕組みです。
- **存在（enrich）** — sid `1000001` は、エンリッチのプローブごとにちょうど 1 回発火します。
- **速度** — ソースごとに 300 秒あたり 30 リクエストという `detection_filter` のレートを超えると、sid `1000002` が発火します。
- **存在（WebFetch）** — sid `1000003` は、norma の WebFetch リクエストごとにちょうど 1 回発火し、同じキャプチャでは enrich の sid は沈黙します。2 つのネットワークシグネチャは、それぞれ存在するだけでなく、互いに特異的です。
- **特異性** — ほかは同一で、User-Agent だけを良性のブラウザに変えたキャプチャは、ARTEX のアラートを **1 件も** 出しません。

[`suricata/gen_pcap.py`](suricata/gen_pcap.py) は [scapy](https://scapy.net) でキャプチャを作ります。固定した 1 つの送信元から、N 本の独立した平文 HTTP リクエスト／レスポンスのフローを作り、それぞれに選んだ User-Agent を載せ、固定した基準タイムスタンプから 1 秒間隔で並べます。ファイルを書くだけで、パケットを送ることも、ネットワークに触れることもありません。

### 実行する

必要なのは Docker だけです。scapy と Suricata はどちらもコンテナで動きます。

```sh
detections/tests/suricata/run.sh
```

想定される出力（抜粋）:

```
  PASS  ruleset loads with zero parse/init errors (suricata -T)
  PASS  sid 1000001 presence: one alert per probe  (got 35, want eq 35)
  PASS  sid 1000002 velocity: fires past 30-in-300s  (got 5, want ge 1)
  PASS  sid 1000003 presence: one alert per WebFetch request  (got 8, want eq 8)
  PASS  enrich sids stay silent on norma traffic (specificity)  (got 0, want eq 0)
  PASS  benign browser UA produces no ARTEX alerts  (got 0, want eq 0)
RESULT: PASS
```

どれかの断言が失敗するとスクリプトは 0 以外で終了するので、CI や pre-commit フックにそのまま入れられます。内部にミラーがある場合は、`SURICATA_IMAGE` / `PYTHON_IMAGE` でイメージを上書きできます。

### 速度の件数を、一致値ではなく下限にする理由

`run.sh` は、存在の件数（`1000001 == 35`、`1000003 == 8`）と良性の件数（`== 0`）を正確に断言します。これらはエンジンのバージョンに依存しないからです。一致するリクエストごとにアラート 1 件、別の User-Agent では不一致です。速度ルールの件数は、その Suricata のリリースが境界で `detection_filter` の閾値をどう解決するかに依存するので、テストは `>= 1` を断言し、基準値は別に記録します。**Suricata 8.0.7** では、基準の実行が sid `1000002` に **5** 件のアラートを出します（300 秒に 30 件の閾値を超えたあとの、フロー 31–35）。

## Sigma — [`sigma/`](sigma/)

[`sigma/run.sh`](sigma/run.sh) は、[`../sigma/`](../sigma/) の Sigma ルールを、構造と、[sigma-cli](https://github.com/SigmaHQ/sigma-cli)（pySigma）によるコンパイルの両方で検証し、5 つの性質を断言します。

- **妥当** — `sigma check` がツリー全体で、エラー 0、条件エラー 0、issue 0 を報告します。
- **コンパイルできる** — `sigma convert -t splunk` が、ツリー全体をエラーなくバックエンドのクエリ言語に変換します。
- **指標が残る** — 各アトミック指標の文字列（`artex-enrich/1.0`、`artex-selfupdate`、ガードマーカー、記録用プロキシの CA ファイル名 `mitmproxy-ca-cert.pem`）が、コンパイルされたクエリに残ります。ルールが、自分の土台になっている文字列を静かに失うことはできません。
- **相関がコンパイルできる** — [`../sigma/correlation/`](../sigma/correlation/) の振る舞いルールは、捨てられずに `event_count` / `value_count` の集約を出します。
- **相関は実体として必要** — 相関ルールを *単独で* 変換すると失敗します。アトミックなベースルールを `id` で参照しており、その参照は飾りではなく強制されるためです。これは、上の Suricata の特異性の断言に相当する、Sigma 側の仕組みです。

これは、[`../README.md`](../README.md) に書かれた構造とコンパイルの検証を、実行でき、断言する形にしたものです。対になる下の [`sigma_match/`](sigma_match/) スイートが、アトミックルールと相関ルールの両方に *マッチング* の半分を足します。代表的な悪意のあるイベント（またはタイムライン）が各ルールを発火させ、良性のものは発火させない、という確認です。これで Sigma ルールも、Suricata ルールと同じように、再現できる検証テストと再現できるマッチングテストの両方を持ちます。（弱い手書きのマッチャがルールを掘り崩す、という以前の懸念は、解析をすべて pySigma に委ねることで扱っています。信頼モデルは次の節で説明します。）

### 実行する

必要なのは Docker だけです。sigma-cli と splunk バックエンドはコンテナで動き、リポジトリには何も書き込みません。

```sh
detections/tests/sigma/run.sh
```

想定される出力（抜粋）:

```
  PASS  sigma check: 0 errors, 0 condition errors, 0 issues
  PASS  whole tree converts to splunk (exit 0)
  PASS  indicator present: artex-enrich/1.0
  PASS  correlation rule fails to convert alone — it requires its atomic base rule
RESULT: PASS
```

どれかの断言が失敗するとスクリプトは 0 以外で終了するので、CI や pre-commit フックにそのまま入れられます。sigma-cli は基準バージョン（`3.1.0`）にピン留めしてあります。内部にミラーがある場合は、`SIGMA_CLI_VERSION` でバージョンを、`PYTHON_IMAGE` でイメージを上書きできます。

## Sigma のライブなイベントマッチング — [`sigma_match/`](sigma_match/)

[`sigma_match/run.sh`](sigma_match/run.sh) は、[`../sigma/`](../sigma/) の Sigma ルール — アトミックルールと、[`../sigma/correlation/`](../sigma/correlation/) の相関ルールの両方 — が、一致するイベント（またはタイムライン）で実際に *発火* し、良性のものでは静かであることを証明します。Suricata スイートがネットワークルールに与える「実行できない検知は主張にすぎない」という保証を、ホスト／ログ層のルールへ広げたものです。アトミックルールに 3 つ、相関に 3 つ、合わせて 6 つの性質を断言します。

- **ルールとサンプルの対応** — すべてのアトミックルールに [`events/<name>.json`](sigma_match/events/) のサンプルファイルがあり、すべてのサンプルファイルはルールへ戻ります。サンプルなしで追加されたルールは、未検証のまま通り過ぎず、ここで失敗します。
- **真陽性** — 各ルールは、自分の悪意のあるサンプルイベントのすべてに一致します。
- **真陰性** — 各ルールは、自分の良性サンプルイベントのどれにも一致しません。たとえば `.mitmproxy/` の下にある単独の `mitmproxy-ca-cert.pem` は、記録用プロキシのルールを発火させ**ません**。その `|all` 修飾子が、ARTEX が書く `_ca/` ディレクトリも要求するからです。その区別を証明するのがマッチングテストです。
- **相関ルールとタイムラインの対応** — すべての相関ルールに [`events/correlation/<name>.json`](sigma_match/events/correlation/) のタイムラインファイルがあり、すべてのタイムラインはルールへ戻ります。各タイムラインのイベントは、相対秒の `ts` フィールドを持ちます。
- **相関の真陽性** — 1 つのグループの中で、ウィンドウ内に閾値が満たされる陽性タイムラインに対して、各ルールが発火します。たとえば、1 つの送信元（`c-ip`）から 10 分以内に 20 の異なるホストへ扇状に広がるリクエストは、エンリッチのファンアウトルールを発火させます。
- **相関の真陰性** — 閾値に届かないとき、閾値には届いてもイベントがウィンドウの外に散らばっているとき、グループをまたいで分かれているとき、あるいは時間的なルールの一方のレッグが欠けているときは、各ルールは沈黙します。とくに、量は多くても幅の狭いバーストはファンアウトルールを発火させ**ません**。信号は量ではなく幅であり、その区別を証明するのがマッチングテストです。

信頼モデルは、手書きのコードではなく pySigma が各ルールを解析する、というものです。アトミックルールは条件木になり（`|contains` はワイルドカード値、`|all` は AND、`1 of selection_*` は OR）、相関ルールは集約の仕様（種別、group-by、timespan、閾値、そして解決済みのアトミックなベースルールへの参照）になります。[`check.py`](sigma_match/check.py) はその木と仕様をたどるだけで、参照されたルールにどのイベントを渡すかは、まったく同じアトミックマッチャで決めます。権威のある Sigma の論理は pySigma の側に残ります。明示的にサポートしていない構文に出会うと、通さずに例外を送出します（fail-closed）。範囲と限界はスクリプトのヘッダに書いてあります。相関のウィンドウは、標準的なスライディングウィンドウの解釈（マッチしたイベントごとに `timespan` 秒の窓を置く）であり、実際の SIEM のウィンドウの切り方は異なることがあります。マッチングは **大文字と小文字を区別しません**（`sigma/` スイートが対象にする splunk バックエンドの既定であり、破壊的コマンドのルール自身の誤検知の注記も、これを前提にしています）。キーワードのマッチングは、全文の部分文字列検索です。これはルールのフィールド、値、条件、集約の論理に対する回帰テストであり、フィールドの正規化が異なりうる、各自の SIEM での検証に代わるものではありません。

### 実行する

必要なのは Docker だけです。pySigma はコンテナで動き、リポジトリには何も書き込みません。

```sh
detections/tests/sigma_match/run.sh
```

想定される出力（抜粋）:

```
  PASS  rule/sample pairing: 5 atomic rules, 5 event files, no orphans
  PASS  artex_enrich_user_agent: 1/1 positive events matched
  PASS  artex_recording_proxy_ca: 2/2 benign events correctly not matched
  PASS  rule/timeline pairing: 4 correlation rules, 4 timeline files, no orphans
  PASS  artex_enrich_fanout: fired — 20 distinct hosts from one source within the 10-minute window
  PASS  artex_enrich_fanout: quiet — high volume, low breadth: 25 requests from one source but only 4 distinct hosts
RESULT: PASS
```

どれかの断言が失敗するとスクリプトは 0 以外で終了するので、CI や pre-commit フックにそのまま入れられます。pySigma は基準バージョン（`2.0.0`）にピン留めしてあります。内部にミラーがある場合は、`PYSIGMA_VERSION` でバージョンを、`PYTHON_IMAGE` でイメージを上書きできます。

## Sigma バックエンドの移植性 — [`sigma_backends/`](sigma_backends/)

[`sigma_backends/run.sh`](sigma_backends/run.sh) は、Sigma テストが動かしている単一の Splunk の例を超えてもルールが変換できることを証明し、[`../README.md`](../README.md) にあるバックエンドごとの対応表を正直に保ちます。Sigma の相関変換はバックエンドに依存するので、README は防御側に、どの `-t` 対象がツリー全体を受け、どの対象がアトミックルールだけを受けるかを伝えます。再実行して初めて信頼できる主張です。2 つの性質を断言します。どちらも肯定形なので、テストが失敗するのは実際の回帰があるときだけです。

- **相関は移植できる** — ツリー全体（アトミック + 相関）が Splunk、Elasticsearch の `eql` 対象、Grafana の `loki` で変換され、enrich の指標が各クエリに残ります。相関ルールが Splunk 専用ではないことを示します。
- **アトミックだけのフォールバックが動く** — 5 つのアトミックルールは、`lucene` と Microsoft の `kusto` バックエンドでも変換されます。ピン留めしたバージョンでは、これらは Sigma の相関変換をサポートしないので、そのバックエンドを使う防御側はアトミックルールを配置し、相関ウィンドウはネイティブの機能で表現できます。

「バックエンド X は相関を扱えない」という否定形は、わざと断言しません。そうすると、バックエンドが *良くなる* ことが赤いビルドになるからです。正直な限界は README にあり、このテストのコマンドがそれを再現します。[`sigma_backends/check.sh`](sigma_backends/check.sh) はコンテナの中の半分です。ピン留めした sigma-cli と 4 つのバックエンドをインストールし、読み取り専用でマウントしたルールツリーを読みます。

### 実行する

必要なのは Docker だけです。sigma-cli とバックエンドはコンテナで動き、リポジトリには何も書き込みません。

```sh
detections/tests/sigma_backends/run.sh
```

想定される出力（抜粋）:

```
  PASS  whole tree (atomic + correlation) converts on 'eql', enrich indicator survives
  PASS  five atomic rules convert on 'kusto', enrich indicator survives
RESULT: PASS
```

どれかの断言が失敗するとスクリプトは 0 以外で終了します。sigma-cli はピン留めしてあります（`3.1.0`、`SIGMA_CLI_VERSION` で上書き）。バックエンドのプラグインは、互換性のある最新版でインストールされます。そのためこのスイートは、上流のバックエンドのリリースに最も敏感です。サポートを落としたプラグインはビルドを赤くし、それが、ピンと README の表を一緒に更新する合図です。

## SigmaHQ の規約リント — [`sigma_lint/`](sigma_lint/)

[`sigma_lint/run.sh`](sigma_lint/run.sh) は、README と `CONTRIBUTING.md` にある「`sigma check` をきれいに通る」という約束が、pySigma の中核の検査だけでなく、SigmaHQ の規約まで含むようにします。素の `sigma check` は `pySigma-validators-sigmahq` プラグインを読み込まないので、タイトルの大文字小文字、フィールド名の分類体系、ログソースの分類体系、参照リンクの規約は検査されずに通ります。このスイートはそのプラグインをインストールし、[`sigma_lint/validators.yml`](sigma_lint/validators.yml) に文書化したベースラインに対して一式を実行します。2 つの性質を断言します。

- **文書化したベースラインはきれい** — `validators.yml` を付けた `sigma check` が、エラー 0、issue 0 を報告します。
- **一式は生きていて、残るのは文書化した除外だけ** — 除外なしですべての SigmaHQ バリデータを実行しても issue が報告され、そのそれぞれは `validators.yml` がわざと無効にしている 4 つの検査のどれかです（それ以外はない）。これは空虚さを防ぐガードです。プラグインの読み込みに失敗すると、全体の実行は何も報告せず、最初の性質が間違った理由で通ってしまうので、既知の除外が現れることを要求します。

4 つの除外は、SigmaHQ のモノレポの置き方（ログソースの接頭辞と `correlation_` のファイル名）と分類体系（汎用の `application` ログソースと、製品名のない `process_creation`）、そしてブランチ対パーマリンクの参照規約を表します。どれも、自分の生きている文書を参照する、小さく独立したルール集には合いません。各除外の理由は `validators.yml` の中に書いてあります。*それ以外* の SigmaHQ の検査はすべて強制されるので、新しい規約の issue を拾ったルール — 大文字小文字の崩れたタイトル、分類体系から外れたフィールド名 — はビルドを赤くします。`pySigma-validators-sigmahq` はピン留めしてあります（`0.21.0`、`SIGMAHQ_VALIDATORS_VERSION` で上書き）。バージョンを上げると新しい規約が現れることがあり、それが、ルールか文書化したベースラインを更新する合図です。

### 実行する

必要なのは Docker だけです。sigma-cli とバリデータのプラグインはコンテナで動き、リポジトリには何も書き込みません。

```sh
detections/tests/sigma_lint/run.sh
```

想定される出力（抜粋）:

```
  PASS  sigma check with the documented baseline: 0 errors, 0 issues
  PASS  every reported issue is one of the four documented exclusions
RESULT: PASS
```

## ATT&CK レイヤー — [`attack/`](attack/)

[`attack/run.sh`](attack/run.sh) は、[`../attack/artex_navigator_layer.json`](../attack/artex_navigator_layer.json) の [ATT&CK カバレッジレイヤー](../attack/) が、カバーすると主張するルールと一致し続けることを確認します。ルール集からずれたカバレッジレイヤーは、レイヤーがない状態よりも害があります。このテストは、「これらのルールがこれらの ATT&CK テクニックをカバーする」を、レビュアーがソースから再実行できるものに変えます。断言するのは次のとおりです。

- **妥当なレイヤー** — ファイルは JSON として解析でき、必須の ATT&CK Navigator v4.x のフィールドを持ち、すべての項目に、形の正しいテクニック ID と有効な ATT&CK の戦術があります。
- **双方向の一致** — スコアの付いたテクニックは、Sigma ルールの `attack.*` テクニックタグと *ちょうど* 一致します。レイヤーに欠けたルールのテクニックもなく、ルールにないレイヤーのテクニックもありません。戦術も同じやり方で一致します。
- **根拠がある** — スコアの付いたすべてのテクニックのコメントが、実在するルールファイルを名指しするので、レイヤーは、名前が変わった、または削除されたルールを引用できません。

これは発火テストではなく、一貫性の検査です。検知バックエンドは要らず、Python の標準ライブラリだけで動くので、Sigma と Suricata のテストと違い、バージョンに依存する件数はありません。[`attack/check.py`](attack/check.py) はコンテナの中の半分です。読み取り専用でマウントした検知ツリーを読み、何も書きません。

### 実行する

必要なのは Docker だけです。検査は Python コンテナで動き、リポジトリには何も書き込みません。

```sh
detections/tests/attack/run.sh
```

想定される出力（抜粋）:

```
  PASS  scored techniques match the rule set exactly (8: T1059, T1105, T1485, T1489, T1557, T1561.002, T1592, T1595)
  PASS  scored tactics match the rule set exactly (collection, command-and-control, credential-access, execution, impact, reconnaissance)
RESULT: PASS
```

どれかの断言が失敗するとスクリプトは 0 以外で終了するので、CI や pre-commit フックにそのまま入れられます。内部にミラーがある場合は、`PYTHON_IMAGE` でイメージを上書きできます。

## 指標の正本 — [`indicators/`](indicators/)

[`indicators/run.sh`](indicators/run.sh) は、上の 3 つのテストが証明しない 1 つのことを証明します。各ルールがピン留めした指標が、いまも ARTEX 自身のソースが実際に出す文字列であることです。Sigma のテストは、指標がルールからクエリへの *コンパイル* を生き延びることを証明します。ATT&CK のテストは、レイヤーがルールのタグと一致することを証明します。Suricata のテストは、ネットワークルールが合成したキャプチャで *発火* することを証明します。どれも、指標が由来すると主張するソースファイルを振り返りません。それらがみな見逃す腐敗は、プローバの User-Agent を `artex-enrich/2.0` に上げる、またはガードマーカーを書き換える、上流の再同期です。それでもルールはすべてコンパイルでき、レイヤーは一致したままで、pcap のテストも発火します。そして、配置されたルールは、実際の ARTEX トラフィックへの一致を静かにやめます。各指標について、双方向に断言します。

- **ソースがいまも出している** — 値が、それを生み出す上流のソースファイルにあること（`enrich/enrich.go` の `artex-enrich/1.0`、`selfupdate/` の `artex-selfupdate`、`guard/guard.go` のガードマーカー）。値がないことは、ルールがまだ追いついていない上流の変更を意味します。
- **ルールがいまもピン留めしている** — 値が、それを土台にしたルールにあること。ルールの編集が、指標をソースから静かに引き離せないようにします。Suricata のルールは `startswith` のプレフィックスで確認します。ワイヤ上で実際に一致するやり方と同じです。
- **拒否リストの対応** — 破壊的コマンドのトークン（`rm -rf`、`mkfs`、`DROP DATABASE`、`FLUSHALL`）が、ARTEX のガードの拒否リスト（`db/db.go`）と、それを映したハンティングルールの両方に現れること。これらは固有の指紋ではなく、一般的なハンティングの手がかりなので、テストが断言するのは、ルールが実際に主張する対応だけです。
- **公開リストが根拠を保つ** — 防御側が取り込む成果物である、機械可読の指標リスト [`detections/indicators/artex_indicators.csv`](../indicators/artex_indicators.csv) を、行ごとに読み直します。すべての値は、引用するソースファイルにいまもあり、引用するルールにピン留めされていなければならず、テストが根拠を確認したすべての指紋は、このリストに現れなければなりません。公開された CSV は、由来すると主張するソースから、どちらの方向にも静かにずれることができません。
- **両方のゲートが、ピン留めした各ソースで発火する** — テストが読むすべての上流ソースは、それを動かす 2 つのゲートに含まれます。CI ワークフローの `push` と `pull_request` の paths フィルタ（[`.github/workflows/detections.yml`](../../.github/workflows/detections.yml)）と、ローカルの pre-commit フックの `files` 正規表現（[`.pre-commit-config.yaml`](../../.pre-commit-config.yaml)）です。必要な集合は指標自身から導かれるので、新しいソースをピン留めしながら（以前の `cmd/artex/main.go` のポートがそうだったように）、*両方* のゲートへ配線しないと、ここで失敗します。そうしないと、そのソースだけに触れる変更は、そのソースを省いたゲートでテストを飛ばします。CI ではマージゲートを green のまま通過し、フックでは「CI と同じソース範囲」と約束しているのに、ローカルでは最後まで捕捉されません。

これは、[`../README.md`](../README.md) の約束 — 「ここにあるすべての指標は、推測ではなく、このリポジトリのソースで検証した文字列に根拠を置く」 — と、CONTRIBUTING の最初の貢献契約を、レビュアーが再実行できるガードに変えます。ATT&CK のテストと同じく、検知バックエンドは要らず、Python の標準ライブラリだけで動きます。[`indicators/check.py`](indicators/check.py) は、ルールツリー、公開された指標リスト、ピン留めしたソースパッケージ、そしてそれを発火させる 2 つのゲート（CI ワークフローと pre-commit の設定）を読み取り専用でマウントして読み、何も書きません。

### 実行する

必要なのは Docker だけです。検査は Python コンテナで動き、リポジトリには何も書き込みません。

```sh
detections/tests/indicators/run.sh
```

想定される出力（抜粋）:

```
  PASS  enrichment prober User-Agent: 'artex-enrich/1.0' emitted by enrich/enrich.go
  PASS  detections/sigma/artex_enrich_user_agent.yml pins 'artex-enrich/1.0'
  PASS  'FLUSHALL' present in both db/db.go and detections/sigma/destructive_command_hunting.yml
  PASS  enrich-user-agent: 'artex-enrich/1.0' grounded in enrich/enrich.go
  PASS  tested fingerprint 'artex-enrich/1.0' is published in the list
  PASS  .github/workflows/detections.yml push paths covers cmd/artex/main.go
  PASS  .pre-commit-config.yaml files covers cmd/artex/main.go
RESULT: PASS
```

どれかの断言が失敗するとスクリプトは 0 以外で終了するので、CI や pre-commit フックにそのまま入れられます。内部にミラーがある場合は、`PYTHON_IMAGE` でイメージを上書きできます。

## MISP エクスポートの一貫性 — [`misp/`](misp/)

[`misp/run.sh`](misp/run.sh) は、指標の 2 つ目の公開形態、すぐにインポートできる MISP イベント [`detections/indicators/artex_indicators.misp.json`](../indicators/artex_indicators.misp.json) を扱います。上の指標テストが CSV をソースに根拠づけるなら、このテストは、防御側が実際に脅威インテリジェンスのプラットフォームへ読み込む成果物である MISP イベントが、その CSV からずれないように保ちます。断言するのは次のとおりです。

- **本当に MISP である** — イベントは [pymisp](https://github.com/MISP/PyMISP) で読み込まれ、そのオブジェクトモデルは、`type` が本物の MISP の型でない属性を拒みます。もっともらしく見えても無効な型はここで失敗するので、「妥当な MISP」は主張ではなく、MISP サーバが使うライブラリによって証明されます。
- **CSV と行ごとの同期** — すべての CSV の行が、意図した型とカテゴリを持つ MISP 属性ちょうど 1 つに対応し（`http.user-agent` → `user-agent`、ガードマーカーの `string` → `pattern-in-file`、`port` → `port`、`ip-dst|port` → 合成した `ip|port` の値を持つ `ip-dst|port`、探索スキーマの `other` → `other`）、CSV の行を持たない MISP 属性は残りません。イベントは CSV と並んで手で維持するので、同じコミットで `artex_indicators.misp.json` を合わせて更新せずに CSV の行を追加、削除、または型変更すると失敗します。
- **`to_ids` が `rule` 列を映す** — ルールに裏づけられた指標は `to_ids: true` です。ルールのないホストフォレンジックの行は、`disable_correlation: true` とともに `to_ids: false` です。CSV が含意するものからフラグをひっくり返すと失敗するので、MISP イベントは、どの指紋が対処可能かを、静かに過大にも過小にも主張できません。
- **ガードマーカーがバイト単位で残り**、`detections/**` が CI の paths フィルタにあるので、CSV やイベントを変えるとこのスイートが発火します。

上の、標準ライブラリだけで動くテストと違い、このスイートはコンテナの中にピン留めした `pymisp` をインストールします（ホストには何もインストールしません）。[`misp/check.py`](misp/check.py) は、CSV、MISP イベント、CI ワークフローを読み取り専用でマウントして読み、何も書きません。

### 実行する

必要なのは Docker だけです。pymisp はコンテナにインストールされ、リポジトリには何も書き込みません。

```sh
detections/tests/misp/run.sh
```

どれかの断言が失敗するとスクリプトは 0 以外で終了します。内部にミラーがある場合は、`PYTHON_IMAGE` でイメージを、`PYMISP_VERSION` でピン留めしたライブラリを上書きできます。

## 貢献

新しい検知ルールは、発火を示すテストがあると、より強くなります。テストは自分の入力を決定的に合成し、エンジンのバージョンに依存しない性質は正確に断言し（それより柔らかい性質は、記録した基準値つきの下限とし）、攻撃の手引きに読める内容は避けてください。[`../../CONTRIBUTING.md`](../../CONTRIBUTING.md) と、[`../README.md`](../README.md) のルール索引を参照してください。

8 つのスイートはすべて、`detections/` に触れるすべての push または pull request で CI として動きます（[`../../.github/workflows/detections.yml`](../../.github/workflows/detections.yml) を参照）。指標テストは、それがピン留めした上流のソースファイル（`enrich/`、`selfupdate/`、`guard/`、`db/`、`cmd/artex/main.go`）が変わったときにも動きます。そのため、指標を落とし、ATT&CK レイヤーからずれ、文書化したバックエンドで変換が止まり、SigmaHQ の規約を壊し、ソースとの同期が外れ、MISP イベントを CSV からずらし、またはワークフローがまだ監視していない新しいソースをピン留めするルールの変更は、マージできる前にビルドを赤くします。
