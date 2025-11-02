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

**コーディング特化モデル（2025年11月最新）**

| モデル | サイズ | メモリ | コンテキスト | 速度（MS-S1 Max） | 特徴 |
|--------|--------|--------|--------------|-------------------|------|
| **qwen3-coder:30b-a3b-q8_0** | **32GB** | **34GB** | **256K tokens** | **22 tokens/s** | **推奨・最高品質** |
| qwen3-coder:14b | 18GB | 20GB | 256K tokens | 28 tokens/s | バランス型 |
| qwen3-coder:7b | 8GB | 10GB | 256K tokens | 42 tokens/s | 軽量・高速 |
| deepseek-coder-v2:16b | 20GB | 22GB | 128K tokens | 25 tokens/s | 代替選択肢 |
| codellama:13b | 7.4GB | 10GB | 16K tokens | 20 tokens/s | 旧世代 |

**汎用モデル（コーディング以外も対応）**

| モデル | サイズ | メモリ | 速度 | 特徴 |
|--------|--------|--------|------|------|
| qwen2.5:14b | 8.9GB | 11.2GB | 18 tokens/s | 汎用・バランス |
| llama3.1:8b | 4.7GB | 6.5GB | 28 tokens/s | 軽量 |
| mixtral:8x7b | 26GB | 28GB | 8 tokens/s | 高性能 |

**📊 モデル選択の判断基準**

どのモデルを選ぶべきか、以下の基準で判断してください：

**1. メモリ容量で選ぶ**
```
あなたのマシンのメモリ: ___GB

【判断基準】
✅ 64GB以上（MS-S1 Maxなど）
   → qwen3-coder:30b-a3b-q8_0（32GB使用）
   　理由: 最高品質、256Kコンテキスト、余裕あり

⚠️ 32GB〜64GB
   → qwen3-coder:14b（18GB使用）
   　理由: バランス型、他のアプリも使える

⚠️ 16GB〜32GB
   → qwen3-coder:7b（8GB使用）
   　理由: 軽量、十分な性能

❌ 16GB未満
   → ローカルLLMは非推奨
   　理由: メモリ不足でシステムが不安定になる
```

**2. 用途で選ぶ**

| 用途 | 推奨モデル | 理由 |
|------|-----------|------|
| **本格的な開発** | qwen3-coder:30b-a3b-q8_0 | 最高品質のコード生成、大規模ファイル対応 |
| **学習・実験** | qwen3-coder:14b | バランス型、コスパ良好 |
| **簡単な質問応答** | qwen3-coder:7b | 高速レスポンス、軽量 |
| **文章作成** | qwen2.5:14b（汎用） | コーディング以外も対応 |

**3. 速度重視 vs 品質重視**

```
【速度重視】（リアルタイム応答が欲しい）
→ qwen3-coder:7b（42 tokens/s）
  - チャットボットのような対話
  - 簡単なコード生成
  - 学習中のクイック質問

【品質重視】（正確性が重要）
→ qwen3-coder:30b-a3b-q8_0（22 tokens/s）
  - 本番コードの生成
  - 複雑なアルゴリズム
  - コードレビュー
  - リファクタリング
```

**4. コンテキスト長で選ぶ**

```
処理したいファイルサイズ: _____行

【判断基準】
✅ 1000行以上の大規模ファイル
   → qwen3-coder:30b-a3b-q8_0（256K tokens）
   　理由: 大規模コードベースに対応

✅ 100〜1000行の中規模ファイル
   → qwen3-coder:14b（256K tokens）
   　理由: 十分なコンテキスト

✅ 100行未満の小規模ファイル
   → qwen3-coder:7b（256K tokens）
   　理由: オーバースペック不要
```

**💡 MS-S1 Maxユーザーへの推奨**

あなたがMS-S1 Max（128GB RAM、96GB VRAM設定可能）を使用している場合：

```bash
# 【推奨】最高品質モデルをダウンロード
ollama pull qwen3-coder:30b-a3b-q8_0

# 理由:
# ✅ メモリ容量に余裕がある（32GB使用、残り64GB以上）
# ✅ 最高品質のコード生成（HumanEval 92.8%）
# ✅ 大規模コンテキスト（256K tokens）
# ✅ 実用的な速度（22 tokens/s）
# ✅ 将来の拡張性（1Mトークンまで対応可能）
```

