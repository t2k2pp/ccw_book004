# 第7章:MS-S1 Max向けパフォーマンス最適化

## 7.1 ハードウェアリソースの最大活用

### 7.1.1 128GB メモリの戦略的活用

MS-S1 Maxの128GB大容量メモリを最大限に活用します。

```bash
# システム全体のメモリ配分戦略
┌─────────────────────────────────────────┐
│ 総メモリ: 128GB                          │
├─────────────────────────────────────────┤
│ OS + システム: 8GB                       │
│ Ollama サービス: 2GB                    │
│ モデルウェイト: 40-80GB (可変)          │
│ コンテキストキャッシュ: 10-30GB         │
│ 作業用バッファ: 10-20GB                 │
│ 予備: 10-20GB                           │
└─────────────────────────────────────────┘
```

### 7.1.2 複数モデルの同時実行

```bash
# 環境変数設定（最大3モデル同時ロード）
sudo nano /etc/systemd/system/ollama.service.d/performance.conf
```

```ini
[Service]
Environment="OLLAMA_MAX_LOADED_MODELS=3"
Environment="OLLAMA_NUM_PARALLEL=4"
Environment="OLLAMA_KEEP_ALIVE=30m"
```

**推奨構成例:**
```bash
# 同時実行可能な組み合わせ
1. qwen2.5:32b (20GB) + qwen2.5:14b (9GB) + qwen2.5:7b (5GB) = 34GB
2. llama3.1:70b (42GB) + qwen2.5:14b (9GB) = 51GB
3. qwen2.5:7b × 3本 + codellama:13b = 23GB
```

### 7.1.3 コンテキストキャッシュの最適化

```bash
# 長時間実行用設定
OLLAMA_KEEP_ALIVE=60m  # モデルを60分メモリ保持
```

```python
# Python での設定
import ollama

# 長いコンテキストを維持
ollama.generate(
    model='qwen2.5:32b',
    prompt='...',
    keep_alive='3600s'  # 1時間
)
```

## 7.2 GPU最適化

### 7.2.1 ROCmメモリ管理

```bash
# ~/.bashrc に追加
export GPU_MAX_HEAP_SIZE=100
export GPU_MAX_ALLOC_PERCENT=100
export HSA_FORCE_FINE_GRAIN_PCIE=1
```

### 7.2.2 バッチサイズの調整

```dockerfile
# Modelfile でのバッチサイズ最適化
FROM llama3.1:70b

PARAMETER num_batch 512    # MS-S1 Maxでは512-1024が最適
PARAMETER num_thread 32     # 全コア活用
PARAMETER num_gpu 1
```

### 7.2.3 Flash Attention の活用

```bash
# Flash Attention 2 を有効化（メモリ効率30%向上）
export OLLAMA_FLASH_ATTENTION=1
```

**効果測定:**
```bash
# Flash Attention なし
time ollama run llama3.1:70b "Write 500 words"
# → 45秒

# Flash Attention あり
OLLAMA_FLASH_ATTENTION=1 time ollama run llama3.1:70b "Write 500 words"
# → 32秒（29%高速化）
```

## 7.3 プロンプト処理の最適化

### 7.3.1 コンテキスト長の適切な設定

```python
# ベンチマーク: コンテキスト長と速度の関係
import ollama
import time

context_lengths = [2048, 4096, 8192, 16384, 32768]

for ctx_len in context_lengths:
    start = time.time()

    ollama.generate(
        model='qwen2.5:14b',
        prompt='Hello' * 100,
        options={'num_ctx': ctx_len}
    )

    elapsed = time.time() - start
    print(f"Context: {ctx_len}, Time: {elapsed:.2f}s")
```

**MS-S1 Max での推奨値:**
| モデル | 推奨 num_ctx | メモリ使用 | 速度 |
|--------|-------------|-----------|------|
| 7B | 32768 | ~12GB | 高速 |
| 14B | 16384 | ~15GB | 高速 |
| 32B | 8192 | ~25GB | 中速 |
| 70B | 8192 | ~50GB | 中速 |

### 7.3.2 プロンプトキャッシング

```python
# 頻繁に使うプロンプトはキャッシュ
import ollama
from functools import lru_cache

@lru_cache(maxsize=100)
def cached_generate(model, prompt):
    return ollama.generate(model=model, prompt=prompt)

# 使用
result1 = cached_generate('qwen2.5:7b', 'What is AI?')  # 実行
result2 = cached_generate('qwen2.5:7b', 'What is AI?')  # キャッシュから取得
```

## 7.4 量子化レベルの最適化

