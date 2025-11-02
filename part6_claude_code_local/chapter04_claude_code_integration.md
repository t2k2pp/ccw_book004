# Chapter 04: Claude Code統合

## 4.1 Claude Code CLIのインストール

### 4.1.1 Node.jsのインストール

Claude Code CLIはNode.js製のツールです。まずNode.jsをインストールします。

**ステップ1: Node.jsバージョン確認**

```bash
# Node.jsがインストールされているか確認
node --version

# npm（パッケージマネージャー）の確認
npm --version
```

**Node.js 18以上が必要です。** インストールされていないか古い場合は以下の手順でインストールします。

**ステップ2: NodeSource経由でインストール（推奨）**

```bash
# Node.js 20 LTSをインストール
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# インストール確認
node --version  # v20.x.x
npm --version   # 10.x.x
```

### 4.1.2 Claude Code CLIのインストール

**重要な注意事項**:
現時点（2025年1月）で、公式のClaude Code CLIはまだリリースされていません。本書では、提供されたZennやMediumの記事に基づき、**カスタム設定によるClaude Code利用**または**類似ツールの使用方法**を解説します。

**方法1: Cursor / Continue.devを使用（推奨）**

Claude CodeライクなAIコーディングアシスタントとして、以下のツールが利用できます：

```bash
# Continue.dev（VSCode拡張機能）のインストール
# VSCodeを開き、拡張機能から"Continue"を検索してインストール

# または、Cursorエディタをダウンロード
wget https://downloader.cursor.sh/linux/appImage/x64 -O cursor.AppImage
chmod +x cursor.AppImage
sudo mv cursor.AppImage /usr/local/bin/cursor
```

**方法2: OpenAI互換CLIツールの使用**

```bash
# LiteLLMのCLIモードを直接使用
pip install openai

# Pythonスクリプトでローカル

LLMを使用
```

### 4.1.3 代替方法: Aider（AIペアプログラミングツール）

Claude Codeと同様の機能を持つ**Aider**を使用する方法を推奨します。

```bash
# Aiderをインストール
pip install aider-chat

# インストール確認
aider --version

# 出力例: aider 0.60.0
```

**Aiderの特徴**
- Git統合
- ファイル編集機能
- OpenAI互換API対応
- ローカルLLM対応
- 複数ファイル操作
- コンテキスト管理

## 4.2 Aiderの設定（Claude Code代替）

### 4.2.1 環境変数の設定

AiderをLiteLLMプロキシ経由でローカルLLMに接続します。

```bash
# ~/.bashrcまたは~/.zshrcに追加
nano ~/.bashrc
```

以下を追加：

```bash
# LiteLLM API設定
export OPENAI_API_BASE="http://localhost:8000/v1"
export OPENAI_API_KEY="sk-local-dev-1234"  # LiteLLMのmaster_key
export OPENAI_API_MODEL="claude-3-5-sonnet-20241022"  # 使用するモデル
```

設定を反映：

```bash
source ~/.bashrc

# 確認
echo $OPENAI_API_BASE
echo $OPENAI_API_KEY
echo $OPENAI_API_MODEL
```

### 4.2.2 Aider設定ファイルの作成

**【必須】Aider専用の設定ファイルを作成します。**

```bash
# ホームディレクトリに設定ファイルを作成
nano ~/.aider.conf.yml
```

**基本設定（コピペ用）**

```yaml
# ~/.aider.conf.yml
# 【基本構成】MS-S1 Max + Qwen3 Coder 30B Q8_0向け

# APIエンドポイント（必須）
openai-api-base: http://localhost:8000/v1
openai-api-key: sk-local-dev-1234

# モデル設定（必須）
model: claude-3-5-sonnet-20241022

# コンテキスト設定（重要）
map-tokens: 4096
max-chat-history-tokens: 8192

# 機能設定（オプション）
auto-commits: true
dirty-commits: true
git: true
stream: true

# エディタ設定（オプション）
editor: nano

# ログ設定（オプション）
verbose: false
show-diffs: true
```

**📖 各設定項目の詳細解説**

**1. API接続設定（必須）**

