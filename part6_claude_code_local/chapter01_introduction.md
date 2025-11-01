# Chapter 01: Claude CodeとローカルLLMの統合

## 1.1 Claude Codeとは

Claude Codeは、Anthropic社が提供するAI支援型開発環境（AI-powered coding assistant）です。通常はAnthropic社のクラウドAPIを使用しますが、適切な設定により**ローカルLLM**を使用することも可能です。

### 1.1.1 Claude Codeの特徴

**主な機能**
- コード生成と編集
- ファイル操作（読み込み、書き込み、検索）
- ターミナルコマンド実行
- プロジェクト全体の理解と分析
- Git操作
- マルチステップタスクの自動実行

**通常の利用（Anthropic API）**
```
[開発者] → [Claude Code CLI] → [Anthropic API] → [Claude 3.5 Sonnet]
                                     ↓
                               (クラウド・有料)
```

**ローカルLLM利用（本書の方法）**
```
[開発者] → [Claude Code CLI] → [LiteLLM Proxy] → [Ollama] → [Qwen2.5 14B]
                                     ↓                ↓
                              (ローカル・無料)   (MS-S1 Max)
```

### 1.1.2 なぜローカルLLMを使うのか

**メリット**
1. **コスト削減**: クラウドAPIの課金を回避（完全無料）
2. **プライバシー**: コードが外部に送信されない
3. **オフライン動作**: インターネット接続不要
4. **カスタマイズ**: 独自モデルやファインチューニングモデルの使用
5. **レスポンス速度**: MS-S1 Maxの高性能を活かした高速応答
6. **データ主権**: 企業の機密情報を守る

**デメリット（制限事項）**
1. Claude固有の機能が一部使えない可能性
2. ローカルモデルの性能に依存
3. 初期セットアップが必要
4. メモリとGPU資源を消費

### 1.1.3 MS-S1 Maxでの優位性

MS-S1 Max（AMD Ryzen AI Max+ 395、128GB RAM）は、ローカルLLM運用に最適な環境です。

**理由**
- **大容量メモリ（128GB）**: 大規模モデルを余裕で実行
- **統合GPU（16GB VRAM）**: ROCm対応で高速推論
- **高性能CPU（16コア）**: LiteLLMプロキシの並列処理
- **低消費電力**: クラウドと比較して圧倒的に低コスト

**推奨モデルと性能**

| モデル | パラメータ | メモリ | 速度（MS-S1 Max） | 用途 |
|--------|-----------|--------|-------------------|------|
| qwen2.5-coder:7b | 7B | 5.8GB | 32 tokens/s | コーディング（高速） |
| qwen2.5-coder:14b | 14B | 11.2GB | 18 tokens/s | コーディング（高品質） |
| qwen2.5:14b | 14B | 11.2GB | 18 tokens/s | 汎用（バランス型） |
| codellama:13b | 13B | 10GB | 20 tokens/s | コード特化 |
| deepseek-coder:6.7b | 6.7B | 5.2GB | 35 tokens/s | 軽量・高速 |

## 1.2 アーキテクチャ概要

### 1.2.1 コンポーネント構成

```
┌─────────────────────────────────────────────────────────┐
│                      開発者                              │
└───────────────────┬─────────────────────────────────────┘
                    │ (コマンド・質問)
                    ↓
┌─────────────────────────────────────────────────────────┐
│                  Claude Code CLI                         │
│  - ファイル操作                                          │
│  - Git統合                                               │
│  - ターミナル実行                                        │
└───────────────────┬─────────────────────────────────────┘
                    │ (OpenAI互換API)
                    ↓
┌─────────────────────────────────────────────────────────┐
│                 LiteLLM Proxy                            │
│  - APIフォーマット変換                                   │
│  - リクエストルーティング                                │
│  - ログ・キャッシュ                                      │
└───────────────────┬─────────────────────────────────────┘
                    │ (Ollama API)
                    ↓
┌─────────────────────────────────────────────────────────┐
│                    Ollama                                │
│  - モデル管理                                            │
│  - 推論エンジン                                          │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│              MS-S1 Max Hardware                          │
│  - AMD Ryzen AI Max+ 395 (16コア、128GB)                │
│  - Radeon 8060S (RDNA 3.5、16GB VRAM)                   │
│  - ROCm 6.4.2                                            │
└─────────────────────────────────────────────────────────┘
```

