# AWS Bedrock AgentCore ハンズオン

## テーマパーク案内エージェントを作ろう！

AgentCore Browser（Web自動閲覧）× AgentCore Memory（会話記憶）を GitHub Codespaces 上で1時間で体験するハンズオン資料です。

```
あなた「今日の開園時間は？」
         ↓
  AgentCore Browser が公式サイトを自動閲覧
         ↓
  「本日の開園時間は○○です」（公式サイトより）
         ↓
  会話を重ねると文脈を引き継いでくれる（Memory）
```

---

## 体験する機能

| 機能 | 内容 |
|------|------|
| **AgentCore Browser** | エージェントが Web サイトを自律的に閲覧して最新情報を取得 |
| **AgentCore Memory** | 会話履歴をクラウドに保存し、セッション内で文脈を引き継ぐ |

---

## ⚠️ 注意事項

本ハンズオンでは AgentCore Browser を使用して外部 Web サイトに自動アクセスします。

- 本ハンズオンは**学習目的**であり、取得した情報を外部に公開・再配布しないでください
- 対象サイトへの過度なアクセス（短時間での大量リクエスト等）は避けてください
- 各サイトの利用規約を確認の上、自己責任でご利用ください

> **パーク選択肢について**
> 当初は USJ・東京ディズニーリゾート・ポケパーク カントーを候補としていましたが、利用規約・robots.txt の調査により以下の理由で差し替えています。
> - **東京ディズニーリゾート**: 利用規約でコンテンツの自動取得・再配布を明確に禁止
> - **USJ**: 利用規約で「software robots, spider, crawlers」等の自動データ収集ツールの使用を明確に禁止
> - **ポケパーク カントー**: robots.txt で ClaudeBot・GPTBot 等の主要 AI クローラーを全面ブロック

---

## 事前に必要なもの

- **AWS アカウント**（コンソールにログインできる状態）
- **GitHub アカウント**

> Python・AWS CLI のインストールは不要です。GitHub Codespaces を使用するため、ブラウザさえあれば手元の PC に何もインストールせずに参加できます。

---

## ファイル構成

```
.
├── .devcontainer/
│   └── devcontainer.json   # Codespaces 環境定義（Python 3.11 + AWS CLI）
├── agent.py                # エージェント本体（冒頭2行を書き換えて使用）
├── setup_memory.py         # AgentCore Memory のセットアップ
├── cleanup.py              # リソースのクリーンアップ
└── README.md               # このファイル
```

---

## タイムライン

| 時間 | 内容 |
|------|------|
| 00〜10分 | Codespaces 起動 & AWS 認証 |
| 10〜15分 | パッケージインストール & Bedrock 確認 |
| 15〜25分 | パーク選択 & `agent.py` 書き換え |
| 25〜45分 | 動作確認（Browser 機能） |
| 45〜55分 | AgentCore Memory セットアップ & 体験 |
| 55〜60分 | クリーンアップ |

---

## 手順

### 準備①：GitHub Codespaces を起動する

以下のハンズオン用リポジトリを開きます：

