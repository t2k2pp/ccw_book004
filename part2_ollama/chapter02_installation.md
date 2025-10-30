# 第2章:インストールとセットアップ

## 2.1 システム要件の確認

### 2.1.1 MS-S1 Max のスペック確認

Ollamaをインストールする前に、システムの詳細を確認します。

```bash
# CPU情報
lscpu | grep -E "Model name|CPU\(s\)|Thread"

# 出力例
Model name: AMD Ryzen AI Max+ 395
CPU(s): 32
Thread(s) per core: 2

# メモリ情報
free -h

# 出力例
              total        used        free      shared  buff/cache   available
Mem:          125Gi       8.2Gi       110Gi       1.5Gi       6.8Gi       115Gi

# GPU情報
lspci | grep VGA

# 出力例
0000:01:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Device 1900
```

### 2.1.2 必要なソフトウェア

#### Ubuntu/Debian

```bash
# システムを最新に更新
sudo apt update && sudo apt upgrade -y

# 必要なパッケージ
sudo apt install -y \
    curl \
    wget \
    ca-certificates \
    gnupg \
    lsb-release
```

### 2.1.3 ディスク空間の確認

```bash
# ディスク使用状況
df -h /

# モデル保存用のスペース確認
# 推奨: 最低100GB以上の空き容量
```

Ollamaのモデルは以下のディレクトリに保存されます：
```
~/.ollama/models/
```

**容量の目安:**
- 7Bモデル（Q4量子化）: 約4GB
- 13Bモデル（Q4量子化）: 約8GB
- 34Bモデル（Q4量子化）: 約20GB
- 70Bモデル（Q4量子化）: 約40GB

## 2.2 Ollama のインストール

### 2.2.1 公式インストールスクリプト

最も簡単な方法は、公式インストールスクリプトを使用することです。

```bash
# 公式インストールスクリプト実行
curl -fsSL https://ollama.com/install.sh | sh
```

**インストール内容:**
```
Installing Ollama...
✓ Downloaded Ollama binary
✓ Created systemd service
✓ Added to PATH
✓ Started Ollama service

Ollama has been installed successfully!
```

### 2.2.2 インストールの確認

```bash
# バージョン確認
ollama --version

# 出力例
ollama version is 0.5.4

# サービス状態確認
systemctl status ollama

# 出力例
● ollama.service - Ollama Service
     Loaded: loaded (/etc/systemd/system/ollama.service; enabled)
     Active: active (running) since...
```

### 2.2.3 手動インストール（オプション）

より細かい制御が必要な場合は、手動でインストールできます。

```bash
# Ollamaバイナリをダウンロード
sudo curl -L https://ollama.com/download/ollama-linux-amd64 -o /usr/local/bin/ollama

# 実行権限を付与
sudo chmod +x /usr/local/bin/ollama

# ollamaユーザーとグループを作成
sudo useradd -r -s /bin/false -U -m -d /usr/share/ollama ollama
sudo usermod -a -G render ollama
sudo usermod -a -G video ollama

# systemdサービスファイル作成
sudo nano /etc/systemd/system/ollama.service
```

**サービスファイルの内容:**
```ini
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/local/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3
Environment="PATH=/usr/local/bin:/usr/bin:/bin"
Environment="OLLAMA_HOST=0.0.0.0:11434"

[Install]
WantedBy=default.target
```

```bash
# サービスを有効化して起動
sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama
```

## 2.3 AMD GPU 認識の確認

### 2.3.1 ROCm インストール状況の確認

Ollamaが AMD GPU を利用するには、ROCm が正しくインストールされている必要があります。

```bash
# ROCmバージョン確認
rocm-smi --version

# 出力例（期待値）
ROCm System Management Interface version: 6.3.0

# GPU情報表示
rocm-smi

# 出力例
========================ROCm System Management Interface========================
GPU  Temp   AvgPwr  SCLK     MCLK     Fan   Perf  PwrCap  VRAM%  GPU%
0    42.0c  15.0W   500Mhz   96Mhz    0%    auto  120.0W  2%     0%
```

### 2.3.2 環境変数の設定

AMD Radeon 8060S (RDNA 3.5) を正しく認識させるための環境変数を設定します。

```bash
# ~/.bashrcに追加
nano ~/.bashrc

# 以下を末尾に追加
export HSA_OVERRIDE_GFX_VERSION=11.0.0
export OLLAMA_DEBUG=1  # デバッグログ有効化（初期設定時のみ）
```

