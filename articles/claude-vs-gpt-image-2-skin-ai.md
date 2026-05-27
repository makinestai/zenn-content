---
title: "【Claude vs GPT】肌診断 AI を実働 5 日で成立させた話 — gpt-image-2 が突破したマルチモーダルの壁"
emoji: "⚡"
type: "tech"
topics: ["ai", "claude", "openai", "computervision", "multimodal"]
published: true
---

# 【Claude vs GPT】肌診断 AI を実働 5 日で成立させた話 — gpt-image-2 が突破したマルチモーダルの壁

:::message
**TL;DR**

- Claude Sonnet 4.5 の vision では、肌特徴の正確な位置検出に **構造的限界** があった（頬の左右対称な bbox になる、画像上に注釈を描けない）
- OpenAI の GPT 系画像生成・編集モデル **`gpt-image-2`** で、画像上に **直接・点線囲み・日本語ラベル・凡例・医師相談注記まで自動描画** できた
- 両者組合せ：**¥30/req · 70-210 秒**、実働 5 日でブレークスルー
- Claude vs gpt-image-2 の同入力 3 枚並び比較画像つき
:::

> 妻と一緒に「美容専門の ChatGPT」を作ろうと **実働 5 日で** 6 軸肌診断 RAG を実装。Claude vision で精度が出ず Deep Research でも「商用 API でも未解決」と確認して一度は諦めかけたが、ChatGPT で **GPT 系画像生成・編集モデル `gpt-image-2`** を試したら突破できた、というストーリー型の実装ノートです。

---

## 1. はじめに：妻と ChatGPT の美容相談から見えたもの

きっかけは、妻が ChatGPT を「チャッピー」と呼んで美容相談に使っていると知ったこと。「『肌をキレイにしたい』って相談したらメラノ CC（市販の美白美容液）勧められて買った」と。**AI の推奨が実購買に直結している**。

一方で妻は ChatGPT に 3 つの不満を持っていた：①情報の鮮度がわからない、②「ダークチョコレートブラウン」と言われても画像がないと分からない、③過去の会話を遡れない。

「**画像で示してほしい**」。**画像を AI に理解させる技術（コンピュータビジョン）** と **大規模言語モデル（LLM、文章を生成・理解する AI）** の両方が扱える Claude Sonnet 4.5 なら踏み込めるはず。そう思って実装したのが、6 軸肌診断 × 化粧品問い合わせの RAG だった。

> **ここまでで分かったこと**：当事者の不満を聞くと、AI に欲しい機能は「画像」「鮮度」「履歴」とハッキリしてくる。

---

## 2. 何を作ろうとしたか：6 軸肌診断 × 化粧品問い合わせの RAG

作ったのは、化粧品 EC・美容医療クリニック向けの AI カスタマーサポート。ここでいう **RAG（検索拡張生成、Retrieval Augmented Generation = 外部知識を検索して AI に持たせる仕組み）** は、社内の FAQ や商品データを引いて回答するパターン。

![図 1：RAG パイプライン全体像](/images/zenn1/diagrams/fig_1_rag_flow.png)

肌の観察は 6 つの軸：ニキビ、ニキビ跡（PIH = 炎症後色素沈着）、シミ・そばかす、赤み、毛穴、乾燥。

20 件のテストケースで評価ハーネスを構築し、テキスト応答部分のベースライン[^cache]を確定：

| 指標 | 実測値（Claude Sonnet 4.5） |
|---|---|
| 質の評価（100 点満点、5 観点） | 平均 73.45 / 中央値 83 |
| 1 件あたりのコスト（テキスト RAG） | ¥3.95 |
| 薬機法違反検出率 | 0% |
| Prompt Caching によるコスト削減 | 約 52% |

[^cache]: **Prompt Caching**：プロンプトの前半（システム指示など）を AI 側にキャッシュさせて、2 回目以降の呼び出しコストを下げる仕組み。Anthropic の場合は `cache_control: ephemeral` を指定するだけ。

テキスト応答は十分良い品質に到達。**問題は、画像入力の処理だった**。

> **ここまでで分かったこと**：テキストの RAG は ¥4/件・薬機法違反ゼロまで安定したが、画像が絡んだ瞬間に難易度が跳ね上がる。

---

## 3. 1 回目の壁：Claude vision で肌特徴の位置検出ができない

Claude Sonnet 4.5 の **vision（画像を AI に理解させる機能）** に「ニキビ跡・シミ・赤みの位置を 0〜1 の座標で返して」と頼むだけの **1-shot Vision**（1 回のプロンプトで指示も評価もする方式）から始めた。合成画像や Unsplash のクローズアップでは動いたが、**実画像で精度が出ない**。**バウンディングボックス（対象を囲む四角形）** が頬の上下に対称配置されてしまい、実際の悩み箇所と一致しない。ランドマーク基準・サイズ上限を追加してもラベル重なりは解消したが、**精度は改善しなかった**。

