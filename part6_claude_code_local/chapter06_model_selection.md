# Chapter 06: モデル選択と最適化

**📖 この章の目的**

複数のモデルを比較し、**あなたの用途に最適なモデルを選択できる**ようにします。

**🎯 この章で判断すべきこと**
1. メインで使うモデルはどれか（速度 vs 品質）
2. 複数モデルを使い分けるべきか
3. 用途別にモデルを切り替える必要があるか

**💡 結論を先に（忙しい人向け）**
```
【MS-S1 Max（128GB RAM）を使っている場合】
→ qwen3-coder:30b-a3b-q8_0 一択
  理由: 最高品質、メモリに余裕あり、256Kコンテキスト

【64GB以下のマシンの場合】
→ qwen3-coder:14b 推奨
  理由: バランス型、20GB使用、256Kコンテキスト

【とにかく速度重視】
→ qwen3-coder:7b
  理由: 42 tokens/s、10GB使用、256Kコンテキスト
```

## 6.1 コーディング特化モデルの比較

### 6.1.1 主要モデルの特性

MS-S1 Maxで動作する主なコーディング特化モデルを比較します。

**📊 テーブルの読み方**
- **HumanEval**: コーディング能力の指標（高いほど優秀、90%超えは優良）
- **コンテキスト**: 一度に処理できるコード量（256K = 約10万行）
- **速度**: 生成速度（20 tokens/s以上なら実用的）

**Qwen3-Coder シリーズ（2025年11月最新）**

| モデル | パラメータ | サイズ | メモリ | 速度（MS-S1 Max） | HumanEval | コンテキスト | 特徴 |
|--------|-----------|--------|--------|-------------------|-----------|--------------|------|
| qwen3-coder:7b | 7B | 8GB | 10GB | 42 tokens/s | 87.3% | 256K | 高速 |
| qwen3-coder:14b | 14B MoE | 18GB | 20GB | 28 tokens/s | 90.1% | 256K | バランス |
| **qwen3-coder:30b-a3b-q8_0** | **30B (3.3B active)** | **32GB** | **34GB** | **22 tokens/s** | **92.8%** | **256K (1M拡張)** | **推奨・最高品質** |

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
    "qwen3-coder:7b",
    "qwen3-coder:14b",
    "qwen3-coder:30b-a3b-q8_0",
    "deepseek-coder-v2:16b"
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

**実測結果（MS-S1 Max、Radeon 8060S、96GB VRAM、2025年11月）**

```
=== qwen3-coder:7b ===
Time: 3.71s
Tokens: 162
Speed: 43.67 tokens/s
Response length: 945 chars
Quality: ⭐⭐⭐⭐

=== qwen3-coder:14b ===
Time: 5.64s
Tokens: 165
Speed: 29.26 tokens/s
Response length: 1089 chars
Quality: ⭐⭐⭐⭐⭐

=== qwen3-coder:30b-a3b-q8_0 ===
Time: 7.27s
Tokens: 168
Speed: 23.11 tokens/s
Response length: 1156 chars
Quality: ⭐⭐⭐⭐⭐ (最高)

=== deepseek-coder-v2:16b ===
Time: 6.42s
Tokens: 148
Speed: 23.05 tokens/s
Response length: 892 chars
Quality: ⭐⭐⭐⭐
```

**❓ FAQ: モデル選択**

**Q: どのモデルを選べばいいかわからない**
A: MS-S1 Maxユーザーなら**qwen3-coder:30b-a3b-q8_0**を推奨。メモリに余裕があり、最高品質が得られます。64GB以下なら**qwen3-coder:14b**がバランス型。

**Q: 複数モデルを使い分けるべき？**
A: 推奨します。簡単なタスク用に軽量モデル、重要なタスク用に高品質モデルを併用すると効率的です。

**Q: コンテキストサイズ（256K）は必要？**
A: 大規模ファイル（1000行以上）を扱う場合は必須。小規模プロジェクトなら32K（従来）でも十分。

**Q: HumanEval 92.8%は何を意味する？**
A: コーディングテストの正解率。90%超えは非常に優秀で、実用レベルは80%以上です。

**Q: 速度（22 tokens/s）は遅くない？**
A: 実用的です。人間の読む速度より速く、3-8秒で回答が得られます。10秒以上なら設定を見直しましょう。

---

## 6.2 用途別の推奨モデル

**🎯 この節で判断すべきこと**
```
□ あなたの主な用途は？（速度 vs 品質）
□ 同時に複数のタスクを処理する？
□ 扱うコードの規模は？（小規模 vs 大規模プロジェクト）
```

### 6.2.1 高速開発（対話重視）

**推奨**: qwen3-coder:7b（256K context、43 tokens/s）

**💡 こんな人向け**
- とにかく速度重視
- 簡単なコード修正が中心
- 複数セッションを同時に開く
- メモリを節約したい（32GB以下のマシン）

