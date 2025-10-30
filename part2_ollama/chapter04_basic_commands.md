# 第4章:基本コマンドと使い方

## 4.1 Ollama CLI の基本構造

### 4.1.1 コマンド一覧

```bash
# ヘルプ表示
ollama --help

# 主要コマンド
ollama serve      # サーバーを起動（通常は自動起動）
ollama run        # モデルを実行
ollama pull       # モデルをダウンロード
ollama push       # モデルをアップロード
ollama list       # ローカルモデル一覧
ollama ps         # 実行中のモデル表示
ollama cp         # モデルをコピー
ollama rm         # モデルを削除
ollama show       # モデル情報表示
ollama create     # カスタムモデル作成
```

### 4.1.2 基本的な文法

```bash
# 基本形式
ollama [command] [model] [options]

# 例
ollama run llama3.1 "Hello"
ollama pull qwen2.5:7b
ollama list
```

## 4.2 モデルの実行（run）

### 4.2.1 インタラクティブモード

```bash
# 対話型セッション開始
ollama run qwen2.5:7b

# プロンプトが表示される
>>> こんにちは！
こんにちは！何かお手伝いできることはありますか？

>>> MS-S1 Maxについて教えて
MS-S1 Maxは、AMD Ryzen AI Max+ 395プロセッサを搭載した...

>>> /bye
```

**インタラクティブモードの特殊コマンド:**

| コマンド | 機能 |
|---------|------|
| `/bye` | セッション終了 |
| `/clear` | 会話履歴をクリア |
| `/save [filename]` | 会話を保存 |
| `/load [filename]` | 会話を読み込み |
| `/set parameter value` | パラメータ変更 |
| `/show` | 現在の設定表示 |

### 4.2.2 ワンショット実行

```bash
# 直接プロンプトを指定
ollama run llama3.1 "What is the capital of Japan?"

# 出力
The capital of Japan is Tokyo.

# 複数行プロンプト
ollama run qwen2.5:14b "
Translate to Japanese:
Hello, how are you?
"
```

### 4.2.3 標準入力からの実行

```bash
# パイプで入力
echo "Explain quantum computing" | ollama run llama3.1

# ファイルから入力
cat prompt.txt | ollama run qwen2.5:7b

# ヒアドキュメント
ollama run llama3.1 << EOF
Please summarize the following:
$(cat article.txt)
EOF
```

### 4.2.4 パラメータ指定

```bash
# --verbose: 詳細情報表示
ollama run --verbose llama3.1 "Hello"

# --nowordwrap: 自動改行無効
ollama run --nowordwrap qwen2.5:7b "Long text..."

# --format json: JSON形式で出力
ollama run --format json llama3.1 "List 3 colors"
```

## 4.3 モデルの管理

### 4.3.1 モデルのダウンロード（pull）

```bash
# 最新版をダウンロード
ollama pull llama3.1

# 特定のタグを指定
ollama pull llama3.1:8b-instruct-q4_K_M

# 複数モデルのダウンロード
for model in llama3.1 qwen2.5:7b mistral; do
    ollama pull $model
done
```

**進行状況の表示:**
```
pulling manifest
pulling 8cf58c9acf79... 100% ▕████████████████████████▏ 4.7 GB
pulling 8ab4849b038c... 100% ▕████████████████████████▏  249 B
pulling 23e0f4461c0c... 100% ▕████████████████████████▏  11 KB
verifying sha256 digest
writing manifest
success
```

### 4.3.2 モデル一覧表示（list）

```bash
# ローカルモデル一覧
ollama list

# 出力例
NAME                              ID              SIZE    MODIFIED
llama3.1:latest                   abcd1234        4.7GB   2 hours ago
qwen2.5:7b                        efgh5678        4.9GB   1 day ago
qwen2.5:14b                       ijkl9012        9.2GB   2 days ago
mistral:latest                    mnop3456        4.1GB   3 days ago
```

### 4.3.3 モデル情報の表示（show）