### 7.4.1 MS-S1 Max向け推奨設定

```bash
# 7B-14Bモデル: Q5_K_M（品質重視）
ollama pull qwen2.5:7b-instruct-q5_K_M   # 5.8GB
ollama pull qwen2.5:14b-instruct-q5_K_M  # 11GB

# 32B-34Bモデル: Q4_K_M（バランス）
ollama pull qwen2.5:32b-instruct-q4_K_M  # 19GB

# 70Bモデル: Q4_K_M（実用的）
ollama pull llama3.1:70b-instruct-q4_K_M # 41GB

# 用途別
# - 本番環境: Q5_K_M（高品質）
# - 開発・テスト: Q4_K_M（バランス）
# - 実験: Q3_K_M（小サイズ）
```

### 7.4.2 速度vs品質のベンチマーク

```bash
#!/bin/bash
# benchmark_quant.sh

MODEL="qwen2.5:7b"
PROMPT="Explain quantum computing in detail."
QUANTS=("q8_0" "q6_K" "q5_K_M" "q4_K_M" "q3_K_M")

echo "Quantization Benchmark"
echo "======================"

for quant in "${QUANTS[@]}"; do
    model_name="${MODEL}-instruct-${quant}"
    echo "Testing: $model_name"

    # ダウンロード
    ollama pull $model_name 2>/dev/null

    # 速度測定
    start=$(date +%s%N)
    response=$(ollama run $model_name "$PROMPT" 2>/dev/null)
    end=$(date +%s%N)

    elapsed=$((($end - $start) / 1000000))  # ms
    length=${#response}

    echo "  Time: ${elapsed}ms"
    echo "  Length: $length chars"
    echo "  Speed: $((length * 1000 / elapsed)) chars/s"
    echo ""
done
```

## 7.5 並行処理の最適化

### 7.5.1 マルチスレッド設定

```python
# concurrent_ollama.py
from concurrent.futures import ThreadPoolExecutor
import ollama

def process_query(query):
    return ollama.generate(model='qwen2.5:7b', prompt=query)

queries = [
    "What is AI?",
    "What is ML?",
    "What is DL?",
    "What is NLP?",
    "What is CV?"
]

# 最大4並行実行（MS-S1 Maxの場合）
with ThreadPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(process_query, queries))

for i, result in enumerate(results):
    print(f"Query {i+1}: {result['response'][:100]}...")
```

### 7.5.2 非同期処理

```python
# async_batch.py
import asyncio
import ollama

async def async_generate(prompt):
    client = ollama.AsyncClient()
    response = await client.generate(
        model='qwen2.5:7b',
        prompt=prompt
    )
    return response['response']

async def batch_process(prompts):
    tasks = [async_generate(p) for p in prompts]
    results = await asyncio.gather(*tasks)
    return results

if __name__ == '__main__':
    prompts = [f"Tell me about topic {i}" for i in range(10)]
    results = asyncio.run(batch_process(prompts))
```

## 7.6 ネットワーク最適化

### 7.6.1 ローカル接続の最適化

```bash
# UNIXソケット使用（TCPより高速）
export OLLAMA_HOST=unix:///tmp/ollama.sock
```

### 7.6.2 API レスポンスの圧縮

```python
import requests
import gzip
import json

# gzip圧縮を有効化
headers = {
    'Content-Type': 'application/json',
    'Accept-Encoding': 'gzip'
}

response = requests.post(
    'http://localhost:11434/api/generate',
    json={'model': 'llama3.1', 'prompt': 'Hello'},
    headers=headers
)
```

## 7.7 システムレベルの最適化

### 7.7.1 CPUガバナー設定

```bash
# パフォーマンスモード
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 永続化
sudo apt install cpufrequtils
sudo nano /etc/default/cpufrequtils

# 追加
GOVERNOR="performance"

sudo systemctl restart cpufrequtils
```

### 7.7.2 NUMA最適化

```bash
# NUMA情報確認
numactl --hardware

# Ollamaを特定のNUMAノードで実行
numactl --cpunodebind=0 --membind=0 ollama serve
```

### 7.7.3 IRQアフィニティ

```bash
# GPU IRQを特定のCPUに固定
# /proc/interrupts でGPU IRQ番号確認
cat /proc/interrupts | grep amdgpu

# IRQアフィニティ設定
echo "4" | sudo tee /proc/irq/[IRQ番号]/smp_affinity_list
```

## 7.8 ベンチマークツール

### 7.8.1 総合ベンチマーク