```bash
# 設定を反映
source ~/.bashrc
```

### 2.3.3 GPU認識テスト

```bash
# Ollamaを再起動して環境変数を適用
sudo systemctl restart ollama

# ログでGPU認識を確認
journalctl -u ollama -f
```

**正常な出力例:**
```
Starting Ollama Service...
Detected GPU: AMD Radeon Graphics (gfx1100)
Using ROCm backend
GPU Memory: 96GB available
Ollama server started on http://0.0.0.0:11434
```

## 2.4 初回モデルのダウンロード

### 2.4.1 小型モデルでテスト

まずは小型モデルで動作確認します。

```bash
# 7Bモデルをダウンロード
ollama pull qwen2.5:7b

# 進行状況が表示される
pulling manifest
pulling 8cf58c9acf79... 100% ▕████████████████████████▏ 4.7 GB
pulling 8ab4849b038c... 100% ▕████████████████████████▏  249 B
pulling 23e0f4461c0c... 100% ▕████████████████████████▏  11 KB
pulling df4c5cf440f3... 100% ▕████████████████████████▏  485 B
verifying sha256 digest
writing manifest
success
```

### 2.4.2 動作確認

```bash
# モデルを実行
ollama run qwen2.5:7b

# プロンプトが表示される
>>> こんにちは！MS-S1 Maxについて教えてください。

# レスポンス（例）
MS-S1 Maxは、AMD Ryzen AI Max+ 395プロセッサを搭載したMinisforumの
高性能ミニPCです。128GBの大容量メモリと強力な統合GPUにより、
大規模な言語モデルをローカルで実行するのに最適な環境を提供します...

>>> /bye
```

## 2.5 Ollama の基本設定

### 2.5.1 環境変数の詳細設定

Ollamaの動作は環境変数で制御できます。

```bash
# systemd環境変数ファイルを作成
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo nano /etc/systemd/system/ollama.service.d/environment.conf
```

**environment.conf の内容:**
```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_ORIGINS=*"
Environment="OLLAMA_NUM_PARALLEL=2"
Environment="OLLAMA_MAX_LOADED_MODELS=3"
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
Environment="OLLAMA_FLASH_ATTENTION=1"
```

**環境変数の説明:**

| 変数 | 説明 | デフォルト | 推奨値（MS-S1 Max） |
|------|------|-----------|---------------------|
| OLLAMA_HOST | APIサーバーのアドレス | 127.0.0.1:11434 | 0.0.0.0:11434 |
| OLLAMA_ORIGINS | CORS許可オリジン | localhost | * |
| OLLAMA_NUM_PARALLEL | 並列リクエスト数 | 1 | 2-4 |
| OLLAMA_MAX_LOADED_MODELS | 同時ロードモデル数 | 1 | 2-3 |
| HSA_OVERRIDE_GFX_VERSION | AMD GPU互換設定 | - | 11.0.0 |
| OLLAMA_FLASH_ATTENTION | Flash Attention有効化 | 0 | 1 |

```bash
# 設定を反映
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### 2.5.2 ネットワーク設定

#### ローカルネットワークからアクセス

```bash
# ファイアウォール設定（UFW使用の場合）
sudo ufw allow 11434/tcp
sudo ufw reload

# ポート確認
sudo netstat -tlnp | grep 11434

# 出力例
tcp  0  0  0.0.0.0:11434  0.0.0.0:*  LISTEN  12345/ollama
```

#### 他のマシンからアクセステスト

```bash
# 別のPC/スマホから
curl http://<MS-S1-MaxのIPアドレス>:11434/api/version

# 出力例
{"version":"0.5.4"}
```

### 2.5.3 ログ設定

```bash
# リアルタイムログ表示
journalctl -u ollama -f

# 過去のログ確認
journalctl -u ollama --since "1 hour ago"

# ログレベル設定（デバッグモード）
sudo nano /etc/systemd/system/ollama.service.d/environment.conf

# 追加
Environment="OLLAMA_DEBUG=1"
```

## 2.6 パフォーマンス確認

### 2.6.1 GPU使用率のモニタリング

```bash
# 別のターミナルでGPU監視
watch -n 1 rocm-smi