```bash
# モデル詳細情報
ollama show llama3.1

# 出力例
Model
  arch              llama
  parameters        8.0B
  quantization      Q4_K_M
  context length    131072
  embedding length  4096

Parameters
  stop    "<|start_header_id|>"
  stop    "<|end_header_id|>"
  stop    "<|eot_id|>"

License
  Meta Llama 3.1 Community License

System
  You are a helpful assistant.
```

```bash
# Modelfile の表示
ollama show --modelfile llama3.1

# 出力
FROM /path/to/model
TEMPLATE """{{ .System }}
{{ .Prompt }}"""
PARAMETER stop "<|start_header_id|>"
PARAMETER stop "<|end_header_id|>"
```

### 4.3.4 実行中のモデル（ps）

```bash
# 実行中・ロード中のモデル確認
ollama ps

# 出力例
NAME              ID        SIZE    PROCESSOR    UNTIL
llama3.1:latest   abc123    4.7GB   100% GPU     4 minutes from now
qwen2.5:7b        def456    4.9GB   100% GPU     4 minutes from now
```

**出力項目の説明:**
- **NAME**: モデル名
- **SIZE**: メモリ使用量
- **PROCESSOR**: GPU/CPUの使用率
- **UNTIL**: アンロードまでの時間（アイドル5分でアンロード）

### 4.3.5 モデルのコピー（cp）

```bash
# モデルのコピー（カスタム名で保存）
ollama cp llama3.1 my-assistant

# 使用
ollama run my-assistant

# 確認
ollama list
# my-assistant が追加されている
```

### 4.3.6 モデルの削除（rm）

```bash
# モデル削除
ollama rm mistral:latest

# 複数削除
ollama rm llama3.1:8b qwen2.5:3b

# 確認プロンプト表示
# deleted 'mistral:latest'
```

**⚠️ 注意:** 削除は元に戻せません。再度使用する場合は`ollama pull`で再ダウンロードが必要です。

## 4.4 実践的な使用例

### 4.4.1 テキスト生成

```bash
# ストーリー生成
ollama run llama3.1 "Write a short story about a robot learning to love."

# コードの説明
ollama run qwen2.5:7b "Explain this Python code:
$(cat script.py)
"

# 要約
ollama run llama3.1 "Summarize in 3 sentences:
$(cat article.txt)
"
```

### 4.4.2 翻訳

```bash
# 英語→日本語
ollama run qwen2.5:14b "Translate to Japanese:
Artificial Intelligence is transforming the world.
"

# 日本語→英語
ollama run llama3.1 "Translate to English:
人工知能は世界を変革しています。
"

# 複数言語翻訳スクリプト
translate() {
    local text="$1"
    local target="$2"
    ollama run qwen2.5:14b "Translate to $target: $text"
}

translate "Hello World" "Japanese"
translate "Hello World" "Spanish"
translate "Hello World" "French"
```

### 4.4.3 コーディング支援

```bash
# コード生成
ollama run codellama:13b "Write a Python function to calculate fibonacci numbers"

# コードレビュー
ollama run qwen2.5:14b "Review this code and suggest improvements:
$(cat mycode.py)
"

# バグ修正
ollama run llama3.1 "Find and fix the bug in this code:
def calculate(x, y):
    return x + x  # Bug here
"

# 出力
# The bug is that it adds x to itself instead of adding x and y.
# Fixed version:
# def calculate(x, y):
#     return x + y
```

### 4.4.4 データ処理

```bash
# JSON生成
ollama run llama3.1 --format json "Generate a JSON object with 3 users containing name, age, and email"

# CSV解析
ollama run qwen2.5:7b "Convert this data to JSON:
$(cat data.csv)
"

# データ抽出
ollama run llama3.1 "Extract all email addresses from this text:
$(cat document.txt)
"
```

### 4.4.5 情報抽出と分析

```bash
# キーワード抽出
ollama run qwen2.5:14b "Extract the main keywords from this article:
$(cat article.txt)
"

# 感情分析
ollama run llama3.1 "Analyze the sentiment (positive/negative/neutral) of this review:
The MS-S1 Max is an amazing machine! The performance is outstanding.
"

# エンティティ認識
ollama run qwen2.5:7b "Extract all person names, locations, and organizations from:
$(cat news.txt)
"
```

