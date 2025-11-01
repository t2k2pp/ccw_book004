# Chapter 06: モデル選択と最適化

## 6.1 コーディング特化モデルの比較

### 6.1.1 主要モデルの特性

MS-S1 Maxで動作する主なコーディング特化モデルを比較します。

**Qwen2.5-Coder シリーズ**

| モデル | サイズ | メモリ | 速度 | HumanEval | コンテキスト | 特徴 |
|--------|--------|--------|------|-----------|--------------|------|
| qwen2.5-coder:3b | 2.2GB | 3.8GB | 42 tokens/s | 65.2% | 32K | 最軽量 |
| qwen2.5-coder:7b | 4.7GB | 5.8GB | 32 tokens/s | 85.5% | 32K | バランス |
| **qwen2.5-coder:14b** | 8.9GB | 11.2GB | 18 tokens/s | 88.9% | 32K | **推奨** |
| qwen2.5-coder:32b | 19GB | 24GB | 8 tokens/s | 92.1% | 32K | 最高品質 |

**DeepSeek-Coder シリーズ**

| モデル | サイズ | メモリ | 速度 | HumanEval | コンテキスト |
|--------|--------|--------|------|-----------|--------------|
| deepseek-coder:1.3b | 1.3GB | 2.5GB | 55 tokens/s | 52.3% | 16K |
| deepseek-coder:6.7b | 3.8GB | 5.2GB | 35 tokens/s | 78.6% | 16K |
| deepseek-coder:33b | 19GB | 23GB | 7 tokens/s | 82.1% | 16K |

**CodeLlama シリーズ**

| モデル | サイズ | メモリ | 速度 | HumanEval | コンテキスト |
|--------|--------|--------|------|-----------|--------------|
| codellama:7b | 3.8GB | 5.0GB | 33 tokens/s | 48.8% | 16K |
| codellama:13b | 7.4GB | 10GB | 20 tokens/s | 50.6% | 16K |
| codellama:34b | 19GB | 23GB | 8 tokens/s | 53.7% | 16K |

**StarCoder2 シリーズ**

| モデル | サイズ | メモリ | 速度 | HumanEval | コンテキスト |
|--------|--------|--------|------|-----------|--------------|
| starcoder2:3b | 1.7GB | 3.0GB | 45 tokens/s | 31.7% | 16K |
| starcoder2:7b | 4.1GB | 5.5GB | 30 tokens/s | 35.4% | 16K |
| starcoder2:15b | 8.7GB | 11.5GB | 16 tokens/s | 46.2% | 16K |

### 6.1.2 実測パフォーマンス（MS-S1 Max）

```python
# benchmark_models.py
import ollama
import time

models = [
    "qwen2.5-coder:7b",
    "qwen2.5-coder:14b",
    "deepseek-coder:6.7b",
    "codellama:13b"
]

prompt = "Write a Python function to implement binary search with type hints and docstring"

for model in models:
    print(f"\n=== {model} ===")

    start = time.time()
    response = ollama.chat(model=model, messages=[
        {"role": "user", "content": prompt}
    ])
    end = time.time()

    content = response['message']['content']
    tokens = response.get('eval_count', 0)
    duration = end - start

    print(f"Time: {duration:.2f}s")
    print(f"Tokens: {tokens}")
    print(f"Speed: {tokens/duration:.2f} tokens/s")
    print(f"Response length: {len(content)} chars")
```

**実測結果（MS-S1 Max、Radeon 8060S）**

```
=== qwen2.5-coder:7b ===
Time: 4.82s
Tokens: 156
Speed: 32.37 tokens/s
Response length: 892 chars

=== qwen2.5-coder:14b ===
Time: 8.45s
Tokens: 158
Speed: 18.70 tokens/s
Response length: 1024 chars

=== deepseek-coder:6.7b ===
Time: 4.51s
Tokens: 142
Speed: 31.49 tokens/s
Response length: 785 chars

=== codellama:13b ===
Time: 7.89s
Tokens: 135
Speed: 17.11 tokens/s
Response length: 723 chars
```

## 6.2 用途別の推奨モデル

### 6.2.1 高速開発（対話重視）

**推奨**: qwen2.5-coder:7b

```bash
ollama pull qwen2.5-coder:7b

# config.yaml
model_list:
  - model_name: gpt-3.5-turbo
    litellm_params:
      model: ollama/qwen2.5-coder:7b
```

**メリット**
- 32 tokens/s の高速応答
- メモリ使用量が少ない（5.8GB）
- 複数セッション可能

**最適な用途**
- クイックプロトタイピング
- 簡単なバグ修正
- コメント追加
- リファクタリング提案

### 6.2.2 バランス型（推奨）

**推奨**: qwen2.5-coder:14b

```bash
ollama pull qwen2.5-coder:14b

# config.yaml
model_list:
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen2.5-coder:14b
```