```yaml
openai-api-base: http://localhost:8000/v1
```

**💡 `openai-api-base` とは？**
- **目的**: LiteLLMプロキシのAPIエンドポイント
- **デフォルト**: `http://localhost:8000/v1`
- **あなたの環境**:
  - **ローカル**: そのまま使用（変更不要）
  - **別マシンのLiteLLM**: `http://<IPアドレス>:8000/v1`
  - **ポート変更時**: `http://localhost:<別ポート>/v1`
- **重要**: 末尾の `/v1` を忘れないこと

```yaml
openai-api-key: sk-local-dev-1234
```

**💡 `openai-api-key` とは？**
- **目的**: LiteLLM認証キー
- **あなたの環境**: LiteLLMの`config.yaml`で設定した`master_key`と**同じ値**を使用
- **例**:
  - LiteLLMで `master_key: sk-1234` なら、ここも `sk-1234`
  - LiteLLMで `master_key: sk-local-dev-1234` なら、ここも `sk-local-dev-1234`
- **確認方法**: `cat ~/litellm/config.yaml | grep master_key`

**2. モデル設定（必須）**

```yaml
model: claude-3-5-sonnet-20241022
```

**💡 `model` パラメータ**
- **目的**: 使用するモデルを指定
- **デフォルト**: `claude-3-5-sonnet-20241022`
- **あなたの環境**:
  - **MS-S1 Max（推奨）**: `claude-3-5-sonnet-20241022`（Qwen3 30B Q8_0にマップ）
  - **速度重視**: `gpt-3.5-turbo`（Qwen3 14Bにマップ）
  - **最速**: `claude-3-haiku-20240307`（Qwen3 7Bにマップ）
- **対応関係**: LiteLLMの`config.yaml`で定義したモデル名を使用

```
【モデル選択の目安】
claude-3-5-sonnet-20241022  ← 最高品質（22 tokens/s）
       ↓                        用途: 本番コード、レビュー
gpt-3.5-turbo               ← 高速（28 tokens/s）
       ↓                        用途: 通常開発、実験
claude-3-haiku-20240307     ← 最速（42 tokens/s）
                                用途: クイック質問、学習
```

**3. コンテキスト設定（重要・パフォーマンスに影響）**

```yaml
map-tokens: 4096
```

**💡 `map-tokens` パラメータ**
- **目的**: コードベースマップ（ファイル一覧・構造）に使うトークン数
- **デフォルト**: 1024
- **あなたの環境**:
  ```yaml
  map-tokens: 2048   # ← 小規模プロジェクト（10ファイル未満）
  map-tokens: 4096   # ← 推奨・中規模（10〜50ファイル）
  map-tokens: 8192   # ← 大規模プロジェクト（50ファイル以上）
  map-tokens: 0      # ← マップ無効化（最軽量）
  ```
- **影響**:
  - 大きい: プロジェクト全体を理解しやすいがメモリ消費増
  - 小さい: メモリ節約だが全体把握が困難
- **MS-S1 Max推奨**: `8192`（余裕あり）

```yaml
max-chat-history-tokens: 8192
```

**💡 `max-chat-history-tokens` パラメータ**
- **目的**: 会話履歴に使うトークン数
- **デフォルト**: 2048
- **あなたの環境**:
  ```yaml
  max-chat-history-tokens: 4096   # ← 短期会話（5〜10往復）
  max-chat-history-tokens: 8192   # ← 推奨・通常（10〜20往復）
  max-chat-history-tokens: 16384  # ← 長期会話（20往復以上）
  max-chat-history-tokens: 32768  # ← 非常に長い（MS-S1 Max向け）
  ```
- **影響**:
  - 大きい: 長い会話の文脈を保持、メモリ消費増
  - 小さい: メモリ節約だが過去の会話を忘れやすい
- **MS-S1 Max推奨**: `16384`（256Kコンテキストを活かす）

**4. Git統合設定（オプション・便利）**

```yaml
auto-commits: true
```

