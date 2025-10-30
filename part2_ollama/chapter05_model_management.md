# 第5章:モデル管理とカスタマイズ

## 5.1 Modelfile の基礎

### 5.1.1 Modelfile とは

**Modelfile** は、カスタムモデルを定義するための設定ファイルです。Dockerfileに似た構文を使用します。

```dockerfile
# 基本的な Modelfile
FROM llama3.1

SYSTEM """
You are a helpful AI assistant.
"""

PARAMETER temperature 0.8
PARAMETER top_p 0.9
```

### 5.1.2 Modelfileの構文

#### FROM命令

```dockerfile
# ベースモデル指定
FROM llama3.1
FROM qwen2.5:14b
FROM mistral:latest

# ローカルのGGUFファイル
FROM ./models/mymodel.gguf

# 量子化レベル指定
FROM llama3.1:8b-instruct-q4_K_M
```

#### SYSTEM命令

```dockerfile
# システムプロンプト（AIの役割を定義）
SYSTEM """
あなたは経験豊富なソフトウェアエンジニアです。
コードレビューを行い、改善提案を提供します。
"""
```

#### PARAMETER命令

```dockerfile
# 推論パラメータ設定
PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER top_k 40
PARAMETER repeat_penalty 1.1
PARAMETER num_predict 2048
PARAMETER stop "<|end|>"
```

#### TEMPLATE命令

```dockerfile
# プロンプトテンプレート
TEMPLATE """{{ .System }}

User: {{ .Prompt }}
Assistant: """
```

#### MESSAGE命令

```dockerfile
# 会話履歴の事前設定
MESSAGE user "こんにちは"
MESSAGE assistant "こんにちは！お手伝いできることはありますか？"
```

### 5.1.3 完全な Modelfile 例

```dockerfile
# MS-S1 Max用カスタムアシスタント
FROM qwen2.5:14b

# システムプロンプト
SYSTEM """
あなたは親切で知識豊富な日本語AIアシスタントです。
以下の特徴があります：
- 正確で詳細な情報を提供
- 分かりやすく丁寧な説明
- 技術的な質問に強い
- MS-S1 Maxのエキスパート
"""

# パラメータ
PARAMETER temperature 0.8
PARAMETER top_p 0.9
PARAMETER top_k 40
PARAMETER repeat_penalty 1.1
PARAMETER num_ctx 8192
PARAMETER num_predict -1

# ストップシーケンス
PARAMETER stop "<|im_end|>"
PARAMETER stop "<|endoftext|>"

# テンプレート
TEMPLATE """<|im_start|>system
{{ .System }}<|im_end|>
<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
"""
```

## 5.2 カスタムモデルの作成

### 5.2.1 基本的な作成手順

```bash
# 1. Modelfile作成
nano my-assistant.Modelfile

# 2. モデル作成
ollama create my-assistant -f my-assistant.Modelfile

# 出力例
parsing modelfile
creating model layers
writing manifest
success

# 3. 確認
ollama list

# 4. 実行
ollama run my-assistant
```

### 5.2.2 専門特化モデルの作成例

#### 日本語翻訳特化

```bash
nano translator-jp.Modelfile
```

```dockerfile
FROM qwen2.5:14b

SYSTEM """
あなたは高精度な翻訳AIです。
以下のルールに従って翻訳してください：
- 自然で読みやすい日本語
- 専門用語は適切に訳す
- 文脈を考慮した翻訳
- 不明な点があれば説明を追加
"""

PARAMETER temperature 0.3
PARAMETER top_p 0.85
PARAMETER num_ctx 16384
```

```bash
# 作成と使用
ollama create translator-jp -f translator-jp.Modelfile
ollama run translator-jp "Translate to Japanese: The quick brown fox..."
```

#### コードレビュー特化

```bash
nano code-reviewer.Modelfile
```

```dockerfile
FROM qwen2.5:32b

SYSTEM """
あなたは経験豊富なシニアエンジニアです。
コードレビューを行う際は：
1. バグや潜在的な問題を指摘
2. パフォーマンス改善提案
3. 可読性向上のアドバイス
4. ベストプラクティスの適用
5. セキュリティ上の懸念事項

レビューは建設的で具体的に。
"""

PARAMETER temperature 0.5
PARAMETER top_p 0.9
PARAMETER num_ctx 16384
```

```bash
ollama create code-reviewer -f code-reviewer.Modelfile
```

#### MS-S1 Maxエキスパート

```bash
nano ms-s1-expert.Modelfile
```

```dockerfile
FROM llama3.1:8b

SYSTEM """
あなたはMS-S1 Maxのエキスパートです。
以下の情報を熟知しています：

【ハードウェア】
- AMD Ryzen AI Max+ 395 (16コア/32スレッド)
- 128GB LPDDR5X-8000
- Radeon 8060S (RDNA 3.5)
- 256GB/sメモリ帯域幅

【最適化】
- ROCm設定
- パフォーマンスチューニング
- 冷却管理

質問に対して、具体的で実践的なアドバイスを提供します。
"""

PARAMETER temperature 0.7
PARAMETER top_p 0.9
```

