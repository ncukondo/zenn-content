---
title: "AI時代の論文執筆環境：Word＋文献管理ソフトから抜け出す"
emoji: "📝"
type: "tech"
topics: ["vscode", "markdown", "pandoc", "ai", "論文執筆"]
published: true
---

## はじめに

(私の周りの)研究者の論文執筆ワークフローは、EndNoteやMendeley、PaperpileといったGUIベースの文献管理ソフトで文献を整理し、Wordで原稿を書く、というのが大半です。このフローは比較的敷居が低く、医師としても働いており時間が無い私の周りの研究者達にも紹介しやすいという理由で、私も長らくこのスタイルを推奨してきました。

一方で、ChatGPTやGeminiのような生成AIが登場し、文献検索の際に、「このテーマの最新レビューを探して」と頼めば、関連論文をリストアップしてくれるようになりました。

しかし、その先に壁があります。

AIが見つけてくれた論文を実際に使うには、自分でWebサイトにアクセスして内容を確認し、文献管理ソフトに手動で取り込み、Wordのプラグインで挿入する必要があります。AIが活躍するのは検索の入り口だけで、そこから先は従来通りの手作業です。

この記事では、検索から執筆までをAIと統合するために筆者が構築してきた環境と、そこで最後のピースになったVS Code拡張機能[Markdown Academic Preview](https://github.com/ncukondo/vscode-markdown-academic-preview)について紹介します。

## refとsearch-hubで文献管理をAIに任せる

この問題に取り組むにあたって、まず文献の検索・管理の部分をAIに任せられるようにしました。詳細は過去の記事に書いたので、ここでは概要だけ触れます。

### reference-manager（refコマンド）

https://github.com/ncukondo/reference-manager

CLIベースの文献管理ツールです。CSL-JSON形式の単一ファイルをデータソースとし、`ref add`で文献を登録、`ref search`で検索、`ref cite`で引用を生成できます。GUIではなくCLIなので、[Claude Code](https://code.claude.com/)のようなAI agentがそのまま操作できます。

詳しくは以下の記事を参照してください。

https://zenn.dev/ncukondo/articles/ai-agent-reference-management-cli

### research-hub（search-hubコマンド）

https://github.com/ncukondo/research-hub

PubMed・Scopus・ERIC・arXivなど複数の学術データベースを横断的に検索するCLIツールです。系統的文献検索に必要な再現可能な検索プロセスを、AI agentと協働で進められます。

詳しくは以下の記事を参照してください。

https://zenn.dev/ncukondo/articles/ai-agent-ux-design-for-cli

### ここまでで解決したこと

この2つのツールにより、「AIに文献を探してもらい、そのままライブラリに登録する」という流れが実現しました。文献管理ソフトのGUIを開いて手動で取り込む作業は不要になっています。

残る課題は「執筆」です。

## Wordで書き続ける限り、AIとの統合には限界がある

refで管理している文献ライブラリ（CSL-JSONファイル）は、そのままではWordから参照できません。Wordで論文を書くなら、結局EndNoteやMendeleyに文献を取り込み直す必要があります。せっかくAIが一貫して管理しているデータを、人間が手動で別のソフトに橋渡しすることになります。

さらに、Wordの`.docx`ファイルはAI agentが直接編集するのも難しい形式でもあります。

AIが読み書きを得意とする、Markdown形式を使えばこの問題は解決できます。

## Markdownで論文を書くという選択肢

Markdownに馴染みのない方のために簡単に説明すると、Markdownはテキストに簡単な記号で書式をつける記法です。`**太字**`で太字、`# 見出し`で見出しになります。GitHubのREADMEやZennの記事もMarkdownで書かれています（この記事もMarkdownです）。

[Pandoc](https://pandoc.org/)は、MarkdownをWord、PDF、LaTeXなどの形式に変換するツールです。Pandocは独自の引用記法をサポートしていて、Markdownの中に`[@smith2020]`と書けば、BibTeXやCSL-JSONといったAIが扱いやすい形式の文献データから自動的に引用と参考文献リストを生成してくれます。

つまり、Markdown + Pandocなら、refのライブラリをそのまま参照ファイルとして使えます。文献管理ソフトへの再取り込みは不要です。

```markdown
---
bibliography: ~/ref-library/references.json
---

先行研究では、この手法の有効性が報告されている[@kondo2023]。
```

これだけで、refのライブラリから`kondo2023`の書誌情報を引いて、正しい引用形式でレンダリングしてくれます。

## 執筆環境としてのVS Code

Markdownで書くにはテキストエディタが必要です。ここで使うのが[VS Code](https://code.visualstudio.com/)（Visual Studio Code）です。

VS Codeはもともとプログラマ向けに開発されたテキストエディタですが、Markdownのプレビュー機能を内蔵しており、「拡張機能」と呼ばれるプラグインで機能を追加できます。無料で使え、Windows・Mac・Linuxのいずれでも動きます。

同様のエディタとして、AI機能を統合した[Cursor](https://www.cursor.com/)やGoogleの[Antigravity](https://antigravity.google/)もあります。これらはVS Codeをベースにしているため、同じ拡張機能がそのまま使えます。AI agentとの協働を重視するなら、こうしたエディタも有力な選択肢です。

## プレビューの問題

ここまでで、Markdownで書いてPandocで変換すればよいことはわかりました。しかし、実際に書き始めると別の問題が出てきます。

論文を書いている途中で、引用や参考文献リストがどう表示されるかを確認したい。これは当然の欲求です。Wordなら書きながらリアルタイムに見えていたものが、プレーンテキストのMarkdownでは見えません。

VS Codeには[Markdown Preview Enhanced](https://marketplace.visualstudio.com/items?itemName=shd101wyy.markdown-preview-enhanced)という拡張機能があり、Pandocと連携してプレビューを表示できます。しかし、これを使うにはまずPandocをインストールし、パスを通し、設定ファイルを整え……というセットアップが必要です。

この「プレビュー環境の構築」が、Markdownで論文を書く上での大きなハードルでした。

## Markdown Academic Previewで解決する

https://github.com/ncukondo/vscode-markdown-academic-preview

[Markdown Academic Preview](https://marketplace.visualstudio.com/items?itemName=ncukondo.vscode-markdown-academic-preview)は、この問題を解決するために開発したVS Code拡張機能です。

Pandocのインストールは不要です。引用や参考文献リストのレンダリングを、JavaScript（[citation-js](https://citation.js.org/)）だけで行うので、拡張をインストールするだけで動きます。

VS Code標準のMarkdownプレビューを拡張する仕組みなので、KaTeXなど他の数式拡張とも共存できます。Pandocの引用記法（`[@key]`や`@key`）をそのまま使えるので、最終出力でPandocを使うワークフローとも矛盾しません。

![プレビューのデモ](/images/markdown-academic-preview-demo.gif)

### セットアップ

VS Codeの拡張機能マーケットプレイスから「Markdown Academic Preview」をインストールします。

refのライブラリをデフォルトの参照先として設定するには、VS Codeの設定画面を開き（`Ctrl+,` / `Cmd+,`）、検索バーに`markdownAcademicPreview`と入力します。「Default Bibliography」の項目にrefのライブラリファイルのパス（例: `~/ref-library/references.json`）(AI Agentに「`ref config get library`でライブラリの場所を教えて」と聞くのがオススメ)を追加してください。

これだけで、すべてのMarkdownファイルでrefのライブラリが自動的に読み込まれます。Markdownファイルの先頭に`bibliography:`を書く必要すらありません。

もちろん、プロジェクトごとに個別の文献ファイルを指定したい場合は、ファイルの先頭で指定することもできます。

```markdown
---
bibliography: ./project-specific-refs.bib
csl: vancouver.csl
---
```

### 引用のプレビュー

Pandocの引用記法がプレビューで正しくレンダリングされます。

```markdown
先行研究では[@tanaka2023]、この手法が有効であることが示されている。
Yamadaら[-@yamada2022]も同様の結果を報告した。
```

### 引用のポップオーバー

プレビュー上の引用だけでなく、エディタ上で`@citationKey`にマウスを重ねたときにも書誌情報がポップアップ表示されます。文献の詳細を確認するために参考文献リストまでスクロールする必要がありません。

![引用のポップオーバー](/images/markdown-academic-popover.gif)

### 引用キーの補完

エディタ上で`@`を入力すると、ライブラリに含まれる文献の候補が表示されます。著者名・年・タイトルが表示されるので、citation keyを暗記しておく必要はありません。

![引用キーの補完](/images/markdown-academic-autocomplete.gif)

引用キーを全く覚えていない場合は、コマンドパレット（`Ctrl+Shift+P` / `Cmd+Shift+P`）から「Markdown Academic: Insert Citation」を実行すれば、著者名やタイトルで文献を検索して挿入することもできます。

![コマンドパレットからの引用挿入](/images/markdown-academic-insert-citation.gif)

### 相互参照

[pandoc-crossref](https://github.com/lierdakil/pandoc-crossref)の記法にも対応しています。図表や数式に番号を付けて参照できます。

```markdown
![](figure1.png)
: 実験結果の概要 {#fig:results}

@fig:resultsに示すように、処理群で有意な改善が見られた。
```

### 参考文献リストと上付き・下付き文字

文書内で引用された文献の参考文献リストを自動生成します。CSLスタイルを指定すれば、APA、Vancouver、IEEEなど任意のスタイルに切り替えられます。

Pandocの記法で`H~2~O`がH₂Oに、`x^2^`がx²にレンダリングされるので、化学式や数式の簡単な表記にも対応しています。

## AI統合された執筆フローの全体像

ここまでのツールを組み合わせると、以下のようなフローが実現します。

```
AIが文献を検索        (search-hub / Claude Codeから直接検索)
    ↓
AIがライブラリに登録   (ref add)
    ↓
AIと協働で執筆        (Markdown + VS Code)
    ↓                  引用は ref のライブラリを直接参照
プレビューで確認       (Markdown Academic Preview)
    ↓
最終出力              (Pandoc → Word / PDF)
```

従来のフローでは、ChatGPT等で検索した後、手動でWebを確認し、文献管理ソフトに手動で取り込み、Wordで執筆し、Wordのプラグインで引用を挿入していました。

AI統合フローでは、AI agentが検索・登録を行い、Markdownで執筆し（AIが直接編集できる）、`@`でライブラリから補完し、プレビューで確認するだけです。

従来のフローにあった「手作業の壁」が、すべて無くなっています。

## 最終出力もAIが助けてくれる

「プレビューはわかったけど、最終的にWordやPDFが必要な場面はどうするの？」という疑問があると思います。ここではPandocが必要になります。

```bash
pandoc paper.md -o paper.docx --citeproc --bibliography references.json
```

このコマンドで、Markdownの原稿がWordファイルに変換されます。引用も参考文献リストも自動生成されます。

Pandocのインストールや実行に不安がある方もいるかもしれませんが、ここは今のAIが得意とするところです。Claude CodeのようなAI agentに「Pandocを使って、この原稿をWordに変換して」と頼めば、Pandocのインストールからコマンドの実行まで面倒を見てくれます。CLIの操作を自分で覚える必要はありません。

## まとめ

[reference-manager](https://github.com/ncukondo/reference-manager)で文献管理をAIに任せ、[research-hub](https://github.com/ncukondo/research-hub)で系統的文献検索をAIと協働で行えるようになりました。そして[Markdown Academic Preview](https://github.com/ncukondo/vscode-markdown-academic-preview)により、Markdownでの執筆中にPandocなしで引用や参考文献リストをプレビューできるようになりました。

これで、文献の検索から執筆まで、一貫してAIと統合された環境が整いました。

Word＋文献管理ソフトのワークフローが悪いわけではありません。指導者や学術誌からはWordでの提出を求められることの方が多いと思います。ただ、AIと協働して研究を進めたいと考えたとき、テキストベースの環境にはGUIベースの環境にはない柔軟性があります。そして、AIを使えば、その移行のハードルも下げることができます。
