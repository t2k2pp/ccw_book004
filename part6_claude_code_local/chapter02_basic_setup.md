# Chapter 02: 基本セットアップ（Ollama）

## 2.1 Ollamaのインストール

### 2.1.1 Ollamaとは

Ollamaは、ローカルでLLMを簡単に実行できるツールです。Dockerのような感覚で、コマンド一つでモデルをダウンロード・実行できます。

**特徴**
- シンプルなCLI
- 自動的なモデル管理
- OpenAI互換API
- GPU自動検出（ROCm対応）
- メモリ効率的な設計

### 2.1.2 インストール手順（Ubuntu）

**ステップ1: システムアップデート**

```bash
# パッケージリストを更新
sudo apt update

# 既存パッケージをアップグレード
sudo apt upgrade -y
```

**ステップ2: 必要なパッケージをインストール**

```bash
# curlがインストールされているか確認
curl --version

# インストールされていなければ
sudo apt install curl -y
```

**ステップ3: Ollamaをインストール**

```bash
# 公式インストールスクリプトを実行
curl -fsSL https://ollama.com/install.sh | sh
```

**実行結果の例**
```
>>> Downloading ollama...
>>> Installing ollama to /usr/local/bin...
>>> Creating ollama user...
>>> Adding ollama user to video group...
>>> Adding ollama user to render group...
>>> Creating ollama systemd service...
>>> Enabling and starting ollama service...
>>> The Ollama API is now available at 127.0.0.1:11434.
>>> Install complete. Run "ollama run llama2" to get started.
```

**ステップ4: インストール確認**

```bash
# バージョン確認
ollama --version

# 出力例: ollama version 0.5.1
```

```bash
# サービス状態確認
systemctl status ollama

# 出力例:
# ● ollama.service - Ollama Service
#      Loaded: loaded (/etc/systemd/system/ollama.service; enabled)
#      Active: active (running) since ...
```

```bash
# API動作確認
curl http://localhost:11434/api/tags

# 出力例:
# {"models":[]}
```

### 2.1.3 MS-S1 Max向けROCm設定

MS-S1 MaxのRadeon 8060SでGPU加速を有効にするため、環境変数を設定します。

**ステップ1: 環境変数ファイルを作成**

```bash
# Ollama用の環境変数ファイルを作成
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo nano /etc/systemd/system/ollama.service.d/override.conf
```

**ステップ2: 以下の内容を記述**

```ini
[Service]
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
Environment="PYTORCH_ROCM_ARCH=gfx1100"
Environment="GPU_MAX_ALLOC_PERCENT=95"
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_KEEP_ALIVE=5m"
Environment="OLLAMA_NUM_PARALLEL=2"
```

**各環境変数の説明**
- `HSA_OVERRIDE_GFX_VERSION=11.0.0`: Radeon 8060S（RDNA 3.5）を認識させる
- `PYTORCH_ROCM_ARCH=gfx1100`: アーキテクチャを指定
- `GPU_MAX_ALLOC_PERCENT=95`: VRAM使用量の上限（95%）
- `OLLAMA_HOST=0.0.0.0:11434`: すべてのネットワークインターフェースで待ち受け
- `OLLAMA_KEEP_ALIVE=5m`: モデルをメモリに5分間保持
- `OLLAMA_NUM_PARALLEL=2`: 並列リクエスト数

**ステップ3: サービスを再起動**

```bash
# systemdデーモンをリロード
sudo systemctl daemon-reload

# Ollamaサービスを再起動
sudo systemctl restart ollama

# 状態確認
sudo systemctl status ollama
```

**ステップ4: GPU認識を確認**

```bash
# ROCmデバイス確認
rocm-smi

# 出力例:
# ====================================== ROCm SMI =======================================
# GPU  Temp   AvgPwr  SCLK     MCLK     Fan   Perf  PwrCap  VRAM%  GPU%
# 0    45.0c  15.0W   800Mhz   1000Mhz  0%    auto  120.0W  5%     0%
# ======================================================================================
```

## 2.2 モデルのダウンロード

### 2.2.1 推奨モデルの選択

MS-S1 Maxでは、以下のモデルが推奨されます。

**コーディング特化モデル**

| モデル | サイズ | メモリ | 速度 | 特徴 |
|--------|--------|--------|------|------|
| qwen2.5-coder:7b | 4.7GB | 5.8GB | 32 tokens/s | 軽量・高速 |
| qwen2.5-coder:14b | 8.9GB | 11.2GB | 18 tokens/s | **推奨**・バランス型 |
| qwen2.5-coder:32b | 19GB | 24GB | 8 tokens/s | 最高品質 |
| deepseek-coder:6.7b | 3.8GB | 5.2GB | 35 tokens/s | 軽量 |
| codellama:13b | 7.4GB | 10GB | 20 tokens/s | Meta製 |

