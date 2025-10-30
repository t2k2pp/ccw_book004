# 第1章:はじめに - Ollamaとローカル大規模言語モデルの革命

## 1.1 Ollamaとは

**Ollama**は、大規模言語モデル(LLM)をローカル環境で簡単に実行できる革新的なオープンソースツールです。コマンドラインインターフェースを中心に設計されており、開発者やパワーユーザーに最適化されています。

### Ollamaの哲学

Ollamaは「Dockerのシンプルさ」をLLMの世界に持ち込むことを目指して開発されました。

```bash
# Dockerのように簡単
docker pull ubuntu
docker run ubuntu

# Ollamaも同様にシンプル
ollama pull llama3.1
ollama run llama3.1
```

この設計思想により、複雑な機械学習の知識がなくても、誰でも簡単に最先端のAIモデルを利用できます。

### なぜOllamaが注目されているのか

#### 1. **圧倒的なシンプルさ**

従来のLLM実行環境は複雑でした。

**従来の方法（Python + Transformers）:**
```bash
# 環境構築だけで数時間
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm6.0
pip install transformers accelerate bitsandbytes
pip install sentencepiece protobuf

# コードを書く必要がある
python inference.py --model meta-llama/Llama-3.1-70B-Instruct --prompt "Hello"
```

**Ollamaの場合:**
```bash
# インストール（1分）
curl -fsSL https://ollama.com/install.sh | sh

# すぐに使える
ollama run llama3.1 "Hello"
```

#### 2. **開発者フレンドリー**

- シンプルなCLI
- REST API標準装備
- OpenAI API互換
- 多言語SDK（Python、JavaScript、Go、Rustなど）

#### 3. **パフォーマンス重視**

- C++で実装された高速推論エンジン（llama.cpp基盤）
- 自動GPU検出とオフロード
- 効率的なメモリ管理
- 複数モデルの並行実行

#### 4. **豊富なモデルライブラリ**

Ollama公式ライブラリには、100以上のモデルが登録されています。

```bash
# 主要なモデルファミリー
ollama pull llama3.1           # Meta Llama 3.1
ollama pull qwen2.5            # Alibaba Qwen2.5
ollama pull mistral            # Mistral AI
ollama pull gemma2             # Google Gemma 2
ollama pull codellama          # コーディング特化
ollama pull phi3               # Microsoft Phi-3
```

## 1.2 LM StudioとOllamaの違い

第一部で学習したLM Studioと、本書のテーマであるOllamaには、それぞれ異なる強みがあります。

### LM Studio の強み

| 特徴 | 詳細 |
|------|------|
| **GUI** | 直感的なグラフィカルインターフェース |
| **初心者向け** | クリック操作で完結 |
| **可視化** | GPU使用率、メモリ使用量の可視化 |
| **チャット重視** | ChatGPT風のインターフェース |
| **プリセット管理** | GUI上で簡単に設定保存 |

### Ollama の強み

| 特徴 | 詳細 |
|------|------|
| **CLI** | コマンドライン中心の設計 |
| **開発者向け** | API、SDK、自動化に最適 |
| **軽量** | リソース効率が良い |
| **スクリプト化** | 自動化、バッチ処理が容易 |
| **サーバーモード** | バックグラウンド動作 |

### 使い分けの指針

```
【LM Studioを選ぶべきケース】
✓ 初めてローカルLLMを試す
✓ 対話型チャットがメイン用途
✓ GUIで設定を調整したい
✓ パフォーマンスを可視化したい
✓ Windowsユーザー

【Ollamaを選ぶべきケース】
✓ 開発者、エンジニア
✓ API経由で他アプリと連携
✓ スクリプトで自動化したい
✓ サーバー用途（常駐）
✓ Linux/macOSユーザー
✓ 複数モデルを切り替えて使う
```

### 併用のすすめ

実際には、両方を併用するのが最も効果的です。

```
MS-S1 Max (128GB メモリ)
├── LM Studio （ポート 1234）
│   └── 対話型チャット用
└── Ollama （ポート 11434）
    └── API/開発用
```

128GBの大容量メモリを持つMS-S1 Maxでは、両方を同時に起動しても余裕があります。