👉 **[https://github.com/yuuka51/AmazonBedrockAgentCoreHandsOn202603](https://github.com/yuuka51/AmazonBedrockAgentCoreHandsOn202603)**

ページ上部の **「Code」→「Codespaces」→「Create codespace on main」** をクリックします。

> 初回起動は 1〜2 分かかります。ブラウザ上に VSCode のような画面が表示されたら起動完了です。

起動後、画面下部のターミナルを開きます（表示されていない場合は `Ctrl + `` `）。

以下でバージョンを確認してください：

```bash
python3 --version   # → Python 3.11.x 以上
aws --version       # → aws-cli/2.32.0 以上
```

---

### 準備②：AWS にログインする

Codespaces はリモート環境のため、`aws login --remote` を使ってブラウザ経由で認証します。

```bash
aws login --remote
```

**リージョンを聞かれた場合は `us-west-2` を入力してください：**

```
AWS Region [us-east-1]: us-west-2
```

その後、以下のような出力が表示されます：

```
Open the following URL in your browser:
https://signin.us-west-2.amazonaws.com/authorize?...

Enter the authorization code:
```

1. 表示された URL をコピーしてブラウザの新しいタブで開く
2. AWS マネジメントコンソールと同じユーザー名・パスワードでログイン
3. ブラウザに表示された**認証コード**をターミナルに貼り付けて Enter

ログインを確認します：

```bash
aws sts get-caller-identity
```

アカウント情報が返ってきたら OK です。

> 取得した認証情報は 15 分ごとに自動ローテーションされます。セッションは最大 12 時間有効なので、ハンズオン中に再ログインを求められることはありません。

---

### 準備③：Bedrock モデルアクセスの確認

エージェントが使用する Claude Sonnet 4.5 が利用可能であることを確認します。

1. [AWS コンソール](https://console.aws.amazon.com/)を開く
2. 右上のリージョンが **「米国西部（オレゴン）us-west-2」** になっていることを確認
3. 検索バーで「Bedrock」を開き、左メニュー「モデルカタログ」から **Claude Sonnet 4.5** が表示されることを確認

> 2026年3月現在、Bedrock のサーバーレス基盤モデルは初回呼び出し時に自動的に有効化されるため、手動でのモデルアクセス有効化は不要です。ただし Anthropic モデルの初回利用時にユースケース情報の送信が求められる場合があります。

---

### 準備④：パッケージのインストール

Codespaces は Python 環境が整っているため、仮想環境の作成は不要です。

```bash
pip install bedrock-agentcore strands-agents strands-agents-tools playwright boto3 nest_asyncio "botocore[crt]"
playwright install chromium
```

> `chromium` のダウンロードが完了したら準備完了です（1〜2 分かかります）。

---

### Step 1：パークを選んで `agent.py` を書き換える

#### パーク選択肢

| 選択肢 | パーク名 | 公式サイト |
|--------|---------|-----------|
| **A** | サンリオピューロランド | https://www.puroland.jp/ |
| **B** | 富士急ハイランド | https://www.fuji-q.com/ |
| **C** | 花やしき | https://www.hanayashiki.net/ |

#### `agent.py` の冒頭 2 行を書き換える

ファイルエクスプローラーから `agent.py` を開き、冒頭の 2 行だけ自分が選んだパークに書き換えて `Ctrl + S` で保存します：

```python
# ======================================================
# 書き換えるのはこの2行だけです！
# ======================================================
PARK_NAME = "サンリオピューロランド"   # ← 自分のパーク名に変更
PARK_URL  = "https://www.puroland.jp/"  # ← 自分のパークURLに変更
# ======================================================
```

**例：富士急ハイランドを選んだ場合**

```python
PARK_NAME = "富士急ハイランド"
PARK_URL  = "https://www.fuji-q.com/"
```

---

### Step 2：エージェントを起動して動作確認する

#### エージェントを起動

```bash
python agent.py
```

以下のように表示されたら起動成功です：

```
INFO:     Started server process [xxxxx]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8080 (Press CTRL+C to quit)
```

#### 別ターミナルから質問する

ターミナル右上の **「＋」または「Split Terminal」アイコン** をクリックして新しいターミナルを開き、以下を実行します。

**質問①：開園・閉園時間**

```bash
curl -s -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "今日の開園時間と閉園時間を教えてください"}' \
  | python3 -c "import sys,json;print(json.dumps(json.load(sys.stdin),ensure_ascii=False,indent=2))"
```

> ⏳ **1〜2分かかります。** エージェントが公式サイトを実際に閲覧しているためです。

**質問②：持ち込みについて**

```bash
curl -s -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "食べ物・飲み物の持ち込みはできますか？"}' \
  | python3 -c "import sys,json;print(json.dumps(json.load(sys.stdin),ensure_ascii=False,indent=2))"
```

**質問③：注意事項**

```bash
curl -s -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "入場時の注意事項を教えてください"}' \
  | python3 -c "import sys,json;print(json.dumps(json.load(sys.stdin),ensure_ascii=False,indent=2))"
```

> ✅ 「（〇〇公式サイトより）」という出典付きで回答が返ってきたら Browser 機能の動作確認完了です。

---

### Step 3：AgentCore Memory を体験する

#### Memory リソースのセットアップ

まずエージェントを `Ctrl + C` で停止してから実行します：

```bash
python setup_memory.py
```

完了すると以下のように表示されます：

```
✅ IAMロールを作成しました
✅ Bedrockアクセス権限を付与しました
✅ セットアップ完了！

export MEMORY_ID=ParkGuideMemory-xxxxxxxxxx
```

表示された `export` コマンドをコピーして実行します：

```bash
export MEMORY_ID=<表示されたMemory ID>
```

**コンソールで作成確認：** AWSコンソール「Bedrock」→「AgentCore」→「Memory」→ `ParkGuideMemory` が **Active** になっていることを確認してください。

#### エージェントを再起動

```bash
python agent.py
```

#### Memory 付きで質問する

**別ターミナルから** `sessionId` を指定して質問します（同じ `sessionId` で複数回質問するのがポイントです）：

**質問④：開園時間を聞く**

```bash
curl -s -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "今日の開園時間を教えてください", "sessionId": "my-session-01"}' \
  | python3 -c "import sys,json;print(json.dumps(json.load(sys.stdin),ensure_ascii=False,indent=2))"
```

**質問⑤：これまでの会話をまとめてもらう**（同じ `sessionId` で）

```bash
curl -s -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "これまでに聞いたことをまとめてください", "sessionId": "my-session-01"}' \
  | python3 -c "import sys,json;print(json.dumps(json.load(sys.stdin),ensure_ascii=False,indent=2))"
```

> ✅ **確認ポイント**: Browserツールを呼ばずに（ログに `Tool #N: browser` が出ずに）前の会話内容を踏まえた回答が返ってきたら Memory 機能の動作確認完了です。2回目以降は Memory から即答するため数秒〜10秒程度で返ってきます。

**コンソールでメトリクス確認：** 「AgentCore」→「Memory」→ `ParkGuideMemory` →「イベント」タブで会話の記録、「記憶」タブで自動抽出されたサマリーを確認できます。

---

### クリーンアップ

ハンズオン終了後は以下を実行してリソースを削除してください。

```bash
# 1. エージェントを停止（起動中のターミナルで）
Ctrl + C

# 2. AgentCore Memory リソースと IAM ロールを削除
python cleanup.py
```

**コンソールで削除確認：** 「AgentCore」→「Memory」を開き、`ParkGuideMemory` が一覧から消えていることを確認してください。

> **利用コストの目安**: Bedrock 推論（Claude Sonnet 4.5）、AgentCore Browser、AgentCore Memory の利用で、1 人あたり数十円〜数百円程度の従量課金が発生します。正確な料金は [Bedrock 料金ページ](https://aws.amazon.com/bedrock/pricing/) および [AgentCore 料金ページ](https://aws.amazon.com/bedrock/agentcore/pricing/) をご確認ください。

---

## トラブルシューティング

### `ModuleNotFoundError` が出る

```bash
pip install bedrock-agentcore strands-agents strands-agents-tools playwright boto3 nest_asyncio "botocore[crt]"
```

### Codespaces が起動しない・画面が固まる

ブラウザをリロードしてください。それでも解決しない場合は GitHub の Codespaces 一覧から該当の Codespace を選択して「Resume」をクリックしてください。

### `aws login --remote` でエラーが出る

```bash
aws --version   # 2.32.0 以上であることを確認
```

バージョンが古い場合は Codespace を一度削除して再作成してください（devcontainer が再実行され最新版がインストールされます）。

### `aws login --remote` で認証コードが通らない

URL をブラウザで開いた際、すでにログインしているアカウントで認証が行われます。意図したアカウントと異なる場合は、一度コンソールからサインアウトしてから再度 URL を開いてください。

### `AccessDeniedException` が出る

```bash
# 認証状態を確認
aws sts get-caller-identity

# リージョンを確認
aws configure get region   # → us-west-2 であること

# IAMユーザーの権限を確認
aws iam list-attached-user-policies --user-name <あなたのユーザー名>
```

アカウント情報が返らない場合は `aws login --remote` を再実行してください。

### Bedrock のモデルアクセスエラーが出る

AWS コンソールで「Amazon Bedrock」→「モデルカタログ」→「Claude Sonnet 4.5」を選択し、画面の指示に従ってユースケース情報を送信してください。

### レスポンスが返ってこない（2 分以上）

- エージェント起動側のターミナルにエラーが出ていないか確認する
- エージェントを `Ctrl + C` で停止し、再起動してから再試行する

---

## 余裕があれば試してほしいこと

- `agent.py` の `SYSTEM_PROMPT` を書き換えて、エージェントの回答スタイルを変えてみる
- `sessionId` を変えて新しいセッションを開始し、Memory の動作を比較してみる
- `agentcore deploy` でクラウドにデプロイして、どこからでも呼び出せるエージェントにする

---

## 参考リンク

- [AgentCore 公式ドキュメント](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/)
- [AgentCore サンプルコード集](https://github.com/awslabs/amazon-bedrock-agentcore-samples)
- [Strands Agents ドキュメント](https://strandsagents.com/)
- [aws login コマンドの紹介（AWS Blog）](https://aws.amazon.com/jp/blogs/news/simplified-developer-access-to-aws-with-aws-login/)

---

*最終更新: 2026年3月 | AWS Bedrock AgentCore GA版（2025年10月〜）| 実行環境: GitHub Codespaces*