### 1.2.2 通信フロー

**1. ユーザーリクエスト**
```
開発者: "このファイルのバグを修正して"
    ↓
Claude Code CLI: ファイルを読み込み、コンテキストを構築
```

**2. API変換**
```
Claude Code → LiteLLM
{
  "model": "claude-3-5-sonnet-20241022",
  "messages": [...],
  "tools": [...]
}
    ↓
LiteLLM → Ollama (変換後)
{
  "model": "qwen2.5-coder:14b",
  "messages": [...],
  "stream": true
}
```

**3. 推論と応答**
```
Ollama (MS-S1 Max) → LiteLLM
{
  "choices": [{
    "message": {
      "content": "修正方法: ...",
      "tool_calls": [...]
    }
  }]
}
    ↓
LiteLLM → Claude Code (OpenAI形式)
    ↓
Claude Code: ファイルを編集、結果を表示
```

## 1.3 必要な前提知識

### 1.3.1 基本的なLinuxコマンド

この書籍を進めるには、以下のLinuxコマンドを理解している必要があります。

```bash
# ディレクトリ操作
cd /path/to/directory    # ディレクトリ移動
mkdir my_folder          # ディレクトリ作成
ls -la                   # ファイル一覧表示

# ファイル操作
cat file.txt             # ファイル内容表示
nano file.txt            # ファイル編集
chmod +x script.sh       # 実行権限付与

# プロセス管理
ps aux | grep ollama     # プロセス検索
kill -9 12345            # プロセス強制終了
systemctl status ollama  # サービス状態確認

# ネットワーク
curl http://localhost:11434/api/tags  # API動作確認
netstat -tuln | grep 8000             # ポート使用確認
```

### 1.3.2 Python基礎

LiteLLMはPythonで実装されています。基本的なPython知識があると理解が深まります。

```python
# 仮想環境作成
python3 -m venv venv
source venv/bin/activate

# パッケージインストール
pip install litellm

# 簡単なスクリプト実行
python3 script.py
```

### 1.3.3 JSONフォーマット

APIリクエスト・レスポンスはJSON形式です。

```json
{
  "model": "qwen2.5-coder:14b",
  "messages": [
    {
      "role": "user",
      "content": "Hello, World!"
    }
  ],
  "temperature": 0.7,
  "max_tokens": 2048
}
```

## 1.4 環境要件

### 1.4.1 ハードウェア要件

**最小要件**
- CPU: 4コア以上
- メモリ: 16GB以上
- ストレージ: 50GB以上の空き容量
- GPU: オプション（CPUでも動作）

**推奨要件（MS-S1 Max）**
- CPU: AMD Ryzen AI Max+ 395（16コア）
- メモリ: 128GB LPDDR5X-8000
- GPU: Radeon 8060S（16GB VRAM）
- ストレージ: 500GB以上のNVMe SSD

### 1.4.2 ソフトウェア要件

**OS**
- Ubuntu 24.04 LTS（推奨）
- Ubuntu 22.04 LTS
- その他のLinuxディストリビューション（要調整）

**必須ソフトウェア**
- Python 3.10以上
- Node.js 18以上（Claude Code CLI用）
- Git 2.30以上
- curl、wget

**AMD ROCm（GPU使用時）**
- ROCm 6.4.2（MS-S1 Max最適化版）

## 1.5 本書の構成