## 4.5 シェルスクリプトでの活用

### 4.5.1 基本的なスクリプト例

```bash
#!/bin/bash
# ollama_translate.sh

MODEL="qwen2.5:14b"

if [ $# -eq 0 ]; then
    echo "Usage: $0 <text_to_translate>"
    exit 1
fi

TEXT="$1"
ollama run $MODEL "Translate to Japanese: $TEXT"
```

```bash
# 使用
chmod +x ollama_translate.sh
./ollama_translate.sh "Hello, how are you?"
```

### 4.5.2 バッチ処理

```bash
#!/bin/bash
# batch_summarize.sh

MODEL="llama3.1"
INPUT_DIR="./articles"
OUTPUT_DIR="./summaries"

mkdir -p "$OUTPUT_DIR"

for file in "$INPUT_DIR"/*.txt; do
    filename=$(basename "$file" .txt)
    echo "Processing: $filename"

    ollama run $MODEL "Summarize this article in 2-3 sentences:
$(cat "$file")
" > "$OUTPUT_DIR/${filename}_summary.txt"

    echo "Saved: $OUTPUT_DIR/${filename}_summary.txt"
done

echo "Batch processing completed!"
```

### 4.5.3 対話型メニュー

```bash
#!/bin/bash
# ollama_menu.sh

while true; do
    echo "==============================="
    echo "  Ollama Assistant Menu"
    echo "==============================="
    echo "1. Chat (Llama 3.1)"
    echo "2. Translate (Qwen 2.5)"
    echo "3. Code Help (CodeLlama)"
    echo "4. Exit"
    echo "==============================="
    read -p "Select option: " choice

    case $choice in
        1)
            ollama run llama3.1
            ;;
        2)
            read -p "Enter text to translate: " text
            ollama run qwen2.5:14b "Translate to Japanese: $text"
            ;;
        3)
            read -p "Describe the code you need: " desc
            ollama run codellama:13b "$desc"
            ;;
        4)
            echo "Goodbye!"
            exit 0
            ;;
        *)
            echo "Invalid option"
            ;;
    esac

    echo ""
    read -p "Press Enter to continue..."
done
```

### 4.5.4 ログ機能付きスクリプト

```bash
#!/bin/bash
# ollama_with_log.sh

MODEL="qwen2.5:7b"
LOG_DIR="$HOME/ollama_logs"
LOG_FILE="$LOG_DIR/chat_$(date +%Y%m%d_%H%M%S).log"

mkdir -p "$LOG_DIR"

echo "Starting Ollama session..."
echo "Log file: $LOG_FILE"
echo "========================================" | tee -a "$LOG_FILE"

while IFS= read -r -p "You: " prompt; do
    [ -z "$prompt" ] && continue
    [ "$prompt" = "exit" ] && break

    echo "You: $prompt" >> "$LOG_FILE"
    echo "Assistant:" | tee -a "$LOG_FILE"

    response=$(ollama run $MODEL "$prompt")
    echo "$response" | tee -a "$LOG_FILE"
    echo "" >> "$LOG_FILE"
done

echo "Session ended. Log saved to: $LOG_FILE"
```

## 4.6 マルチモデル活用

### 4.6.1 モデルの使い分け

```bash
# 小型・高速モデル（簡単なタスク）
FAST_MODEL="qwen2.5:7b"

# 中型・バランスモデル（一般的なタスク）
BALANCED_MODEL="qwen2.5:14b"

# 大型・高精度モデル（複雑なタスク）
POWERFUL_MODEL="qwen2.5:32b"

# 用途に応じた使い分け
ollama run $FAST_MODEL "What is 2+2?"
ollama run $BALANCED_MODEL "Explain quantum entanglement"
ollama run $POWERFUL_MODEL "Write a detailed business plan for a startup"
```

### 4.6.2 専門モデルの活用

