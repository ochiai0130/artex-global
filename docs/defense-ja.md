# 自律型 AI 攻撃の防御と検知

> この文書は、ARTEX のような **自律型 AI ペネトレーションエージェント** がどのように動作するかを **防御側** が理解し、そうした攻撃を **検知し遮断する能力** を育てるためのものです。攻撃を実行するための手引きではありません。ここにある内容はすべて、自分が所有するシステム、またはテストについて明示的な書面による許可を得ているシステムを守るためにだけ使ってください。許可なく他人の情報通信網を探査または攻撃する行為は、それ自体が犯罪です（[トップレベル README のセキュリティと悪用に関する警告](../README.md#️-最初にお読みください--許可された利用と法的告知)を参照）。
>
> English: **[Defense and Detection Guide (defense-en.md)](defense-en.md)** · 한국어: **[방어·탐지 가이드 (defense-ko.md)](defense-ko.md)**.

自律型 AI の攻撃ツールは、かつては一人の運用者が手で進めていたペネトレーションテストを、LLM エージェントが **自ら目標を分解し、実際のツールを実行し、発見を蓄積する**、24 時間 365 日の自動プロセスに変えます。防御側が向き合う相手は、「熟練した攻撃者 1 人」から「疲れず、休まないエージェントの群れ」へと移ります。このガイドは、その変化が検知と対応に何を求めるかを示します。

---

## 1. 自律型 AI 攻撃は従来のスキャナとどう違うか

従来の脆弱性スキャナ（たとえば固定シグネチャのツール）は、あらかじめ定めた点検項目を順番に投げて終わります。ARTEX のような自律エージェントは、構造が違います。[システムアーキテクチャ](../README.md#システムアーキテクチャ)で説明しているとおり、次の要素が組み合わさって、**人の介入なしに多段の攻撃チェーンを完遂**します。

- **役割を分けたマルチエージェント。** 作業は、目標を分解する `goals`、次の方向を決める唯一の intent 生成者である `planner`、1 つの intent を実際のツールで実行する複数の `worker`、人が介在する点である `mainagent` に分かれます。planner が唯一の intent 生成者であり、worker たちがその intent を並列に実行します。
- **デュアルグラフに状態を蓄積する。** 「何があるか」（資産グラフ）と「どこまでテストしたか」（探索グラフ）を別々に築き、アンカーで結びます。その結果、攻撃は **段階的に深まり**、同じ資産を新しい角度から再訪し、先行する観察の上に次のステップを積みます。
- **イベント駆動の閉じたループ。** グラフが変わるたびに planner が起床して次の intent を割り当て、worker の書き込みが次のラウンドを引き起こします。この循環は、目標が証明されるまで止まりません。
- **直列の攻撃チェーンが安定して進む。** planner は依存の順序 — 「注入点を見つける → 認証情報を得る → 横展開 → 権限昇格」— を共有 todolist に一度だけ記録し、先行ステップが満たされてから次のステップにだけ intent を割り当てます。そのため、ステートレスなセッション環境でも、攻撃チェーンは脱線せず完遂します。
- **LLM が文脈ごとにペイロードを変える。** 実行するツールとペイロードはコードに固定された定数ではなく、LLM が状況を読んで生成する値なので、個々のリクエストの形は少しずつ変わります。

### 検知がより難しい理由

- **固定シグネチャは合いにくい。** ペイロードが文脈とともに変わるため、既知の悪意ある文字列を正確に一致させるルール（WAF シグネチャ）はすり抜けられやすいです。
- **ゆっくりと、人のように断続して進める。** エージェントはラウンドの間に休み、発見に応じて方向を変えるので、「短い時間に多数のリクエスト」だけを見るレートベースの検知では見逃すことがあります。
- **偵察と侵入が一つの流れで進む。** 人が偵察結果を見て、数日後に手動で攻撃していた隙間がなくなるため、最初の接触からデータの持ち出しまでの時間が劇的に縮みます。

### それでも検知できる理由 — 挙動は隠しにくい

静的な指紋（User-Agent、特定のペイロード文字列）は、運用者の裁量で変えられます。しかし **自律エージェントの挙動パターン** は攻撃の本質であり、変えにくいものです。下記 2 節から 4 節は、この挙動ベースの見方に重きを置きます。

- 単一の送信元（または少数の回転する送信元）が、**性質の異なるリクエストを連鎖させ**（偵察 → 列挙 → 認証試行 → エクスプロイト）、**一貫したセッションの中で** 進める。
- 人が疲労で手を止める時間帯でも、**途切れずに続く** 探索。
- 失敗した応答のヒントを使って **次のリクエストを体系的に変形する**、無作為ではない適応的な進行。

---

## 2. 防御側が観察できる指紋（IoC とシグネチャ）

指紋を二つの見方に分けます。**(a) 標的（被害側）の視点** — 自システムに向けられた ARTEX のトラフィックで見えるもの。**(b) 運用者・フォレンジックの視点** — ARTEX が実際に動作したホスト（または侵害された中継）で見えるもの。この二つを混ぜないことが重要です。標的側から見える静的な指紋は限られており、挙動の指紋が中核です。

### (a) 標的側の視点 — 自システムへ向けられたトラフィック

- **エンリッチ照会の User-Agent `artex-enrich/1.0`。** ARTEX が資産を自動でエンリッチするとき（DNS と HTTP の確認）、この User-Agent で標的に直接 `GET` を送ります（`enrich/enrich.go`）。この経路は LLM とは無関係に ARTEX 自身が生成し、**リダイレクトを追わず、keep-alive を無効にし、応答の先頭だけを読んで `<title>` を抜き出す** のが特徴です。デフォルトの同時実行数は 4 です。したがって、`artex-enrich/1.0` の UA で **短命な接続を開き、GET を 1 回だけ送り、タイトルだけ読んで切断する** 照会が、複数の資産に同時に届くと、ARTEX 系のエンリッチトラフィックを強く示唆します。ただし運用者は UA を変えられるので、**不在は安全を意味しません**。
- **実際の攻撃トラフィックは、ツールのデフォルト指紋に従う。** worker は、自分が実行する実際のツール（Bash 経由の外部ツール、および HTTP）を通してリクエストを送ります。このトラフィックを自前の記録プロキシへ通すため、ARTEX はサブプロセスの環境変数に `HTTP_PROXY` とプロキシ CA のパスを注入しますが、**攻撃トラフィックに ARTEX 固有の User-Agent を強制しません**。そのため標的が見る User-Agent とヘッダーは、**そのとき実行されたツールのデフォルト**（各種コマンドラインツールのデフォルト UA）です。運用者がカスタマイズしていなければ、ありふれた自動化ツールの指紋が残ります。カスタマイズしていれば、通常のブラウザに見せかけて偽装されていることもあります。したがって、**単一の UA 照合に頼らず、挙動ベースの検知と組み合わせてください**。
- **worker 内蔵の WebFetch ツールは `norma/0.4` の User-Agent を残す。** worker が Bash 経由で実行する外部ツール（上記）とは異なり、ARTEX が norma SDK（`github.com/Autumn-27/norma`）を通して直接行う HTTP 照会 — その WebFetch ツール — は、攻撃段階でその SDK のデフォルト User-Agent `norma/0.4` を載せます。平文 HTTP の回線上（または TLS を終端する地点で検査するとき）に観測でき、[Suricata ルール（sid 1000003）](../detections/suricata/README.md) がこの接頭辞（`norma/`）に一致します。ただしこの UA は **ARTEX 固有ではありません**。norma SDK の上に作られた他のツールも共有します。そのため `artex-enrich/1.0` と同様、証明ではなく補助的な手がかりであり、Bash で実行するツールは各自の UA を使うので、**不在は安全を意味しません**。
- **組み込みのレート制限はない。** ARTEX 自身には標的トラフィックのレート制限がなく、リクエストの速さは LLM が駆動する外部ツールが決めます。代わりに、デフォルトではタスクあたり 3 つの worker が並列に動くので、**複数の intent が 1 つの標的に対して同時に進む** ことがあります。つまり攻撃は「遅い単一セッション」としても「複数の角度が同時に進む」としても現れ得るため、固定の閾値一つでは捉えにくいです。
- **挙動シグネチャ（最も重要）。** 下記のパターンが **同時に現れる** ことが、自律エージェントを指します。
  - 一つの送信元（または少数の回転する送信元）から、**偵察 → ディレクトリ／エンドポイントの列挙 → パラメータ探査 → 認証／注入の試行が、短い間隔で連鎖する**。
  - 同じエンドポイントへの連続リクエストが、応答の **ステータスコードと長さに応じて体系的に変形する**（無作為なファジングではなく、適応的）。
  - 人の通常の稼働時間帯の外で、**長い時間途切れずに続く** セッション。
  - 失敗（401/403/429）の後も止まらず、**回避の変形** を試し続ける粘り。

### (b) 運用者・フォレンジックの視点 — ARTEX が動作したホスト

侵入調査の中で、中継・経由ホストに ARTEX がインストールされ実行された痕跡を探すときに使います。

- **デフォルトの待ち受けポート `:8787`。** これは ARTEX サーバーのデフォルト HTTP 待ち受けアドレスです（`cmd/artex/main.go`、`--addr` で変更可能）。内部ホストがこのポートで管理 UI（ダッシュボード、タスク、資産グラフ）を提供していれば、ARTEX インスタンスを疑う根拠になります。
- **記録型 MITM プロキシ `127.0.0.1:8788`。** worker の Bash/HTTP 実行をすべて傍受し、端から端まで記録するローカルプロキシのデフォルトアドレスです（`cmd/artex/main.go` の `--proxy` のデフォルト、ループバックのみ）。TLS を復号して記録するために自前の CA を生成するので（`mitmproxy-ca-cert.pem`）、手がかりは二つあります。ホストが **ARTEX によってインストールされた信頼 CA 証明書** を持っているか、そしてサブプロセスへ `HTTP_PROXY` とプロキシ CA パスの環境変数が注入された痕跡があるかです。この注入は ARTEX が起動するすべての worker ツールに入り、変数名はソースにハードコードされています（`agent/worker.go`）。そのため、**実行中のプロセスがプロキシ変数と mitmproxy の CA 信頼変数を同時に持っていること** は、ポート単体よりも特異的な手がかりです。[ホストトリアージツール](../detections/triage/README.md) は、`/proc`（フォレンジックイメージの場合は、取得した環境変数ダンプ）でこの組み合わせを確認します。
- **自己更新のコールバック `artex-selfupdate`。** 自己更新機能が GitHub のリリースを問い合わせるときに使う User-Agent です（`selfupdate/`）。送信（egress）ログに、この UA でコードリポジトリのホストへ出ていくリクエストがあれば、ARTEX バイナリの存在を示唆します。
- **PostgreSQL 上のデュアルグラフ。** `exploration_nodes`、`assets`、`companies`、`activity` のようなテーブルと、`agent_prompts` のシードを持つデータベースは、ARTEX のデータストアに特徴的です。
- **DB に置かれた正規表現の承認ルールと監査ログ。** ツール呼び出しを評価する intercept ルールは DB に保存され、優先度順に正規表現として評価されます（`intercept/`）。遮断された呼び出しは、監査ログ（`GET /api/audit`）に、`【ARTEX 平台管控·非目标防御】` で始まる統制の枠組み（「プラットフォーム統制 / 標的以外の防御」という枠組み）とともに記録されます。したがって、侵害されたホストの監査記録にこの文字列があれば、そこで ARTEX が動作したという結論を支えます。
- **破壊的コマンドのハンティング指標。** ARTEX 自身のガードが遮断対象として埋め込んでいるコマンドパターンは、実質的に、自律エージェントが **試みるかもしれない** コマンド群の鏡像です。ホストのコマンド監査では、次をハンティング指標として扱ってください。`rm -rf`、`mkfs`、`dd of=/dev/`、`shred`/`wipe`、SQL の `DROP DATABASE`/`DROP TABLE`/`TRUNCATE`、MongoDB の `drop`/`dropDatabase`、Redis の `FLUSHALL`/`FLUSHDB`、`curl`/`wget` の `-X DELETE`、そして `curl … | nc …` の形をしたデータ持ち出しパイプです。ただし、末尾のデータ持ち出しパイプは種類が違います。破壊コマンドではなく、データが外へ運ばれている（持ち出し）信号であり、上の破壊パターンとは異なり、ARTEX のガードはこのルール一つだけをデフォルトで無効にして埋め込んでいます（その curl/wget/nc のパイプパターンは、正当なペネトレーションテストのリバースシェルやデータ転送パイプで誤検知します）。デプロイ可能な破壊的コマンドのハンティングルール（[`destructive_command_hunting.yml`](../detections/sigma/destructive_command_hunting.yml)）は破壊に範囲を限定しており、これを含みません。持ち出しパイプは直ちに遮断せず、十分に調整した別の指標としてハントしてください。

> 要するに、**標的側の防御は挙動の指紋に置き、静的な UA（`artex-enrich/1.0`、`norma/0.4` など）は補助的な手がかりとしてだけ使う** ことです。運用者・フォレンジックの指紋（`:8787`、`127.0.0.1:8788`、`artex-selfupdate`、DB スキーマ、監査ログの枠組み）は、**侵害された経由ホストを調査するとき** に有効です。

### IP アドレスの遮断が弱い第一線の防御である理由

セキュリティインシデントが知られると、「攻撃者 IP の共有リストがある。ファイアウォールで遮断せよ」という投稿が、ソーシャルメディアやコミュニティフォーラムに出回りがちです。共有する側に善意があっても、**出所が不明な非公式 IP リストをそのまま遮断ルールへ入れることは推奨しません**。自律型 AI 攻撃に対しては、特にそうです。

- **出所を検証しにくい。** 個人が投稿した非公式リストでは、誰が何を根拠に集めたのか確認できず、インシデントと無関係な IP が混ざっていても取り除く手段がありません。
- **すぐに古くなる。** 自律エージェントは、VPN、クラウドインスタンス、乗っ取られた中継サーバーを通して、送信元 IP を絶えず回転させます（上記 (a) の「少数の回転する送信元」）。昨日観測した攻撃 IP は、今日にはすでに捨てられている可能性が高く、リストを遮断しても攻撃者は別の IP から戻ってきます。
- **誤遮断のリスクが高い。** リストに共有アドレス帯、CDN、正当なクラウド IP が混ざっていると、それらを遮断した瞬間に、健全な顧客トラフィックや内部サービスまで切り離します。自動遮断（下記 6 節）と組み合わさると、誤遮断の被害はさらに速く広がります。

これは、IP 遮断が無意味だという意味ではありません。**対応機関または信頼できる脅威インテリジェンスから公式の侵害指標（IoC）を受け取り、有効期間と誤検知の可能性を検討したうえで適用するとき** に意味を持ちます。しかし単一の IP を遮断するだけでは、回転する送信元を追いかける応急措置にすぎません。長く効くのは、変えにくい **挙動**（2〜4 節の挙動ベース検知）と、**攻撃面の縮小**（3 節と 5 節のハードニング。露出した資産の整理、パッチ、MFA）です。このガイドの前提 — 指紋は変えられても、挙動は隠しにくい — は、ここでも当てはまります。

---

## 3. 攻撃者が狙う侵入口とハードニング

自律エージェントは、人の攻撃者と同じ **弱点** を狙いますが、より速く、より執拗に繰り返します。以下は、防御側から見た優先度の高いハードニング箇所です。

### 3.1 外部に露出した攻撃面と既知（n-day）の脆弱性

自律エージェントが最初に、そして最も確実に狙う侵入口は、巧妙なゼロデイではなく、**露出したまま未パッチの、既知で既に公開された脆弱性** です。典型的な標的は、境界デバイス（VPN、ファイアウォール）、外部から到達できる管理・運用コンソール、アプリケーションサーバー、ミドルウェア、フレームワーク（広く悪用される WebLogic や Struts 級のソフトウェアなど）、そして **本サービスではなく、パートナー、採用、従業員向けに付随させた補助システム** です。自律エージェントは資産グラフから露出面を自動で列挙し、公開エクスプロイト（PoC）のある n-day を、**パッチが適用される前に**、数百の資産に対して一斉に狙います。露出した弱点を見つけては試すこのループの速さに、人の攻撃者との非対称が開きます。

- **攻撃面を減らす。** インターネットに露出した資産、管理コンソール、補助システムの棚卸しを継続的に維持し、外部にある必要のないものは内部ネットワーク、VPN、または許可リストの背後へ移します。
- **既知の脆弱性を速くパッチする。** 境界デバイス、Web サーバー、アプリケーションサーバー、ミドルウェアの公開済み脆弱性（n-day）は自律エージェントの最優先標的なので、公開 PoC のあるコンポーネントから、パッチ適用の間隔をできる限り短く保ちます。何を先にパッチするかを決めるには、実際の悪用が確認された脆弱性のカタログを優先順位の入力に使います。[CISA KEV（Known Exploited Vulnerabilities）カタログ](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) を、7.1 の国内勧告（KISA / 保護ナラ）と突き合わせてください。特定の CVE 番号を文書に固定するのではなく、このような生きた公式リストに依拠することで、自律エージェントが次に流通する n-day へ移っても遅れにくくなります。
- **露出を避けられないインターフェースを締める。** 露出を避けられない管理・運用インターフェースには、送信元制限（IP 許可リスト）、MFA、VPN を加え、未認証の列挙そのものを止めます。
- **補助システムを本サービスと同じ基準で管理する。** パートナー、採用、従業員向けの補助システムを、本サービスと同じパッチと監視の水準に保ちます。自律エージェントが通り抜ける侵入口は、本サービスではなく、こうした補助経路の一つであることが多いです。認証フローそのもののハードニングは、下記 3.2 に続きます。

検知については、特定の脆弱性の既知のパス（URL、パラメータ）を狙うリクエストが、外部から短い間隔の連鎖で届くことが、n-day スキャンの信号です。この信号は、2 節の挙動指紋および 4 節の同一送信元・多段相関ルールと組み合わせたときに最もよく現れます。

### 3.2 補助的な認証と本人確認のフロー

**本サービスとは別の経路で付いている認証と本人確認のフロー** — アドオンサービス、パートナーチャネル、採用チャネル — は、検証が緩いことが多く、迂回の標的になります。自律エージェントはこれらの経路を自動で列挙し、応答の違いを読んで迂回条件を体系的に探します。

- 本人確認と認証の手順を **本サービスと同じ強度に統一** し、迂回され得る補助経路をすべて監査します。
- 認証状態の遷移（未認証 → 認証済み、ユーザー → 特権）を **サーバーで再検証** し、クライアントが送る信頼の印（Cookie、ヘッダー、パラメータ）を盲信しません。
- 本人確認トークンとワンタイムコードの **寿命、再利用、推測しやすさ** を確認します。

### 3.3 API の認証と認可（IDOR と権限昇格）

- すべてのオブジェクトアクセスに **サーバー側の所有権／認可チェック** を強制します（識別子だけを変えると他人のリソースが開く IDOR を止めます）。
- 水平および垂直の権限昇格経路を自分で列挙します。自律エージェントは識別子を機械的に増減させて一括で試すため、**手動テスト 1 回では見逃した穴** をすぐに見つけます。

### 3.4 クレデンシャルスタッフィング

漏洩した ID／パスワードのリストを再送する攻撃は、自律エージェントによって **速度と分散** で増幅されます。

- ログインと本人確認のエンドポイントに、**適応型レート制限**（IP、アカウント、デバイス、挙動に基づく）を適用します。
- 機密操作に **多要素認証（MFA）** を強制します。スタッフィングで正しいパスワードに当たっても、第 2 要素がそれを止めます。
- **侵害された認証情報の検知**（既知の漏洩リストとの照合、異常なログイン場所や速度）で先回りして遮断します。
- ログインの失敗と成功の **分布の変化**（突然の低速・広範囲な試行）を警報します。

### 3.5 セッション、トークン、シークレットの管理

- セッショントークンの **範囲、寿命、更新** を最小化し、機密な遷移のたびに再認証します。
- API キーや内部トークンを、応答、ログ、エラーメッセージに **出さない**（自律エージェントはエラー応答から手がかりを積極的に集めます）。

---

## 4. 検知ルールとログのパターン（実務）

特定の製品に依存しない **疑似ルール** として書いてあります。自組織の WAF/IPS/SIEM の構文に置き換えてください。以下のルールのうち静的な指紋に基づくものは、すぐにデプロイできる [Sigma ルール（`detections/sigma/`）](../detections/README.md) として同梱しています。中核となる挙動と相関の検知（4.1 と 4.2）も単一のルールには還元できませんが、ARTEX のソースに根拠を持つ挙動指標は、デプロイ可能な [Sigma 相関ルール（`detections/sigma/correlation/`）](../detections/README.md) として同梱しています。すなわち、エンリッチの速度、エンリッチのファンアウト、ガード遮断のバースト、および同一ホストでのガードマーカーと破壊的コマンドの同時出現です。純粋な Web の多段相関（列挙 → 探査 → 認証）は、その多段パターンが単一の ARTEX 固有 User-Agent に還元されないため、環境ごとのベースルールが引き続き必要です。すぐ調整できる汎用の Sigma ベーステンプレートを下記 4.2 に置いてあるので、自組織の SIEM とベースラインに合わせて出発点にしてください。回線上で観測できる ARTEX の User-Agent は二つあり、エンリッチプローバの `artex-enrich/1.0`（sid 1000001–1000002）と、norma SDK の WebFetch ツールが攻撃段階で送る `norma/0.4`（sid 1000003）です。どちらも [Suricata ルール（`detections/suricata/`）](../detections/suricata/README.md) として同梱しています。

### 4.1 WAF/IPS（挙動ベース）

- 単一の送信元からの **性質の異なるリクエスト**（静的リソースのリクエストの割合が低く、列挙、パラメータ探査、認証試行の割合が高い）が **一つのセッションとして続く** とき、スコアを上げます。
- ステータスコードとボディ長に **応じて変形する連続リクエスト**（エントロピーは高いが、無作為ではない適応的なパターン）を重み付けします。
- `artex-enrich/1.0` のような既知の自動化 UA は **直ちに高リスク** としてタグ付けしますが、UA の不在を安全と読まないでください。

### 4.2 SIEM 相関ルール

- **同一送信元の多段相関:** 同じ IP/ASN/セッションから、(a) ディレクトリ／エンドポイントの列挙、(b) パラメータ探査、(c) 認証／注入の試行が **短い時間窓の中ですべて観測された** とき、「自律攻撃の疑い」の警報を上げます。
- **時間帯の異常:** サービスの通常のトラフィック分布の外で、**長い時間途切れずに続く** 単一セッション。
- **失敗後の持続:** 403/429 を受けても止まらず、**回避の変形** を続ける送信元。

上の **同一送信元の多段相関** について、デプロイ可能なベーステンプレートを以下に置きます。攻撃トラフィックは ARTEX 固有の User-Agent を持たないため、このテンプレートは `detections/sigma/` 配下の ARTEX のソースに根拠を持つルールとは異なり、**汎用の挙動ルール** です。見るのは挙動だけ — 「一つの送信元が、短い時間窓の中で列挙、探査、認証を実行する」— であり、いずれの攻撃ツールの指紋でもありません。自己完結しています。3 つのサブルールに加え、一つのクライアントが時間窓の中で 3 つすべてを満たしたときだけ発火する temporal 相関です。

```yaml
# ── 汎用の挙動テンプレート（ARTEX 固有のシグネチャではない）──
# 防御ガイド 4.2 節の、同一送信元による多段 Web パターン
# （列挙 → 探査 → 認証）。ARTEX の攻撃トラフィックには ARTEX の
# 指紋がないため、detections/sigma/ 配下のルールとは異なり、これは
# ARTEX のソースに根拠を置かない汎用の挙動の出発点である。フィールド名
# （SigmaHQ の webserver 分類）と閾値・時間窓は、自組織のログと
# ベースラインに合わせて必ず調整すること。自己完結: 3 つのサブルールに加え、
# 1 つのクライアントが時間窓内で 3 つすべてを満たしたときだけ発火する
# temporal 相関。
title: Web Endpoint Enumeration Burst From One Source
id: f03c360c-dc33-4a8a-afa8-821b1ff5c4e3
status: experimental
description: |
    Stage 1 of the same-source multi-stage pattern in the ARTEX defense guide section 4.2: a
    burst of endpoint or directory enumeration from a single client, seen as a high rate of 404
    and 400 responses in a short window. This is generic behaviour, not an ARTEX-specific
    signature; tune the count and window to your own baseline. On its own this leg is low signal
    and earns weight only inside the correlation below.
references:
    - https://github.com/jiwoochris/artex-global/blob/main/docs/defense-ko.md
    - https://github.com/jiwoochris/artex-global/blob/main/docs/defense-en.md
author: artex-global defense guide (generic template)
date: 2026-10-07
tags:
    - attack.reconnaissance
    - attack.t1595
logsource:
    category: webserver
detection:
    enum_misses:
        sc-status:
            - 404
            - 400
    condition: enum_misses
falsepositives:
    - Broken links, authorised vulnerability scanners, or misconfigured clients that generate
      many 404 responses.
level: low
---
title: Web Parameter Or Path Injection Probe From One Source
id: f9296e55-6b7a-4030-b5f7-5f7b146233be
status: experimental
description: |
    Stage 2 of the same-source multi-stage pattern: parameter or path probing, matched here as
    query strings carrying common injection or traversal markers. This leg is unavoidably
    signature-like and noisy on its own, so it is scored low and earns weight only inside the
    correlation below. Extend the marker list to your own probe corpus and WAF categories; it is
    a coarse proxy for the broader "adapts requests to responses" behaviour the guide describes.
references:
    - https://github.com/jiwoochris/artex-global/blob/main/docs/defense-ko.md
    - https://github.com/jiwoochris/artex-global/blob/main/docs/defense-en.md
author: artex-global defense guide (generic template)
date: 2026-10-07
tags:
    - attack.initial-access
    - attack.t1190
logsource:
    category: webserver
detection:
    probe_markers:
        cs-uri-query|contains:
            - '../'
            - "' or "
            - ' union select '
            - '<script'
            - '; drop '
    condition: probe_markers
falsepositives:
    - Legitimate request payloads that resemble probe markers; tune the marker list to your
      application.
level: low
---
title: Authentication Or Identity-Verification Attempt From One Source
id: 2614cacb-7455-46ad-9af2-7b9633f12d7b
status: experimental
description: |
    Stage 3 of the same-source multi-stage pattern: requests to login, authentication, or
    identity-verification endpoints, or 401 and 403 responses. Map the paths and your own
    authentication-event fields to your application; auxiliary, affiliate, and broker channels
    often expose weaker identity-verification endpoints than the main service and belong here
    too. This leg is broad by design and is only meaningful inside the correlation below.
references:
    - https://github.com/jiwoochris/artex-global/blob/main/docs/defense-ko.md
    - https://github.com/jiwoochris/artex-global/blob/main/docs/defense-en.md
author: artex-global defense guide (generic template)
date: 2026-10-07
tags:
    - attack.credential-access
    - attack.t1110
logsource:
    category: webserver
detection:
    auth_path:
        cs-uri-stem|contains:
            - '/login'
            - '/auth'
            - '/verify'
            - '/otp'
    auth_deny:
        sc-status:
            - 401
            - 403
    condition: auth_path or auth_deny
falsepositives:
    - Ordinary users signing in; this leg is broad and only meaningful inside the correlation.
level: low
---
title: Same-Source Multi-Stage Web Attack (Enumeration, Probe, Auth)
id: 9b7c7b86-702f-42b8-be99-3e60a188ec5b
status: experimental
description: |
    The behaviour-based core of ARTEX defense guide section 4.2 as a deployable template: one
    client runs endpoint enumeration, parameter or path probing, and an authentication or
    identity-verification attempt within the same short window. This is the pattern an autonomous
    agent drives at machine speed and keeps driving past 403 and 429 responses. It is UA-free and
    carries no ARTEX fingerprint, so it is a GENERIC behavioural rule, not one of the
    ARTEX-source-grounded rules under detections/sigma/. Normalise the client field (c-ip, or a
    session identifier if you have one) and tune the window to your baseline. If three legs are
    too strict and miss cases, relax to any two of the three.
references:
    - https://github.com/jiwoochris/artex-global/blob/main/docs/defense-ko.md
    - https://github.com/jiwoochris/artex-global/blob/main/docs/defense-en.md
author: artex-global defense guide (generic template)
date: 2026-10-07
tags:
    - attack.initial-access
    - attack.t1190
correlation:
    type: temporal
    rules:
        - f03c360c-dc33-4a8a-afa8-821b1ff5c4e3
        - f9296e55-6b7a-4030-b5f7-5f7b146233be
        - 2614cacb-7455-46ad-9af2-7b9633f12d7b
    group-by:
        - c-ip
    timespan: 10m
falsepositives:
    - An authorised vulnerability scan or QA run from a single source; allow-list its address.
level: high
```

このテンプレートを使うときの注意:

- このブロックは、検知パックが使うのと同じツールで検証済みです。`sigma check` を SigmaHQ の規約一式で実行するとエラー 0、issue 0 で通過し、`sigma convert -t splunk` はクエリを生成します（3 つのサブルールを 10 分窓で `c-ip` ごとにまとめ、3 つすべてが揃ったときに発火）。ARTEX のソースに根拠を置けないため、テスト対象の `detections/` ルールツリーには入れていません。それは、そのツリーが「推定ではなく、このリポジトリのソースで確認されたものだけを載せる」という約束を守るためです。これらの事実は一度きりの計測ではありません。[`detections/tests/check-defense-template.sh`](../detections/tests/check-defense-template.sh) が、このガイドが変わるたびに CI で再実行します（`sigma check` の通過、splunk/eql/loki への変換、lucene/kusto の拒否、そして韓国語版と英語版がバイト単位で同一のルール本体を持つこと）。そのため、ここの記述が上流の `sigma-cli` の変更で静かに古くなることはありません。
- このテンプレートは相関ルールなので、`sigma convert` がテンプレート全体を出すか、3 つのサブルールだけを出すかは、バックエンドが Sigma の相関変換をサポートするかに依存します。同じ固定版の `sigma-cli` 3.1.0 で実測すると、テンプレート全体は Splunk（`-t splunk`）、Elasticsearch EQL（`-t eql`）、Grafana Loki（`-t loki`）で変換されます。Microsoft の `kusto` バックエンド（Sentinel と Defender）と Elasticsearch Lucene（`-t lucene`）では相関は変換されません（`Backend does not support correlation rules`）。そのため 3 つのサブルールだけを変換し、10 分・同一 `c-ip` の相関は製品の中でネイティブに表現します（たとえば Sentinel のスケジュール分析の `summarize ... by bin(TimeGenerated, 10m), <client>`）。これは検知パックが文書化しているのと同じ移植性であり、実測したサポートの表は [検知ルール一覧](../detections/README.md) にあります。
- 段階 2（探査）は注入／トラバーサルのマーカー一覧に依っており、単独では粗く、ノイズの多い信号です。そのため 3 つのサブルールは `low` とし、3 つすべてが一つの送信元から揃う相関だけが高い警報を上げます。
- クライアントは `c-ip` でまとめます。プロキシや CDN の背後では、`X-Forwarded-For` から復元した実際のクライアントアドレス、またはセッション識別子に切り替えます。3 段階すべてを要求するのが厳しすぎて見逃す場合は、3 つのうち任意の 2 つに緩めてください。

### 4.3 認証ログ

- アカウント／IP ごとの **ログイン失敗率の急な変化**、広い範囲のアカウントに散らばる **低速の分散試行**（スタッフィングに特徴的）、**失敗から成功への遷移速度の異常**。
- 本人確認とワンタイムコードのエンドポイントへの **列挙型のアクセス**。

### 4.4 送信（egress）とフォレンジック

- 内部ホストから、`artex-selfupdate` の UA でコードリポジトリのホストへ出ていくリクエスト。
- 内部で `:8787`（管理 UI）または `127.0.0.1:8788`（記録プロキシ）にバインドされたプロセス。
- `artex-enrich/1.0` の UA で、短い時間に多数の外部資産を照会する DNS/HTTP のエンリッチパターン。

---

## 5. ハードニングチェックリスト

防御するチームがすぐ確認できるよう、要約します。

- [ ] ログイン、本人確認、機密 API に **適応型レート制限**（IP、アカウント、デバイス、挙動）を適用している。
- [ ] 機密操作に **MFA** を強制している。
- [ ] すべてのオブジェクトアクセスに **サーバー側の所有権／認可チェック** がある（IDOR を止める）。
- [ ] **認証状態の遷移をサーバーで再検証** し、クライアントの信頼の印を盲信しない。
- [ ] **補助／パートナー／採用チャネル** の本人確認の強度を、本サービスと統一した。
- [ ] 漏洩した認証情報について **侵害された認証情報の検知／照合** を運用している。
- [ ] WAF を **挙動ベースのモード** で動かし、固定シグネチャだけに頼らない。
- [ ] **出回っている非公式 IP 遮断リストをそのまま適用せず**、信頼できる出典から公式の侵害指標（IoC）を受け取り、有効期間と誤遮断リスクを検討したうえで適用する。
- [ ] SIEM に **同一送信元の多段相関ルール** を追加した。
- [ ] **認証、アクセス、送信のログを十分な期間保持する**（自律攻撃は速いので、事後に追跡する材料が重要）。
- [ ] **インターネットに露出した資産、管理コンソール、補助システムの棚卸しを維持して攻撃面を減らし**、境界デバイス、アプリケーションサーバー、ミドルウェアの **既知（n-day）の脆弱性を速くパッチする**。
- [ ] ネットワークの **セグメンテーション** で、横展開と権限昇格の波及範囲を狭めた。
- [ ] シークレット（キー、トークン）を応答、ログ、エラーメッセージに **出さない**。
- [ ] **自動遮断／隔離** の対応を用意した（人の承認だけを待っていては、自律攻撃の速度に追いつけない）。

---

## 6. インシデント対応の要約

自律型 AI 攻撃の決定的な特徴は **速度** です。人が 1 年かける侵入と持ち出しを、エージェントははるかに短い時間で完遂できます。対応は、この速度を前提に設計します。

- **まず自動遮断。** 疑わしい送信元の隔離、セッションの無効化、レートの急激な制限のような措置を、人の承認の前に **自動で発火** できるよう用意します。すべてを人の承認ループに縛ると、攻撃の速度に追いつけません。
- **どのログを保持するかを事前に決める。** 認証ログ、アクセスログ（取得できる範囲のリクエストボディを含む）、送信ログ、DNS クエリ。自律攻撃は痕跡を速く積むので、あとから攻撃チェーンを再構成するにはこの材料が必要です。
- **侵害の範囲を資産ごとに追う。** 攻撃は資産グラフに沿って広がるので、最初の侵入口の資産から横展開と権限昇格までの **経路全体** を再構成し、再侵入を防がなければなりません。

### 6.1 疑わしいホストまたはトラフィックのトリアージ手順

ARTEX の関与が疑われるとき、最初に何を確認するかを順に示します。2 節が指紋を二つの視点に分けたように、トリアージも **(a) 自サービスが標的になったか** と **(b) あるホストで ARTEX が動作したか** に分かれます。どの段階でも、単一のヒットだけで結論しないでください。複数の指標と挙動の信号が一緒に現れるかで判断します。静的指標がすべて不在でも、挙動の信号が出ていれば調査を続けます。

**(a) 標的側 — 自サービスが ARTEX の標的になったか**

1. アクセス／認証ログで、エンリッチの User-Agent `artex-enrich/1.0` を検索します。リダイレクトを追わない単発の `GET` が、短い間隔で複数の資産に同時に届いているかを確認します（2 節 (a)）。運用者は User-Agent を変えられるので、一致がなくても次の段階へ進みます。
2. 同じ送信元（または少数の回転する送信元）からの **多段の連鎖** を探します。偵察からエンドポイント列挙、パラメータ探査、認証／注入の試行へ短い間隔でつながり、応答コードと長さに適応し、401/403/429 の後も回避の変形を止めないものです。この挙動の信号は、静的な User-Agent より長く残ります（2 節 (a) の挙動シグネチャ）。
3. SIEM を運用しているなら、この挙動を [Sigma 相関ルール](../detections/README.md)（エンリッチの速度、ファンアウト、ガード遮断のバースト、ガードマーカーと破壊的コマンドの同時出現）で捉え、一致した送信元を、上の「まず自動遮断」の原則に従って隔離とセッション無効化へエスカレーションします。

**(b) ホストフォレンジック — あるホストで ARTEX が動作したか**

疑わしいホストで次を確認します。各指標の根拠は 2 節 (b) と、機械可読の [指標リスト](../detections/indicators/artex_indicators.csv) にあります。下記 5 つの読み取り専用チェックのうち最初の 4 つ（待ち受けポート、送信ログ、監査ログ、状態と記録のストア）は、[ホストトリアージスクリプト](../detections/triage/README.md)（`detections/triage/artex_host_triage.py`）が一度に実行します。5 つ目のコマンド監査は、スクリプトが自動化しません。破壊的コマンドは ARTEX の指紋ではなく、正当な管理者も実行するハンティングの手がかりなので、自動指標から意図的に外してあります。この項目は、ホストのコマンド履歴と手で照合してください。SIEM がなくシェルアクセスがあるときは、まずスクリプトを実行し、各ヒットは下記のとおり手がかりとして扱います。

1. **待ち受けポート。** デフォルトのサーバーポート `:8787` と、ループバックのトラフィック記録プロキシ `127.0.0.1:8788` が開いているかを、ホスト自身で確認します。
   ```sh
   ss -ltnp | grep -E ':8787|:8788'   # ss が使えない場合は netstat -ltnp を使う
   ```
   この二つのポートは `--addr` と `--proxy` のフラグで変更できるので、この検索が空でも、開いているポート全体と、内部の管理 UI が起動しているかをあわせて確認します。
2. **送信ログ。** 自己更新の User-Agent `artex-selfupdate` で、コードリポジトリのホスト（GitHub のリリース）へ出ていったリクエストが送信ログにあるかを確認します（`selfupdate/`）。これは、ホスト上で ARTEX バイナリが動作したことを示唆します。
3. **監査ログ。** ガードの統制マーカー `【ARTEX 平台管控·非目标防御】` が監査記録に現れていれば、ARTEX が実行されたという判断を支えます（`guard/guard.go`）。遮断されたツール呼び出しのたびに、この枠組みで書かれます。
4. **状態と記録のストア。** ARTEX は探索グラフを PostgreSQL に保持し（`exploration_nodes`、`assets`、`companies`、`activity` のテーブルと `agent_prompts` のシード）、実行ファイルの隣のデータディレクトリ（`cmd/artex/main.go` の `--data` のデフォルト）に状態と記録を残します。そのディレクトリの直下には、タスクごとの `tasks/` と会話の `transcripts/` サブディレクトリがあり、記録プロキシの成果物は一段深い `traffic/` サブディレクトリにあります（`server/manager.go` が、データディレクトリ配下の `traffic/` をレコーダのストアとして開きます）。したがって信頼 CA 証明書は `traffic/_ca/mitmproxy-ca-cert.pem`、トラフィック索引は `traffic/_index/index.sqlite`、記録されたリクエスト／応答ボディは `traffic/_blobs/` にあります。これらが `tasks/` および `transcripts/` と一緒に現れると、状況証拠は強くなります。
5. **コマンド監査。** 破壊的コマンドのハンティング指標（2 節 (b) の末尾。`rm -rf`、`DROP DATABASE`、`FLUSHALL`、持ち出しパイプなど）を、ホストのコマンド履歴と照合します。正当な管理者も同じコマンドを使うので、手がかりとしてだけ扱います。

静的指標（ポート、User-Agent、マーカー）は、運用者が変更または削除できます。したがって **不在は安全を意味せず**、自律型 AI 攻撃のトリアージの要は、(a) の挙動信号と (b) のホスト痕跡を集めて一緒に判断することです。

---

## 7. 韓国の公式チャネル: 侵害指標、勧告、報告義務

韓国の防御チームは、侵害指標とセキュリティ勧告を公式チャネルから受け取り、インシデントが起きたときは法律が定める報告義務を果たすべきです。出回る非公式リストではなく、下記の公式出典を第一の参照にしてください。

### 7.1 侵害指標と勧告の入手先

- **KISA（韓国インターネット振興院、Korea Internet & Security Agency）の [保護ナラ（Boho Nara）/ KrCERT/CC](https://www.boho.or.kr)** は、セキュリティ勧告、脆弱性通知、インシデント対応情報を公開し、C-TAS（サイバー脅威分析・共有システム、Cyber Threat Analysis and Sharing。公開ポータルとは異なり、加入機関の間で共有され、別途申請が必要です）を通して、組織間で脅威インテリジェンスを共有します。
- **[金融保安院（FSI、Financial Security Institute）](https://www.fsec.or.kr)** は、金融分野の侵入・脅威情報を分野横断で共有します（金融分野の ISAC）。金融分野であれば、このチャネルも見てください。
- **[個人情報保護委員会（PIPC、Personal Information Protection Commission）](https://www.pipc.go.kr)** は、漏えい通知の基準と保護措置のガイダンスを公開します。

これらのチャネルが、5 節のハードニングチェックリストでいう「信頼できる出典から公式の侵害指標を受け取る」の中身です。公式の IoC であっても、有効期間と誤遮断リスクを検討したうえで適用します。これは 2 節「IP アドレスの遮断が弱い第一線の防御である理由」で説明したのと同じ原則です。

### 7.2 韓国法上の報告義務

自律攻撃は速く広がるので、法定の報告手順を 6 節のインシデント対応手順にあらかじめ組み込んでください。以下は要約です。正確な適用範囲、期限、条件は、各当局の現行の規定で確認してください。

- **個人情報の漏えい:** 個人情報保護法（Personal Information Protection Act）第34条に基づき、漏えいを知ったときから 72 時間以内に PIPC または KISA へ報告し、影響を受けた情報主体へ通知します（報告条件には、情報主体 1,000 人以上への漏えい、機微情報または固有識別情報の漏えい、不法な外部アクセスによる漏えいが含まれます）。
- **セキュリティインシデント:** 情報通信網法（Network Act。「情報通信網利用促進及び情報保護等に関する法律」）に基づき、情報通信サービス提供者は、インシデントを知ったときから 24 時間以内に科学技術情報通信部（Ministry of Science and ICT）と KISA（KrCERT/CC）へ報告します。
- **金融会社:** 金融分野の監督規定に基づき、金融監督院（Financial Supervisory Service）や FSI などの機関へ追加で報告しなければならない場合があるので、それらの規定も確認してください。

実際の届出は、セキュリティインシデントなら [保護ナラ](https://www.boho.or.kr) または市外局番なしの 118（KISA のサイバー相談センター）へ、個人情報の漏えいなら PIPC の [個人情報ポータル](https://www.privacy.go.kr) から行います。期限が短いので、担当者と連絡経路を 6 節のインシデント対応手順にあらかじめ記録しておきます。

---

## 参考

- 上流プロジェクト: [Autumn-27/ARTEX](https://github.com/Autumn-27/ARTEX)（AGPL-3.0 · 現在アクセス不可、2026-10 確認）。この文書は、そのグローバル版リポジトリの防御資料です。
- トップレベルの [README のセキュリティと悪用に関する警告、利用範囲、法的告知](../README.md#️-最初にお読みください--許可された利用と法的告知)。
- この版は韓国向けにローカライズされています。個人情報が関わる場合は、韓国法（情報通信網法と個人情報保護法）が適用されます。それとは別に、無許可のテストはほとんどの法域で犯罪です。必ず書面による許可と、合意された範囲を先に確保してください。
- 一般的な Web セキュリティのハードニングに関する標準的な参考資料: [OWASP Top 10](https://owasp.org/www-project-top-ten/)、[OWASP ASVS（Application Security Verification Standard）](https://owasp.org/www-project-application-security-verification-standard/)、[OWASP API Security Top 10](https://api-security.owasp.org/)。

> このガイドは、防御と検知の能力を支えるために継続的に拡充されます。追加の検知ルールやハードニング項目の提案は、リポジトリの issue として歓迎します。