**💡 `auto-commits` パラメータ**
- **目的**: コード変更時に自動でGitコミット
- **デフォルト**: false
- **あなたの環境**:
  - `true`: **推奨**・変更を自動記録、履歴管理が楽
  - `false`: 手動コミット、自分でタイミング制御
- **メリット**: 変更履歴が自動で残る、ロールバック簡単
- **デメリット**: コミットが増える（gitログが多い）

```yaml
dirty-commits: true
```

**💡 `dirty-commits` パラメータ**
- **目的**: 未コミットの変更がある状態でもコミット可能
- **デフォルト**: false
- **あなたの環境**:
  - `true`: **推奨**・柔軟に作業可能
  - `false`: クリーンな状態のみコミット（厳格）
- **推奨**: `auto-commits: true`なら`dirty-commits: true`も有効化

```yaml
git: true
```

**💡 `git` パラメータ**
- **目的**: Git統合機能を有効化
- **デフォルト**: true
- **あなたの環境**:
  - `true`: **推奨**・Git機能を使う
  - `false`: Git不使用（Gitリポジトリでない場合）
- **前提**: プロジェクトが`git init`済みであること

**5. ユーザー体験設定（オプション）**

```yaml
stream: true
```

**💡 `stream` パラメータ**
- **目的**: レスポンスをリアルタイム表示
- **デフォルト**: true
- **あなたの環境**:
  - `true`: **推奨**・タイプライター風表示、待ち時間短く感じる
  - `false`: 完全生成後に一括表示
- **推奨**: `true`（ユーザー体験向上）

```yaml
editor: nano
```

**💡 `editor` パラメータ**
- **目的**: `/editor`コマンドで使うエディタ
- **デフォルト**: システムのデフォルト
- **あなたの環境**:
  - `nano`: 初心者向け（簡単）
  - `vim`: Vimユーザー向け
  - `code`: VSCode使用者向け
  - `emacs`: Emacsユーザー向け
- **変更方法**: 好みのエディタ名を指定

```yaml
verbose: false
show-diffs: true
```

**💡 `verbose` / `show-diffs` パラメータ**
- **verbose**: デバッグ情報を表示
  - `false`: **推奨**・通常使用
  - `true`: トラブルシューティング時のみ
- **show-diffs**: 変更前後の差分を表示
  - `true`: **推奨**・何が変わったか確認しやすい
  - `false`: 差分非表示（シンプル）

**🔧 環境別のカスタマイズ例**

**ケース1: MS-S1 Max向け最適化（推奨）**

```yaml
# 大容量メモリを活かした設定
openai-api-base: http://localhost:8000/v1
openai-api-key: sk-local-dev-1234
model: claude-3-5-sonnet-20241022

# コンテキストを大きく
map-tokens: 8192                    # ← 大規模プロジェクト対応
max-chat-history-tokens: 16384      # ← 長い会話対応

# Git自動化
auto-commits: true
dirty-commits: true
git: true
stream: true
editor: nano
verbose: false
show-diffs: true
```

**ケース2: メモリ節約型（64GB以下）**

```yaml
# 軽量設定
openai-api-base: http://localhost:8000/v1
openai-api-key: sk-local-dev-1234
model: gpt-3.5-turbo                # ← 軽量モデル

# コンテキストを抑える
map-tokens: 2048                    # ← 小規模対応
max-chat-history-tokens: 4096       # ← 短期会話

# 基本機能のみ
auto-commits: true
git: true
stream: true
```

**ケース3: 速度最優先**

```yaml
# 高速設定
openai-api-base: http://localhost:8000/v1
openai-api-key: sk-local-dev-1234
model: claude-3-haiku-20240307      # ← 最速モデル

# コンテキスト最小
map-tokens: 0                       # ← マップ無効化
max-chat-history-tokens: 2048       # ← 最小

auto-commits: false                 # ← 手動制御で高速化
git: true
stream: true
```

**❓ よくある質問**

**Q: すべての設定を書く必要がありますか？**
A: いいえ。`openai-api-base`、`openai-api-key`、`model`の3つのみ必須です。他は省略するとデフォルト値が使われます。

**Q: 設定を間違えたらどうなりますか？**
A: Aider起動時にエラーメッセージが表示されます。typoに注意してください。