```bash
# コーディング: CodeLlama
ollama run codellama:13b "Write a sorting algorithm in Python"

# 数学: Qwen2.5-Math
ollama pull qwen2.5-math:7b
ollama run qwen2.5-math:7b "Solve: ∫x²dx"

# 日本語特化: ELYZA
ollama pull elyza:jp8b
ollama run elyza:jp8b "日本の歴史について教えて"
```

### 4.6.3 パイプライン処理

```bash
# ステップ1: 要約
SUMMARY=$(ollama run llama3.1 "Summarize in one sentence: $(cat article.txt)")

# ステップ2: 翻訳
TRANSLATION=$(ollama run qwen2.5:14b "Translate to Japanese: $SUMMARY")

# ステップ3: キーワード抽出
KEYWORDS=$(ollama run qwen2.5:7b "Extract 3 keywords from: $TRANSLATION")

echo "Summary: $SUMMARY"
echo "Translation: $TRANSLATION"
echo "Keywords: $KEYWORDS"
```

## 4.7 パフォーマンス測定

### 4.7.1 実行時間測定

```bash
# time コマンドで測定
time ollama run llama3.1 "Write a 100-word essay on AI"

# 出力例
real    0m8.234s
user    0m0.042s
sys     0m0.028s
```

### 4.7.2 トークン速度の計算

```bash
#!/bin/bash
# measure_speed.sh

MODEL="$1"
PROMPT="$2"
NUM_TOKENS=100

START=$(date +%s.%N)
ollama run $MODEL "$PROMPT" > /dev/null
END=$(date +%s.%N)

ELAPSED=$(echo "$END - $START" | bc)
SPEED=$(echo "scale=2; $NUM_TOKENS / $ELAPSED" | bc)

echo "Model: $MODEL"
echo "Time: ${ELAPSED}s"
echo "Speed: ${SPEED} tokens/s"
```

### 4.7.3 複数モデルの比較

```bash
#!/bin/bash
# compare_models.sh

MODELS=("qwen2.5:7b" "qwen2.5:14b" "llama3.1" "mistral")
PROMPT="Explain machine learning in 50 words"

echo "Model Comparison Results:"
echo "========================="

for model in "${MODELS[@]}"; do
    echo -n "Testing $model... "
    START=$(date +%s.%N)
    ollama run $model "$PROMPT" > /dev/null 2>&1
    END=$(date +%s.%N)
    ELAPSED=$(echo "$END - $START" | bc)
    echo "Time: ${ELAPSED}s"
done
```

## 4.8 トラブルシューティング

### 4.8.1 コマンドが見つからない

```bash
# エラー
ollama: command not found

# 解決
echo $PATH  # /usr/local/bin が含まれているか確認

# 含まれていない場合
export PATH=/usr/local/bin:$PATH

# 永続化
echo 'export PATH=/usr/local/bin:$PATH' >> ~/.bashrc
```

### 4.8.2 モデルが起動しない

```bash
# エラー例
Error: model 'llama3.1' not found

# 解決1: モデルをダウンロード
ollama pull llama3.1

# 解決2: 正しい名前を確認
ollama list
```

### 4.8.3 応答が遅い

```bash
# GPU使用確認
rocm-smi

# GPU使用率が低い場合、CPUで動作している可能性
# 第3章を参照してROCm設定を確認
```

## 4.9 本章のまとめ

本章では、以下の内容を学習しました。

✅ **基本コマンド**
- run, pull, list, show, ps, rm など

✅ **実行モード**
- インタラクティブモード
- ワンショット実行
- 標準入力からの実行

✅ **モデル管理**
- ダウンロード、削除、コピー
- 情報表示

✅ **実践的な使い方**
- テキスト生成、翻訳、コーディング支援
- データ処理、情報抽出

✅ **スクリプト化**
- シェルスクリプトでの自動化
- バッチ処理、ログ機能

次章では、Modelfileを使ったカスタムモデルの作成と、モデル管理の高度なテクニックを学びます。

---

**前章へ**: [第3章 ROCm設定とAMD GPU最適化](chapter03_rocm_optimization.md)
**次章へ**: [第5章 モデル管理とカスタマイズ](chapter05_model_management.md)
