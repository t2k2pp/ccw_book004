# Chapter 03: LiteLLMのセットアップ

## 3.1 LiteLLMとは

### 3.1.1 LiteLLMの役割

LiteLLMは、様々なLLMプロバイダー（Ollama、OpenAI、Anthropic、Azure OpenAI等）を**統一インターフェース**で扱えるPythonライブラリです。

**なぜLiteLLMが必要か**

```
Claude Code → 期待するAPI形式: OpenAI互換
    ↓
Ollama → 提供するAPI形式: Ollama独自（一部OpenAI互換）
```

LiteLLMがこのギャップを埋めます：

```
Claude Code → [LiteLLM Proxy] → Ollama
                  ↓
              完全なOpenAI互換APIに変換
              + 追加機能（キャッシュ、ログ、フォールバック）
```

**主な機能**
1. **APIフォーマット変換**: OllamaのレスポンスをOpenAI形式に変換
2. **プロキシサーバー**: 独立したAPIサーバーとして動作
3. **ロードバランシング**: 複数モデルへのリクエスト分散
4. **キャッシング**: 同じリクエストの結果を再利用
5. **ログ記録**: すべてのリクエスト・レスポンスを記録
6. **フォールバック**: モデルが失敗したら別モデルで再試行

### 3.1.2 アーキテクチャ

```
┌──────────────────┐
│  Claude Code     │
└────────┬─────────┘
         │ HTTP POST /v1/chat/completions
         │ {model: "claude-3-5-sonnet", messages: [...]}
         ↓
┌──────────────────┐
│  LiteLLM Proxy   │
│  (Port 8000)     │
│                  │
│  - Model Mapping │ ← config.yaml
│  - Cache         │
│  - Logging       │
└────────┬─────────┘
         │ HTTP POST /api/chat
         │ {model: "qwen3-coder:30b-a3b-q8_0", messages: [...]}
         ↓
┌──────────────────┐
│  Ollama          │
│  (Port 11434)    │
└────────┬─────────┘
         │
         ↓
┌──────────────────────────┐
│  Qwen3 Coder 30B Q8_0   │
│  (MS-S1 Max: 96GB VRAM) │
└──────────────────────────┘
```

## 3.2 LiteLLMのインストール

### 3.2.1 Pythonバージョン確認

```bash
# Pythonバージョン確認（3.10以上が必要）
python3 --version

# 出力例: Python 3.11.6

# 3.10未満の場合はアップグレード
sudo apt install python3.11 python3.11-venv -y
```

### 3.2.2 仮想環境の作成

```bash
# 作業ディレクトリを作成
mkdir -p ~/litellm
cd ~/litellm

# 仮想環境を作成
python3 -m venv venv

# 仮想環境を有効化
source venv/bin/activate

# プロンプトが変わることを確認
# (venv) user@ms-s1-max:~/litellm$
```

### 3.2.3 LiteLLMのインストール

```bash
# pipをアップグレード
pip install --upgrade pip

# LiteLLMをインストール（プロキシ機能付き）
pip install 'litellm[proxy]'

# インストール確認
litellm --version

# 出力例: litellm 1.55.7
```

**インストール時間**: 約2-3分

**必要なディスク容量**: 約500MB

## 3.3 設定ファイルの作成

### 3.3.1 基本的な設定ファイル

LiteLLMはYAML形式の設定ファイルで動作を制御します。

```bash
# 設定ファイルを作成
nano ~/litellm/config.yaml
```

**最小構成（config.yaml）**

```yaml
model_list:
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      api_base: http://localhost:11434
      temperature: 0.7
      max_tokens: 4096
      num_ctx: 262144  # 256K tokens context

general_settings:
  master_key: sk-1234  # 任意のキー（ローカルなので簡易的でOK）
```

**設定の説明**
- `model_name`: Claude Codeが指定するモデル名
- `litellm_params.model`: 実際に使用するOllamaモデル
- `api_base`: OllamaのエンドポイントURL
- `master_key`: LiteLLMプロキシの認証キー

### 3.3.2 高度な設定ファイル

複数モデル、キャッシュ、ログを有効にした設定：