**メリット**
- 高い精度（HumanEval 88.9%）
- 日本語品質が高い
- 18 tokens/s の実用的な速度
- 128GBメモリで余裕の運用

**最適な用途**
- 通常の開発作業全般
- コードレビュー
- テスト生成
- ドキュメント作成

### 6.2.3 最高品質

**推奨**: qwen2.5-coder:32b

```bash
ollama pull qwen2.5-coder:32b

# config.yaml
model_list:
  - model_name: gpt-4
    litellm_params:
      model: ollama/qwen2.5-coder:32b
```

**メリット**
- 最高精度（HumanEval 92.1%）
- 複雑な推論が得意
- MS-S1 Maxでも動作（24GB）

**最適な用途**
- アーキテクチャ設計
- 複雑なアルゴリズム実装
- セキュリティ監査
- 最終的なコードレビュー

### 6.2.4 複数モデルの併用（推奨）

```yaml
# config.yaml（最適構成）
model_list:
  # 高速タスク用
  - model_name: gpt-3.5-turbo
    litellm_params:
      model: ollama/qwen2.5-coder:7b
      temperature: 0.3

  # 通常タスク用（デフォルト）
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen2.5-coder:14b
      temperature: 0.7

  # 高品質タスク用
  - model_name: gpt-4
    litellm_params:
      model: ollama/qwen2.5-coder:32b
      temperature: 0.5

  # 汎用タスク用
  - model_name: claude-3-haiku-20240307
    litellm_params:
      model: ollama/qwen2.5:14b
      temperature: 0.7
```

**使い分け例**

```bash
# 簡単なタスク
aider --model gpt-3.5-turbo
> Add type hints to this function

# 通常タスク
aider --model claude-3-5-sonnet-20241022
> Refactor this module for better readability

# 複雑なタスク
aider --model gpt-4
> Design a scalable architecture for this microservice
```

## 6.3 MS-S1 Max最適化

### 6.3.1 ROCm環境変数の最適化

```bash
# /etc/systemd/system/ollama.service.d/override.conf

[Service]
# GPU認識
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
Environment="PYTORCH_ROCM_ARCH=gfx1100"

# メモリ最適化
Environment="GPU_MAX_ALLOC_PERCENT=95"
Environment="HSA_XNACK=1"

# パフォーマンス最適化
Environment="OLLAMA_NUM_PARALLEL=2"
Environment="OLLAMA_MAX_LOADED_MODELS=2"
Environment="OLLAMA_KEEP_ALIVE=5m"

# ROCmパス
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
Environment="ROCM_PATH=/opt/rocm"
```

再起動：

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### 6.3.2 同時実行の最適化

MS-S1 Maxの128GBメモリを活用して、複数モデルを同時実行できます。

```bash
# モデル1: 開発用（Port 11434）
systemctl start ollama

# モデル2: レビュー用（Port 11435）
# 別のOllamaインスタンスを起動
OLLAMA_HOST=0.0.0.0:11435 ollama serve &

# LiteLLM config.yaml
model_list:
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen2.5-coder:14b
      api_base: http://localhost:11434  # 開発用

  - model_name: gpt-4
    litellm_params:
      model: ollama/qwen2.5-coder:32b
      api_base: http://localhost:11435  # レビュー用
```

**メモリ使用量**
- qwen2.5-coder:14b: 11.2GB
- qwen2.5-coder:32b: 24GB
- 合計: 35.2GB（128GBの27%）

### 6.3.3 キャッシュ戦略

```yaml
# config.yaml
litellm_settings:
  cache: true
  cache_params:
    type: redis
    host: localhost
    port: 6379
    ttl: 3600

  # セマンティックキャッシュ（類似クエリをキャッシュ）
  enable_semantic_caching: true
  semantic_cache_threshold: 0.95
```

**効果**

```python
# 初回（キャッシュなし）
> Add error handling to this function
# 応答時間: 8.2秒

# 2回目（キャッシュヒット）
> Add error handling to that function  # ほぼ同じ
# 応答時間: 0.3秒（27倍高速）
```

### 6.3.4 コンテキストサイズの調整

```yaml
# ~/.aider.conf.yml

# デフォルト
map-tokens: 4096
max-chat-history-tokens: 8192

# MS-S1 Max最適化（大容量メモリ活用）
map-tokens: 12288  # 3倍
max-chat-history-tokens: 24576  # 3倍

# 注意: トークン数を増やすと推論時間が増加
# 必要に応じて調整
```

## 6.4 パフォーマンスチューニング

### 6.4.1 温度（Temperature）の調整

```yaml
# config.yaml
model_list:
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen2.5-coder:14b
      temperature: 0.7  # デフォルト

      # 用途別の推奨値:
      # 0.0-0.3: 決定論的（コード生成）
      # 0.3-0.7: バランス（通常開発）
      # 0.7-1.0: 創造的（アイデア出し）
```