## 1.3 MS-S1 Max × Ollama の最強の組み合わせ

### AMD Ryzen AI Max+ 395 の圧倒的アドバンテージ

MS-S1 MaxのRyzen AI Max+ 395は、Ollamaにとって理想的なハードウェアです。

#### 1. **大容量統合メモリ（128GB）**

```
通常のPC (16GB):
  └── 7Bモデルのみ実行可能

ハイエンドPC (64GB):
  └── 34Bモデルまで

MS-S1 Max (128GB):
  ├── 70Bモデルを快適に実行
  ├── 複数の34Bモデルを同時実行
  ├── 13Bモデル×3本 + 7Bモデル×2本
  └── 超長コンテキスト（128K+）も余裕
```

#### 2. **AMD Radeon 8060S GPU（ROCm対応）**

OllamaはROCmをフルサポートしており、AMD GPUで高速推論が可能です。

```bash
# 自動GPU検出
ollama run llama3.1:70b

# 出力例
>>> 推論速度: 15-20 tokens/s（70Bモデル、Q4量子化）
>>> GPU使用率: 85-95%
>>> VRAM使用量: 42GB
```

#### 3. **高メモリ帯域幅（256GB/s）**

クアッドチャネルLPDDR5X-8000により、大規模モデルのロードと推論が高速です。

```
メモリ帯域幅の影響:

DDR4-3200 (51GB/s):
  - 70Bモデルロード: 90秒
  - プロンプト処理: 350 t/s

LPDDR5X-8000 (256GB/s):
  - 70Bモデルロード: 18秒（5倍高速）
  - プロンプト処理: 1200 t/s（3.4倍高速）
```

### Ollamaに最適なハードウェア特性

MS-S1 Maxは、Ollamaの以下の特性と完璧にマッチします。

#### マルチモデル並行実行

```bash
# ターミナル1: 翻訳タスク（34Bモデル）
ollama run qwen2.5:32b "Translate to Japanese: Hello World"

# ターミナル2: コーディング（13Bモデル）
ollama run codellama:13b "Write a Python function for..."

# ターミナル3: チャット（7Bモデル）
ollama run llama3.1 "What is the meaning of life?"

# 全て同時実行可能！
# 総メモリ使用量: 約65GB → 128GBで十分余裕
```

#### 長時間稼働

Ollamaはバックグラウンドサービスとして動作します。MS-S1 Maxの効率的な冷却システムにより、24時間365日の安定稼働が可能です。

```
MS-S1 Max Balance モード（130W）:
  - 温度: 65-75℃（安定）
  - ファン音: 許容範囲
  - 推論性能: 高速

→ サーバー用途に最適
```

## 1.4 Ollamaでできること

### 1.4.1 対話型チャット

最もシンプルな使い方です。

```bash
# インタラクティブモード
ollama run qwen2.5:32b

>>> こんにちは！MS-S1 Maxについて教えてください。
MS-S1 Maxは、AMD Ryzen AI Max+ 395を搭載した強力なミニPCです。
主な特徴として...

>>> /bye
```

### 1.4.2 ワンショット実行

スクリプトから利用する場合。

```bash
# シェルスクリプトでの利用
RESPONSE=$(ollama run llama3.1 "Summarize: $(cat article.txt)")
echo "$RESPONSE" > summary.txt

# パイプラインでの利用
echo "Translate to English: こんにちは" | ollama run qwen2.5:14b
```

### 1.4.3 REST API サーバー

Ollamaは起動時に自動的にREST APIサーバーを起動します。

```bash
# Ollamaサービスは自動起動（デフォルト: ポート11434）
# curl で利用
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.1",
  "prompt": "Why is the sky blue?"
}'

# ストリーミングレスポンス（リアルタイム）
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5:32b",
  "prompt": "Write a long story...",
  "stream": true
}'
```

### 1.4.4 OpenAI API 互換モード

既存のOpenAI APIクライアントがそのまま使えます。