```yaml
# ~/litellm/config.yaml (フル機能版)

model_list:
  # Claudeモデルのマッピング（最高品質・30B Q8_0）
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      api_base: http://localhost:11434
      temperature: 0.7
      max_tokens: 4096
      num_ctx: 262144  # 256K tokens context

  # GPT-4モデルのマッピング（同じく30B使用）
  - model_name: gpt-4
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      api_base: http://localhost:11434
      temperature: 0.7
      num_ctx: 262144

  # 軽量モデル（高速用・14B）
  - model_name: gpt-3.5-turbo
    litellm_params:
      model: ollama/qwen3-coder:14b
      api_base: http://localhost:11434
      temperature: 0.7
      num_ctx: 262144

  # 超軽量モデル（最速・7B）
  - model_name: claude-3-haiku-20240307
    litellm_params:
      model: ollama/qwen3-coder:7b
      api_base: http://localhost:11434
      temperature: 0.7
      num_ctx: 262144

litellm_settings:
  # デバッグログを有効化
  set_verbose: true

  # キャッシュ設定
  cache: true
  cache_params:
    type: redis
    host: localhost
    port: 6379
    ttl: 3600  # 1時間キャッシュ

  # タイムアウト設定
  request_timeout: 600  # 10分

  # ストリーミング設定
  stream: true

general_settings:
  # 認証キー
  master_key: sk-local-dev-1234

  # 許可するモデル
  allowed_models:
    - claude-3-5-sonnet-20241022
    - gpt-4
    - gpt-3.5-turbo
    - claude-3-haiku-20240307

  # データベース（ログ保存用）
  database_url: sqlite:///litellm.db

  # ログ設定
  success_callback: ["langfuse"]
  failure_callback: ["langfuse"]

router_settings:
  # ロードバランシング戦略
  routing_strategy: latency-based-routing

  # リトライ設定
  num_retries: 2
  timeout: 300

  # フォールバック
  fallbacks:
    - claude-3-5-sonnet-20241022: [gpt-4, gpt-3.5-turbo]
```

### 3.3.3 設定ファイルの検証

```bash
# 設定ファイルの構文チェック
python3 << EOF
import yaml

with open('config.yaml', 'r') as f:
    config = yaml.safe_load(f)
    print("Config loaded successfully!")
    print(f"Number of models: {len(config['model_list'])}")
    for model in config['model_list']:
        print(f"  - {model['model_name']} → {model['litellm_params']['model']}")
EOF
```

**出力例**
```
Config loaded successfully!
Number of models: 4
  - claude-3-5-sonnet-20241022 → ollama/qwen3-coder:30b-a3b-q8_0 (256K context)
  - gpt-4 → ollama/qwen3-coder:30b-a3b-q8_0 (256K context)
  - gpt-3.5-turbo → ollama/qwen3-coder:14b (256K context)
  - claude-3-haiku-20240307 → ollama/qwen3-coder:7b (256K context)
```

## 3.4 LiteLLMプロキシの起動

### 3.4.1 基本的な起動方法

```bash
# 仮想環境を有効化（まだの場合）
cd ~/litellm
source venv/bin/activate

# LiteLLMプロキシを起動
litellm --config config.yaml --port 8000 --host 0.0.0.0
```

**起動ログ例（2025年11月最新構成）**
```
INFO: Starting LiteLLM Proxy Server
INFO: Loaded config from config.yaml
INFO: Loaded 4 models
INFO:   - claude-3-5-sonnet-20241022 -> ollama/qwen3-coder:30b-a3b-q8_0 (256K ctx)
INFO:   - gpt-4 -> ollama/qwen3-coder:30b-a3b-q8_0 (256K ctx)
INFO:   - gpt-3.5-turbo -> ollama/qwen3-coder:14b (256K ctx)
INFO:   - claude-3-haiku-20240307 -> ollama/qwen3-coder:7b (256K ctx)
INFO: Uvicorn running on http://0.0.0.0:8000
INFO: Proxy server started successfully!
```

### 3.4.2 バックグラウンド起動

プロキシをバックグラウンドで実行し、ターミナルを占有しないようにします。

**方法1: nohupを使用**

```bash
# バックグラウンドで起動
nohup litellm --config config.yaml --port 8000 --host 0.0.0.0 > litellm.log 2>&1 &

# プロセスIDを確認
echo $!

# ログを確認
tail -f litellm.log
```

**方法2: systemdサービスとして起動**

```bash
# サービスファイルを作成
sudo nano /etc/systemd/system/litellm.service
```

```ini
[Unit]
Description=LiteLLM Proxy Server
After=network.target ollama.service

[Service]
Type=simple
User=your_username
WorkingDirectory=/home/your_username/litellm
Environment="PATH=/home/your_username/litellm/venv/bin:/usr/bin"
ExecStart=/home/your_username/litellm/venv/bin/litellm --config /home/your_username/litellm/config.yaml --port 8000 --host 0.0.0.0
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**重要**: `your_username`を実際のユーザー名に置き換えてください。

```bash
# サービスを有効化・起動
sudo systemctl daemon-reload
sudo systemctl enable litellm
sudo systemctl start litellm

