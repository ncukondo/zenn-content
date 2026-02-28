---
title: "AI agentに文献管理をさせるために必要だったのは、良いCLIだった"
emoji: "📚"
type: "tech"
topics: ["cli", "mcp", "ai", "claude", "論文管理"]
published: true
---

## はじめに

研究者の文献管理ワークフローは、「文献を検索→取り込み→閲覧→論文に統合」という流れになっています。このワークフロー全体をAI agentにサポートしてもらえたら、と考えたのが開発のきっかけでした。

Endnote、Zotero、Mendeleyといった既存の文献管理ソフトには、共通する制約があります。**GUIでの操作を前提としている**ことです。人間がマウスでクリックして操作する設計になっているため、AI agentが介入する余地がありません。

「この論文に関連する文献を探して、ライブラリに追加して、APAスタイルで引用を生成して」

こう頼めるAI agentがほしいと思いました。そのためには、AI agentが操作できるインターフェースを持つ文献管理ツールが必要でした。ということで[Claude Code](https://code.claude.com/)を使って作りました。

https://github.com/ncukondo/reference-manager

## reference-manager(refコマンド)の設計

[reference-manager](https://github.com/ncukondo/reference-manager)(refコマンド)は、CSL-JSONを単一のデータソースとする文献管理CLIツールです。

設計上の重要な判断として、**CSL-JSON形式の単一ファイル**をデータの唯一の保存先にしました。CSL-JSONは[Pandoc](https://pandoc.org/)が直接読める標準フォーマットなので、論文執筆パイプラインにそのまま組み込めます。データベースもサーバーも不要で、JSONファイル1つですべてが完結します。

インターフェースとしてCLIとMCPを用意し、すべての操作をプログラムから実行可能にしました。

## 当初の構想：MCPサーバー

AI agentとの連携を実現するために、当初は[MCP（Model Context Protocol）](https://modelcontextprotocol.io/)サーバーをメイン機能として開発しました。MCPはAI agentに構造化されたツールを提供できるプロトコルで、search、add、cite、fulltext等のツールをMCPサーバーとして公開しました。

Claude DesktopやClaude CodeからMCP経由で文献を操作できました。

しかし、MCPの動作確認のために用意していた**CLI機能**が、想定外の主役になることになります。

## 転機：AI agentはCLIを上手に使える

MCPの開発と並行して、Claude Codeが`gh` CLIやBashコマンドを使いこなす様子を日常的に見ていました。AIは`gh pr create`や`git log`を適切に組み合わせて作業を進めます。

ふと思い立って、Claude Codeに「`ref`コマンドを使って文献を登録して」と頼んでみました。

**それだけで動いた。**

![Claude Codeがrefコマンドで文献を登録する様子](/images/ref-add-demo.gif)

MCPサーバーのセットアップすら不要でした。Claude Codeは`ref --help`を読み、`ref add`でPMIDやDOIから文献を登録し、`ref search`で検索し、`ref cite`で引用を生成しました。MCPの型定義された美しいインターフェースと同じことが、CLIだけで実現できたのです。

## Skills：複合ワークフローのラッパー

CLIだけでも基本操作はできますが、実際の研究ワークフローでは複数のステップを組み合わせた操作が頻繁に発生します。

例えば：「文献を登録して、フルテキストも取得して、取得できなければURLを教えて」

このような複合操作は、Claude CodeのSkillsとして定義しておくと便利です。Skillsは、AI agentに対して「このタスクはこの手順で実行して」と自然言語で指示するテンプレートです。

MCPツールが「1つの操作」を提供するのに対し、Skillsは「複数のCLIコマンドの組み合わせによるワークフロー」を定義できます。PMIDやDOIを含まないURLから文献を登録したいときや、登録と同時にフルテキスト取得まで行いたいときなど、定型的だが複数ステップにまたがる操作をSkillでまとめています。

例えば、以下は実際に使っている「文献検索・登録」Skillです。`.claude/skills/find-paper/SKILL.md`として配置すると、Claude Codeで`/find-paper [URL / キーワード / PMID / DOI]`と呼び出すだけで、論文の特定から登録、フルテキスト取得までを自動で実行してくれます。

:::details Skill定義の全文（SKILL.md）
```markdown
---
name: find-paper
description: 学術文献の検索・登録・フルテキスト入手を行う。記事URL、キーワード、PMID、PubMed URL、DOIなどの入力から関連する査読済み論文を特定し、refコマンドで登録してフルテキストを取得する。
argument-hint: "[URL / キーワード / PMID / DOI]"
---

# 文献検索・登録スキル

ユーザーの入力 `$ARGUMENTS` から学術文献を特定し、`ref` コマンドで登録、フルテキストの取得まで自動で行う。

## 入力の判定

まず入力の種類を判定する：

1. **PMID** — 数字のみ（例: `41015033`）
2. **PubMed URL** — `pubmed.ncbi.nlm.nih.gov` を含むURL → PMIDを抽出
3. **PMC URL** — `pmc.ncbi.nlm.nih.gov` を含むURL → PMCIDを抽出
4. **DOI** — `10.` で始まる文字列、または `doi.org/` を含むURL
5. **arXiv** — `arxiv.org` を含むURL、または `arXiv:` で始まるID
6. **一般URL** — 上記以外のURL（ブログ記事、ニュース記事など）
7. **キーワード** — URL・数字・DOIのいずれでもないテキスト

## 処理フロー

### パターン A: PMID / DOI / arXiv ID が判明している場合

直接登録に進む（→「登録と取得」セクション）。

### パターン B: 一般URL（ブログ記事等）の場合

1. WebFetchでURLの内容を取得し、引用されている論文・DOI・参考文献を抽出する
2. 取得できない場合（403等）はWebSearchで記事タイトル・内容を検索する
3. 記事内で引用されている論文、または記事のテーマに最も関連する査読済み論文をWebSearchでPubMed・arXiv等から検索する
4. 候補論文を特定し、パターンAに進む

### パターン C: キーワードの場合

1. WebSearchで `{キーワード} site:pubmed.ncbi.nlm.nih.gov OR site:arxiv.org` を検索する
2. 必要に応じて追加の検索クエリを試す（同義語、英語変換など）
3. 候補論文を特定し、パターンAに進む

### 候補が複数ある場合

最も関連性の高い論文を最大3件まで提示し、ユーザーに登録対象を確認する。
明らかに1件のみが該当する場合は確認なしで登録に進む。

## 登録と取得

### ステップ1: ref add で登録

識別子の種類に応じてコマンドを実行する：

ref add --fetch-fulltext {PMID}
ref add --fetch-fulltext {DOI}
ref add --fetch-fulltext {arXiv_ID}

`--fetch-fulltext` を付けて、登録と同時にOAフルテキストの自動取得を試みる。

コマンドの出力からcitation keyを記録する。

### ステップ2: フルテキスト取得の確認

`ref add --fetch-fulltext` でフルテキストが取得できなかった場合：

1. `ref fulltext discover {citation_key}` でOA利用可能性を確認する
2. 利用可能なソースがあれば `ref fulltext fetch {citation_key}` を実行する
3. それでも取得できない場合は、以下のURLをユーザーに提示する：
   - DOIがある場合: `https://doi.org/{DOI}`
   - PMCIDがある場合: `https://pmc.ncbi.nlm.nih.gov/articles/{PMCID}/`
   - PubMedがある場合: `https://pubmed.ncbi.nlm.nih.gov/{PMID}/`

## 出力フォーマット

### 論文情報を表形式で出力

| 項目 | 詳細 |
|------|------|
| **タイトル** | （論文タイトル） |
| **著者** | （全著者名） |
| **雑誌** | （ジャーナル名） |
| **年** | （出版年） |
| **DOI** | （DOI） |
| **PMID** | （PubMed ID、あれば） |
| **Citation Key** | （ref addで割り当てられたkey） |

### アブストラクトの要約

日本語で3-5文程度に要約する。

### 登録・取得結果

- ref addの結果（成功/失敗、重複検出など）
- フルテキスト取得の結果
  - 自動取得できた場合: 「フルテキストを取得しました」
  - 手動取得が必要な場合: アクセス用URLを提示

## 注意事項

- 入力が一般URLの場合、元記事が学術論文でない場合はその旨を明記する
- `ref add` が重複を検出した場合はユーザーに通知し、`--force` での上書き登録をするか確認する
```
:::

ポイントは、Skill定義がすべて自然言語で書かれていることです。入力の判定ロジック、フォールバック処理、ユーザーへの出力フォーマットまで、プログラミングなしで定義できます。AI agentはこの指示を読んで、`ref add`や`ref fulltext fetch`等のCLIコマンドを適切に組み合わせて実行します。

MCPはClaude Desktopのようなターミナルのない環境では依然として有用です。ただ、Claude Codeのようなターミナルベースの環境では、Skills + CLIの方がセットアップ不要で柔軟性も高いです。

## AI agentに使いやすいCLIとは

Skills + CLIで文献管理をする中で、**AI agentにとって使いやすいCLIとは何か**について、いくつかの知見を得ました。

### コマンド名を慣習に揃える

AI agentは一般的なCLIでよくあるコマンド名を試そうとします。一覧を見たいときは`list`、追加は`add`、検索は`search`。特別な名前を発明する必要はありません。AI agentも人間も、`list`と言えば一覧が出ることを期待します。

### helpメッセージに具体例を入れる

AI agentはツールを使う前に`--help`を読みます。ここに具体例があるかどうかで、正しいコマンドを組み立てられるかが大きく変わります。

```
EXAMPLES
  $ ref search "machine learning"
  $ ref search author:Smith year:2020
  $ ref search author:"John Smith" title:introduction
  $ ref search tag:review --sort published --order desc
```

オプションの一覧だけでなく、実際のユースケースに基づいた例を入れることで、AI agentは初回から正しいコマンドを実行できます。

### エラーメッセージでnext stepを示す

これが最も大きな学びでした。

`ref fulltext fetch`は、OA（オープンアクセス）のフルテキストを自動取得するコマンドです。内部では複数のプロバイダ（PubMed Central、Unpaywall等）を順に試し、見つかればダウンロードします。

**当初のエラーメッセージ**は単純でした。

```
Error: Failed to download fulltext
```

このメッセージを受け取ったAI agentは、プロバイダを個別に指定して再試行したり、再度URLを探しに行ったりと、**すでに失敗した操作を形を変えて繰り返しました**。`ref fulltext fetch`はすでにすべてのプロバイダを試しているのに、です。

**改善後のエラーメッセージ**は、何を試し、何がスキップされ、次に何ができるかを含めるようにしました。

```
Error: No OA sources found for ali-2024
  Checked: pmc
  Skipped: arxiv (no arXiv ID available), unpaywall (unpaywallEmail not configured),
  core (coreApiKey not configured)
  Hint: open to download manually:
  https://doi.org/10.1111/eje.12937
  https://pubmed.ncbi.nlm.nih.gov/37550893/
```

AI agentの行動が明確に変わりました。

![改善後のエラーメッセージを受けて、AI agentが設定の問題を指摘し、手動ダウンロードも提案する様子](/images/fulltext-error-nextstep.gif)

- エラーメッセージからUnpaywallのメール未設定に気づき、**設定を促した**
- 同時に、**手動でのダウンロードもオプションとして提示した**
- すでに全プロバイダを試したことを理解し、**無駄な再試行をしなくなった**

**AI agentにとってのエラーメッセージは、人間にとってのそれと同じです。** 「何を試して、なぜ失敗し、次に何をすべきか」がわかれば、適切な行動を取れます。ただ「失敗した」とだけ伝えると、闇雲に再試行します。

## まとめ

AI agentに文献管理をさせるために必要だったのは、高度なプロトコル実装ではなく、**良いCLIだった**。

- 慣習に沿ったコマンド名
- 具体例を含むhelpメッセージ
- next stepにつながるエラーメッセージ

これらはAI agentのためだけの設計ではありません。人間にとっても使いやすいCLIの条件そのものです。AI agentは、良いCLIをそのまま使えます。特別なインターフェースを用意しなくても。

MCPが不要だとは思いません。ターミナルのない環境（Claude Desktop等）では依然として有用です。ただ、AI agentとの連携を考えるなら、まず**良いCLIを作ること**が最もコストパフォーマンスの高いアプローチだと感じています。

---

reference-managerの詳細な使い方やインストール方法は、GitHubリポジトリを参照してください。

https://github.com/ncukondo/reference-manager