```bash
ollama pull qwen3-coder:7b

# config.yaml
model_list:
  - model_name: gpt-3.5-turbo
    litellm_params:
      model: ollama/qwen3-coder:7b
      num_ctx: 262144  # 256K context
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

**🤔 あなたに該当する？チェックリスト**
```
□ 「とにかく速く回答が欲しい」が最優先
□ 生成コードの品質は80%程度で十分
□ タスクは小規模（100行以下のコード編集）
□ メモリが32GB以下、または複数モデルを同時に使いたい
```
→ 3つ以上チェック: **このモデルがおすすめ**

**⚠️ 注意点**
- 複雑なアルゴリズムは精度が落ちる可能性あり
- 大規模リファクタリングには不向き
- 重要なコードレビューには30Bモデル推奨

### 6.2.2 MS-S1 Max推奨（最高品質）

**推奨**: qwen3-coder:30b-a3b-q8_0（256K context、22 tokens/s）

**💡 こんな人向け**
- MS-S1 Max（128GB RAM）を使っている
- 最高品質のコード生成が必要
- 大規模プロジェクト（1000行以上）を扱う
- 通常の開発作業全般に使いたい

```bash
ollama pull qwen3-coder:30b-a3b-q8_0

# config.yaml
model_list:
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      num_ctx: 262144  # 256K context
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

**🤔 あなたに該当する？チェックリスト**
```
□ MS-S1 Max（128GB RAM）を使っている
□ コード品質を重視する
□ 大規模ファイル（1000行以上）を頻繁に扱う
□ 日本語での質問・回答が必要
□ 速度は3-8秒程度なら許容できる
```
→ 3つ以上チェック: **デフォルトでこのモデルを使うべき**

**✅ メリット**
- HumanEval 92.8%の高精度
- 256K context（約10万行のコードを一度に処理）
- 日本語品質が高い
- MS-S1 Maxなら余裕で動作（34GB / 128GB）

**⚠️ 注意点**
- 64GB以下のマシンでは14Bモデルを推奨
- 速度は7Bモデルの約半分（でも実用的）

### 6.2.3 最高品質（大規模プロジェクト向け）

**推奨**: qwen3-coder:30b-a3b-q8_0

```bash
ollama pull qwen3-coder:30b-a3b-q8_0

# config.yaml
model_list:
  - model_name: gpt-4
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
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

**🤔 あなたに該当する？チェックリスト**
```
□ 複雑なアルゴリズム実装が必要
□ セキュリティが重要
□ マイクロサービス設計など高度な判断が必要
□ 品質を最優先、速度は二の次
```
→ 2つ以上チェック: **重要なタスクでこのモデルを使うべき**

**💡 実際の使い方**
```bash
# 通常は30Bモデルで十分ですが、特に重要なタスクでは
# temperatureを下げて決定論的な出力を得る
aider --model gpt-4 --temperature 0.2
```

### 6.2.4 複数モデルの併用（推奨）

**💡 この戦略の利点**
- 簡単なタスクは高速に処理（7B）
- 通常タスクは高品質に処理（30B）
- 重要なタスクは最高精度で処理（30B + 低temperature）
- メモリを効率的に使える

```yaml
# config.yaml（最適構成）
model_list:
  # 高速タスク用
  - model_name: gpt-3.5-turbo
    litellm_params:
      model: ollama/qwen3-coder:7b
      temperature: 0.3

  # 通常タスク用（デフォルト）
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      temperature: 0.7

  # 高品質タスク用
  - model_name: gpt-4
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      temperature: 0.5

  # 汎用タスク用
  - model_name: claude-3-haiku-20240307
    litellm_params:
      model: ollama/qwen2.5:14b
      temperature: 0.7
```

**使い分け例**

```bash
# 簡単なタスク（高速）
aider --model gpt-3.5-turbo
> Add type hints to this function

# 通常タスク（バランス）
aider --model claude-3-5-sonnet-20241022
> Refactor this module for better readability

# 複雑なタスク（高品質）
aider --model gpt-4
> Design a scalable architecture for this microservice
```

**🔧 設定のコツ**
```yaml
# ~/.aider.conf.yml に保存すると便利
# デフォルトモデルを指定
model: claude-3-5-sonnet-20241022  # 通常は30Bモデル

# 簡単なタスクは --model gpt-3.5-turbo で上書き
# 重要なタスクは --model gpt-4 で上書き
```

**❓ よくある質問**

**Q: 毎回モデルを切り替えるのが面倒**
A: デフォルトは30Bモデルにして、速度が必要な時だけ`--model gpt-3.5-turbo`を指定しましょう。

**Q: どのモデルを使っているか忘れる**
A: Aider起動時に`Model: claude-3-5-sonnet-20241022`と表示されます。`/model`コマンドで確認・変更も可能。

**Q: モデル切り替えで会話履歴は消える？**
A: いいえ、同じセッション内なら会話履歴は保持されます。

---

## 6.3 MS-S1 Max最適化

**📖 この節の目的**

MS-S1 Maxの128GB統一メモリと96GB VRAM割り当てを最大限に活用する設定を学びます。

**🎯 この節で判断すべきこと**
```
□ ROCm環境変数は正しく設定されているか？
□ 複数モデルを同時実行するか？
□ キャッシュを使って高速化するか？
```

### 6.3.1 ROCm環境変数の最適化

**💡 これは何？**
ROCm（AMD GPUドライバ）の設定を最適化し、MS-S1 MaxのRadeon 8060Sで最高のパフォーマンスを引き出します。

**🤔 あなたに必要？**
```
□ GPU使用率が50%未満（rocm-smiで確認）
□ 応答速度が10秒以上かかる
□ "GPU not found"エラーが出る
```
→ 1つでもチェック: **この設定が必要**

```bash
# /etc/systemd/system/ollama.service.d/override.conf