# 状態確認
sudo systemctl status litellm
```

**方法3: tmuxを使用（推奨）**

```bash
# tmuxセッションを作成
tmux new -s litellm

# LiteLLMを起動
litellm --config config.yaml --port 8000 --host 0.0.0.0

# Ctrl+B → D でデタッチ（バックグラウンド化）

# 後で再接続するには
tmux attach -t litellm
```

### 3.4.3 起動の確認

**ステップ1: プロセス確認**

```bash
# LiteLLMプロセスを確認
ps aux | grep litellm

# 出力例:
# user  12345  1.5  0.8 456789 102400 ?  Sl  10:30  0:05 /home/user/litellm/venv/bin/python3 /home/user/litellm/venv/bin/litellm ...
```

**ステップ2: ポート確認**

```bash
# ポート8000が待ち受けているか確認
sudo netstat -tuln | grep 8000

# 出力例:
# tcp  0  0  0.0.0.0:8000  0.0.0.0:*  LISTEN
```

**ステップ3: エンドポイント確認**

```bash
# ヘルスチェック
curl http://localhost:8000/health

# 出力例:
# {"status":"healthy"}
```

**ステップ4: モデル一覧取得**

```bash
curl http://localhost:8000/v1/models \
  -H "Authorization: Bearer sk-local-dev-1234"

# 出力例:
# {
#   "data": [
#     {"id": "claude-3-5-sonnet-20241022", "object": "model"},
#     {"id": "gpt-4", "object": "model"},
#     {"id": "gpt-3.5-turbo", "object": "model"},
#     {"id": "claude-3-haiku-20240307", "object": "model"}
#   ]
# }
```

## 3.5 動作テスト

### 3.5.1 curlでのテスト

```bash
# Chat Completions APIをテスト
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-local-dev-1234" \
  -d '{
    "model": "claude-3-5-sonnet-20241022",
    "messages": [
      {
        "role": "user",
        "content": "Write a Python function to calculate Fibonacci numbers"
      }
    ],
    "temperature": 0.7,
    "max_tokens": 500
  }'
```

**成功時の出力例**
```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1705315200,
  "model": "claude-3-5-sonnet-20241022",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Here's a Python function to calculate Fibonacci numbers:\n\n```python\ndef fibonacci(n):\n    \"\"\"\n    Calculate the nth Fibonacci number.\n    \n    Args:\n        n (int): The position in the Fibonacci sequence\n    \n    Returns:\n        int: The Fibonacci number at position n\n    \"\"\"\n    if n <= 0:\n        return 0\n    elif n == 1:\n        return 1\n    else:\n        a, b = 0, 1\n        for _ in range(2, n + 1):\n            a, b = b, a + b\n        return b\n\n# Example usage\nprint(fibonacci(10))  # Output: 55\n```\n\nThis function uses an iterative approach for efficiency."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 15,
    "completion_tokens": 142,
    "total_tokens": 157
  }
}
```

### 3.5.2 Pythonスクリプトでのテスト

```python
# test_litellm.py
import requests
import json

url = "http://localhost:8000/v1/chat/completions"
headers = {
    "Content-Type": "application/json",
    "Authorization": "Bearer sk-local-dev-1234"
}

payload = {
    "model": "claude-3-5-sonnet-20241022",
    "messages": [
        {
            "role": "system",
            "content": "You are a helpful coding assistant."
        },
        {
            "role": "user",
            "content": "Explain what a Python decorator is with a simple example."
        }
    ],
    "temperature": 0.7,
    "max_tokens": 1000
}

print("Sending request to LiteLLM proxy...")
response = requests.post(url, headers=headers, json=payload)

if response.status_code == 200:
    result = response.json()
    print("\n=== Response ===")
    print(result['choices'][0]['message']['content'])
    print(f"\n=== Usage ===")
    print(f"Prompt tokens: {result['usage']['prompt_tokens']}")
    print(f"Completion tokens: {result['usage']['completion_tokens']}")
    print(f"Total tokens: {result['usage']['total_tokens']}")
else:
    print(f"Error: {response.status_code}")
    print(response.text)
```

**実行**

```bash
python3 test_litellm.py
```

### 3.5.3 ストリーミングのテスト

```python
# test_streaming.py
import requests
import json