**⚠️ 複数モデルの併用（オプション）**

用途に応じて複数モデルを使い分けることも可能です（ディスク容量に余裕がある場合）：

```bash
# メインモデル（高品質・本番用）
ollama pull qwen3-coder:30b-a3b-q8_0  # 32GB

# サブモデル（高速・実験用）
ollama pull qwen3-coder:7b             # 8GB

# 合計使用量: 約40GB
# MS-S1 Maxなら問題なく両方使用可能
```

**📝 テーブルの読み方**

| 項目 | 意味 | あなたへの影響 |
|------|------|----------------|
| **サイズ** | ディスク使用量 | ダウンロードに必要な空き容量 |
| **メモリ** | RAM使用量（実行時） | システムメモリから消費される量 |
| **コンテキスト** | 一度に処理できるトークン数 | 大きいほど長いコードを処理可能 |
| **速度** | 生成速度（tokens/秒） | 早いほど待ち時間が短い |

**❓ よくある質問**

**Q: どれを選べばいいかわからない**
A: **MS-S1 Maxなら迷わずqwen3-coder:30b-a3b-q8_0を選んでください。**メモリに余裕があるので最高品質モデルを使わない理由はありません。

**Q: 複数モデルをインストールしても大丈夫？**
A: はい。ディスク容量があれば複数インストール可能です。切り替えて使えます。

**Q: 後からモデルを変更できる？**
A: はい。`ollama pull <別モデル>`で追加ダウンロードし、設定ファイルで切り替えるだけです。

**Q: 汎用モデルとコーディング特化モデルの違いは？**
A: コーディング特化モデルはプログラミング言語のデータで追加訓練されており、コード生成の精度が高いです。文章作成なら汎用モデル、プログラミングならコーディング特化モデルを選んでください。

### 2.2.2 モデルのダウンロード手順

**【必須】以下の手順を実行してください**

**ステップ1: qwen3-coder:30b-a3b-q8_0をダウンロード**

```bash
# Qwen3 Coder 30B Q8_0をダウンロード（初回は時間がかかります）
ollama pull qwen3-coder:30b-a3b-q8_0
```

**💡 このコマンドは何をしているのか？**
- `ollama pull`: Ollamaにモデルをダウンロードするよう指示
- `qwen3-coder:30b-a3b-q8_0`: ダウンロードするモデルの正確な名前
  - `qwen3-coder`: モデルのベース名（Qwen3のコーディング特化版）
  - `30b`: パラメータ数（30billion = 300億）
  - `a3b`: アクティブパラメータ（MoEで実際に動作する部分は3.3B）
  - `q8_0`: 量子化レベル（Q8 = 8bit量子化、品質と速度のバランス）

**出力例**
```
pulling manifest
pulling 7a3d9f8c2b1e... 100% ▕████████████████▏ 32 GB
pulling 98c4ac90a64b... 100% ▕████████████████▏  147 B
pulling d8f8a0e2ee96... 100% ▕████████████████▏  11 KB
pulling 0ba8f0e314b4... 100% ▕████████████████▏  487 B
verifying sha256 digest
writing manifest
success
```

**📌 何が起きているのか？**
1. **pulling manifest**: モデルの構成情報をダウンロード
2. **pulling 7a3d9f8c2b1e... 32 GB**: メインのモデルファイル（最も時間がかかる）
3. **その他のファイル**: 設定ファイル、メタデータ
4. **verifying sha256 digest**: ダウンロードが正常か検証
5. **success**: 完了

**ダウンロード時間の目安**
- 光回線（100Mbps）: 約45-60分
- 高速回線（1Gbps）: 約5-8分

**ステップ2: ダウンロード確認**

```bash
# インストール済みモデルを確認
ollama list

# 出力例:
# NAME                         ID              SIZE    MODIFIED
# qwen3-coder:30b-a3b-q8_0     7a3d9f8c2b1e    32 GB   2 minutes ago
```

### 2.2.3 追加モデルのダウンロード（オプション）

**軽量モデル（高速動作・メモリ制約時）**

```bash
# Qwen3 Coder 7B（高速・軽量）
ollama pull qwen3-coder:7b

# Qwen3 Coder 14B（バランス型）
ollama pull qwen3-coder:14b
```

**代替モデル**