決定打は、**Claude vision に「画像上に注釈を描いて返して」と頼んでも、概念図イラストを別途生成してくるだけ**で、入力画像そのものは編集できないこと。Claude（GPT-4o / Gemini Pro vision も同じ）は「画像を読んで **テキスト** で返す」モデルであり、「画像を編集して **画像** で返す」モデルではない。

### Deep Research で確認した「商用 API でも未解決問題」

ピボットを決める前に、Claude Research と別の Deep Research の **2 本** で「肌診断 AI の商用製品・研究モデル」を調査させた。結果は厳しい：

- **B2B（企業間取引、Business to Business）プラットフォーム**（Haut.AI / Revieve / Perfect Corp）が既に成熟。**OSS（オープンソースソフトウェア、誰でも改変・再配布できる公開ソフト）**で同じ土俵では勝てない
- 臨床検証された論文がある商用エンジンは La Roche-Posay SpotScan+ のみ（皮膚科医評価との一致率 68%）。Perfect Corp の「95%」や Haut.AI の「98%」は **マーケ数値で独立検証なし**
- **スマホセルフィーでの PIH・老人性色素斑・そばかすの完全分離は学術的にも未解決**。公開 OSS で実装するならシニア CV エンジニア 6-9 ヶ月で **MVP**（Minimum Viable Product、最小限の機能で動くプロトタイプ）、¥100-400 万のデータ収集コスト

90 日で「商用品質の肌診断 OSS」を作るのは絶対に不可能。**OSS 化を諦めて別領域に方向転換しよう**——と一度は結論を出した。

> **ここまでで分かったこと**：汎用 LLM の vision で「肌画像の位置検出」は構造的限界がある。商用 API ですら未解決なので、自前実装の前にまず Deep Research で「土俵を確認する」べきだった。

---

## 4. 2 回目の壁の突破：GPT の gpt-image-2 で画像上に直接描画できた

転換点は、妻に試しに ChatGPT に同じ画像を投げてもらったときだった。

> 「ニキビ跡やニキビ、シミ、毛穴の開きを特定して、それぞれをどの症状がどこにあるかを画像上に表示して」

ChatGPT（推定 **GPT 系の `gpt-image-2`**、OpenAI の画像編集モデル）が返したのは、**入力画像そのものに 4 軸を色分け点線で囲み、日本語ラベル + 凡例 + 「医学的診断ではない」注記まで描画**された画像だった。汎用 LLM の vision でできなかったことが、画像編集モデルでは実用レベルで動く。**汎用包丁から専用包丁に持ち替えた感覚**。

![図 3：AI で生成 → AI で注釈、2 段階生成フロー](/images/zenn1/diagrams/fig_3_two_stage_generation.png)

### 仕組み + 実装：30 行

`gpt-image-2` は OpenAI が 2025 年リリースの画像編集モデル。**API（Application Programming Interface、ソフトウェア同士をつなぐ接続口）** の **エンドポイント（特定機能を呼び出す URL）** `POST /v1/images/edits` に画像 + 編集指示を送ると編集後画像が返る。解像度は 1024x1024 / 1024x1536 / 1536x1024、品質は low / medium / high、料金 $0.04 / $0.07 / $0.19 per image。OpenAI Python SDK は Python 3.8 で動かないため、**`httpx`（Python の HTTP 通信ライブラリ）** で直接叩く。

```python
import base64, os, httpx
from pathlib import Path

PROMPT = """\
Annotate the skin photo with dotted-line shapes:
red=活動性ニキビ, orange=ニキビ跡・PIH, brown=シミ・色素沈着, blue=毛穴・キメ.
Preserve original photo. Japanese labels with leader lines.
Legend bottom-left, disclaimer bottom-right.
"""

def annotate(in_path: Path, out_path: Path) -> None:
    r = httpx.post(
        "https://api.openai.com/v1/images/edits",
        files={"image": (in_path.name, in_path.read_bytes(), "image/jpeg")},
        data={"model": "gpt-image-2", "prompt": PROMPT,
              "size": "1024x1536", "quality": "high", "n": "1"},
        headers={"Authorization": f"Bearer {os.environ['OPENAI_API_KEY']}"},
        timeout=300.0,
    )
    out_path.write_bytes(base64.b64decode(r.json()["data"][0]["b64_json"]))
```

### 入力画像も AI で作る、というメタな構成

