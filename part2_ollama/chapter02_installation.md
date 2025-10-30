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

## 2.9 本章のまとめ

本章では、以下の内容を学習しました。

✅ **システム要件の確認**
- MS-S1 Maxのスペック確認方法
- 必要なディスク容量

✅ **Ollamaのインストール**
- 公式スクリプトによる簡単インストール
- 手動インストール方法

✅ **AMD GPU認識**
- ROCm確認
- 環境変数設定（HSA_OVERRIDE_GFX_VERSION）

✅ **初期設定**
- 環境変数の詳細設定
- ネットワーク設定
- ログ設定

✅ **動作確認**
- 初回モデルダウンロード
- パフォーマンステスト

次章では、ROCmの詳細設定とAMD GPU最適化について深く掘り下げます。

---

**前章へ**: [第1章 はじめに](chapter01_introduction.md)
**次章へ**: [第3章 ROCm設定とAMD GPU最適化](chapter03_rocm_optimization.md)