```bash
ollama create ms-s1-expert -f ms-s1-expert.Modelfile
```

## 5.3 高度なパラメータ設定

### 5.3.1 Temperature（創造性）

```dockerfile
# 低温度: 確定的、正確（0.1-0.5）
PARAMETER temperature 0.3  # 翻訳、要約、コードに最適

# 中温度: バランス（0.6-0.8）
PARAMETER temperature 0.7  # 一般的な会話

# 高温度: 創造的（0.9-1.5）
PARAMETER temperature 1.2  # 創作、ブレインストーミング
```

### 5.3.2 Top-P（確率閾値）

```dockerfile
# 保守的: 高確率トークンのみ
PARAMETER top_p 0.7

# バランス
PARAMETER top_p 0.9

# 多様性重視
PARAMETER top_p 0.95
```

### 5.3.3 Context Length（コンテキスト長）

```dockerfile
# 短いコンテキスト（高速）
PARAMETER num_ctx 2048

# 標準
PARAMETER num_ctx 8192

# 長いコンテキスト（MS-S1 Maxで可能）
PARAMETER num_ctx 32768

# 超長コンテキスト（大容量メモリ活用）
PARAMETER num_ctx 131072
```

**MS-S1 Maxでの推奨設定:**

| モデルサイズ | 推奨 num_ctx | メモリ使用量 |
|------------|-------------|-------------|
| 7B | 32768 | ~12GB |
| 14B | 16384 | ~15GB |
| 32B | 8192 | ~25GB |
| 70B | 8192 | ~45GB |

### 5.3.4 Repeat Penalty（繰り返し抑制）

```dockerfile
# 繰り返しを許容
PARAMETER repeat_penalty 1.0

# 標準的な抑制
PARAMETER repeat_penalty 1.1

# 強い抑制
PARAMETER repeat_penalty 1.3
```

### 5.3.5 完全なパラメータリスト

```dockerfile
PARAMETER num_predict 2048        # 最大生成トークン数
PARAMETER temperature 0.8          # 創造性（0.0-2.0）
PARAMETER top_p 0.9               # 核サンプリング（0.0-1.0）
PARAMETER top_k 40                # Top-Kサンプリング
PARAMETER repeat_penalty 1.1      # 繰り返しペナルティ
PARAMETER repeat_last_n 64        # ペナルティ適用範囲
PARAMETER num_ctx 8192            # コンテキストウィンドウ
PARAMETER num_batch 512           # バッチサイズ
PARAMETER num_gpu 1               # GPU数
PARAMETER num_thread 16           # CPUスレッド数
PARAMETER stop "<|end|>"          # ストップシーケンス
PARAMETER mirostat 0              # Mirostatサンプリング（0=無効）
PARAMETER mirostat_eta 0.1        # Mirostat学習率
PARAMETER mirostat_tau 5.0        # Mirostat目標エントロピー
```

## 5.4 テンプレートのカスタマイズ

### 5.4.1 基本的なテンプレート変数

```dockerfile
{{ .System }}   # システムプロンプト
{{ .Prompt }}   # ユーザー入力
{{ .Response }} # アシスタントの応答（メッセージ履歴用）
```

### 5.4.2 Chat ML形式

```dockerfile
TEMPLATE """<|im_start|>system
{{ .System }}<|im_end|>
<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
"""
```

### 5.4.3 Llama 3形式

```dockerfile
TEMPLATE """<|begin_of_text|><|start_header_id|>system<|end_header_id|>

{{ .System }}<|eot_id|><|start_header_id|>user<|end_header_id|>

{{ .Prompt }}<|eot_id|><|start_header_id|>assistant<|end_header_id|>

"""
```

### 5.4.4 カスタムフォーマット

```dockerfile
# Markdown風
TEMPLATE """# System
{{ .System }}

## User
{{ .Prompt }}

## Assistant
"""

# 構造化形式
TEMPLATE """[SYSTEM]
{{ .System }}
[/SYSTEM]

[USER]
{{ .Prompt }}
[/USER]

[ASSISTANT]
"""
```

## 5.5 マルチモーダルモデル

### 5.5.1 ビジョンモデルの作成

```bash
# LLaVAベース（画像+テキスト）
nano vision-assistant.Modelfile
```

```dockerfile
FROM llava:13b

SYSTEM """
あなたは画像を理解し、詳細に説明できるAIアシスタントです。
"""

PARAMETER temperature 0.7
PARAMETER num_ctx 4096
```

```bash
# 作成
ollama create vision-assistant -f vision-assistant.Modelfile

# 使用（画像を渡す）
ollama run vision-assistant "Describe this image" image.jpg
```

### 5.5.2 コード理解特化

```bash
nano code-understanding.Modelfile
```

```dockerfile
FROM qwen2.5-coder:7b

SYSTEM """
コードを分析し、以下を提供します：
1. 機能の説明
2. 複雑度分析
3. 改善提案
4. ドキュメント生成
"""

PARAMETER temperature 0.5
PARAMETER num_ctx 16384
```

## 5.6 モデルバージョン管理

### 5.6.1 タグ付けシステム