# 推論実行中の出力例
GPU  Temp   AvgPwr  SCLK     MCLK     Fan   Perf  PwrCap  VRAM%  GPU%
0    68.0c  95.0W   2900Mhz  1000Mhz  65%   auto  120.0W  45%    92%
```

### 2.6.2 ベンチマークテスト

```bash
# Python スクリプトでベンチマーク
nano benchmark_ollama.py
```

**benchmark_ollama.py:**
```python
import ollama
import time

models = ["qwen2.5:7b", "qwen2.5:14b"]

for model in models:
    print(f"\n=== Testing {model} ===")

    # ウォームアップ
    ollama.generate(model=model, prompt="Hello")

    # ベンチマーク
    start = time.time()
    response = ollama.generate(
        model=model,
        prompt="Write a detailed explanation of quantum computing in 100 words.",
        options={"num_predict": 100}
    )
    elapsed = time.time() - start

    tokens = 100
    speed = tokens / elapsed
    print(f"Time: {elapsed:.2f}s")
    print(f"Speed: {speed:.1f} tokens/s")
```

```bash
# 実行
python3 benchmark_ollama.py
```

**期待される結果（MS-S1 Max）:**
```
=== Testing qwen2.5:7b ===
Time: 2.50s
Speed: 40.0 tokens/s

=== Testing qwen2.5:14b ===
Time: 4.55s
Speed: 22.0 tokens/s
```

## 2.7 複数バージョンの管理

### 2.7.1 モデルのタグ指定

```bash
# 特定のバージョンをダウンロード
ollama pull llama3.1:8b-instruct-q4_K_M
ollama pull llama3.1:8b-instruct-q8_0

# モデルリスト表示
ollama list

# 出力例
NAME                              ID              SIZE    MODIFIED
llama3.1:8b-instruct-q4_K_M      abcd1234        4.9GB   2 minutes ago
llama3.1:8b-instruct-q8_0        efgh5678        8.5GB   1 minute ago
qwen2.5:7b                        ijkl9012        4.7GB   10 minutes ago
```

### 2.7.2 エイリアスの作成

```bash
# カスタムタグでモデルをコピー
ollama cp qwen2.5:7b my-assistant

# 使用
ollama run my-assistant
```

## 2.8 トラブルシューティング

### 2.8.1 サービスが起動しない

```bash
# エラーログ確認
sudo journalctl -u ollama --no-pager

# 一般的な問題
```

**問題1: ポート既使用**
```bash
# ポート使用確認
sudo lsof -i :11434

# 解決法: ポート変更
sudo nano /etc/systemd/system/ollama.service.d/environment.conf
# OLLAMA_HOST=0.0.0.0:11435 に変更
```

**問題2: GPUアクセス権限**
```bash
# renderグループに追加
sudo usermod -a -G render $USER
sudo usermod -a -G video $USER

# 再ログイン
exit
# 再度SSH/ログイン
```

### 2.8.2 GPU が認識されない

```bash
# ROCm確認
/opt/rocm/bin/rocminfo | grep "Name:"

# 環境変数確認
echo $HSA_OVERRIDE_GFX_VERSION  # 11.0.0 であるべき

# Ollamaログ確認
journalctl -u ollama | grep -i gpu
```

**解決法:**
```bash
# 環境変数を永続化
sudo nano /etc/environment

# 追加
HSA_OVERRIDE_GFX_VERSION=11.0.0

# システム再起動
sudo reboot
```

### 2.8.3 モデルダウンロードが失敗する

```bash
# ネットワーク確認
curl -I https://ollama.com

# プロキシ設定（必要な場合）
sudo nano /etc/systemd/system/ollama.service.d/environment.conf

# 追加
Environment="HTTP_PROXY=http://proxy.example.com:8080"
Environment="HTTPS_PROXY=http://proxy.example.com:8080"
```

## 2.9 Docker による Ollama 環境構築

### 2.9.1 Docker Desktop との統合

**Docker Desktop の Models 機能**

Docker Desktop（Windows/Mac版）には、バージョン4.26以降、**Models**という統合機能が追加されました。これにより、Ollamaを含む複数のAIモデル実行環境を簡単に管理できます。

```
Docker Desktop → 左サイドバー → Models
→ Ollama、OpenLLM、vLLMなどから選択
→ ワンクリックでインストール・起動
```

**特徴:**
- GUI からの簡単なセットアップ
- モデルの一元管理
- コンテナのリソース割り当て設定
- ログの統合表示

### 2.9.2 Docker Compose による Ollama セットアップ

**基本的な docker-compose.yml**

```yaml
version: '3.8'

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_ORIGINS=*
      - OLLAMA_NUM_PARALLEL=2
      - OLLAMA_MAX_LOADED_MODELS=3
    restart: unless-stopped