**汎用モデル**

| モデル | サイズ | メモリ | 速度 | 特徴 |
|--------|--------|--------|------|------|
| qwen2.5:14b | 8.9GB | 11.2GB | 18 tokens/s | 汎用・バランス |
| llama3.1:8b | 4.7GB | 6.5GB | 28 tokens/s | 軽量 |
| mixtral:8x7b | 26GB | 28GB | 8 tokens/s | 高性能 |

**初心者への推奨: qwen2.5-coder:14b**
- Claude Code用に最適化
- 日本語・英語ともに高品質
- MS-S1 Maxで快適に動作

### 2.2.2 モデルのダウンロード手順

**ステップ1: qwen2.5-coder:14bをダウンロード**

```bash
# モデルをダウンロード（初回は時間がかかります）
ollama pull qwen2.5-coder:14b
```

**出力例**
```
pulling manifest
pulling 4e16af9b34d5... 100% ▕████████████████▏ 8.9 GB
pulling 98c4ac90a64b... 100% ▕████████████████▏  147 B
pulling d8f8a0e2ee96... 100% ▕████████████████▏  11 KB
pulling 0ba8f0e314b4... 100% ▕████████████████▏  487 B
verifying sha256 digest
writing manifest
success
```

**ダウンロード時間の目安**
- 光回線（100Mbps）: 約12-15分
- 高速回線（1Gbps）: 約2-3分

**ステップ2: ダウンロード確認**

```bash
# インストール済みモデルを確認
ollama list

# 出力例:
# NAME                    ID              SIZE    MODIFIED
# qwen2.5-coder:14b       4e16af9b34d5    8.9 GB  2 minutes ago
```

### 2.2.3 追加モデルのダウンロード（オプション）

**軽量モデル（高速動作）**

```bash
# Qwen2.5 Coder 7B（より高速）
ollama pull qwen2.5-coder:7b

# DeepSeek Coder 6.7B（最軽量）
ollama pull deepseek-coder:6.7b
```

**高性能モデル（より高品質）**

```bash
# Qwen2.5 Coder 32B（最高品質）
ollama pull qwen2.5-coder:32b
```

**汎用モデル（コーディング以外も対応）**

```bash
# Qwen2.5 14B（汎用）
ollama pull qwen2.5:14b

# Llama 3.1 8B（軽量・汎用）
ollama pull llama3.1:8b
```

## 2.3 動作確認

### 2.3.1 CLIでの対話テスト

**ステップ1: モデルを起動**

```bash
# Qwen2.5 Coder 14Bを起動
ollama run qwen2.5-coder:14b
```

**ステップ2: 質問をする**

```
>>> Hello! Can you write a Python function to calculate factorial?

Certainly! Here's a Python function to calculate the factorial of a number:

def factorial(n):
    """
    Calculate the factorial of a non-negative integer n.

    Args:
        n (int): A non-negative integer

    Returns:
        int: The factorial of n

    Raises:
        ValueError: If n is negative
    """
    if n < 0:
        raise ValueError("Factorial is not defined for negative numbers")
    elif n == 0 or n == 1:
        return 1
    else:
        result = 1
        for i in range(2, n + 1):
            result *= i
        return result

# Example usage:
print(factorial(5))  # Output: 120

This function calculates the factorial by multiplying all positive integers
from 1 to n. It also includes error handling for negative inputs.
```

**ステップ3: 終了**

```
>>> /bye
```

### 2.3.2 API経由でのテスト

Ollamaは`http://localhost:11434`でREST APIを提供します。

**ステップ1: curlでテスト**

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5-coder:14b",
  "prompt": "Write a hello world program in Python",
  "stream": false
}'
```

**出力例**
```json
{
  "model": "qwen2.5-coder:14b",
  "created_at": "2025-01-15T10:30:00.000Z",
  "response": "Here's a simple Hello World program in Python:\n\n```python\nprint(\"Hello, World!\")\n```\n\nThis program uses the `print()` function to output the text \"Hello, World!\" to the console.",
  "done": true,
  "total_duration": 2340000000,
  "load_duration": 450000000,
  "prompt_eval_count": 12,
  "prompt_eval_duration": 340000000,
  "eval_count": 45,
  "eval_duration": 1550000000
}
```

**ステップ2: Pythonスクリプトでテスト**

```python
# test_ollama.py
import requests
import json