**Q: 後から設定を変更できますか？**
A: はい。`~/.aider.conf.yml`を編集し、Aiderを再起動すれば反映されます。

**Q: 起動時にオプションで上書きできますか？**
A: はい。例: `aider --model gpt-3.5-turbo --map-tokens 2048`で一時的に変更可能です。

**Q: どの設定が自分に必要か判断できない**
A: **MS-S1 Maxなら上記の「ケース1: 最適化」をそのままコピペしてください。**他の環境なら、まず基本設定で試してから調整してください。

### 4.2.3 動作確認

**ステップ1: Aiderを起動**

```bash
# プロジェクトディレクトリに移動
mkdir -p ~/test-project
cd ~/test-project

# Gitリポジトリを初期化
git init
git config user.name "Your Name"
git config user.email "[email protected]"

# Aiderを起動
aider
```

**起動時のログ例**
```
Aider v0.60.0
Model: claude-3-5-sonnet-20241022 using http://localhost:8000/v1
Git repo: /home/user/test-project
Repository map: disabled (no files)

Use /help to see available commands.

>
```

**ステップ2: 簡単なテスト**

```
> /add test.py

> Create a Python script that prints "Hello, Local LLM!"

# Aiderがコードを生成
# test.pyが作成される
```

**ステップ3: 確認**

```bash
# 生成されたファイルを確認
cat test.py

# 実行
python3 test.py

# 出力: Hello, Local LLM!
```

## 4.3 詳細な使用方法

### 4.3.1 ファイル操作

**ファイルをコンテキストに追加**

```
> /add main.py utils.py

# 複数ファイルを一度に追加
> /add src/*.py

# ディレクトリ全体を追加
> /add src/
```

**ファイルをコンテキストから削除**

```
> /drop utils.py
```

**現在のコンテキストを確認**

```
> /ls

# 出力例:
# Files in chat:
#   main.py (125 lines)
#   utils.py (45 lines)
```

### 4.3.2 コード生成・編集

**新しいファイルを作成**

```
> Create a FastAPI server with a /hello endpoint that returns "Hello, World!"

# Aiderがmain.pyを生成
```

**既存ファイルを編集**

```
> /add main.py

> Add error handling to the /hello endpoint

# Aiderがmain.pyを編集
```

**差分を確認**

```
> /diff

# または自動で表示される（show-diffs: true）
```

### 4.3.3 Git統合

**変更をコミット**

```
> /commit

# メッセージを入力
Commit message: Add error handling to hello endpoint
```

**自動コミットを有効化**

```yaml
# ~/.aider.conf.ymlで設定済み
auto-commits: true
dirty-commits: true
```

有効化すると、Aiderが変更を加えるたびに自動でコミットされます。

**コミット履歴を確認**

```bash
git log --oneline

# 出力例:
# a1b2c3d Add error handling to hello endpoint
# d4e5f6g Create initial FastAPI server
```

### 4.3.4 高度な機能

**複数ファイルの一括編集**

```
> /add src/*.py

> Refactor all functions to use type hints

# すべてのPythonファイルにタイプヒントを追加
```

**コマンド実行**

```
> /run python3 main.py

# スクリプトを実行して結果を確認
```

**コードレビュー**

```
> /add main.py

> Review this code and suggest improvements

# Aiderがコードレビューを提供
```

**バグ修正**

```
> /add buggy_function.py

> This function crashes when input is empty. Fix the bug.

# Aiderがバグを特定して修正
```

### 4.3.5 モデルの切り替え

複数のモデルを使い分けることができます。

```
# 軽量モデルに切り替え（高速）
> /model gpt-3.5-turbo

# 高品質モデルに切り替え
> /model claude-3-5-sonnet-20241022

# または起動時に指定
aider --model gpt-3.5-turbo
```

**config.yamlで定義したモデルマッピング（2025年11月最新）**
- `gpt-3.5-turbo` → `qwen3-coder:14b`（高速・256K context）
- `claude-3-5-sonnet-20241022` → `qwen3-coder:30b-a3b-q8_0`（最高品質・256K context）
- `gpt-4` → `qwen3-coder:30b-a3b-q8_0`（最高品質・256K context）
- `claude-3-haiku-20240307` → `qwen3-coder:7b`（最速・256K context）