```bash
# バージョン付きモデル作成
ollama create my-assistant:v1.0 -f assistant-v1.Modelfile
ollama create my-assistant:v1.1 -f assistant-v1.1.Modelfile
ollama create my-assistant:latest -f assistant-latest.Modelfile

# バージョン指定実行
ollama run my-assistant:v1.0
ollama run my-assistant:latest
```

### 5.6.2 モデルのエクスポートとインポート

```bash
# Modelfileをエクスポート
ollama show --modelfile my-assistant > my-assistant.exported.Modelfile

# 別のマシンでインポート
ollama create my-assistant -f my-assistant.exported.Modelfile
```

### 5.6.3 モデルの共有

```bash
# Ollama Hubにプッシュ（アカウント必要）
ollama push username/my-assistant:latest

# 他のユーザーが利用
ollama pull username/my-assistant
```

## 5.7 量子化レベルの選択

### 5.7.1 量子化の基礎

量子化は、モデルサイズと精度のトレードオフです。

```
Q8_0:  8ビット量子化（最高品質、大サイズ）
Q6_K:  6ビット量子化（高品質）
Q5_K_M: 5ビット量子化（バランス）
Q4_K_M: 4ビット量子化（推奨、良好なバランス）
Q3_K_M: 3ビット量子化（小サイズ、品質低下）
Q2_K:  2ビット量子化（最小サイズ、大幅な品質低下）
```

### 5.7.2 MS-S1 Maxでの推奨設定

```bash
# 7B-14Bモデル: Q4_K_M または Q5_K_M
ollama pull qwen2.5:7b-instruct-q4_K_M   # 4.9GB
ollama pull qwen2.5:7b-instruct-q5_K_M   # 5.8GB

# 32B-34Bモデル: Q4_K_M
ollama pull qwen2.5:32b-instruct-q4_K_M  # 19GB

# 70Bモデル: Q4_K_M（128GBメモリで快適）
ollama pull llama3.1:70b-instruct-q4_K_M # 41GB
```

### 5.7.3 量子化レベルの比較

```bash
# 比較スクリプト
#!/bin/bash
MODEL_BASE="qwen2.5:7b"
QUANTS=("q8_0" "q5_K_M" "q4_K_M" "q3_K_M")
PROMPT="Write a detailed explanation of quantum computing."

for quant in "${QUANTS[@]}"; do
    model="${MODEL_BASE}-instruct-${quant}"
    echo "Testing: $model"

    ollama pull $model
    time ollama run $model "$PROMPT"
    echo "---"
done
```

## 5.8 モデルのファインチューニング

### 5.8.1 LoRAアダプタの使用

```dockerfile
# ベースモデル + LoRAアダプタ
FROM llama3.1:8b

# LoRAアダプタ（別途学習したもの）
ADAPTER ./adapters/japanese-qa-lora.bin

SYSTEM """
日本語Q&Aに特化したモデルです。
"""
```

### 5.8.2 カスタムGGUFモデルの統合

```bash
# 1. GGUFモデルを配置
mkdir -p ~/.ollama/models/custom
cp my-finetuned-model.gguf ~/.ollama/models/custom/

# 2. Modelfile作成
nano custom-model.Modelfile
```

```dockerfile
FROM ~/.ollama/models/custom/my-finetuned-model.gguf

SYSTEM """
カスタムファインチューニングモデル
"""

PARAMETER temperature 0.7
```

```bash
# 3. インポート
ollama create custom-model -f custom-model.Modelfile
```

## 5.9 トラブルシューティング

### 5.9.1 モデル作成エラー

```bash
# エラー例
Error: failed to parse modelfile

# 解決: 構文確認
# - FROM行が最初にあるか
# - クオートが正しく閉じられているか
# - パラメータ名が正しいか
```

### 5.9.2 メモリ不足

```bash
# エラー
Error: failed to allocate memory

# 解決1: 小さい量子化を使用
FROM llama3.1:8b-q4_K_M  # q8_0ではなく

# 解決2: コンテキスト長を削減
PARAMETER num_ctx 4096  # 16384ではなく
```

### 5.9.3 パフォーマンス低下

```bash
# 症状: カスタムモデルが遅い

# 確認1: ベースモデルの量子化レベル
ollama show --modelfile my-model

# 確認2: パラメータ設定
# num_ctxが大きすぎないか確認
```

## 5.10 本章のまとめ

本章では、以下の内容を学習しました。

✅ **Modelfileの基礎**
- 構文と命令
- カスタムモデル作成

✅ **パラメータ最適化**
- Temperature, Top-P, Context Length
- MS-S1 Max向け推奨設定

✅ **テンプレートカスタマイズ**
- Chat ML形式
- カスタムフォーマット

✅ **量子化レベル**
- Q4_K_M vs Q8_0
- サイズと品質のトレードオフ

✅ **モデル管理**
- バージョン管理
- エクスポート/インポート

次章では、Ollama APIの活用と、他のアプリケーションとの統合方法を学びます。

---

**前章へ**: [第4章 基本コマンドと使い方](chapter04_basic_commands.md)
**次章へ**: [第6章 API活用と統合](chapter06_api_integration.md)