url = "http://localhost:8000/v1/chat/completions"
headers = {
    "Content-Type": "application/json",
    "Authorization": "Bearer sk-local-dev-1234"
}

payload = {
    "model": "claude-3-5-sonnet-20241022",
    "messages": [
        {
            "role": "user",
            "content": "Count from 1 to 10 in English."
        }
    ],
    "stream": True  # ストリーミング有効化
}

print("Streaming response:")
print("-" * 50)

response = requests.post(url, headers=headers, json=payload, stream=True)

for line in response.iter_lines():
    if line:
        line_str = line.decode('utf-8')
        if line_str.startswith('data: '):
            data_str = line_str[6:]  # "data: " を除去
            if data_str == '[DONE]':
                break
            try:
                data = json.loads(data_str)
                content = data['choices'][0]['delta'].get('content', '')
                print(content, end='', flush=True)
            except json.JSONDecodeError:
                pass

print("\n" + "-" * 50)
```

## 3.6 ログとモニタリング

### 3.6.1 ログの確認

**リアルタイムログ**

```bash
# systemdサービスの場合
sudo journalctl -u litellm -f

# nohupの場合
tail -f ~/litellm/litellm.log

# tmuxの場合
tmux attach -t litellm
```

**ログの内容例**
```
INFO: Request: POST /v1/chat/completions
INFO: Model: claude-3-5-sonnet-20241022 -> ollama/qwen3-coder:30b-a3b-q8_0
INFO: Context: 256K tokens available
INFO: Prompt tokens: 15, Completion tokens: 142
INFO: Response time: 7.54s (22 tokens/s)
INFO: Status: 200
```

### 3.6.2 データベースログ

設定ファイルで`database_url`を指定すると、すべてのリクエストがデータベースに保存されます。

```bash
# SQLiteデータベースを確認
sqlite3 ~/litellm/litellm.db

# テーブル一覧
.tables

# リクエスト履歴を表示
SELECT model, status_code, response_time, created_at
FROM logs
ORDER BY created_at DESC
LIMIT 10;

# 終了
.quit
```

### 3.6.3 モニタリングダッシュボード

LiteLLMには組み込みダッシュボードがあります。

```bash
# ダッシュボード付きで起動
litellm --config config.yaml --port 8000 --host 0.0.0.0 --ui
```

ブラウザで`http://localhost:8000/ui`にアクセスすると、以下が確認できます：
- リクエスト数
- レスポンスタイム
- エラー率
- モデル使用状況

## 3.7 トラブルシューティング

### 3.7.1 起動エラー

**エラー: "Port 8000 is already in use"**

```bash
# ポートを使用しているプロセスを特定
sudo lsof -i :8000

# プロセスを終了
sudo kill -9 <PID>

# または別のポートを使用
litellm --config config.yaml --port 8001
```

**エラー: "Could not connect to Ollama"**

```bash
# Ollamaが起動しているか確認
sudo systemctl status ollama

# 起動していなければ起動
sudo systemctl start ollama

# API確認
curl http://localhost:11434/api/tags
```

### 3.7.2 リクエストエラー

**エラー: "Unauthorized"**

```bash
# Authorization ヘッダーを確認
# Bearer トークンがconfig.yamlのmaster_keyと一致するか
```

**エラー: "Model not found"**

```bash
# モデル名が正しいか確認
curl http://localhost:8000/v1/models \
  -H "Authorization: Bearer sk-local-dev-1234"

# config.yamlのmodel_listに存在するか確認
```

## 3.8 まとめ

本章では、LiteLLMプロキシをセットアップしました。

**達成したこと**
✅ LiteLLMのインストール
✅ config.yamlの作成
✅ プロキシサーバーの起動
✅ API動作確認（curl、Python）
✅ ストリーミングテスト

**現在の構成**
```
Ollama (Port 11434) ← LiteLLM Proxy (Port 8000)
```

**次のステップ**
次章では、Claude Code CLIをインストールし、LiteLLMプロキシに接続します。これでローカルLLMをClaude Codeから使えるようになります！

**確認チェックリスト**
- [ ] `litellm --version`が動作する
- [ ] config.yamlが正しく作成されている
- [ ] プロキシが起動している（Port 8000）
- [ ] `/health`エンドポイントが応答する
- [ ] curlでチャット補完が動作する
- [ ] ログが正しく出力されている

すべてチェックできたら、Chapter 04へ進みましょう！