顔写真の調達は意外と難しい（Unsplash で症状が明確な顔は少ない / クリニックサイトの患者写真はライセンスがグレー / 家族はプライバシー的に NG）。そこで **入力画像も `gpt-image-2` で生成する** 2 段構成にした。ライセンス完全クリア、モデルリリース不要、人種・性別・症状軸を意図的にコントロール可能。「AI で生成 → AI で注釈」というメタなストーリーが記事の核になった。

検証用に 5 枚生成し、症状軸の典型例として 3 枚を採用：

| | 入力（AI 生成） | gpt-image-2 注釈出力（1024x1536 / high） |
|---|---|---|
| **ニキビ系**（アジア女性） | ![入力 3](/images/zenn1/gpt/input_3.png) | ![注釈 3](/images/zenn1/gpt/annotated_3.png) |
| **シミ・そばかす系**（白人女性） | ![入力 5](/images/zenn1/gpt/input_5.png) | ![注釈 5](/images/zenn1/gpt/annotated_5.png) |
| **PIH 系**（東南アジア女性） | ![入力 7](/images/zenn1/gpt/input_7.png) | ![注釈 7](/images/zenn1/gpt/annotated_7.png) |

3 枚いずれも凡例の 4 軸が綺麗に読め、日本語注記も完璧、症状箇所を色分け点線で囲む。**凡例まで自動で書いてくる**のは正直驚いた。

### 試行錯誤で見えた実装ノウハウ

**アスペクト比は 1024x1024 ではなく 1024x1536（顔写真）/ 1536x1024（横長）必須**——日本語ラベルのレンダリングが大きく改善する。quality も high 必須（medium だと文字崩れ）。Safety System は確率的で、医療表現で初回ブロックされても再試行で通る。5 枚生成 + 3 枚注釈で合計約 ¥250。

> **ここまでで分かったこと**：「Claude では構造的に無理」が、別カテゴリのモデル（`gpt-image-2`）だと実用品質で動く。**モデル選択そのものがアーキテクチャ**。

---

## 5. Claude vs gpt-image-2：出力・コスト・速度の比較

![図 2：Claude vision と gpt-image-2 の入出力形式の違い](/images/zenn1/diagrams/fig_2_model_comparison.png)

### 同じ 3 枚を両方で処理した実画像比較

| 症状軸 | Claude Sonnet 4.5 vision（座標 → Pillow 描画） | gpt-image-2（high、1024x1536、直接描画） |
|---|---|---|
| **ニキビ系** | ![claude_3](/images/zenn1/claude/annotated_3.png) | ![gpt_3](/images/zenn1/gpt/annotated_3.png) |
| **シミ・そばかす系** | ![claude_5](/images/zenn1/claude/annotated_5.png) | ![gpt_5](/images/zenn1/gpt/annotated_5.png) |
| **PIH 系** | ![claude_7](/images/zenn1/claude/annotated_7.png) | ![gpt_7](/images/zenn1/gpt/annotated_7.png) |

Claude vision 側はバウンディングボックスが頬の左右に対称配置されがち（1-shot Vision の限界）、矩形は塗りつぶしで輪郭にフィットせず、凡例も注記もない。対して `gpt-image-2` 側は症状の輪郭に追従する自由曲線、4 色色分け点線、凡例 + 日本語注記まで自動描画。**同じ入力画像でここまで違う**事実そのものが、最大の発見だった。

### スペック + コスト

| 観点 | Claude Sonnet 4.5 vision | gpt-image-2（high、1024x1536） |
|---|---|---|
| 出力形式 | テキスト（座標 + ラベル） | **画像（注釈レイヤー直接描画）** |
| 後処理 | 必要（Pillow 等） | 不要（API 1 回完結） |
| 1 回あたりコスト | ¥8-10（実測） | **¥28.5** |
| 1 回あたり所要時間 | 30-60 秒 | **70-210 秒** |

両者組合せで **¥36-38/req**。Claude を「テキスト応答 + 商品推奨」、`gpt-image-2` を「画像注釈の可視化」に分業する設計が現実的。Prompt Caching で Claude 側は ¥2-3/req まで下がるので合計 ¥30/req 程度。化粧品 EC なら「テキストは数秒で返し、可視化画像は 2-3 分後にメール通知」のハイブリッド UX が現実的。

> **ここまでで分かったこと**：マルチモーダル AI は **「テキスト出力モデル」と「画像出力モデル」を分業させる**のが現実解。コストは加算されるが ¥30/req なら B2B 用途は十分許容範囲。

---

## 6. 学びと、次に作るもの

![図 4：実働 5 日の判断プロセスタイムライン](/images/zenn1/diagrams/fig_4_timeline.png)