## 4.4 実践例

### 4.4.1 WebアプリケーションRestructure

**シナリオ**: 既存のFlaskアプリをFastAPIにリファクタリング

```bash
# プロジェクトディレクトリに移動
cd ~/my-flask-app

# Aiderを起動
aider

# ファイルをコンテキストに追加
> /add app.py

# リファクタリング依頼
> Convert this Flask application to FastAPI. Keep the same endpoints and functionality.

# Aiderがコードを変換
# 差分を確認
> /diff

# コミット
> /commit
# Commit message: Refactor Flask to FastAPI
```

**生成されるコード例**

**Before (Flask):**
```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/hello')
def hello():
    return jsonify({"message": "Hello, World!"})

if __name__ == '__main__':
    app.run(debug=True)
```

**After (FastAPI):**
```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class HelloResponse(BaseModel):
    message: str

@app.get('/hello', response_model=HelloResponse)
def hello():
    return {"message": "Hello, World!"}

# Run with: uvicorn main:app --reload
```

### 4.4.2 テスト生成

```bash
> /add main.py

> Generate pytest tests for all functions in this file

# Aiderがtest_main.pyを生成
```

**生成されるテスト例**

```python
# test_main.py
import pytest
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_hello_endpoint():
    response = client.get('/hello')
    assert response.status_code == 200
    assert response.json() == {"message": "Hello, World!"}

def test_hello_response_schema():
    response = client.get('/hello')
    assert "message" in response.json()
    assert isinstance(response.json()["message"], str)
```

**テスト実行**

```bash
> /run pytest test_main.py -v

# Aiderが実行結果を表示
```

### 4.4.3 ドキュメント生成

```bash
> /add src/*.py

> Generate a README.md file that documents all modules, functions, and their usage

# README.mdが生成される
```

## 4.5 パフォーマンス最適化

### 4.5.1 コンテキストサイズの調整

MS-S1 Maxの大容量メモリを活かして、大きなコンテキストを使用できます。

```yaml
# ~/.aider.conf.yml

# デフォルト（標準）
map-tokens: 4096
max-chat-history-tokens: 8192

# MS-S1 Max向け最適化（大容量）
map-tokens: 8192
max-chat-history-tokens: 16384

# 注意: コンテキストを大きくすると応答時間が長くなる
```

### 4.5.2 ストリーミングの活用

```yaml
# ~/.aider.conf.yml
stream: true  # リアルタイムでレスポンスを表示
```

ストリーミングを有効にすると、生成中のコードをリアルタイムで確認できます。

### 4.5.3 キャッシュの活用

LiteLLM側でRedisキャッシュを有効にすると、同じ質問への応答が高速化されます。

```bash
# Redisインストール（まだの場合）
sudo apt install redis-server -y

# Redis起動
sudo systemctl start redis-server

# config.yamlでキャッシュを有効化（Chapter 03参照）
```

**キャッシュのメリット**
- 同じコードレビュー: 即座に応答
- 繰り返しのリファクタリング: 高速化
- ドキュメント生成: 初回のみ時間がかかる

## 4.6 トラブルシューティング

### 4.6.1 接続エラー

**エラー: "Could not connect to API endpoint"**

```bash
# LiteLLMプロキシが起動しているか確認
curl http://localhost:8000/health

# 起動していなければ起動
cd ~/litellm
source venv/bin/activate
litellm --config config.yaml
```

**エラー: "Unauthorized"**

```bash
# APIキーを確認
echo $OPENAI_API_KEY

# config.yamlのmaster_keyと一致するか確認
```

### 4.6.2 モデルエラー

**エラー: "Model not found"**

```bash
# モデル一覧を確認
curl http://localhost:8000/v1/models \
  -H "Authorization: Bearer sk-local-dev-1234"

# モデル名が正しいか確認
echo $OPENAI_API_MODEL

# ~/.aider.conf.ymlのmodel:設定を確認
```

