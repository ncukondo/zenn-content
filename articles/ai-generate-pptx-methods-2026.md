---
title: "生成AIでパワポを作る方法一覧【2026年3月版】"
emoji: "📊"
type: "tech"
topics: ["ai", "powerpoint", "pptx", "claude", "生成AI"]
published: true
---

## はじめに

生成AIを使ったスライド作成ツールの紹介記事は増えていますが、その多くはPDFやHTML出力が中心です。しかし実際の仕事では、上司や共著者が追加編集する、提出物としてパワポ形式(PPTX)が指定される、といった理由からPPTXでの出力が必要な場面が少なくありません。

本記事では、2026年3月時点で利用可能な方法をGUI完結型・Markdown型・コード生成型の3カテゴリに分けて、PPTX出力の観点から整理します。

## PPTX出力の「編集可能性」に注意

まず前提として、**PPTX出力に対応していても、中身が編集可能とは限らない**という点を押さえておく必要があります。

たとえば[NotebookLM](https://notebooklm.google.com/)は2026年2月にPPTXダウンロードに対応しましたが、各スライドが画像として埋め込まれた「一枚絵」です。スライド内の画像生成に[Nano Banana Pro](https://blog.google/innovation-and-ai/products/nano-banana-pro/)という画像生成AIが使われており、テキストも含めてすべてが画像になっています。PowerPointで開いてもテキストを直接編集することはできません。

[Gamma](https://gamma.app/)はテキスト自体は編集可能なPPTXを出力しますが、独自のカードベースレイアウトとPowerPointのスライド構造の違いにより、フォントのずれやテキストボックスの位置ずれが頻繁に発生します。

この「PPTX出力対応」と「編集可能なPPTX出力」の違いを意識した上で、以下の方法を見ていきます。

## 1. GUI完結型：プロンプトを入力するだけ

### Gamma

https://gamma.app/

[Gamma](https://gamma.app/)は、テキストプロンプトやファイルからスライド全体を生成するAIツールです。テンプレートが約30種類用意されており、デザイン品質が高いのが特徴です。無料枠は400クレジット（1生成あたり約40クレジット）。

PPTX出力に対応しており、テキスト自体は編集可能です。ただし前述の通り、レイアウト崩れが起きやすい点には注意が必要です。Gammaで作ったスライドをそのままプレゼンに使う分には問題ありませんが、PPTXに出力して細かく編集する用途では手直しが発生します。

### NotebookLM

https://notebooklm.google.com/

[NotebookLM](https://notebooklm.google.com/)は、Googleが提供するAIツールです。PDF等の資料をアップロードし、それをソースとしてスライドを自動生成します。ソース資料のみに基づいて生成するためハルシネーションが少なく、無料版でも日3回利用できます。

2026年2月にPPTXダウンロードに対応しましたが、前述の通り各スライドは画像埋め込みです。つまり直接テキストを編集できません。「AI生成スライドの構成を参考にして、自分でスライドを作り直す」という使い方が現実的です。PPTXとしての編集が必要な場合は、[Codia AI NoteSlide](https://www.codia.ai/)などの外部ツールで再構築する手段もあります。

### Claude.ai（PPTX Skill）

https://claude.ai/

[Claude.ai](https://claude.ai/)は、ブラウザ上でPPTXファイルを直接生成できます。内部的には[python-pptx](https://python-pptx.readthedocs.io/)ライブラリを使ったコード実行により、テキストが編集可能な正規のPPTXを出力します。

- テンプレートPPTXを添付すれば、デザインを維持したまま内容を更新可能
- 生成後にQAパスを実行し、レイアウトの問題をチェックする機能も備える
- 無料プランを含む全プランで利用可能（コード実行とファイル作成がデフォルトで有効）

GUI完結型の中では、編集可能なPPTXを確実に出力できる点で、PPTX出力の観点からは現時点で最も実用的な選択肢です。

![Claude.aiでPPTXを生成している画面。15枚のスライド構成が表示され、右側のArtifactsパネルにPPTXファイルとプレビューが確認できる](/images/claude-ai-pptx-skill.png)
*Claude.aiのPPTX Skill で15枚のスライドを生成した例*

### Claude in PowerPoint（アドイン）

https://support.claude.com/en/articles/13521390-use-claude-for-powerpoint

2026年2月にリリースされた[Claude in PowerPoint](https://support.claude.com/en/articles/13521390-use-claude-for-powerpoint)は、PowerPointアプリ内で直接Claudeを使えるアドインです。既存スライドの編集支援やトークスクリプトの生成が可能です。ゼロから生成するというよりは、既存のパワポを編集・改善する用途に向いています。現時点ではベータ版で、Pro以上のプランが必要です。

### その他

- [Microsoft Copilot](https://copilot.microsoft.com/)：[Microsoft 365](https://www.microsoft.com/ja-jp/microsoft-365)に統合されたAIアシスタント。PowerPoint内でスライドの下書きを生成でき、当然ながらPPTX編集性は問題ない
- [Canva AI](https://www.canva.com/)：豊富なデザインテンプレートとAI補助を組み合わせたスライド作成。PPTX出力対応で編集可能
- [SlidesGPT](https://slidesgpt.com/)：[ChatGPT](https://chatgpt.com/)技術を活用したスライド生成。編集可能なPPTXを出力できるが、ダウンロードにはProプラン（$9.99/月）が必要

## 2. Markdown型：テキストファイルでスライドを管理

Markdown形式でスライドを記述し、AIにそのMarkdownを生成させるアプローチです。スライドの内容がテキストファイルになるため、Git管理ができ、差分が明確になります。

### Marp + AI

https://marp.app/

[Marp](https://marp.app/)は、Markdownからプレゼンテーションを生成するエコシステムです。HTML、PDF、そしてPPTXへの変換に対応しています。

```markdown
---
marp: true
theme: default
---

# スライドタイトル

- 箇条書き1
- 箇条書き2

---

# 次のスライド

内容をここに書く
```

AIとの組み合わせ方はシンプルです。Claude（やChatGPT）に「次の内容をMarp形式のMarkdownで出力してください」と指示するだけで、スライドの下書きが得られます。Markdownなので修正も容易です。[VS Code](https://code.visualstudio.com/)拡張でリアルタイムプレビューも可能で、開発者にとっては馴染みやすいワークフローです。

```bash
# Marp CLIでPPTXに変換
npx @marp-team/marp-cli slide.md --pptx
```

ただし、MarpのPPTX出力にも注意点があります。デフォルトの`--pptx`オプションで生成されるPPTXは、各スライドが事前レンダリングされた画像として埋め込まれます。NotebookLMと同様に、テキストを直接編集することはできません。Marp開発チームは、複雑なHTML/CSSベースのレイアウトを維持しつつ編集可能なPPTXを生成するのは困難であるとしており、デザインの再現性を優先した設計です。

編集可能なPPTXが必要な場合は、実験的オプション`--pptx-editable`を使用できます。

```bash
# 編集可能なPPTXを生成（実験的）
npx @marp-team/marp-cli slide.md --pptx --pptx-editable
```

ただし、このオプションにはいくつかの制限があります。

- デザインの再現性が通常のPPTX出力より低い
- 複雑なスタイルを使用しているとエラーや不完全な出力になることがある
- [LibreOffice Impress](https://www.libreoffice.org/discover/impress/)のインストールが必要
- 発表者ノートが非対応

Marpは「Markdownで書いてGit管理できる」という点が最大の強みであり、PPTX出力はあくまで補助的な位置づけです。編集可能なPPTXが最終成果物として求められる場面では、この制限を把握した上で使う必要があります。

### Slidev（参考：PPTX非対応）

[Slidev](https://sli.dev/)は[Vue.js](https://vuejs.org/)ベースの開発者向けスライド作成ツールで、表現力が高いのですが、出力はHTMLとPDFが中心でPPTX出力には対応していません。ブラウザ上でプレゼンする前提のツールです。PPTX提出が必要な場合は選択肢から外れます。

## 3. コード生成型：プログラムでスライドを組み立てる

スライドの各要素をプログラムで制御する方法です。手軽さでは劣りますが、デザインの一貫性・完全な制御・再現性において最も優れています。コードで生成するため、出力は当然ながら編集可能な正規のPPTXです。

### Claude Code + PptxGenJS

筆者が実際にワークショップの事前動画用スライドを作成した際のアプローチです。[PptxGenJS](https://github.com/gitbrent/PptxGenJS)（TypeScript/JavaScript向けPPTX生成ライブラリ）を[Claude Code](https://github.com/anthropics/claude-code)と組み合わせ、PPTX生成からナレーション付きMP4動画までを自動化しました。ソースコードは[GitHub](https://github.com/ncukondo/presentation-generator)で公開しています。

https://github.com/ncukondo/presentation-generator

#### プロジェクト構造

```
presentation-generator/
├── slides.yaml              # スライド内容＋ナレーション原稿
├── CLAUDE.md                # Claude Codeへのプロジェクト説明
├── generate.ts              # PPTX生成
├── screenshot.ts            # PPTX → PNG変換
├── tts.ts                   # Gemini TTSでナレーション音声生成
├── video.ts                 # PNG + 音声 → MP4動画生成
├── lib/
│   ├── theme.ts             # カラー、フォント、サイズ等の定数
│   ├── helpers.ts           # 再利用可能なスライドビルダー関数
│   ├── slides-data.ts       # slides.yamlの読み込み・取得
│   ├── cite.ts              # 引用管理（APA形式）
│   └── types.ts             # 型定義
├── pages/
│   ├── slide01-title.ts     # タイトルスライド
│   ├── slide02-background.ts
│   └── ...                  # スライドごとのビルダー
├── presentation.pptx        # 生成されたPPTX
├── output_images/           # 生成されたPNG
└── voice_output/            # 生成された音声ファイル
```

ポイントは`slides.yaml`（内容・ナレーション定義）とコード（デザイン実装）の分離です。

#### slides.yaml：スライド内容とナレーションの一元管理

`slides.yaml`にスライドの構成、内容、ナレーション原稿をまとめて記述します。デザインの指定は含みません。

```yaml
slides:
  - id: background
    title: "背景：なぜ今、生成AIで教材を作るのか"
    subtitle: "教育現場が直面する3つの課題"
    cards:
      - heading: "知識量の増大と認知負荷"
        body: "医学知識の量と複雑性はワーキングメモリの限界を超えやすい"
        detail: "情報量の増大に対し、構造化された提示が必要"
        cites: ["sweller2011"]
      - heading: "教育者の時間的制約"
        body: "臨床業務と教材作成の両立は困難"
        detail: "限られた時間で質の高い教材を準備する必要がある"
        cites: ["prince2004"]
    narration: |
      教育現場では3つの大きな課題があります。
      まず、医学知識の急速な増大により...
```

YAML形式にすることで各スライドに`id`を付与でき、TypeScript側から`getSlide("background")`のように型安全にデータを取得できます。ナレーション原稿も同じファイルに含められるため、内容と読み上げテキストの整合性を保ちやすい構造です。

#### theme.ts：デザイン定数の集約

色、フォント、フォントサイズなどのデザイン定数を一箇所に集約します。WCAG AAのコントラスト比も注釈しています。

```typescript
export const C = {
  primary: "1E88E5",       // Material Blue 600 — 4.05:1 ✓
  accent: "FF9800",        // Material Orange 500
  text: "37474F",          // Blue Grey 800
  white: "FFFFFF",
  warmBg: "FFF9F2",        // warm cream for slide backgrounds
  step1: "1E88E5",         // カードごとの色分け用
  step2: "43A047",
  step3: "AB47BC",
  // ...
} as const;

export const FS = {
  slideTitle: 28,
  heading: 24,
  body: 22,        // 最小フォントサイズ
} as const;
```

デザインルールが定数として定義されているため、Claude Codeが新しいスライドを追加する際にも一貫したデザインになります。

#### helpers.ts：再利用可能なビルダー関数

スライドの共通パターンをヘルパー関数として抽出します。

```typescript
/** タイトルバー付きのコンテンツスライドを生成 */
export function addContentSlide(pres: Pres, title: string): Slide {
  const slide = pres.addSlide();
  slide.addShape(pres.ShapeType.rect, {
    x: 0, y: 0, w: SLIDE_W, h: SLIDE_H,
    fill: { color: C.warmBg },
  });
  slide.addShape(pres.ShapeType.rect, {
    x: 0, y: 0, w: SLIDE_W, h: 1.05,
    fill: { color: C.primary },
  });
  slide.addText(title, {
    x: MARGIN.left, y: 0.15, w: CONTENT_W, h: 0.75,
    fontSize: FS.slideTitle, fontFace: FONT, color: C.white,
    bold: true, align: "left", valign: "middle",
  });
  return slide;
}
```

`addCard`（カラーバンド付きカード）、`threeColLayout`（3カラムレイアウト）なども用意しておくことで、各スライドのビルダーは内容の配置に集中できます。

#### pages/：スライドごとのビルダー

各スライドは独立したファイルに1関数として定義します。`getSlide(id)`でYAMLからデータを取得し、レイアウトだけをコードで記述します。

```typescript
// pages/slide02-background.ts
import { getSlide } from "../lib/slides-data";

export function buildSlide02(pres: Pres) {
  const d = getSlide("background");
  const slide = addContentSlide(pres, d.title);
  if (d.narration) slide.addNotes(d.narration.trim());

  const { xs, colW } = threeColLayout(0.35);
  const cards = d.cards as Array<{ heading: string; body: string; detail: string }>;

  cards.forEach((card, i) => {
    addCard(slide, xs[i], cardY, colW, cardH,
      card.heading,
      [{ text: card.body, options: { fontSize: FS.small } }],
      CARD_COLORS[i],
    );
  });
}
```

各ビルダーはコンテンツをコードに直接書かず、YAMLから取得します。これにより、内容の変更はYAMLの編集だけで完結します。

#### CLAUDE.md：AIへのプロジェクト説明

`CLAUDE.md`にはプロジェクト構造、ビルドコマンド、デザインルールを記載します。

```markdown
## Build Commands
bun run generate      # PPTX生成のみ
bun run build         # generate + PNG変換
bun run tts           # ナレーション音声生成
bun run video         # PNG + 音声 → MP4動画生成

## Typography Rules
- **最小フォントサイズ: 22pt**（FS.body 以上を使用すること）
- テキストがボックスからはみ出す場合は内容を削減・簡潔にする
```

このファイルがあることで、Claude Codeは「slides.yamlを読んで内容を把握し、既存のtheme.tsとhelpers.tsのパターンに従って新しいスライドを追加する」という作業を自律的に行えます。

#### ワークフロー

最初から最後までClaude Codeと対話しながら進めます。

1. **構成づくり**：小さなアイデアや既存の資料をClaude Codeに渡し、一緒に`slides.yaml`を組み立てる
2. **スライド生成**：Claude Codeがtheme.ts、helpers.tsのパターンに合わせてpages/以下にビルダーを生成し、`bun run build`でPPTXとPNGを出力
3. **フィードバックと修正**：生成されたスクリーンショットを見て「この図を大きく」「文言を変えて」とClaude Codeに伝え、修正を繰り返す
4. **ナレーション確認**：`bun run tts`で[Gemini TTS](https://ai.google.dev/gemini-api/docs/speech-generation)による音声を生成し、読み上げの出来栄えを確認。不自然な箇所はslides.yamlのナレーション原稿を調整
5. **動画生成**：`bun run video`でPNGと音声を合成しMP4動画を生成
6. **共著者からのフィードバック反映**：共著者がPowerPointで直接編集したPPTXが戻ってきた場合は、そのファイルをClaude Codeに読み込ませ、変更内容を`slides.yaml`に反映させる

デザインの修正は`theme.ts`や`helpers.ts`を変えれば全スライドに一括反映されます。内容・ナレーション・デザインがすべてテキストファイルで管理されているため、どの段階からでもClaude Codeと一緒に修正を回せます。

#### なぜこのアプローチを選んだか

GUI完結型と比べると初期の手間はかかります。しかし、以下の場面では明確に優位性があります。

- デザインの一貫性が重要な場合：theme.tsで色・フォント・サイズを一元管理するため、スライド間でデザインがブレない
- 繰り返し生成する場合：内容だけ変えてビルドし直せる。同じテンプレートで複数のプレゼンを作れる
- Git管理したい場合：全てがテキストファイルなので差分管理が容易
- 確実に編集可能なPPTXが必要な場合：PptxGenJSは正規のPPTXを生成するため、出力後にPowerPointで自由に編集できる
- ナレーション付き動画まで自動化したい場合：slides.yamlにナレーション原稿を含め、Gemini TTSで音声生成、[ffmpeg](https://ffmpeg.org/)でMP4動画まで一気通貫で生成できる
- 学術用途で文献参照が必要な場合：slides.yamlに引用キーを記述し、[ref](https://github.com/ncukondo/ref)（文献管理CLIツール）と連携してAPA形式の引用を自動生成できる。GUI完結型やMarkdown型のツールでは、文献リストの管理と参照の挿入を手作業で行う必要があり、スライド枚数が増えると整合性の維持が難しくなる

### Claude Code + Office PowerPoint MCP（参考）

[Office PowerPoint MCPサーバー](https://github.com/GongRzhe/Office-PowerPoint-MCP-Server)を使うと、Claude Codeから32種類のPowerPoint操作ツールを利用できます。python-pptxベースで、CLAUDE.mdにデザインルールを定義しておくことで、自然言語の指示からPPTXを自動生成できます。

PptxGenJSのようなプロジェクト構造を自分で設計する必要がない分、手軽に始められます。「コード生成型を試してみたいが、プロジェクト構成を一から作るのは大変」という場合の入り口として検討できるアプローチです。

## 比較まとめ

| 方法 | 手軽さ | カスタマイズ性 | PPTX編集可能性 | 再現性 |
|---|---|---|---|---|
| Gamma | ◎ | △ | △ レイアウト崩れ | △ |
| NotebookLM | ◎ | △ | ✗ 画像埋め込み | △ |
| Claude.ai PPTX Skill | ○ | ○ | ○ | ○ |
| Marp + AI | ○ | ○ | △ 要`--pptx-editable` | ◎ |
| PptxGenJS + Claude Code | △ | ◎ | ◎ | ◎ |

## おわりに

生成AIでスライドを作るツールは数多くありますが、編集可能なPPTXを出力できるかという観点で見ると、選択肢はかなり絞られます。

「とりあえず1回作りたい」ならClaude.aiのPPTX Skillが手軽で確実です。「Markdownで管理したい」ならMarpがPPTX出力にも対応しています。「デザインの一貫性と再現性を重視する」ならPptxGenJSのようなコード生成型が適しています。

パワポでの提出が求められる場面は今後も続くでしょう。自分のワークフローに合った方法を見つけて、スライド作成の時間を本来の内容づくりに振り向けていきましょう。