### Chapter 01: 導入（本章）
- Claude Codeとは
- ローカルLLM統合のメリット
- アーキテクチャ概要

### Chapter 02: 基本セットアップ
- Ollamaインストール
- モデルダウンロード
- 動作確認

### Chapter 03: LiteLLMのセットアップ
- LiteLLMインストール
- 設定ファイル作成
- プロキシ起動

### Chapter 04: Claude Code統合
- Claude Code CLIインストール
- 設定ファイル編集
- 初回起動と動作確認

### Chapter 05: 機能の互換性と制限
- 動作する機能
- 動作しない機能
- 回避策と代替手段

### Chapter 06: モデル選択と最適化
- コーディング特化モデルの比較
- MS-S1 Maxでの性能チューニング
- メモリ・GPU最適化

### Chapter 07: 実践例
- プロジェクト作成
- バグ修正
- リファクタリング
- テスト生成

### Chapter 08: トラブルシューティング
- よくある問題と解決方法
- ログの見方
- デバッグ手順

### Chapter 09: 高度な設定とベストプラクティス
- 複数モデルの使い分け
- キャッシュ活用
- パフォーマンスモニタリング
- コスト比較（Claude API vs ローカル）

## 1.6 セットアップの全体像

本書を通じて、以下の手順で環境を構築します。

**ステップ1: Ollama準備（Chapter 02）**
```bash
# Ollamaインストール
curl -fsSL https://ollama.com/install.sh | sh

# モデルダウンロード
ollama pull qwen2.5-coder:14b
```

**ステップ2: LiteLLM設定（Chapter 03）**
```bash
# LiteLLMインストール
pip install litellm[proxy]

# 設定ファイル作成
nano litellm_config.yaml

# プロキシ起動
litellm --config litellm_config.yaml
```

**ステップ3: Claude Code設定（Chapter 04）**
```bash
# Claude Code CLIインストール
npm install -g @anthropic-ai/claude-code

# 設定ファイル編集
nano ~/.config/claude-code/config.json

# 起動
claude-code
```

**ステップ4: 動作確認**
```bash
# Claude Codeで質問
You: "Hello, can you help me with Python?"
Assistant: "Of course! I'm running on Qwen2.5-Coder locally on your MS-S1 Max..."
```

## 1.7 期待される成果

本書の内容を実践することで、以下が実現できます。

### 1.7.1 コスト削減

**Claude API利用時（月額概算）**
- 軽度利用（10万トークン/月）: $3-5
- 中程度利用（100万トークン/月）: $30-50
- ヘビー利用（1000万トークン/月）: $300-500

**ローカルLLM利用時**
- 初期費用: $0（既存のMS-S1 Max利用）
- 月額費用: 電気代のみ（約$3-5、24時間稼働想定）
- **年間節約額**: $360-$6,000

### 1.7.2 プライバシー保護

- **社内コード**: 外部に送信されない
- **機密情報**: ローカルで完結
- **知的財産**: 完全に保護される

### 1.7.3 開発効率向上

**実測データ（MS-S1 Max + Qwen2.5-Coder 14B）**
- コード生成速度: 18 tokens/s
- 応答時間: 平均2-5秒
- 同時処理: 複数セッション対応（128GBメモリ）

**生産性向上**
- コーディング時間: 30-50%短縮
- バグ修正: 2-3倍高速化
- ドキュメント作成: 自動化

## 1.8 まとめ

本章では、Claude CodeとローカルLLMを統合する意義と全体像を学びました。

**重要ポイント**
1. Claude CodeはLiteLLM経由でOllamaを使用できる
2. MS-S1 Maxは大規模モデルの運用に最適
3. 完全無料でプライバシーを保護しながら高品質な開発支援が可能
4. 適切な設定と理解が必要

次章では、実際にOllamaをインストールし、モデルをダウンロードする手順を学びます。手を動かしながら、ステップバイステップで環境を構築していきましょう。
