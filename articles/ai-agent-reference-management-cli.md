---
title: "AI agentに文献管理をさせるために必要だったのは、良いCLIだった"
emoji: "📚"
type: "tech"
topics: ["cli", "mcp", "ai", "claude", "論文管理"]
published: true
---

## はじめに

研究者の文献管理ワークフローは、「文献を検索→取り込み→閲覧→論文に統合」という流れになっている。このワークフロー全体をAI agentにサポートしてもらえたら、と考えたのが開発のきっかけだった。

Endnote、Zotero、Mendeleyといった既存の文献管理ソフトには、共通する制約がある。**GUIでの操作を前提としている**ことだ。人間がマウスでクリックして操作する設計になっているため、AI agentが介入する余地がない。

「この論文に関連する文献を探して、ライブラリに追加して、APAスタイルで引用を生成して」

こう頼めるAI agentがほしかった。そのためには、AI agentが操作できるインターフェースを持つ文献管理ツールが必要だった。ということで作った。

https://github.com/ncukondo/reference-manager

## reference-manager(refコマンド)の設計

[reference-manager](https://github.com/ncukondo/reference-manager)(refコマンド)は、CSL-JSONを単一のデータソースとする文献管理CLIツールだ。

設計上の重要な判断として、**CSL-JSON形式の単一ファイル**をデータの唯一の保存先にした。CSL-JSONは[Pandoc](https://pandoc.org/)が直接読める標準フォーマットなので、論文執筆パイプラインにそのまま組み込める。データベースもサーバーも不要で、JSONファイル1つですべてが完結する。

インターフェースとしてCLIとMCPを用意し、すべての操作をプログラムから実行可能にした。

## 当初の構想：MCPサーバー

AI agentとの連携を実現するために、当初は[MCP（Model Context Protocol）](https://modelcontextprotocol.io/)サーバーをメイン機能として開発した。MCPはAI agentに構造化されたツールを提供できるプロトコルで、search、add、cite、fulltext等のツールをMCPサーバーとして公開した。

Claude DesktopやClaude CodeからMCP経由で文献を操作できた。

しかし、MCPの動作確認のために用意していた**CLI機能**が、想定外の主役になることになる。

## 転機：AI agentはCLIを上手に使える

MCPの開発と並行して、Claude Codeが`gh` CLIやBashコマンドを使いこなす様子を日常的に見ていた。AIは`gh pr create`や`git log`を適切に組み合わせて作業を進める。

ふと思い立って、Claude Codeに「`ref`コマンドを使って文献を登録して」と頼んでみた。

**それだけで動いた。**

![Claude Codeがrefコマンドで文献を登録する様子](/images/ref-add-demo.gif)

MCPサーバーのセットアップすら不要だった。Claude Codeは`ref --help`を読み、`ref add`でPMIDやDOIから文献を登録し、`ref search`で検索し、`ref cite`で引用を生成した。MCPの型定義された美しいインターフェースと同じことが、CLIだけで実現できた。

## Skills：複合ワークフローのラッパー

CLIだけでも基本操作はできるが、実際の研究ワークフローでは複数のステップを組み合わせた操作が頻繁に発生する。

例えば：「文献を登録して、フルテキストも取得して、取得できなければURLを教えて」

このような複合操作は、Claude CodeのSkillsとして定義しておくと便利だ。Skillsは、AI agentに対して「このタスクはこの手順で実行して」と自然言語で指示するテンプレートだ。

MCPツールが「1つの操作」を提供するのに対し、Skillsは「複数のCLIコマンドの組み合わせによるワークフロー」を定義できる。PMIDやDOIを含まないURLから文献を登録したいときや、登録と同時にフルテキスト取得まで行いたいときなど、定型的だが複数ステップにまたがる操作をSkillでまとめている。

MCPはClaude Desktopのようなターミナルのない環境では依然として有用だ。ただ、Claude Codeのようなターミナルベースの環境では、Skills + CLIの方がセットアップ不要で柔軟性も高い。

## AI agentに使いやすいCLIとは

Skills + CLIで文献管理をする中で、**AI agentにとって使いやすいCLIとは何か**について、いくつかの知見を得た。

### コマンド名を慣習に揃える

AI agentは一般的なCLIでよくあるコマンド名を試そうとする。一覧を見たいときは`list`、追加は`add`、検索は`search`。特別な名前を発明する必要はない。AI agentも人間も、`list`と言えば一覧が出ることを期待する。

### helpメッセージに具体例を入れる

AI agentはツールを使う前に`--help`を読む。ここに具体例があるかどうかで、正しいコマンドを組み立てられるかが大きく変わる。

```
EXAMPLES
  $ ref search "machine learning"
  $ ref search author:Smith year:2020
  $ ref search author:"John Smith" title:introduction
  $ ref search tag:review --sort published --order desc
```

オプションの一覧だけでなく、実際のユースケースに基づいた例を入れることで、AI agentは初回から正しいコマンドを実行できる。

### エラーメッセージでnext stepを示す

これが最も大きな学びだった。

`ref fulltext fetch`は、OA（オープンアクセス）のフルテキストを自動取得するコマンドだ。内部では複数のプロバイダ（PubMed Central、Unpaywall等）を順に試し、見つかればダウンロードする。

**当初のエラーメッセージ**は単純だった。

```
Error: Failed to download fulltext
```

このメッセージを受け取ったAI agentは、プロバイダを個別に指定して再試行したり、再度URLを探しに行ったりと、**すでに失敗した操作を形を変えて繰り返した**。`ref fulltext fetch`はすでにすべてのプロバイダを試しているのに、だ。

**改善後のエラーメッセージ**は、何を試し、何がスキップされ、次に何ができるかを含めるようにした。

```
Error: No OA sources found for ali-2024
  Checked: pmc
  Skipped: arxiv (no arXiv ID available), unpaywall (unpaywallEmail not configured),
  core (coreApiKey not configured)
  Hint: open to download manually:
  https://doi.org/10.1111/eje.12937
  https://pubmed.ncbi.nlm.nih.gov/37550893/
```

AI agentの行動が明確に変わった。

![改善後のエラーメッセージを受けて、AI agentが設定の問題を指摘し、手動ダウンロードも提案する様子](/images/fulltext-error-nextstep.gif)

- エラーメッセージからUnpaywallのメール未設定に気づき、**設定を促した**
- 同時に、**手動でのダウンロードもオプションとして提示した**
- すでに全プロバイダを試したことを理解し、**無駄な再試行をしなくなった**

**AI agentにとってのエラーメッセージは、人間にとってのそれと同じだ。** 「何を試して、なぜ失敗し、次に何をすべきか」がわかれば、適切な行動を取れる。ただ「失敗した」とだけ伝えると、闇雲に再試行する。

## まとめ

AI agentに文献管理をさせるために必要だったのは、高度なプロトコル実装ではなく、**良いCLIだった**。

- 慣習に沿ったコマンド名
- 具体例を含むhelpメッセージ
- next stepにつながるエラーメッセージ

これらはAI agentのためだけの設計ではない。人間にとっても使いやすいCLIの条件そのものだ。AI agentは、良いCLIをそのまま使える。特別なインターフェースを用意しなくても。

MCPが不要だとは思わない。ターミナルのない環境（Claude Desktop等）では依然として有用だ。ただ、AI agentとの連携を考えるなら、まず**良いCLIを作ること**が最もコストパフォーマンスの高いアプローチだと感じている。

---

reference-managerの詳細な使い方やインストール方法は、GitHubリポジトリを参照してほしい。

https://github.com/ncukondo/reference-manager