**温度別の挙動**

```
# Temperature 0.0（決定論的）
> Generate a function to sort a list
→ 常に同じ実装（クイックソート）

# Temperature 0.7（バランス）
> Generate a function to sort a list
→ 時々異なるアプローチ（マージソート、ヒープソート等）

# Temperature 1.0（創造的）
> Generate a function to sort a list
→ 多様な実装、時々独創的
```

### 6.4.2 Batch処理の活用

```python
# batch_code_review.py
import ollama
from concurrent.futures import ThreadPoolExecutor
import glob

def review_file(file_path):
    """単一ファイルをレビュー"""
    with open(file_path, 'r') as f:
        code = f.read()

    response = ollama.chat(
        model="qwen2.5-coder:14b",
        messages=[{
            "role": "user",
            "content": f"Review this Python code:\n\n{code}"
        }]
    )

    return {
        "file": file_path,
        "review": response['message']['content']
    }

# 並列レビュー（MS-S1 Maxの16コアを活用）
files = glob.glob("src/**/*.py", recursive=True)

with ThreadPoolExecutor(max_workers=4) as executor:
    reviews = list(executor.map(review_file, files))

for review in reviews:
    print(f"\n=== {review['file']} ===")
    print(review['review'])
```

### 6.4.3 プロンプトの最適化

**❌ 非効率なプロンプト**

```
> fix bug
```

**✅ 最適化されたプロンプト**

```
> This function has a bug where it crashes on empty input.
> Fix it by:
> 1. Adding input validation
> 2. Returning None for empty input
> 3. Adding a docstring explaining the behavior
```

**効果**: 応答精度が大幅に向上、やり直しが減少

## 6.5 モニタリングとベンチマーク

### 6.5.1 リアルタイムモニタリング

```bash
# GPU使用率を監視
watch -n 1 rocm-smi

# メモリ使用量を監視
watch -n 1 free -h

# Ollamaプロセスを監視
watch -n 1 'ollama ps'
```

### 6.5.2 ベンチマークスクリプト

```python
# comprehensive_benchmark.py
import ollama
import time
import statistics

def benchmark_model(model, test_cases):
    """包括的なベンチマーク"""
    results = {
        'model': model,
        'response_times': [],
        'tokens_per_second': [],
        'response_lengths': []
    }

    for i, test_case in enumerate(test_cases, 1):
        print(f"Test {i}/{len(test_cases)}: {test_case['name']}")

        start = time.time()
        response = ollama.chat(
            model=model,
            messages=[{"role": "user", "content": test_case['prompt']}]
        )
        end = time.time()

        duration = end - start
        tokens = response.get('eval_count', 0)
        content = response['message']['content']

        results['response_times'].append(duration)
        results['tokens_per_second'].append(tokens / duration if duration > 0 else 0)
        results['response_lengths'].append(len(content))

    # 統計
    print(f"\n=== Results for {model} ===")
    print(f"Avg Response Time: {statistics.mean(results['response_times']):.2f}s")
    print(f"Avg Tokens/s: {statistics.mean(results['tokens_per_second']):.2f}")
    print(f"Avg Response Length: {int(statistics.mean(results['response_lengths']))} chars")

    return results

# テストケース
test_cases = [
    {"name": "Simple function", "prompt": "Write a function to reverse a string"},
    {"name": "Complex algorithm", "prompt": "Implement quicksort with type hints"},
    {"name": "Bug fix", "prompt": "Fix the IndexError in this code: def f(l): return l[10]"},
    {"name": "Refactoring", "prompt": "Refactor this nested loop to use list comprehension"},
    {"name": "Documentation", "prompt": "Add comprehensive docstrings to this module"}
]

# ベンチマーク実行
models = ["qwen2.5-coder:7b", "qwen2.5-coder:14b"]
for model in models:
    benchmark_model(model, test_cases)
```

## 6.6 まとめ

本章では、MS-S1 Maxに最適なモデル選択と最適化を学びました。

**推奨構成**
- **高速タスク**: qwen2.5-coder:7b（32 tokens/s）
- **通常タスク**: qwen2.5-coder:14b（18 tokens/s）← **デフォルト推奨**
- **高品質タスク**: qwen2.5-coder:32b（8 tokens/s）

**MS-S1 Max活用のポイント**
- 128GBメモリで複数モデル同時実行
- ROCm最適化でGPU加速
- Redisキャッシュで高速化
- 並列処理で生産性向上

**次のステップ**
次章では、実際のプロジェクトでの具体的な活用例を学びます。

**確認チェックリスト**
- [ ] 用途に合ったモデルを選択できる
- [ ] ROCm環境変数を最適化している
- [ ] パフォーマンスをモニタリングできる
- [ ] 複数モデルを使い分けられる

すべてチェックできたら、Chapter 07へ進みましょう！
