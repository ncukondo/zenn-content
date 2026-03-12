---
title: "AI agentのためのUX設計"
emoji: "🔍"
type: "tech"
topics: ["cli", "ai", "claude", "ux", "論文検索"]
published: true
---

## はじめに：MCPからCLIへ、そしてその先

[前回の記事](https://zenn.dev/ncukondo/articles/ai-agent-reference-management-cli)では、文献管理CLIツール[reference-manager](https://github.com/ncukondo/reference-manager)の開発を通じて、「AI agentに仕事をさせるために必要だったのは、良いCLIだった」というお話しをしました。当初はMCPサーバーを主軸に開発していましたが、実際にはCLIの方がAI agentとの相性が良かったという経験です。

この流れは個人的な発見ではなく、業界的なトレンドであるようです。MicrosoftのPlaywrightも、[MCP版](https://github.com/anthropics/anthropic-quickstarts/tree/main/mcp-server-playwright)から[CLI版](https://github.com/microsoft/playwright-cli)への移行を進めています。その理由として、CLIはMCPに比べてトークン効率が良く、大きなツールスキーマやアクセシビリティツリーをコンテキストに読み込む必要がないことを挙げています。

> CLI invocations are more token-efficient: they avoid loading large tool schemas and verbose accessibility trees into the model context.

MCPが不要というわけではありません。ターミナルのない環境や、持続的なブラウザセッションを扱う特化型エージェントには依然として有用です。しかし、コーディングエージェントが大規模なコードベースと推論を限られたコンテキストウィンドウの中でやりくりする場面では、CLIの方が適しているという認識が広がっています。

前回のreference-managerでは、慣習に沿ったコマンド名、具体例を含むhelpメッセージ、next stepを示すエラーメッセージという3つの原則でAI agentはCLIを使いこなせるようになりました。

しかし、次に開発した[research-hub](https://github.com/ncukondo/research-hub)では、その3原則だけでは足りませんでした。

research-hubは、系統的文献検索を行うCLIツールです。PubMed・Scopus・ERIC・arXivといった複数の学術データベースを横断的に検索し、scoping reviewやsystematic reviewに必要な再現可能な検索プロセスを支援します。reference-managerとの連携により、検索から引用管理までを一気通貫で行えます。

https://github.com/ncukondo/research-hub

reference-managerでの文献管理は、基本的に単発のコマンド実行で完結します。`ref add`で登録し、`ref search`で検索する。一つひとつの操作は独立しています。

一方、系統的文献検索は多段階の自律的判断が必要です。

1. 検索クエリを作成する
2. MeSH用語などの制御語彙をバリデーションする
3. 各データベースへの翻訳結果を確認する
4. ヒット数を確認し、多すぎれば修正する
5. プレビューでサンプルタイトルを確認する
6. 本検索を実行する
7. 結果を精査し、必要ならクエリを修正して繰り返す

AI agentにこのワークフロー全体を任せるには、単に「良いCLI」を作るだけでは不十分でした。**AIにとってのUXを意識的に設計する**必要がありました。

## 原則：AIは出力から次の行動を決める

AI agentの行動パターンは単純です。コマンドを実行し、出力を読み、次のコマンドを決める。このループの繰り返しです。

ここで問題になる事があります。ほとんどの場合、**AI agentはドキュメントを読みません。** READMEも読まず、`--help`すら読まずに、いきなりコマンドを試すことも頻繁にあります。

実は、これは人間も同じです。新しいツールを渡されたとき、マニュアルを最初から読む人はほとんどいません。触って、試して、結果を見て覚える。この点ではAIも人間も同じ「マニュアルを読まないユーザー」です。

しかし決定的な違いがあります。人間はUIの視覚的手がかりから次の操作を推測できます。ボタンの配置、メニューの階層構造、グレーアウトされた項目といった手がかりが「次に何ができるか」を暗黙に伝えます。GUIであればウィザードやプログレスバーが「今どこにいて、次はどこに行くか」を示してくれます。

AIにはそれがありません。**テキスト出力がUIの全て**です。

だからこそ、CLIの全ての出力、成功時も、エラー時も、途中経過も、が「次に何をすべきか」を含む必要がありました。

## 設計1：成功時に次のステップを明示する

前回の記事では「エラーメッセージにnext stepを」と書きました。search-hubではこれを**成功時にも拡張**しました。

### suggestionsシステム

search-hubには25以上のルールからなるsuggestionsシステムがあります。コマンドの実行結果とワークフローの現在地に応じて、次に実行すべきコマンドを動的に提案します。

例えば、`search --count-only`（ヒット数のみ確認）が成功した後の出力です。

```
Query: diabetes-ai.yaml (count only)

  pubmed:        12,450 hits
  scopus:         8,234 hits
  ────────────────────────
  total:         20,684 hits (before deduplication)

Next:
  search-hub search ./query.yaml --preview     # Preview sample titles
  search-hub search ./query.yaml               # Execute full search

See also:
  search-hub query translate ./query.yaml      # Inspect translated queries
```

ヒット数の報告だけで終わらず、「次はプレビューか本実行ですよ」「クエリの翻訳結果も確認できますよ」と伝えます。

### ワークフロー位置に応じた動的提案

suggestionsは固定ではありません。ワークフローの現在地によって内容が変わります。検索前ならクエリ検証を、検索後なら結果分析を、レビュー中ならスクリーニングの続行を提案します。

レビューでバッチ処理をしている場合は、残り件数と正確な継続コマンドまで提示します。

```
105 articles remaining — extract next batch:
  search-hub review extract --session <id> --offset 100 --limit 50
```

### 人間のUXとの対比

人間向けUIでは、ウィザードやプログレスバーが「現在地と次のステップ」を示します。search-hubのsuggestionsシステムは、そのCLI版と言えます。

ただし、決定的な違いがあります。人間はワークフローを一度覚えれば、次回からは案内なしで操作できます。しかしAI agentはセッションごとにコンテキストが初期化されるため、**常に案内し続ける必要があります**。人間向けUIなら「初回だけ表示するチュートリアル」で済むところを、AIには「毎回表示するナビゲーション」として設計する必要があるのです。

search-hubのメインヘルプにも、ワークフロー全体の地図を表示するようにしています。

```
Workflow:
  1. query init → edit → validate / --dry-run        Query preparation
  2. search --preview → search                       Preview & execute
  3. results / summary / diff / check                Inspect & verify
  4. review init → extract → merge → status          Systematic review
  5. register / export                               Output
```

## 設計2：エラーメッセージで代替手段を示す

前回の記事で、reference-managerの`ref fulltext fetch`のエラーメッセージを改善した話を書きました。「何を試し、なぜ失敗し、次に何ができるか」を含めることで、AI agentが闇雲な再試行をしなくなったという経験です。

search-hubではこの原則をさらに徹底しました。

### 全プロバイダ失敗時

```
All providers failed:
  pubmed: Connection timeout
  scopus: API rate limit exceeded

Suggested actions:
  → Run with --dry-run to inspect translated queries
  → Check provider configuration: search-hub config
  → Use --db <provider> to test a single provider
```

「失敗しました」ではなく、「どのプロバイダが・なぜ失敗し・次に何ができるか」を列挙します。dry-runでクエリ翻訳だけ確認する、設定を見直す、プロバイダを個別にテストするという3つの代替手段を提示します。

### 不正なプロバイダ名

```
Invalid provider(s): pubMed. Valid: pubmed, eric, arxiv, scopus
```

AIが大文字小文字を間違えた場合も、正しい選択肢の全リストを表示します。

### 人間のUXとの対比

人間はエラーに遭遇すると、「ググってみよう」「設定ファイルを探してみよう」と自力で推論します。ドキュメントを読みに行ったり、Stack Overflowで検索したりできます。

AI agentは違います。**提示された選択肢の中から最適なものを選ぶのは非常に得意ですが、提示されていない選択肢を自力で発見するのは苦手です。** だからエラーメッセージには「何が失敗したか」だけでなく、**「何が選択肢として残っているか」を列挙すること**が重要になります。

人間向けのエラーメッセージが「何が起きたか」の説明を重視するのに対し、AI向けのエラーメッセージは「次に何ができるか」のリストを重視します。同じ「良いエラーメッセージ」でも、力点が異なるのです。

## 設計3：AIが陥りがちなミスを仕組みで防ぐ

AI agentも人間もミスをします。しかし、**ミスの性質が根本的に異なります。**

人間のミスは主にタイポや操作ミスです。`pubmed`を`pubMed`と打つ、オプションの順番を間違える、といった類のものです。

AIのミスは「もっともらしいが存在しないものを自信を持って入力する」、いわゆるハルシネーションです。この違いに応じたバリデーションが必要でした。

### MeSHバリデーション：AIの幻覚を仕組みで防ぐ

系統的文献検索では、MeSH（Medical Subject Headings）などの制御語彙を使って検索精度を高めます。問題は、AI agentがMeSH用語を「もっともらしく生成する」ことです。存在しない用語を自信を持って入力してきます。

`query validate`コマンドは、NLM（米国国立医学図書館）のAPIに問い合わせて用語の実在性を検証します。


```
Controlled vocabulary:
  ✓ mesh: "Diabetes Mellitus"
  ✗ mesh: "Diabetus" — not found
    Did you mean: "Diabetes Mellitus Type 1", "Diabetes Mellitus Type 2"
  ✗ mesh: "AI in Medicine" — not found
    Did you mean: "Artificial Intelligence", "Medicine"
```

単に「無効です」と返すのではなく、Levenshtein距離やプレフィックスマッチを使って**候補を提示**します。AIは選択肢を提示されれば、正しいものを選べます。

人間向けのバリデーションは入力形式のチェック（メールアドレスの形式、数値の範囲など）が中心です。しかしAI向けには、**入力内容の実在性チェック**がより重要になります。形式的には完璧だが内容が存在しないというのが、AI特有のエラーパターンです。

### 短いキーワード警告

AIは略語を好んで使う傾向があります。しかし1-2文字のキーワードは学術データベースで大量の偽陽性を生みます。

```
⚠ Query contains short keywords: AI, ML
  Short terms may match unrelated acronyms. Consider:
  - Adding full phrases (e.g., "Artificial Intelligence")
  - Using exclude terms to filter false matches
```

問題を指摘するだけでなく、具体的な対処法を示します。

### 段階的実行：取り返しのつかないミスを防ぐ

search-hubの検索は3段階で実行できます。

1. `--count-only`：ヒット数のみ確認（API負荷最小）
2. `--preview`：サンプルタイトルを数件確認
3. オプションなし：本検索を実行

この設計には人間のUXとの興味深い対比があります。

人間向けUIでは「元に戻す（Undo）」が安全策の基本です。操作後に取り消せれば、気軽に試せます。しかしAPIリクエストを伴う検索は取り消せません。数万件のAPIリクエストを発行した後に「やっぱり違った」では遅いのです。

AI agentは試行錯誤を通じて学びますが、取り返しのつかない試行は困ります。段階的実行は、**Undoの代わりに「コミット前のプレビュー」を提供する**設計です。gitの`git diff --staged`から`git commit`への流れに近い発想と言えます。

## 設計4：AIが触れるファイルにヒントを埋め込む

AI agentがCLIの出力だけでなく、ファイルを直接読み書きする場面があります。search-hubでは、AIが触れるファイル自体にもヒントを埋め込みました。

### YAMLテンプレートのコメント

`query init`コマンドが生成するクエリテンプレートには、各フィールドの意味と有効な値をコメントとして記載しています。

```yaml
# yaml-language-server: $schema=./query.schema.json
name: my_search

query:
  - id: concept-1             # Unique block identifier
    field: title_abstract     # title, abstract, title_abstract, author, keyword, all
    terms:
      keywords:
        - "search term 1"
      # mesh:                 # PubMed MeSH terms (optional)
      #   - "MeSH Heading"
      # eric:                 # ERIC Descriptors (optional, ERIC only)
      #   - "ERIC Descriptor"
      exclude: []             # Terms to exclude (NOT operator)
      # Tip: Use exclude to filter out false matches from short keywords/acronyms
```

AI agentはこのファイルを読んだ時点で、フィールドの選択肢（`title`, `abstract`, `title_abstract`...）やオプション項目の書き方を把握できます。ドキュメントを読みに行く必要がありません。

### 人間のUXとの対比

テンプレートやscaffoldにコメントを入れる手法は、人間向けでも定番です。`.env.example`やフレームワークが生成する設定ファイルでよく見られます。この点では、AIにとって有効な手法は人間にとっても有効です。

ただし、微妙な違いがあります。人間はファイルの「周囲の文脈」から情報を読み取れます。ディレクトリ名、隣にあるファイル、プロジェクト全体の構造といった暗黙の手がかりから情報を読み取れます。

AIは明示的に読み込んだものしか見えません。だからこそ、**ファイルの中に自己完結した情報を持たせる**ことがより重要になります。周囲のファイルを見なくても、そのファイルだけで何をすべきかわかる状態が理想です。

## 人間のUXとAIのUX：同じ原則、異なる力点

ここまで4つの設計判断を見てきました。ここで、人間のUXとAIのUXの関係を整理します。

### 共通する原則

どちらも「マニュアルを読まない」前提で設計すべきです。直感的な命名、段階的開示、明確なフィードバックといった原則は、ドナルド・ノーマンが『誰のためのデザイン？』で述べたデザイン原則そのものです。AI agentのために発明された新しい概念ではありません。

実際、search-hubで実装した設計の多くは、人間にとっても使いやすいCLIの条件そのものです。慣習に沿ったコマンド名、具体例を含むhelp、代替手段を示すエラーメッセージ。これらは良いCLI設計の教科書に載っている内容です。

### 異なる力点1：持続性

人間は学習します。一度覚えたワークフローは、次回からは案内なしで実行できます。初回はウィザードが必要でも、慣れればショートカットで操作します。

AI agentはセッションごとにコンテキストが初期化されます。前回のセッションで学んだことは、次回には持ち越されません。だから案内は「初回向けのチュートリアル」ではなく、「**毎回表示するナビゲーション**」として設計する必要があります。

### 異なる力点2：エラーの性質

人間のエラーはタイポや操作ミスが中心です。AIのエラーはハルシネーション、つまり形式的には正しいが内容が存在しないものの生成が特徴的です。

防ぐべきミスの種類が異なるため、バリデーションの設計も変わります。人間向けの入力形式チェックに加え、AI向けには**内容の実在性チェック**（MeSHバリデーションのような）が重要になります。

### 異なる力点3：文脈の獲得

人間は視覚的・空間的にUIを把握します。画面全体を一望し、ボタンの位置関係やメニュー構造から操作の全体像を掴めます。

AIはテキスト出力のみが情報源です。**テキストに載せない情報は、AIにとって存在しません。** だからこそ、コマンドの出力に「現在地」と「次の行き先」を毎回含める必要があります。

### 異なる力点4：選択肢の発見

人間はメニューを開いたり、タブ補完を試したり、類似コマンドを推測したりして、未知の選択肢を探索的に発見できます。

AI agentは**提示された選択肢から最適なものを選ぶのは得意ですが、提示されていない選択肢を発見するのは苦手**です。エラーメッセージやsuggestionsに選択肢を列挙することの重要性は、ここにあります。

### まとめると

AI agentのためのUX設計は、「良いUXの原則を新たに発明すること」ではありません。**既存の良い設計原則を、暗黙知に頼らず、愚直に徹底すること**です。人間なら文脈から補完できることを、すべて明示する。人間なら一度で学べることを、毎回伝え直す。人間なら自力で探索できる選択肢を、すべて列挙する。

人間のUXにおいて「あったら親切」なものが、AIのUXでは「ないと機能しない」ものになります。これが両者の関係です。

## 実践：AI agentとの協働

これらの設計判断が実際にどう機能するか、scoping reviewのワークフローを例に示します。

以下は、Claude Codeで使っているSkill定義の一部です。AI agentに「このテーマでscoping reviewの検索をして」と依頼すると、search-hubのコマンドを組み合わせてワークフローを自律的に進めます。

```
1. query init でテンプレート生成
2. YAMLを編集してクエリ作成（テンプレートのコメントがガイドになる）
3. query validate でMeSH用語を検証（ハルシネーション防止）
4. search --count-only でヒット数確認
5. 多すぎれば修正して再実行
6. search --preview でサンプル確認
7. search で本検索
8. results -q で結果をフィルタリング
9. check で既知論文のカバレッジ確認
10. register で reference-manager に登録
```

各ステップで表示されるsuggestionsが、AI agentを次のステップへ自然に導きます。AI agentはワークフロー全体を事前に知らなくても、**出力に従うだけで正しい手順を踏める**のです。

reference-managerとの連携も、`register`コマンド一つで実現できます。search-hubで見つけた文献をreference-managerのライブラリに一括登録し、そこから先は`ref cite`で引用を生成できます。検索から執筆までが一つの流れになります。

## まとめ

前回のreference-managerで得た3原則、すなわち慣習に沿ったコマンド名、具体例を含むhelp、next stepを示すエラーメッセージは、AI agentのためのCLI設計の出発点でした。

search-hubの開発では、複雑なワークフローを扱う中で、さらに4つの設計指針を得ました。

1. **成功時にも次のステップを明示する**：全てのコマンド出力をナビゲーションにする
2. **エラー時に代替手段を列挙する**：何が失敗したかではなく、何ができるかを示す
3. **AIのミスを仕組みで防ぐ**：ハルシネーションには実在性チェック、段階的実行で取り返しのつかないミスを回避
4. **ファイルにヒントを埋め込む**：AIが触れるものすべてに自己完結した情報を持たせる

これらは「AI専用の特殊な設計」ではありません。良いUXの原則を、暗黙知に頼らず徹底したものです。人間のUXにおいて「あったら親切」なものが、AIのUXでは「ないと機能しない」ものになります。この違いを意識することが、AI agentと協働するツールを設計する鍵だと感じています。

---

research-hubの詳細な使い方やインストール方法は、GitHubリポジトリを参照してください。

https://github.com/ncukondo/research-hub