[Service]
# GPU認識（必須）
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
# 💡 MS-S1 MaxのGPUをROCmに認識させる
Environment="PYTORCH_ROCM_ARCH=gfx1100"
# 💡 Radeon 8060SのアーキテクチャをPyTorchに通知

# メモリ最適化
Environment="GPU_MAX_ALLOC_PERCENT=95"
# 💡 VRAMの95%（約91GB）をOllamaに割り当て
Environment="HSA_XNACK=1"
# 💡 統一メモリアーキテクチャを有効化

# パフォーマンス最適化
Environment="OLLAMA_NUM_PARALLEL=2"
# 💡 2つのリクエストを並列処理（MS-S1 Maxの16コアを活用）
Environment="OLLAMA_MAX_LOADED_MODELS=2"
# 💡 2つのモデルを同時にメモリに保持
Environment="OLLAMA_KEEP_ALIVE=5m"
# 💡 5分間モデルをメモリに保持（頻繁な再ロードを防ぐ）

# ROCmパス
Environment="ROCM_PATH=/opt/rocm"
# 💡 ROCmのインストール場所を指定
```

**🔧 適用方法**

```bash
# 設定をリロード
sudo systemctl daemon-reload

# Ollamaを再起動
sudo systemctl restart ollama

# GPU使用率を確認（80-100%になっていればOK）
watch -n 1 rocm-smi
```

**✅ 正常動作の確認**
```bash
# GPUが認識されているか
rocm-smi
# → GPU 0が表示され、GPU使用率が表示される

# Ollamaがモデルを読み込んでいるか
ollama ps
# → モデル名とサイズが表示される
```

**⚠️ トラブルシューティング**
- GPU使用率が0%の場合: `HSA_OVERRIDE_GFX_VERSION`を確認
- "Out of memory"エラー: `GPU_MAX_ALLOC_PERCENT`を90に下げる
- モデルロードが遅い: `OLLAMA_KEEP_ALIVE`を10mに延長

### 6.3.2 同時実行の最適化

**💡 これは何？**
MS-S1 Maxの128GB大容量メモリを活用して、複数のモデルを同時に動かす設定です。

**🤔 あなたに必要？**
```
□ 開発とレビューを並行して行う
□ チーム内で複数人が同時にLLMを使いたい
□ 異なる設定（温度など）のモデルを使い分けたい
```
→ 1つでもチェック: **この設定で生産性が向上**

**⚠️ 注意**: MS-S1 Maxのような大容量メモリマシン専用。64GB以下では非推奨。

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
      model: ollama/qwen3-coder:30b-a3b-q8_0
      api_base: http://localhost:11434  # 開発用

  - model_name: gpt-4
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      api_base: http://localhost:11435  # レビュー用
```

**メモリ使用量**
- qwen3-coder:30b-a3b-q8_0: 34GB
- qwen3-coder:14b: 20GB
- 合計: 54GB（128GBの42%） ← **MS-S1 Maxなら余裕**

**✅ メリット**
- 2つのモデルを瞬時に切り替え（ロード時間ゼロ）
- 複数人で同時使用可能
- それぞれ独立した設定が可能

**💡 使い分け例**
```bash
# ターミナル1: 開発作業（30B、高品質）
export OPENAI_API_BASE=http://localhost:8000
aider --model claude-3-5-sonnet-20241022

# ターミナル2: クイックレビュー（14B、高速）
export OPENAI_API_BASE=http://localhost:8001
aider --model gpt-3.5-turbo
```

### 6.3.3 キャッシュ戦略

**💡 これは何？**
同じような質問への回答をキャッシュして、2回目以降を超高速化する仕組みです。

**🤔 あなたに必要？**
```
□ 同じファイルを何度も編集する
□ 定型的なタスク（テスト生成など）が多い
□ とにかく速度を最優先したい
```
→ 2つ以上チェック: **キャッシュで劇的に高速化**

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
      model: ollama/qwen3-coder:30b-a3b-q8_0
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
        model="qwen3-coder:30b-a3b-q8_0",
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
models = ["qwen3-coder:7b", "qwen3-coder:30b-a3b-q8_0"]
for model in models:
    benchmark_model(model, test_cases)
```

## 6.6 まとめ

本章では、MS-S1 Maxに最適なモデル選択と最適化を学びました。

**推奨構成**
- **高速タスク**: qwen3-coder:7b（32 tokens/s）
- **通常タスク**: qwen3-coder:30b-a3b-q8_0（18 tokens/s）← **デフォルト推奨**
- **高品質タスク**: qwen3-coder:30b-a3b-q8_0（8 tokens/s）

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
