# スクリーンショットの案内

日本語 · [English below](#english)

このフォルダには、別々の二組の画面が入っています。混ざって見えないように、先にどちらが何かを示します。

- **`en/`: デフォルトの英語 UI をキャプチャした、ARTEX の実際の画面。** このリポジトリが届ける成果物です。
  メイン文書の [`../README.md`](../README.md)、[`../README.en.md`](../README.en.md)、[`../README.ko.md`](../README.ko.md) が
  この画像を使います。届ける英語の画面だけを見るときは、[`en/`](en/) フォルダを開いてください。
- **ルートの `*.png`: 上流の原本（中国語版）の UI キャプチャ。** 上流リポジトリと 1:1 で突き合わせられるように **原本のまま保存** したもので、保存された中国語 README の [`../README.zh.md`](../README.zh.md) がこの画像を参照します。日本語・英語・韓国語の文書では使いません。
- **`wx.png`: 上流の作者による WeChat 公式アカウントの宣伝画像。** [`../README.zh.md`](../README.zh.md) が原本のまま描画されるよう、一緒に保存してあります。このフォークとは無関係です。

つまり、ルートで中国語の画面が先に見えるのは上流の保全のためであり、**届ける英語の画面は [`en/`](en/) の中にあります。** このリポジトリがなぜ原本を一緒に置くのか（ローカライズの透明性、上流との対照）は、防御・検知の位置づけと同じ文脈です。詳しい背景はルートの README を参照してください。

---

## English

This folder holds two separate sets of captures. To keep them from looking mixed, here
is what each one is.

- **`en/` — real UI of ARTEX in the default English display language.** This is what the fork delivers. The
  main docs [`../README.md`](../README.md) (Japanese), [`../README.en.md`](../README.en.md), and [`../README.ko.md`](../README.ko.md) use
  these images. For the delivered screens, open [`en/`](en/).
- **The `*.png` files in the root — upstream (Chinese edition) UI captures.** They are
  **preserved as-is** so the fork can be diffed against the upstream repository 1:1, and
  the preserved Chinese README [`../README.zh.md`](../README.zh.md) references them. The
  Japanese, English, and Korean docs do not use them.
- **`wx.png` — the upstream author's WeChat channel promo image.** Kept so that
  [`../README.zh.md`](../README.zh.md) still renders unchanged. It is unrelated to this
  fork.

In short, the Chinese screens you see first in the root are there for upstream
preservation; the **delivered English screens live in [`en/`](en/).** See the root README
for why this fork keeps the originals alongside (localization transparency and upstream
comparison).