```bash
# DeepSeek Coder V2 16B（MoE、128Kコンテキスト）
ollama pull deepseek-coder-v2:16b
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
# Qwen3 Coder 30B Q8_0を起動
ollama run qwen3-coder:30b-a3b-q8_0
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
  "model": "qwen3-coder:30b-a3b-q8_0",
  "prompt": "Write a hello world program in Python",
  "stream": false
}'
```

**出力例**
```json
{
  "model": "qwen3-coder:30b-a3b-q8_0",
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
    "model": "qwen3-coder:30b-a3b-q8_0",
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
# Tokens per second: 22.34
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
    model = "qwen3-coder:30b-a3b-q8_0"
    prompt = "Write a Python function to implement binary search algorithm with comments"

    benchmark_ollama(model, prompt, iterations=5)
```

**MS-S1 Maxでの実測結果（2025年11月）**

```
=== Benchmark Results for qwen3-coder:30b-a3b-q8_0 ===
Prompt: Write a Python function to implement binary search...

Total Time:
  Average: 7.82s
  Median:  7.65s
  Min:     7.23s
  Max:     8.54s

Tokens per Second:
  Average: 22.15
  Median:  22.34

Prompt Eval Time: 0.28s (160 tokens/s)
Generation Time:  7.54s (22 tokens/s)
```

## 2.4 Ollamaの基本操作

### 2.4.1 モデル管理コマンド

**モデル一覧表示**

```bash
ollama list

# 出力例:
# NAME                         ID              SIZE    MODIFIED
# qwen3-coder:30b-a3b-q8_0     7a3d9f8c2b1e    32 GB   5 minutes ago
# qwen3-coder:7b               b1c2d3e4f5g6    8 GB    10 minutes ago
```

**モデル削除**

```bash
# 不要なモデルを削除してディスク容量を節約
ollama rm qwen3-coder:7b

# 確認
# Deleted 'qwen3-coder:7b'
```

**モデル情報表示**

```bash
# モデルの詳細情報を表示
ollama show qwen3-coder:30b-a3b-q8_0

# 出力例:
# Model
#   architecture        qwen3
#   parameters          30B (3.3B active MoE)
#   quantization        Q8_0
#   context length      262144  # 256K tokens
#   embedding length    5120
```

**実行中のモデル確認**

```bash
# 現在メモリにロードされているモデルを確認
ollama ps

# 出力例:
# NAME                         ID              SIZE      UNTIL
# qwen3-coder:30b-a3b-q8_0     7a3d9f8c2b1e    34 GB     5 minutes from now
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
ollama pull qwen3-coder:30b-a3b-q8_0
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

**メモリ不足の場合（MS-S1 Maxでは通常不要）**

```bash
# より小さいモデルを使用（メモリ制約がある場合のみ）
ollama pull qwen3-coder:7b  # 10GB
ollama pull qwen3-coder:14b  # 20GB

# またはKEEP_ALIVEを短縮
sudo nano /etc/systemd/system/ollama.service.d/override.conf
# Environment="OLLAMA_KEEP_ALIVE=1m"

# 注: MS-S1 Maxは128GBメモリと96GB VRAM設定可能なので、
#     Qwen3 Coder 30B Q8_0 (32GB) を問題なく実行できます
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
✅ MS-S1 Max向けROCm設定（96GB VRAM対応）
✅ qwen3-coder:30b-a3b-q8_0のダウンロード（32GB、256Kコンテキスト）
✅ CLI/APIでの動作確認
✅ パフォーマンス測定（22 tokens/s生成速度確認）

**次のステップ**
次章では、LiteLLMをインストールし、OllamaとClaude Codeの橋渡しを設定します。LiteLLMプロキシを起動すれば、Claude CodeからQwen3 Coder 30B Q8_0をローカルで使えるようになります。

**確認チェックリスト**
- [ ] `ollama --version`が動作する
- [ ] `ollama list`でqwen3-coder:30b-a3b-q8_0が表示される
- [ ] `ollama run qwen3-coder:30b-a3b-q8_0`で対話できる
- [ ] `rocm-smi`でGPUが認識されている（Radeon 8060S）
- [ ] APIテスト（curl）が成功する（22 tokens/s前後）
- [ ] 256Kコンテキストが利用可能

すべてチェックできたら、Chapter 03へ進みましょう！