実働 5 日（5/19-5/24）で得た学びは 3 つ。

**①限界は「モデル」か「タスク設計」か見極める**：Claude vision で精度が出なかったのは Claude のせいではなく、「テキストモデルに画像編集を求めた」設計の問題。**別カテゴリのモデルで再定義する**と突破口が見える。

**②失敗→検証→ピボットの判断プロセスを記録する**：「Claude vision 試行 → Deep Research → 諦めかけ → gpt-image-2 偶然発見 → 再ピボット成立」。ネガティブな結果も含めて記録することで、後追いの時間を節約できるし判断の透明性も上がる。

**③評価ハーネスは題材を変えても活きる**：5 軸ルーブリック + Sonnet judge + Prompt Caching は肌診断でも社内 wiki RAG でも流用できる資産。**「題材を作る」より「測る仕組みを作る」ほうが長期リターンが大きい**。

### 次に作るもの：中堅企業の社内問い合わせ AI

評価ハーネスを別ドメインに流用し、情シス / 人事 / 経費 / マニュアル の社内問い合わせ AI を Dify で実装した記事を、続編として公開しました（[Dify で社内問い合わせ AI を作ってみた](https://zenn.dev/makinestai/articles/dify-vs-claude-ops-rag-20-cases)）。コードと再現手順は [`ops-rag-template`](https://github.com/makinestai/ops-rag-template) にまとめています。v0.2 では、社内 PDF マニュアルの図表検索（本記事の画像理解の応用）を実装予定です。

> **ここまでで分かったこと**：題材ピボットの判断プロセスそのものが技術ブログとして価値を持つ。**評価ハーネスは題材横断の資産**。

---

## おわりに

「動かないと思っていたものが、モデルを変えたら動いた」。実働 5 日で得た一番の学びはこれです。成功事例の裏には「失敗→検証→別アプローチ→再検証」の試行錯誤があり、今回は妻の **dogfooding（自社製品を自分で使って検証すること）** + Deep Research + ChatGPT 偶発試行が重なってブレークスルーに繋がりました。マルチモーダル AI 導入を検討中の方は、**まず複数モデルの出力形式の違いを 1 枚の画像で実機比較する**ことを強くおすすめします。実装コード（`skin_annotate_gpt.py`）と検証手順は近日 GitHub で公開予定。

---

### こちらの記事もどうぞ

- 📘 **続編**：[Dify で社内問い合わせ AI を作ってみた — 情シス・人事・経費の質問に答える 5 つの Knowledge](https://zenn.dev/makinestai/articles/dify-vs-claude-ops-rag-20-cases) — 評価ハーネスを別ドメインに流用
- 🛠️ **GitHub**：[`cs-rag-template`](https://github.com/makinestai/cs-rag-template) — 本記事の評価ハーネス + Caching の参考実装
- 🛠️ **GitHub**：[`ops-rag-template`](https://github.com/makinestai/ops-rag-template) — 社内 wiki RAG テンプレ（Zenn 2 の題材）
- 🌐 **MakiNest AI 公式**：[https://makinestai.github.io](https://makinestai.github.io) — 屋号サイト

役に立ったらいいねやコメント、Zenn / X でのシェアをいただけると嬉しいです。同じ壁にぶつかっている方の時間節約になれば。

---

### 執筆者プロフィール

- AI エンジニア / コンピュータビジョン専門
- **CVPR**（Computer Vision and Pattern Recognition、画像認識・AI 分野で世界最大級の査読付き国際学会）の **2025 年 Workshop に研究採択**
- 経歴：製造業現場 3 年 → 大学（CV・機械学習専攻）→ 企業の研究開発職
- **MakiNest AI**：仕事を巻き取り、可能性を育てる AI の巣。散らばった情報や繰り返し作業を AI で整理し、人とチームがもっと創造的に働ける余白をつくります。[公式サイト →](https://makinestai.github.io)

### 画像クレジットと注意事項

入力画像（input_3/5/7）は ChatGPT 経由で `gpt-image-2` 生成、注釈出力（gpt/annotated_*）は `skin_annotate_gpt.py`、Claude vision 出力（claude/annotated_*）は `minimal_multimodal.py` で生成。**入力も AI 生成のため、実際の肌悩みを反映しているわけではない**点にご留意ください。実際の肌悩みは皮膚科医にご相談を。

### 参考リサーチ

- Seité S, et al. *Experimental Dermatology* 2019; 28(11): 1252–1257（SpotScan+ 臨床評価）
- Haut.AI / Revieve / Perfect Corp 公式ドキュメント、OpenAI Images API ドキュメント
