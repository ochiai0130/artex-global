# 上流の同期と翻訳ドリフト防止（メンテナー向け）

日本語 · [English](MAINTAINING.en.md) · [한국어](MAINTAINING.ko.md)

この文書は、**メンテナー**が上流リポジトリ [Autumn-27/ARTEX](https://github.com/Autumn-27/ARTEX)（現在アクセス不可、2026-10 確認）の
変更に追従しながら、多言語ローカライズを保つ手順をまとめたものです。貢献の範囲・法的責任・
ローカライズ方針は [CONTRIBUTING.md](CONTRIBUTING.md) に、ユーザー向けの案内は [README.md](README.md) に
あるので、この文書はその方針を **実際にどう適用するか** だけに集中します。

ローカライズの中心にある目標は、一文で言い表せます。原本の **判断性能はそのまま保ち、ユーザーに
見える出力だけを設定した言語に変える** ことです。上流が更新されるたびにこの境界はぼやけやすいので、
以下の手順と検査で翻訳ドリフトを防ぎます。

---

## 1. ローカライズの構造をひと目で見る

このリポジトリは、上流の ARTEX を **フォーク** し、その履歴の上に韓国語ローカライズのコミットを積み上げた構造です。
上流 `main` のすべてのコミットがこのリポジトリの履歴に含まれており、その上にローカライズのコミットが
加わっています。したがって上流の変更を取り込む作業は、「上流 `main` との差分を確認し、残すものと
翻訳するものを切り分けて反映する作業」になります。

成果物は、次の三つに分かれます。

- **原文のまま置く資産**（下記 2 節）。翻訳すると、性能が損なわれるか、上流との差分が取りにくくなります。
- **コードに固定した出力言語の強制**。`agent/prompt.go` の `langDirective()` が、各ロールの system
  prompt の末尾に「ユーザー向けの出力は韓国語で書け」という指示を付け加えます。
- **韓国語に翻訳するユーザー向け文字列**。UI は `web/messages/ko.json` に、サーバーの
  ユーザー向け応答文は各 Go ファイルの名前付き定数に置きます。

---

## 2. 原文のまま残す資産（翻訳禁止）

次の資産は翻訳せず、原文（中国語または英語）を維持します。上流の変更がこれらの資産に触れたら、
**翻訳せずそのまま反映**します。

- **エージェント内部の推論プロンプト（ブレイン本体）。** `agent/promptcatalog.go` と DB シード
  `agent_prompts` にある行動指示の本体です。原文（中国語）でベンチマークされた挙動を維持する必要がある
  ので、翻訳すると判断にドリフトが生じます。
- **表示とエージェント入力を兼ねる文字列。** アクティビティタイムラインに表示されると同時に、planner・reporter の
  入力コンテキストへフィードバックされる一部の文言（タスク中断理由、インターセプトの遮断メッセージ、トラフィック証拠ヘルパー
  など）は、一つのレコードが二つの用途を兼ねるため原文を保ちます。判定の根拠は
  `work/DECISIONS-FOR-JIWOO.md` の「두뇌 경계 기록」（ブレイン境界の記録）に、件ごとに書かれています。
- **原本の中国語ドキュメントと文字列。** ドキュメントは `README.zh.md` に、UI 文字列は `web/messages/zh.json` に
  原文をそのまま残し、上流の変更と突き合わせやすくします。韓国語訳は `web/messages/ko.json`
  にだけ記入します。
- **コマンド・ペイロード・コード・URL・識別子・ログの原文。** 分析に必要な原本なので翻訳しません。
  Go のコードコメントも優先度は最も低く、上流との突き合わせが終わるまでは原文のままにしておきます。

---

## 3. 上流を追跡するベース

上流のリモートは、次のように設定されている必要があります。なければ追加します。

```bash
git remote add upstream https://github.com/Autumn-27/ARTEX
git remote -v   # upstream が表示されるか確認
```

現在、ローカライズの反映が終わっている上流のベースコミットは、次のとおりです。

- **ベース = `d003372`**（上流 `main`、2026-10-03、PR #189 `fix/sse-same-origin` のマージ）。

この値は、「このコミットまでの上流の変更は、すべてこのリポジトリに取り込まれている」という意味です。上流の変更を新しく
反映するたびに、7 節の方法でこのベースを更新します。

---

## 4. 上流の変更を取り込む手順

### 4.1 上流を取得して差分を確認する

```bash
git fetch upstream
git rev-list --count d003372..upstream/main        # 未反映の上流コミット数
git log --oneline d003372..upstream/main           # 未反映のコミット一覧
```

`git fetch` は上流のリモート追跡ブランチだけを更新するので、作業ツリーと `HEAD` には影響しません。
未反映のコミットが 0 なら上流と同期しているので、これ以上することはありません。

### 4.2 変更されたファイルを分類する

未反映のコミットがどのファイルに触れたかを見て、2 節の保存資産と翻訳対象に分けます。

```bash
git log --name-status --oneline d003372..upstream/main
```

分類の基準は、次のとおりです。

- `agent/promptcatalog.go`・`agent_prompts` シード、および 2 節の表示と入力を兼ねる文字列が変わっていたら
  → **翻訳せずそのまま反映**します。
- Go バックエンドのロジック（`db/`・`llmrec/`・`server/` など）が変わっていたら → ロジックはそのまま反映しつつ、
  **新しく増えたユーザー向け応答文**（`writeErr` など）がないか確認し、韓国語の定数に翻訳します。
- UI（`web/src/**`）が変わって **新しい画面文字列** が増えていたら → ハードコードせず、
  `web/messages/zh.json`（原文）と `web/messages/ko.json`（翻訳）に同じキーで追加します。
- **検知ルールが固定している上流の指標**（`enrich/enrich.go` のプローバ User-Agent、`selfupdate/` の
  自己更新 User-Agent、`guard/guard.go` の監査マーカー、`db/db.go` の破壊的コマンド deny リスト、
  `cmd/artex/main.go` のデフォルトのリッスンポートと記録プロキシのポート）が変わっていたら → `detections/` の
  Sigma・Suricata ルールと ATT&CK レイヤー、そして `detections/indicators/artex_indicators.csv` の
  値も新しい値に合わせます。これらの指標は翻訳対象ではなく **検知の根拠** なので、上流が値を変えると
  ルールは気づかないうちに古くなります。5.4 の指標一致テストが、このずれを自動で捉えます。

### 4.3 反映する

機能単位でマージするか cherry-pick したあと、4.2 で切り分けた新しい文字列を韓国語に翻訳します。
マージの過程で `ko.json`・`zh.json` のキーがずれたり、ユーザー向けの箇所に原文が漏れ込んだりし
やすいので、反映の直後には必ず 5 節の検査を実行します。

> **例（2026-10-05 時点の未反映コミット）。** `git fetch upstream` の結果、上流 `main` は
> `b55ceb1` まで進んでおり、ベース `d003372` に対してコミット 2 件（`86729b6` のモデルフォールバック承認トークン計量
> 機能 + マージコミット `b55ceb1`）が未反映です。このコミットは `db/llm_usage.go`・`llmrec/llmrec.go`・
> `server/intercept.go`・`server/server.go` といった Go のロジックと、`web/src/app/(main)/system/intercept/page.tsx`・
> `web/src/lib/api.ts`・`web/src/lib/mock/handler.ts`・`web/src/lib/types.ts` に触れます。
> したがってメンテナーは Go のロジックをそのまま反映し、intercept の設定ページに新しく増えた画面
> 文字列だけを `ko.json`・`zh.json` のキーとして取り出して翻訳すれば足ります。（この 2 件のコミットは、この文書を書いた時点では
> まだ反映していないので、ベースは `d003372` のままにしておきます。）

---

## 5. 翻訳の対称性とドリフトの検査

上流の反映や翻訳作業のあと、次の三つを確認します。

### 5.1 ko と zh のメッセージ対称性とユーザー向け CJK

`ko.json` と `zh.json` のキーは完全に一致し、`ko.json` の値に中国語の漢字が残っていてはいけません。
次のスクリプトが、三つの数値を出力します。

```bash
python3 - <<'PY'
import json, re
ko = json.load(open('web/messages/ko.json'))
zh = json.load(open('web/messages/zh.json'))
def flatten(d, p=''):
    out = {}
    if isinstance(d, dict):
        for k, v in d.items(): out.update(flatten(v, p + '/' + k))
    elif isinstance(d, list):
        for i, v in enumerate(d): out.update(flatten(v, p + '/' + str(i)))
    else: out[p] = d
    return out
fk, fz = flatten(ko), flatten(zh)
han = re.compile(r'[㐀-鿿]')
print('ko leaf keys :', len(fk))
print('zh leaf keys :', len(fz))
print('key symdiff  :', len(set(fk) ^ set(fz)))        # 0 であること
print('ko vals w/CJK:', sum(1 for v in fk.values() if isinstance(v, str) and han.search(v)))  # 0 であること
PY
```

基準値（2026-10-05）: `ko leaf keys = 2950`、`zh leaf keys = 2950`、`key symdiff = 0`、
`ko vals w/CJK = 0`。キー数は上流の反映で増えることがありますが、ko と zh は常に同じでなければならず、
`key symdiff` と `ko vals w/CJK` は常に 0 でなければなりません。

### 5.2 ブレイン資産が原文のままであることを確認する

ブレイン本体は中国語の原文を維持するので、次の検査で **漢字の行数が 0 まで落ちたら**、むしろ
ブレインが誤って翻訳され、汚染されたというサインです。

```bash
python3 -c "import re; han=re.compile(r'[㐀-鿿]'); t=open('agent/promptcatalog.go').read(); print('promptcatalog.go CJK lines =', sum(1 for l in t.splitlines() if han.search(l)))"
```

基準値（2026-10-05）: `promptcatalog.go CJK lines = 70`。この数が大きく減ったら、ブレイン本体が翻訳されていないか
確認します。

### 5.3 ビルド成果物に原文が漏れていないか

UI を静的にエクスポートしたあと、プリレンダーされた HTML に中国語が見えれば、翻訳漏れです。

```bash
cd web && npm ci && NEXT_EXPORT=1 npm run build   # out/ を生成
# out/**/*.html の表示されているテキストに、中国語の漢字が 0 件であることを確認
```

### 5.4 検知指標が上流ソースとまだ一致しているか

`detections/` のルールは、上流が実際に出す文字列（プローバ User-Agent・自己更新
User-Agent・監査マーカー・破壊的コマンドの deny リスト）に基づいています。上流の再同期がこの値を変えると、
翻訳検査はすべて通るのに、配布されたルールだけが気づかないうちに一致しなくなります。次のテストは、各指標が
上流ソースとルールの両方にまだあるかを双方向に確認するので、再同期のあとに一緒に実行します。

```bash
detections/tests/indicators/run.sh   # Docker で隔離して実行。RESULT: PASS なら一致
```

失敗すると、どの指標がずれているかと、その方向（上流ソースが変わったのか、ルールが変わったのか）を
出力するので、4.2 の最後の分類基準に従ってルールとレイヤーを新しい値に合わせます。このテストはリポジトリの
CI（[`.github/workflows/detections.yml`](.github/workflows/detections.yml)）でも、ルールツリーか
上記の上流ソースファイルが変わった push・PR のたびに自動で実行され、再同期のドリフトをマージゲートで捉えます。

新しい指標を追加するにあたり **新しい上流ソースファイルを固定した** 場合（例: `cmd/artex/main.go` のポート指標を
入れるとき）、そのファイルを必ず上記ワークフローの `push`・`pull_request` の `paths` フィルタにも追加します。
入れ忘れると、そのソースだけを変えた PR は指標テストをトリガーできず、ドリフトがマージゲートを気づかないうちに
通過します。この同期自体も、指標テストが自動で確認します（5 番目の検査 "CI triggers this
test when any pinned source changes"）。テストが読む `detections/` 以外のソースが、両方の `paths`
ブロックにすべて列挙されていなければテストは失敗するので、ソースの固定と CI のトリガー条件がずれたままマージされる
ことはありません。

### 5.5 検知テストのツールピンを上げるとき

検知テストは、`sigma-cli`・SigmaHQ の検証プラグイン（`pySigma-validators-sigmahq`）・Suricata イメージを
固定バージョンで実行します（各 `run.sh` のデフォルト値。環境変数で上書きできます）。このピンを上げると、上流
ソースではなく **ツール側のドリフト** が生じることがあります。特に SigmaHQ の検証器は、バージョンが上がるたびに新しい規約
検査を追加するので、`detections/tests/sigma_lint/run.sh` が新しい問題を赤で出すことがあります。
そのときはルールを新しい規約に合わせるか、単独のルールセットには合わない規約であればその理由を書いて
[`detections/tests/sigma_lint/validators.yml`](detections/tests/sigma_lint/validators.yml) の除外
リストに追加します。バックエンドプラグインがサポートを変えると、`sigma_backends` テストが同じサインを出します。

---

## 6. ビルドとテストで仕上げの検証をする

反映と翻訳のあと、[CONTRIBUTING.md の開発環境](CONTRIBUTING.md#開発環境) の手順に従ってバックエンドと
フロントエンドを検証します。ローカルに Go がなければ、Docker で同じように実行できます。

```bash
docker run --rm -v "$PWD":/src -w /src \
  -v artexko-gomod:/go/pkg/mod -v artexko-gocache:/root/.cache/go-build \
  golang:1.26 sh -c 'go build ./... && go vet ./... && go test ./... -count=1'
```

ユーザー向けの文言を翻訳するときは、その文言をアサートする回帰テスト（`*_localized_test.go`）も一緒に
置き、あとから上流の変更が再び中国語を引き込んでもテストが捉えるようにします。翻訳の検証は、必ず
能力のある（フロンティア級の）モデルで行います。安価で小型のモデルは、出力が原文に戻ることがあるので、
翻訳が適用されたかどうかを、その出力だけを見て判断してはいけません。

---

## 7. ベース更新の記録

上流の変更を反映し、検証まで終えたら、この文書の 3 節にある **ベースコミットの値を新しい上流コミットに
更新** し、その変更を同じコミットか、その後のコミットに含めます。こうしておけば、次のメンテナーが
「どこまで反映済みか」を、この文書の一か所で確認できます。

コミットメッセージは [CONTRIBUTING.md のコミットメッセージ](CONTRIBUTING.md#コミットメッセージ) に従います。
たとえば上流同期のコミットは、次のように書きます。説明文は日本語です。

```
chore(upstream): 上流 d003372..b55ceb1 を反映 (intercept のトークン計量) + 新しい UI 文字列の翻訳
```

---

## 8. レビューと検証の心得（よくある落とし穴）

上流の反映・翻訳・文書の補強を点検するとき、メンテナーが繰り返し陥る落とし穴を二つ書いて
おきます。どちらも「検査の方法そのものが間違っていて、正常なものを壊れていると誤認する」場合なので、不要な
差し戻しを防ぐために、心得として固定します。

### 8.1 リポジトリの CI 状態は、リポジトリを指定して確認する

このリポジトリは上流 ARTEX のフォークなので、ローカルの `git remote` には `origin`（jiwoochris/artex-global）と
`upstream`（Autumn-27/ARTEX）が一緒に登録されています（3 節を参照）。この状態で `gh` コマンドに
リポジトリを指定しないと、`gh` は **上流リポジトリをデフォルトに選び**、こちらのワークフローがない
上流の実行結果を表示します。すると、上流の CI が緑になっているのを見て **こちらの CI が通過したと
思い込む** か、こちらのワークフロー（`ci.yml`・`detections.yml`）を "HTTP 404 … not found" と誤って
判断することがあります。

そのため CI を確認するときは、常にリポジトリを明示します。

```bash
gh run list -R jiwoochris/artex-global --workflow ci.yml --limit 5
gh run list -R jiwoochris/artex-global --workflow detections.yml --limit 5
```

一度設定しておけば、`-R` を省いてもこのリポジトリをデフォルトとして見るように変えられます。ただしこの
設定は **ローカルの gh 設定** であり、リポジトリにはコミットされないので、新しいマシンや新しいチェックアウトでは、再び
指定する必要があります。

```bash
gh repo set-default jiwoochris/artex-global
gh repo set-default --view   # jiwoochris/artex-global が表示されるか確認
```

### 8.2 文書の外部リンクは、ブラウザと同じく GET で確認する

防御ガイド（[日本語](docs/defense-ja.md) · [English](docs/defense-en.md) · [한국어](docs/defense-ko.md)）の
7 節には、韓国の公的チャネル（boho.or.kr・fsec.or.kr・pipc.go.kr）へのリンクが載っています。このリンクが生きて
いるかを確認するとき、`curl -I`（HEAD リクエスト）やデフォルトの User-Agent だけで確認すると、**正常なリンクを切れている
ものと誤認**します。韓国の公共機関・セキュリティ機関のサイトは、次の三つの理由で単純な確認を拒む
からです。

- **HEAD リクエストを拒否します。** たとえば fsec.or.kr は、`curl -I`（HEAD）に 400 を返します。
- **デフォルトの `curl` User-Agent を遮断します。** fsec.or.kr と pipc.go.kr は、デフォルトの UA で送った
  GET リクエストにも 400 を返します（ブラウザの UA で送ると 200）。
- **別のアドレスへリダイレクトします。** pipc.go.kr は `www.pipc.go.kr` から `pipc.go.kr/np/` へ
  2 回リダイレクトするので、リダイレクトをたどらないと最終的な状態を見逃します。

したがってリンクの確認は、**ブラウザの User-Agent で、GET で、リダイレクトをたどりながら** 行います。

```bash
UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0 Safari/537.36'
for u in https://www.boho.or.kr https://www.fsec.or.kr https://www.pipc.go.kr; do
  curl -sS -L -A "$UA" -o /dev/null -w "$u -> %{http_code} %{url_effective}\n" "$u"
done
```

最終的なステータスコードが 200 なら、リンクは有効です。ステータスコードが 400・403 で返ってきたら、リンクが切れた
のではなく、**確認方法がサーバーのアクセスポリシーに阻まれた** のではないかとまず疑い、HEAD・デフォルト
UA・リダイレクトをたどらないこと、といった要因を一つずつ外して、もう一度確認します。（2026-10-06 の確認時点では、
三つのリンクはいずれも上記の方法で 200 であり、pipc.go.kr は 2 回のリダイレクトのあと 200 です。）

この手動の手順は、`scripts/check-external-links.py` がそのまま自動化します。追跡されているすべての `.md`
から、コードフェンスとインラインコードの外にある外部リンクを集め（予約済みホストとプレースホルダのホストは除く）、上と
同じようにブラウザ UA・GET・リダイレクト追跡で状態を確認し、ネットワークエラー・5xx・429 は再試行して
一時的なちらつきと本当の障害を切り分けます。結果は四つに分けます。OK（2xx・3xx） ·
RESTRICTED（401・403・405・429。ホストは生きていて、阻まれているのは確認方法だけ） · ALLOWED
（`scripts/external-links-allowlist.txt` に書かれた、こちらでは直せない、上流から引き継いだ死んだリンク） ·
DOWN（404・410・5xx・接続エラー。切れている可能性が高い）。

- ネットワークなしで、点検対象だけを事前に見る: `python3 -I scripts/check-external-links.py --list`
- リリース時・定期の点検（新しく切れたリンクがあれば異常終了）: `python3 -I scripts/check-external-links.py --strict`

外部リンクの生存確認は不安定（flaky）なので、**マージゲートには入れません**。代わりに、非ブロッキングのワークフロー
[`external-links`](.github/workflows/external-links.yml) が毎週月曜日と手動実行で
`--strict` を実行し、allowlist にない DOWN が新しく出ると赤くなります。上流の原文を保存している
ファイルが引き継いだ死んだリンク（例: `CHANGELOG.zh.md` がクレジットしている、すでに消えたコントリビューターのアカウント）は、こちらでは
直せないので、理由とともに allowlist に書き、strict 点検から外します。

---

## 9. リリース発行パイプライン

バージョンタグ（`v*`）を push すると、[`.github/workflows/release.yml`](.github/workflows/release.yml) が五つの
プラットフォーム向けバイナリと、条件を満たすときのマルチアーキテクチャ Docker イメージを作ります。このフォークはまだ
リリースタグを切ったことがなく、このワークフローは一度も実行されていないので、この節ではパイプラインが
何を前提とし、何を出力するか、そしてその前提が今のリポジトリ構造と合っているかをまとめます。
タグを push すると公開リポジトリに GitHub Release が作られるので、リリースを切るのは、発行すると決めたあとに
します。

### 9.1 リリースを切る方法

`v` で始まるタグを push すると、ワークフローが起動します。

```bash
git tag v0.3.15
git push origin v0.3.15
```

### 9.2 パイプラインが行うこと

ワークフローは、五つのジョブに分かれます。

- **frontend.** フロントエンドを静的に一度エクスポートし（`web/out`）、その成果物を `web-dist`
  アーティファクトとしてアップロードします。下の binaries ジョブが、対象ごとにこの成果物を取り直して再利用します。
- **binaries.** 五つの対象（linux amd64・arm64、darwin amd64・arm64、windows amd64）をクロス
  コンパイルし、対象ごとに zip にまとめます。linux amd64 のバイナリには `artex -h` のスモークテストを
  実行し、バイナリが実際に動くか確認します。
- **release.** すべての zip を集め、`SHA256SUMS` のチェックサムを作り、GitHub Release を作成して zip と
  チェックサムを添付します。
- **docker.** binaries がクロスコンパイルした linux バイナリを受け取り、マルチアーキテクチャ（amd64・arm64）のイメージを
  ビルドして、GitHub Container Registry の `ghcr.io/jiwoochris/artex-global` にリリースタグと `latest`
  で push します。ログインはワークフロー既定の `GITHUB_TOKEN`（ジョブの権限 `packages: write`）で行うので、別に
  用意するシークレットはありません。

### 9.3 ビルドの前提はリポジトリ構造と合っているか

パイプラインは次の三つの前提の上で動き、この前提が今のリポジトリ構造とすべて合っているかを、
ローカルで binaries ジョブを直接再現して確認しました。

- **フロントエンドの埋め込み。** frontend ジョブがアップロードした `web-dist`（= `web/out` の中身）を binaries ジョブが
  `server/webui/dist` として受け取り、`server/webui_embed.go` の `//go:embed all:webui/dist` がその場所を
  バイナリに埋め込みます。そのため binaries ジョブはフロントエンドを再ビルドせず、
  `ARTEX_SKIP_FRONTEND=1` で [`build.sh`](build.sh) を呼び出します。
- **バイナリとパッケージのパス。** `build.sh --target <os>/<arch>` は、`dist/artex-<os>-<arch>/artex`
  バイナリと、`dist/` 配下の zip パッケージを作ります。zip にはバイナリとともに起動スクリプト（Linux・
  macOS は `start.sh`、Windows は `start.bat`）、`skills/`、`config.example.json`、`README.md` が
  入ります。
- **Docker イメージへのバイナリのコピー。** binaries ジョブは linux バイナリを `bin-linux-<arch>`
  アーティファクトとして別にアップロードし、docker ジョブはこれを `dist/<arch>/artex` として受け取ります。
  [`Dockerfile`](Dockerfile) の `COPY dist/${TARGETARCH}/artex` が、マルチアーキテクチャビルドで buildx
  が各プラットフォームに合わせて埋める `TARGETARCH` によって、そのパスを参照します。
  [`.dockerignore`](.dockerignore) は `dist/` を除外しないので、バイナリはビルドコンテキストに
  含まれます。

### 9.4 Docker イメージの名前空間

イメージは、このフォークが所有する `ghcr.io/jiwoochris/artex-global` に発行し、`docker-compose.yml` もこの
イメージを取得します（2026-10-11 の決定、`work/DECISIONS-FOR-JIWOO.md` の項目 8）。上流の
`autumn27/artex` は Docker Hub から取り下げられており、このフォークには push 権限もないので使いません。
GHCR のパッケージは、最初の発行時に非公開で作られることがあるので、最初のリリースのあと、パッケージ設定で公開
（public）かどうかを確認します。非公開だと、`docker compose pull` が認証エラーで失敗します。

### 9.5 タグなしでローカルに事前検証する

公開リリースを切らずにパイプラインの前提だけを確認するには、binaries ジョブをローカルで再現します。
ローカルに Go がなければ、Docker で同じように実行できます。

```bash
# 1) フロントエンドの静的エクスポート（release.yml の frontend ジョブに相当）
cd web && npm ci && npm run build:static && cd ..
# 2) binaries ジョブがアーティファクトを受け取る位置へ置く
rm -rf server/webui/dist && mkdir -p server/webui/dist && cp -a web/out/. server/webui/dist/
# 3) 一つの対象だけ、binaries ジョブと同じ環境でビルドする
docker run --rm -v "$PWD":/app -w /app \
  -e ARTEX_SKIP_FRONTEND=1 -e ARTEX_SKIP_NPM_CI=1 \
  -e ARTEX_COMPRESS=0 -e ARTEX_PACKAGE=1 -e ARTEX_PACKAGE_DIR=dist \
  -e ARTEX_BUILD_VERSION=v0.0.0-local \
  golang:1.26 bash -c 'apt-get update && apt-get install -y zip && ./build.sh --target linux/amd64'
# 4) 成果物の確認: dist/artex-linux-amd64/artex · dist/*.zip · dist/SHA256SUMS
```

`dist/artex-linux-amd64/artex` は静的リンクされた ELF で、`-h` を付けると使い方を出力したあと、終了
コード 0 で終わります。これが、binaries ジョブのスモークテストが確認する挙動です。ビルド成果物
（`dist/`・`server/webui/dist/`）はリポジトリにコミットしません（`.gitignore` で除外されます）。