### 4.6.3 パフォーマンス問題

**応答が遅い**

```bash
# Ollamaの状態を確認
ollama ps

# GPUが使われているか確認
rocm-smi

# GPU使用率が0%の場合、ROCm環境変数を確認
sudo systemctl status ollama
```

**メモリ不足**

```bash
# システムメモリ確認
free -h

# より軽量なモデルに切り替え（MS-S1 Maxでは通常不要）
> /model gpt-3.5-turbo  # qwen3-coder:14b (256K context)
> /model claude-3-haiku-20240307  # qwen3-coder:7b (最軽量)
```

## 4.7 Continue.dev（VSCode拡張）の設定

Claude Code代替として、VSCodeの**Continue**拡張機能もおすすめです。

### 4.7.1 Continueのインストール

```bash
# VSCodeを起動
code

# 拡張機能で"Continue"を検索してインストール
# または
code --install-extension continue.continue
```

### 4.7.2 Continueの設定

Ctrl+Shift+P → "Continue: Open config.json"

```json
{
  "models": [
    {
      "title": "Qwen3 Coder 30B Q8_0 (Best - 256K ctx)",
      "provider": "openai",
      "model": "claude-3-5-sonnet-20241022",
      "apiBase": "http://localhost:8000/v1",
      "apiKey": "sk-local-dev-1234"
    },
    {
      "title": "Qwen3 Coder 14B (Fast - 256K ctx)",
      "provider": "openai",
      "model": "gpt-3.5-turbo",
      "apiBase": "http://localhost:8000/v1",
      "apiKey": "sk-local-dev-1234"
    },
    {
      "title": "Qwen3 Coder 7B (Fastest - 256K ctx)",
      "provider": "openai",
      "model": "claude-3-haiku-20240307",
      "apiBase": "http://localhost:8000/v1",
      "apiKey": "sk-local-dev-1234"
    }
  ],
  "tabAutocompleteModel": {
    "title": "Tab Autocomplete",
    "provider": "openai",
    "model": "gpt-3.5-turbo",
    "apiBase": "http://localhost:8000/v1",
    "apiKey": "sk-local-dev-1234"
  },
  "embeddingsProvider": {
    "provider": "ollama",
    "model": "mxbai-embed-large",
    "apiBase": "http://localhost:11434"
  }
}
```

### 4.7.3 Continueの使用

**1. チャット機能**
- Ctrl+L: チャットを開く
- コードを選択 → Ctrl+L: 選択範囲について質問

**2. インラインエディット**
- Ctrl+I: インライン編集
- 「この関数にエラーハンドリングを追加」と入力

**3. タブ補完**
- コードを書き始めると自動で提案
- Tab: 提案を受け入れ

## 4.8 まとめ

本章では、ローカルLLMをClaude Code風のツールで使用する方法を学びました。

**達成したこと**
✅ Aiderのインストールと設定
✅ LiteLLMプロキシへの接続
✅ 基本的なコード生成・編集
✅ Git統合
✅ Continue.dev（VSCode）の設定

**実現できること**
- AIペアプログラミング
- コードレビュー
- リファクタリング
- テスト生成
- ドキュメント作成
- バグ修正

**MS-S1 Maxでの優位性（2025年11月最新）**
- 128GBメモリ（96GB VRAM割当可能） → 大容量コンテキスト（256K tokens）
- Radeon 8060S（40 RDNA 3.5 CU） → 高速推論（22 tokens/s）
- Qwen3 Coder 30B Q8_0 → 最高品質のコード生成
- 完全ローカル → プライバシー保護
- コスト$0 → 無制限に使用可能

**次のステップ**
次章では、ローカルLLMとクラウドLLMの互換性と制限事項について詳しく解説します。何ができて何ができないのか、どう回避するかを学びます。

**確認チェックリスト**
- [ ] Aiderが起動する
- [ ] 環境変数が正しく設定されている
- [ ] ローカルLLMに接続できる
- [ ] コード生成が動作する
- [ ] Git統合が動作する
- [ ] Continue.dev（オプション）が設定されている

すべてチェックできたら、Chapter 05へ進みましょう！