volumes:
  ollama_data:
    driver: local
```

**起動方法:**

```bash
# コンテナ起動
docker-compose up -d

# ログ確認
docker-compose logs -f ollama

# モデル実行
docker exec -it ollama ollama run qwen2.5:7b
```

### 2.9.3 GPU と CPU の分散配置戦略

**なぜ GPU/CPU を分けるのか？**

MS-S1 Max の128GBメモリと Radeon 8060S を活用する際、以下のシナリオで GPU/CPU 分散が有効です：

```
シナリオ1: 複数モデル同時実行
  → GPU: 主力モデル（7B-14B）
  → CPU: 補助的な軽量モデル（3B以下）

シナリオオ2: 異なる優先度のタスク
  → GPU: リアルタイム対話（低レイテンシ）
  → CPU: バックグラウンド処理（バッチ処理）

シナリオ3: リソース効率最大化
  → GPUをメインタスクに専念
  → CPUで軽量タスクを並列処理
```

### 2.9.4 GPU 専用コンテナの設定

**AMD GPU 対応 docker-compose.yml**

```yaml
version: '3.8'

services:
  ollama-gpu:
    image: ollama/ollama:latest
    container_name: ollama-gpu
    ports:
      - "11434:11434"
    volumes:
      - ollama_gpu_data:/root/.ollama
    devices:
      - /dev/kfd
      - /dev/dri
    group_add:
      - video
      - render
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - HSA_OVERRIDE_GFX_VERSION=11.0.0
      - OLLAMA_NUM_PARALLEL=2
      - OLLAMA_MAX_LOADED_MODELS=2
      - OLLAMA_FLASH_ATTENTION=1
    deploy:
      resources:
        reservations:
          devices:
            - driver: amd
              capabilities: [gpu]
              count: all
    restart: unless-stopped

volumes:
  ollama_gpu_data:
    driver: local
```

**重要ポイント:**

```yaml
devices:
  - /dev/kfd      # AMD GPU カーネルドライバ
  - /dev/dri      # Direct Rendering Infrastructure

group_add:
  - video         # ビデオグループ権限
  - render        # レンダーグループ権限

environment:
  - HSA_OVERRIDE_GFX_VERSION=11.0.0  # RDNA 3.5 対応
```

### 2.9.5 CPU 専用コンテナの設定

**CPU 最適化 docker-compose.yml**

```yaml
version: '3.8'

services:
  ollama-cpu:
    image: ollama/ollama:latest
    container_name: ollama-cpu
    ports:
      - "11435:11434"  # 異なるポート番号
    volumes:
      - ollama_cpu_data:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_NUM_PARALLEL=4
      - OLLAMA_MAX_LOADED_MODELS=2
    deploy:
      resources:
        limits:
          cpus: '8.0'        # 8コアに制限
          memory: 32G        # 32GBに制限
        reservations:
          cpus: '4.0'        # 最低4コア確保
          memory: 16G        # 最低16GB確保
    restart: unless-stopped

volumes:
  ollama_cpu_data:
    driver: local
```

### 2.9.6 マルチコンテナ構成（GPU + CPU）

**統合 docker-compose.yml**

```yaml
version: '3.8'

services:
  # GPU コンテナ - メインの推論用
  ollama-gpu:
    image: ollama/ollama:latest
    container_name: ollama-gpu
    ports:
      - "11434:11434"
    volumes:
      - ollama_gpu_data:/root/.ollama
    devices:
      - /dev/kfd
      - /dev/dri
    group_add:
      - video
      - render
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - HSA_OVERRIDE_GFX_VERSION=11.0.0
      - OLLAMA_NUM_PARALLEL=2
      - OLLAMA_MAX_LOADED_MODELS=2
      - OLLAMA_FLASH_ATTENTION=1
    deploy:
      resources:
        reservations:
          devices:
            - driver: amd
              capabilities: [gpu]
              count: all
    restart: unless-stopped
    networks:
      - ollama-network

  # CPU コンテナ - 軽量モデル・バックグラウンド用
  ollama-cpu:
    image: ollama/ollama:latest
    container_name: ollama-cpu
    ports:
      - "11435:11434"
    volumes:
      - ollama_cpu_data:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_NUM_PARALLEL=4
      - OLLAMA_MAX_LOADED_MODELS=3
    deploy:
      resources:
        limits:
          cpus: '8.0'
          memory: 32G
        reservations:
          cpus: '4.0'
          memory: 16G
    restart: unless-stopped
    networks:
      - ollama-network

  # Open WebUI - Web インターフェース
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "3000:8080"
    volumes:
      - open_webui_data:/app/backend/data
    environment:
      - OLLAMA_BASE_URLS=http://ollama-gpu:11434;http://ollama-cpu:11434
    depends_on:
      - ollama-gpu
      - ollama-cpu
    restart: unless-stopped
    networks:
      - ollama-network