url = "http://localhost:11434/api/generate"

payload = {
    "model": "qwen2.5-coder:14b",
    "prompt": "Write a function to reverse a string in Python",
    "stream": False
}

response = requests.post(url, json=payload)

if response.status_code == 200:
    result = response.json()
    print("Response:")
    print(result['response'])
    print(f"\nTokens per second: {result['eval_count'] / (result['eval_duration'] / 1e9):.2f}")
else:
    print(f"Error: {response.status_code}")
```

**実行**

```bash
python3 test_ollama.py

# 出力例:
# Response:
# Here's a simple function to reverse a string in Python:
#
# ```python
# def reverse_string(s):
#     return s[::-1]
#
# # Example usage
# text = "Hello, World!"
# reversed_text = reverse_string(text)
# print(reversed_text)  # Output: !dlroW ,olleH
# ```
#
# Tokens per second: 18.45
```

### 2.3.3 パフォーマンス測定

MS-S1 Maxでの実際の性能を測定します。

**ベンチマークスクリプト**

```python
# benchmark_ollama.py
import requests
import time
import statistics

def benchmark_ollama(model, prompt, iterations=5):
    """Ollamaのパフォーマンスをベンチマーク"""
    url = "http://localhost:11434/api/generate"

    results = {
        'total_times': [],
        'tokens_per_second': [],
        'prompt_eval_times': [],
        'eval_times': []
    }

    for i in range(iterations):
        print(f"Iteration {i+1}/{iterations}...")

        start_time = time.time()

        response = requests.post(url, json={
            "model": model,
            "prompt": prompt,
            "stream": False
        })

        end_time = time.time()

        if response.status_code == 200:
            data = response.json()

            total_time = end_time - start_time
            tokens_per_sec = data['eval_count'] / (data['eval_duration'] / 1e9)

            results['total_times'].append(total_time)
            results['tokens_per_second'].append(tokens_per_sec)
            results['prompt_eval_times'].append(data['prompt_eval_duration'] / 1e9)
            results['eval_times'].append(data['eval_duration'] / 1e9)

    # 統計計算
    print(f"\n=== Benchmark Results for {model} ===")
    print(f"Prompt: {prompt[:50]}...")
    print(f"\nTotal Time:")
    print(f"  Average: {statistics.mean(results['total_times']):.2f}s")
    print(f"  Median:  {statistics.median(results['total_times']):.2f}s")
    print(f"  Min:     {min(results['total_times']):.2f}s")
    print(f"  Max:     {max(results['total_times']):.2f}s")
    print(f"\nTokens per Second:")
    print(f"  Average: {statistics.mean(results['tokens_per_second']):.2f}")
    print(f"  Median:  {statistics.median(results['tokens_per_second']):.2f}")
    print(f"\nPrompt Eval Time: {statistics.mean(results['prompt_eval_times']):.2f}s")
    print(f"Generation Time:  {statistics.mean(results['eval_times']):.2f}s")

# 実行
if __name__ == "__main__":
    model = "qwen2.5-coder:14b"
    prompt = "Write a Python function to implement binary search algorithm with comments"

    benchmark_ollama(model, prompt, iterations=5)
```

**MS-S1 Maxでの実測結果**

```
=== Benchmark Results for qwen2.5-coder:14b ===
Prompt: Write a Python function to implement binary search...

Total Time:
  Average: 8.45s
  Median:  8.32s
  Min:     7.89s
  Max:     9.12s

Tokens per Second:
  Average: 18.23
  Median:  18.45

Prompt Eval Time: 0.42s
Generation Time:  8.03s
```

## 2.4 Ollamaの基本操作

### 2.4.1 モデル管理コマンド

**モデル一覧表示**

```bash
ollama list

# 出力例:
# NAME                    ID              SIZE    MODIFIED
# qwen2.5-coder:14b       4e16af9b34d5    8.9 GB  5 minutes ago
# qwen2.5-coder:7b        a2b3c4d5e6f7    4.7 GB  10 minutes ago
```

**モデル削除**

```bash
# 不要なモデルを削除してディスク容量を節約
ollama rm qwen2.5-coder:7b

# 確認
# Deleted 'qwen2.5-coder:7b'
```

**モデル情報表示**

```bash
# モデルの詳細情報を表示
ollama show qwen2.5-coder:14b

