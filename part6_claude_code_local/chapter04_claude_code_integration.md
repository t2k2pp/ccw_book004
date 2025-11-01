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

Aider専用の設定ファイルを作成します。

```bash
# ホームディレクトリに設定ファイルを作成
nano ~/.aider.conf.yml
```

```yaml
# ~/.aider.conf.yml

# APIエンドポイント
openai-api-base: http://localhost:8000/v1
openai-api-key: sk-local-dev-1234

# モデル設定
model: claude-3-5-sonnet-20241022

# コンテキスト設定
map-tokens: 4096  # マップ用トークン数
max-chat-history-tokens: 8192  # チャット履歴

# 機能設定
auto-commits: true  # 自動コミット
dirty-commits: true  # 変更をコミット
git: true  # Git統合を有効化
stream: true  # ストリーミング有効化

# エディタ設定
editor: nano  # またはvim、code等

# ログ設定
verbose: false
show-diffs: true
```

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

**config.yamlで定義したモデルマッピング**
- `gpt-3.5-turbo` → `qwen2.5-coder:7b`（高速）
- `claude-3-5-sonnet-20241022` → `qwen2.5-coder:14b`（バランス）
- `gpt-4` → `qwen2.5-coder:14b`（高品質）

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

# より軽量なモデルに切り替え
> /model gpt-3.5-turbo  # qwen2.5-coder:7b
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
      "title": "Qwen2.5 Coder 14B (Local)",
      "provider": "openai",
      "model": "claude-3-5-sonnet-20241022",
      "apiBase": "http://localhost:8000/v1",
      "apiKey": "sk-local-dev-1234"
    },
    {
      "title": "Qwen2.5 Coder 7B (Fast)",
      "provider": "openai",
      "model": "gpt-3.5-turbo",
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

**MS-S1 Maxでの優位性**
- 128GBメモリ → 大きなコンテキスト
- Radeon 8060S → 高速推論（18 tokens/s）
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