```python
# OpenAI Python SDK をそのまま利用
from openai import OpenAI

client = OpenAI(
    base_url='http://localhost:11434/v1',
    api_key='ollama'  # ダミー（必須だが値は無視される）
)

response = client.chat.completions.create(
    model="llama3.1",
    messages=[
        {"role": "user", "content": "Hello!"}
    ]
)
print(response.choices[0].message.content)
```

### 1.4.5 Modelfile によるカスタマイズ

独自のモデルバリエーションを作成できます。

```dockerfile
# Modelfile
FROM llama3.1

# システムプロンプト
SYSTEM """
あなたは親切で知識豊富な日本語アシスタントです。
MS-S1 Maxのエキスパートとして、詳しく丁寧に説明します。
"""

# パラメータ
PARAMETER temperature 0.8
PARAMETER top_p 0.9
PARAMETER top_k 40

# テンプレート
TEMPLATE """{{ .System }}

User: {{ .Prompt }}
Assistant:"""
```

```bash
# カスタムモデル作成
ollama create my-assistant -f Modelfile

# 使用
ollama run my-assistant "Ryzen AI Max+ 395について教えて"
```

## 1.5 本書で学ぶこと

### 第二部：Ollama完全ガイド（本書）

本書は以下の9章で構成されています。

**第1章（本章）**: Ollamaの概要とMS-S1 Maxの優位性
**第2章**: インストールとセットアップ
**第3章**: ROCm設定とAMD GPU最適化
**第4章**: 基本的なコマンドと使い方
**第5章**: モデル管理とカスタマイズ
**第6章**: API活用と統合
**第7章**: MS-S1 Max向けパフォーマンス最適化
**第8章**: マルチモデル運用と同時実行
**第9章**: 高度なテクニックとトラブルシューティング

### 学習の進め方

#### 初心者（LLM初体験）の方

1. 第1章、第2章で基礎を理解
2. 第3章でGPU設定を完璧に
3. 第4章で実際に触って体験
4. 第5章で様々なモデルを試す

#### 中級者（LM Studio経験者）の方

1. 第2章でインストール
2. 第3章でROCm設定確認
3. 第6章でAPI活用を習得
4. 第7章で最適化テクニック
5. 第8章でマルチモデル運用

#### 上級者（開発者）の方

1. 第2章、第3章で環境構築
2. 第6章のAPI統合を詳細に学習
3. 第7章で徹底的に最適化
4. 第8章、第9章で実践的な運用方法
5. 自動化スクリプト、アプリケーション開発

### 前提知識

本書を最大限活用するために、以下の知識があると理想的です。

```
【必須】
✓ 基本的なLinuxコマンド（cd, ls, cat など）
✓ テキストエディタの使用（nano, vim など）
✓ ターミナル操作の基礎

【推奨】
✓ シェルスクリプトの基礎
✓ REST APIの概念
✓ JSON形式の理解

【あると便利】
✓ Python, JavaScript の基礎
✓ Docker の経験
✓ Git の使用経験
```

ただし、上級の知識がなくても、順を追って学習すれば十分に理解できるよう構成しています。

## 1.6 Ollamaのエコシステム

Ollamaは単なるツールではなく、エコシステムの中心となっています。

### 公式ツール

#### 1. **Ollama CLI**
```bash
# コアツール
ollama run, pull, push, list, rm, etc.
```

#### 2. **Ollama Web UI (旧称: Ollama WebUI)**
```bash
# ブラウザベースのチャットUI
docker run -d -p 3000:8080 ghcr.io/open-webui/open-webui
```

#### 3. **公式SDK**
```bash
# Python
pip install ollama

# JavaScript/TypeScript
npm install ollama

# Go
go get github.com/ollama/ollama/api

# Rust
cargo add ollama-rs
```

### サードパーティ統合

```
【人気のある統合先】
✓ LangChain / LangSmith
✓ LlamaIndex
✓ Continue.dev (VSCode拡張)
✓ Jan (デスクトップアプリ)
✓ Open WebUI
✓ Obsidian (ノートアプリ)
✓ Raycast (macOS ランチャー)
```

### コミュニティモデル

```bash
# Hugging Faceからのインポート
ollama create mymodel -f Modelfile

# 独自の微調整モデル
ollama create my-finetuned -f custom.Modelfile

# 共有
ollama push myusername/mymodel
```