# 出力例:
# Model
#   architecture        qwen2
#   parameters          14.7B
#   quantization        Q4_K_M
#   context length      32768
#   embedding length    5120
```

**実行中のモデル確認**

```bash
# 現在メモリにロードされているモデルを確認
ollama ps

# 出力例:
# NAME                    ID              SIZE      UNTIL
# qwen2.5-coder:14b       4e16af9b34d5    11.2 GB   5 minutes from now
```

### 2.4.2 サービス管理

**サービス起動・停止**

```bash
# サービス停止
sudo systemctl stop ollama

# サービス起動
sudo systemctl start ollama

# サービス再起動
sudo systemctl restart ollama

# 自動起動設定
sudo systemctl enable ollama

# 自動起動無効化
sudo systemctl disable ollama
```

**ログ確認**

```bash
# リアルタイムログ表示
sudo journalctl -u ollama -f

# 最新100行を表示
sudo journalctl -u ollama -n 100

# エラーのみ表示
sudo journalctl -u ollama -p err
```

### 2.4.3 設定のカスタマイズ

**メモリ使用量の調整**

```bash
# 環境変数ファイルを編集
sudo nano /etc/systemd/system/ollama.service.d/override.conf

# OLLAMA_KEEP_ALIVEを変更
# 短く: Environment="OLLAMA_KEEP_ALIVE=2m"  # メモリをすぐ解放
# 長く: Environment="OLLAMA_KEEP_ALIVE=10m" # モデルを長く保持
```

**並列リクエスト数の調整**

```bash
# 同時に処理できるリクエスト数を変更
Environment="OLLAMA_NUM_PARALLEL=4"  # デフォルト: 1

# 注意: 並列数を増やすとメモリ使用量が増加
```

**ポート番号の変更**

```bash
# デフォルトの11434以外を使用したい場合
Environment="OLLAMA_HOST=0.0.0.0:8080"
```

変更後は必ず再起動：

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

## 2.5 トラブルシューティング

### 2.5.1 よくある問題

**問題1: "ollama: command not found"**

```bash
# 解決方法: パスを確認
which ollama

# もしくは
ls -la /usr/local/bin/ollama

# ない場合は再インストール
curl -fsSL https://ollama.com/install.sh | sh
```

**問題2: GPU認識されない**

```bash
# ROCmインストール確認
rocm-smi

# 環境変数確認
sudo systemctl show ollama | grep Environment

# 正しく設定されていない場合は再設定
sudo nano /etc/systemd/system/ollama.service.d/override.conf
```

**問題3: モデルダウンロードが失敗**

```bash
# ネットワーク確認
curl -I https://ollama.com

# プロキシ設定が必要な場合
export HTTP_PROXY=http://proxy.example.com:8080
export HTTPS_PROXY=http://proxy.example.com:8080

# 再試行
ollama pull qwen2.5-coder:14b
```

**問題4: "Error: could not connect to ollama server"**

```bash
# サービス状態確認
sudo systemctl status ollama

# 起動していない場合
sudo systemctl start ollama

# ポート確認
sudo netstat -tuln | grep 11434
```

### 2.5.2 パフォーマンス最適化

**メモリ不足の場合**

```bash
# より小さいモデルを使用
ollama pull qwen2.5-coder:7b

# またはKEEP_ALIVEを短縮
sudo nano /etc/systemd/system/ollama.service.d/override.conf
# Environment="OLLAMA_KEEP_ALIVE=1m"
```

**速度が遅い場合**

```bash
# GPU使用確認
rocm-smi

# GPU使用率が0%の場合、ROCm設定を確認
echo $HSA_OVERRIDE_GFX_VERSION
```

## 2.6 まとめ

本章では、Ollamaのインストールとセットアップを完了しました。

**達成したこと**
✅ Ollamaのインストール
✅ MS-S1 Max向けROCm設定
✅ qwen2.5-coder:14bのダウンロード
✅ CLI/APIでの動作確認
✅ パフォーマンス測定

**次のステップ**
次章では、LiteLLMをインストールし、OllamaとClaude Codeの橋渡しを設定します。LiteLLMプロキシを起動すれば、Claude CodeからローカルLLMを使えるようになります。

**確認チェックリスト**
- [ ] `ollama --version`が動作する
- [ ] `ollama list`でqwen2.5-coder:14bが表示される
- [ ] `ollama run qwen2.5-coder:14b`で対話できる
- [ ] `rocm-smi`でGPUが認識されている
- [ ] APIテスト（curl）が成功する

すべてチェックできたら、Chapter 03へ進みましょう！