networks:
  ollama-network:
    driver: bridge

volumes:
  ollama_gpu_data:
  ollama_cpu_data:
  open_webui_data:
```

**起動と確認:**

```bash
# すべてのコンテナを起動
docker-compose up -d

# 状態確認
docker-compose ps

# 出力例
NAME                IMAGE                              STATUS
ollama-gpu          ollama/ollama:latest               Up 2 minutes
ollama-cpu          ollama/ollama:latest               Up 2 minutes
open-webui          ghcr.io/open-webui/open-webui:main Up 2 minutes

# GPU コンテナでモデル実行
docker exec -it ollama-gpu ollama run qwen2.5:14b

# CPU コンテナでモデル実行
docker exec -it ollama-cpu ollama run qwen2.5:3b
```

### 2.9.7 ベストプラクティス

#### 1. モデル配置戦略（MS-S1 Max向け）

**GPU コンテナ（高性能・低レイテンシ）:**
```yaml
推奨モデル:
  - qwen2.5:7b-instruct     (4.8GB) - メインチャット
  - qwen2.5:14b-instruct    (9GB)   - 高品質応答
  - deepseek-coder:16b      (10GB)  - コーディング

合計: 約24GB
GPU使用率: 70-90%
レスポンス: 即座（< 0.5秒）
```

**CPU コンテナ（軽量・バックグラウンド）:**
```yaml
推奨モデル:
  - qwen2.5:3b-instruct     (2GB)   - 高速応答
  - phi3:3.8b               (2.3GB) - 簡単なタスク
  - gemma2:2b               (1.6GB) - 分類タスク

合計: 約6GB
CPU使用率: 40-60%（8コア使用時）
レスポンス: 1-2秒
```

#### 2. リソース割り当ての最適化

**推奨構成:**

```yaml
# GPU コンテナ
OLLAMA_NUM_PARALLEL=2      # 同時リクエスト数
OLLAMA_MAX_LOADED_MODELS=2 # ロード済みモデル数
→ メモリ使用: 20-30GB

# CPU コンテナ
OLLAMA_NUM_PARALLEL=4      # CPUは並列度高め
OLLAMA_MAX_LOADED_MODELS=3 # 軽量モデルは多く
CPU制限: 8コア
メモリ制限: 32GB
→ メモリ使用: 10-20GB

残りリソース:
  - CPU: 8コア（システム用）
  - メモリ: 70-98GB（他のアプリ用）
```

#### 3. ロードバランシング戦略

**Nginx によるロードバランシング:**

```nginx
# nginx.conf
upstream ollama_backend {
    # GPU コンテナ - 重いリクエスト用（重み付け高）
    server ollama-gpu:11434 weight=3 max_fails=3 fail_timeout=30s;

    # CPU コンテナ - 軽いリクエスト用
    server ollama-cpu:11434 weight=1 max_fails=3 fail_timeout=30s;
}

server {
    listen 11436;

    location / {
        proxy_pass http://ollama_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        # タイムアウト設定（長時間推論対応）
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
    }
}
```

**docker-compose.yml に追加:**

```yaml
  nginx:
    image: nginx:alpine
    container_name: ollama-loadbalancer
    ports:
      - "11436:11436"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - ollama-gpu
      - ollama-cpu
    restart: unless-stopped
    networks:
      - ollama-network
```

#### 4. パフォーマンス測定

**各コンテナの性能比較スクリプト:**

```python
# benchmark_containers.py
import requests
import time
import json

containers = {
    "GPU": "http://localhost:11434",
    "CPU": "http://localhost:11435"
}

prompt = "Explain quantum computing in 50 words."