## 1.7 Ollamaのライセンスとビジネス利用

### ライセンス

- **Ollama本体**: MIT License（商用利用可）
- **モデル**: 各モデルのライセンスに準拠

### 主要モデルのライセンス

| モデル | ライセンス | 商用利用 |
|--------|-----------|---------|
| Llama 3.1 | Meta License | ✓ 可能 |
| Qwen2.5 | Apache 2.0 | ✓ 可能 |
| Mistral | Apache 2.0 | ✓ 可能 |
| Gemma 2 | Gemma License | ✓ 可能（制限あり）|
| Phi-3 | MIT License | ✓ 可能 |

**⚠️ 注意**: ビジネス利用前に、必ず使用するモデルのライセンスを確認してください。

### プライベート利用のメリット

```
【企業での利用】
✓ データが外部に送信されない
✓ 機密情報を安全に処理
✓ カスタマイズ可能
✓ ランニングコスト削減
✓ レート制限なし
✓ オフライン動作可能

【個人での利用】
✓ プライバシー保護
✓ 月額料金不要
✓ 学習・実験に最適
✓ 自由なカスタマイズ
```

## 1.8 Ollamaのバージョンと互換性

### 最新バージョン（2025年現在）

```bash
# バージョン確認
ollama --version

# 出力例
ollama version is 0.5.4
```

### 主要なマイルストーン

```
v0.1.0 (2023年8月)
  - 初回リリース
  - 基本的なモデル実行機能

v0.2.0 (2023年11月)
  - REST API追加
  - マルチモダル対応

v0.3.0 (2024年2月)
  - Modelfile サポート
  - カスタムモデル作成

v0.4.0 (2024年6月)
  - OpenAI API互換モード
  - パフォーマンス改善

v0.5.0 (2024年10月)
  - AMD ROCm完全サポート
  - マルチGPU対応強化
  - コンテキストキャッシング

v0.5.4 (2025年現在)
  - RDNA 3.5最適化
  - メモリ管理改善
  - 新モデルフォーマット対応
```

### MS-S1 Max対応状況

```bash
# MS-S1 Maxで必要なバージョン
Ollama: v0.4.0以降（v0.5.4推奨）
ROCm: 6.1以降（6.3推奨）
Linux Kernel: 6.5以降
```

## 1.9 本書の表記規則

### コマンド表記

```bash
# コメント: 説明文
command --option value

# 出力例
>>> 結果の表示
```

### 環境変数

```bash
# 設定
export VARIABLE_NAME=value

# 使用
echo $VARIABLE_NAME
```

### ファイル編集

```bash
# ファイルパスの表記
~/.bashrc
/etc/systemd/system/ollama.service
```

### API例（curl）

```bash
curl http://localhost:11434/api/endpoint \
  -H "Content-Type: application/json" \
  -d '{
    "key": "value"
  }'
```

### 重要な情報

> **💡 TIP**: 便利なテクニックやヒント

> **⚠️ 注意**: 注意が必要な事項

> **🚨 警告**: 重大な警告

## 1.10 本章のまとめ

本章では、以下の内容を学習しました。

✅ **Ollamaの特徴と哲学**
- Dockerライクなシンプルさ
- 開発者フレンドリーな設計
- 高性能な推論エンジン

✅ **LM StudioとOllamaの違い**
- GUI vs CLI
- 使い分けの指針
- 両方の併用も可能

✅ **MS-S1 Maxとの相性**
- 128GB大容量メモリの活用
- AMD Radeon 8060S + ROCm
- マルチモデル並行実行

✅ **Ollamaでできること**
- 対話型チャット
- REST API
- OpenAI互換API
- カスタムモデル作成

✅ **エコシステムとライセンス**
- 豊富な統合ツール
- コミュニティサポート
- 商用利用可能

次章では、実際にOllamaをMS-S1 Maxにインストールし、初期設定を行います。ROCmの設定、AMD GPU認識、そして最初のモデル実行まで、ステップバイステップで解説します。

---

**次章へ**: [第2章 インストールとセットアップ](chapter02_installation.md)