```python
# comprehensive_benchmark.py
import ollama
import time
import statistics
import json

def benchmark_model(model, test_cases, num_runs=3):
    results = {
        'model': model,
        'tests': {}
    }

    for test_name, prompt in test_cases.items():
        print(f"Testing {model} - {test_name}...")

        speeds = []
        for i in range(num_runs):
            start = time.time()

            response = ollama.generate(
                model=model,
                prompt=prompt
            )

            elapsed = time.time() - start
            tokens = response.get('eval_count', 0)
            speed = tokens / elapsed if elapsed > 0 else 0
            speeds.append(speed)

        results['tests'][test_name] = {
            'avg_speed': statistics.mean(speeds),
            'std_dev': statistics.stdev(speeds) if len(speeds) > 1 else 0,
            'min_speed': min(speeds),
            'max_speed': max(speeds)
        }

    return results

# テストケース
test_cases = {
    'short': 'Hello',
    'medium': 'Explain quantum computing in 100 words.',
    'long': 'Write a detailed 500-word essay on artificial intelligence.',
    'code': 'Write a Python function to sort a list of numbers.'
}

models = ['qwen2.5:7b', 'qwen2.5:14b', 'qwen2.5:32b']

all_results = []
for model in models:
    result = benchmark_model(model, test_cases)
    all_results.append(result)

# 結果を保存
with open('benchmark_results.json', 'w') as f:
    json.dump(all_results, f, indent=2)

print("\nBenchmark complete! Results saved to benchmark_results.json")
```

### 7.8.2 メモリプロファイリング

```python
# memory_profile.py
import ollama
import psutil
import time

def profile_memory(model, prompt):
    process = psutil.Process()

    # 開始時のメモリ
    mem_before = process.memory_info().rss / 1024**3  # GB

    start = time.time()

    response = ollama.generate(model=model, prompt=prompt)

    elapsed = time.time() - start

    # 終了時のメモリ
    mem_after = process.memory_info().rss / 1024**3  # GB
    mem_used = mem_after - mem_before

    return {
        'model': model,
        'time': elapsed,
        'memory_used_gb': mem_used,
        'tokens': response.get('eval_count', 0)
    }

# プロファイル実行
models = ['qwen2.5:7b', 'qwen2.5:14b', 'qwen2.5:32b']
prompt = "Write a 200-word essay."

for model in models:
    result = profile_memory(model, prompt)
    print(f"{model}:")
    print(f"  Time: {result['time']:.2f}s")
    print(f"  Memory: {result['memory_used_gb']:.2f}GB")
    print(f"  Speed: {result['tokens']/result['time']:.1f} tokens/s")
    print()
```

## 7.9 リアルタイム監視

### 7.9.1 Grafana + Prometheus

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'ollama'
    static_configs:
      - targets: ['localhost:11434']
```

### 7.9.2 カスタムダッシュボード

```python
# metrics_exporter.py
from prometheus_client import start_http_server, Gauge, Counter
import ollama
import time
import threading

# メトリクス定義
requests_total = Counter('ollama_requests_total', 'Total requests')
request_duration = Gauge('ollama_request_duration_seconds', 'Request duration')
tokens_per_second = Gauge('ollama_tokens_per_second', 'Generation speed')
memory_usage = Gauge('ollama_memory_usage_bytes', 'Memory usage')

def collect_metrics():
    while True:
        # メモリ使用量取得
        import psutil
        process = psutil.Process()
        memory_usage.set(process.memory_info().rss)

        time.sleep(5)

# メトリクス収集スレッド
threading.Thread(target=collect_metrics, daemon=True).start()

# Prometheusエクスポーター起動
start_http_server(8001)
print("Metrics available at http://localhost:8001/metrics")

# アプリケーションループ
while True:
    time.sleep(1)
```

## 7.10 本章のまとめ

本章では、以下の内容を学習しました。

✅ **メモリ最適化**
- 128GB活用戦略
- 複数モデル同時実行

✅ **GPU最適化**
- ROCm設定
- Flash Attention

✅ **量子化最適化**
- MS-S1 Max向け推奨設定

✅ **並行処理**
- マルチスレッド
- 非同期処理

✅ **システム最適化**
- CPUガバナー
- NUMA設定

✅ **監視とベンチマーク**
- 総合ベンチマーク
- リアルタイム監視

次章では、マルチモデル運用と同時実行の実践的なテクニックを学びます。

---

**前章へ**: [第6章 API活用と統合](chapter06_api_integration.md)
**次章へ**: [第8章 マルチモデル運用と同時実行](chapter08_multi_model.md)