for name, url in containers.items():
    print(f"\n=== Testing {name} Container ===")

    # API呼び出し
    start = time.time()
    response = requests.post(
        f"{url}/api/generate",
        json={
            "model": "qwen2.5:7b",
            "prompt": prompt,
            "stream": False
        }
    )
    elapsed = time.time() - start

    if response.status_code == 200:
        data = response.json()
        tokens = data.get("eval_count", 0)
        speed = tokens / elapsed if elapsed > 0 else 0

        print(f"Time: {elapsed:.2f}s")
        print(f"Tokens: {tokens}")
        print(f"Speed: {speed:.1f} tokens/s")
    else:
        print(f"Error: {response.status_code}")
```

**期待される結果（MS-S1 Max）:**

```
=== Testing GPU Container ===
Time: 1.8s
Tokens: 50
Speed: 27.8 tokens/s

=== Testing CPU Container ===
Time: 6.2s
Tokens: 50
Speed: 8.1 tokens/s

→ GPU は CPU の約3.4倍高速
```

### 2.9.8 Docker 環境のトラブルシューティング

#### 問題1: AMD GPU が認識されない

```bash
# コンテナ内でGPU確認
docker exec -it ollama-gpu rocm-smi

# エラーが出る場合
# 1. ホストでROCmインストール確認
rocm-smi

# 2. デバイスファイルの権限確認
ls -la /dev/kfd /dev/dri

# 3. ユーザーのグループ確認
groups

# 解決法: render/videoグループに追加
sudo usermod -aG render,video $USER
# 再ログインまたは再起動
```

#### 問題2: コンテナ間でモデルが重複ダウンロード

**解決法: 共有ボリューム使用**

```yaml
volumes:
  # 共有モデルストレージ
  shared_models:
    driver: local

services:
  ollama-gpu:
    volumes:
      - shared_models:/root/.ollama

  ollama-cpu:
    volumes:
      - shared_models:/root/.ollama
```

**注意**: 同時に異なるモデルをダウンロードする場合は競合が発生する可能性があるため、順次ダウンロード推奨。

#### 問題3: メモリ不足

```bash
# Docker Desktop のリソース設定確認
# Settings → Resources → Advanced
# Memory: 100GB以上に設定（MS-S1 Maxの場合）

# または docker-compose.yml で制限調整
deploy:
  resources:
    limits:
      memory: 28G  # 減らす
```

### 2.9.9 本節のまとめ

Docker 環境での Ollama 実行について学習しました：

✅ **Docker Desktop 統合**
- Models 機能での簡単セットアップ
- GUI からの管理

✅ **GPU/CPU 分散配置**
- GPU: 高性能モデル（7B-14B）
- CPU: 軽量モデル（3B以下）
- リソース効率最大化

✅ **マルチコンテナ構成**
- 専用コンテナで役割分担
- ロードバランシング
- 128GBメモリの戦略的活用

✅ **ベストプラクティス**
- モデル配置戦略
- リソース割り当て最適化
- パフォーマンス測定

**💡 推奨構成（MS-S1 Max）:**
```
GPU コンテナ: 14B + 7Bモデル（24GB使用）
CPU コンテナ: 3B × 2-3モデル（10GB使用）
残りメモリ: 94GB（他のアプリに使用可能）
→ 柔軟なマルチモデル環境を構築
```

## 2.10 本章の総まとめ

本章では、以下の内容を学習しました。

✅ **システム要件の確認**
- MS-S1 Maxのスペック確認方法
- 必要なディスク容量

✅ **Ollamaのインストール**
- 公式スクリプトによる簡単インストール
- 手動インストール方法
- Docker による構築

✅ **AMD GPU認識**
- ROCm確認
- 環境変数設定（HSA_OVERRIDE_GFX_VERSION）

✅ **初期設定**
- 環境変数の詳細設定
- ネットワーク設定
- ログ設定

✅ **Docker 環境**
- Docker Desktop 統合
- GPU/CPU 分散配置戦略
- マルチコンテナ構成

✅ **動作確認**
- 初回モデルダウンロード
- パフォーマンステスト

次章では、ROCmの詳細設定とAMD GPU最適化について深く掘り下げます。

---

**前章へ**: [第1章 はじめに](chapter01_introduction.md)
**次章へ**: [第3章 ROCm設定とAMD GPU最適化](chapter03_rocm_optimization.md)
