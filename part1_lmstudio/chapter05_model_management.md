# 第5章：モデルのダウンロードと管理

## 5.1 LLMモデルの基礎知識

### 5.1.1 モデルファミリーとは

**モデルファミリー**は、同じベースアーキテクチャを共有する一連のモデル群です。

#### 主要なモデルファミリー（2025年）

**1. Llama（Meta）**
```
開発元: Meta AI
特徴: 汎用性が高い、オープンソース
バージョン: Llama 3.1, Llama 3.2
サイズ展開: 1B, 3B, 8B, 70B, 405B
ライセンス: Llama 3 Community License
日本語能力: 中〜高（ファインチューン版で向上）
```

**2. Qwen（Alibaba）**
```
開発元: Alibaba Cloud
特徴: 多言語対応、日本語に強い
バージョン: Qwen2.5
サイズ展開: 0.5B, 1.5B, 3B, 7B, 14B, 32B, 72B
ライセンス: Apache 2.0
日本語能力: 非常に高い
推奨: 日本語ユーザーに最適 ⭐
```

**3. Mistral（Mistral AI）**
```
開発元: Mistral AI（フランス）
特徴: 高性能、効率的
バージョン: Mistral 7B, Mixtral 8x7B, Mixtral 8x22B
サイズ展開: 7B, 8x7B（MoE）, 8x22B（MoE）
ライセンス: Apache 2.0
日本語能力: 中程度
```

**4. Gemma（Google）**
```
開発元: Google DeepMind
特徴: 軽量、高効率
バージョン: Gemma 2
サイズ展開: 2B, 9B, 27B
ライセンス: Gemma Terms of Use
日本語能力: 中程度
```

**5. DeepSeek（DeepSeek AI）**
```
開発元: DeepSeek（中国）
特徴: コーディングに特化
バージョン: DeepSeek-Coder V2
サイズ展開: 16B, 236B
ライセンス: DeepSeek License
日本語能力: 中〜高
推奨: プログラミング用途 ⭐
```

### 5.1.2 量子化（Quantization）とは

**量子化**は、モデルの重みのデータ精度を下げてサイズを削減する技術です。

#### 量子化レベルの比較

| 量子化 | ビット数 | サイズ比 | 品質 | 推奨用途 |
|--------|---------|---------|------|----------|
| F32 | 32ビット | 100% | 最高 | 研究用（通常使用しない） |
| F16 | 16ビット | 50% | 最高 | 完全な品質が必要な場合 |
| Q8_0 | 8ビット | 25% | 非常に高 | 高品質推論 |
| Q6_K | 6ビット | 19% | 高 | バランス重視（高品質） |
| Q5_K_M | 5ビット | 16% | 高 | バランス重視 |
| **Q4_K_M** | 4ビット | **12.5%** | **良好** | **最も推奨** ⭐ |
| Q4_K_S | 4ビット | 12% | 良好 | サイズ優先 |
| Q3_K_M | 3ビット | 9% | 中 | メモリ不足時 |
| Q2_K | 2ビット | 6% | 低 | 非推奨 |

**K-Quants（K量子化）**

`K`サフィックスは、より高度な量子化手法を示します。

```
_M (Medium): バランス型（推奨）
_S (Small): サイズ優先
_L (Large): 品質優先
```

**💡 TIP**: MS-S1 Maxの128GBメモリでは、**Q4_K_M**が最適なバランスです。品質を維持しながらサイズを削減できます。

### 5.1.3 GGUF形式

**GGUF（GPT-Generated Unified Format）**は、LLMの標準的なファイル形式です。

**特徴:**
- 単一ファイルで完結
- メタデータ埋め込み
- 高速ロード
- クロスプラットフォーム対応

**ファイル命名規則:**
```
qwen2.5-7b-instruct-q4_k_m.gguf
│    │  │  │        │
│    │  │  │        └─ 量子化レベル
│    │  │  └────────── モデルタイプ（instruct = チャット用）
│    │  └───────────── パラメータ数（7B = 70億）
│    └──────────────── バージョン
└───────────────────── モデルファミリー
```

## 5.2 モデルの検索とダウンロード

### 5.2.1 LM Studio内での検索

**基本的な検索手順**

1. **Searchタブを開く**
   ```
   LM Studio起動 → 左側タブから「🔍 Search」をクリック
   ```

2. **検索フィールドに入力**
   ```
   例: "qwen2.5 7b q4"
   ```

3. **フィルタリング**
   ```
   サイズ: 1B-10B / 10B-30B / 30B+
   量子化: Q4 / Q5 / Q6 / Q8
   モデルタイプ: Instruct / Base / Chat
   ```

4. **並び替え**
   ```
   人気順 / 新着順 / サイズ順 / 名前順
   ```

### 5.2.2 推奨モデルカタログ（MS-S1 Max向け）

#### 初心者向け（3B-7B）

**Qwen2.5 7B Instruct**
```
モデル名: Qwen/Qwen2.5-7B-Instruct-GGUF
ファイル: qwen2.5-7b-instruct-q4_k_m.gguf
サイズ: 4.8GB
メモリ: 約6GB
速度: 35-45 t/s
特徴:
  ✓ 日本語に非常に強い
  ✓ 高速レスポンス
  ✓ 汎用性が高い
推奨用途: 日常的なチャット、質問応答、簡単な文章生成
```

**Llama 3.2 3B Instruct**
```
モデル名: meta-llama/Llama-3.2-3B-Instruct-GGUF
ファイル: llama-3.2-3b-instruct-q4_k_m.gguf
サイズ: 2.0GB
メモリ: 約3GB
速度: 50-60 t/s
特徴:
  ✓ 超高速
  ✓ 軽量
  ✓ 最新アーキテクチャ
推奨用途: 高速応答が必要な用途、リソース節約
```

#### 中級者向け（13B-34B）

**Qwen2.5 14B Instruct**
```
モデル名: Qwen/Qwen2.5-14B-Instruct-GGUF
ファイル: qwen2.5-14b-instruct-q4_k_m.gguf
サイズ: 9.0GB
メモリ: 約12GB
速度: 20-25 t/s
特徴:
  ✓ 7Bより高品質
  ✓ 複雑な指示理解
  ✓ 日本語の文脈理解が優秀
推奨用途: 本格的な文章生成、翻訳、要約
```

**Qwen2.5 32B Instruct**
```
モデル名: Qwen/Qwen2.5-32B-Instruct-GGUF
ファイル: qwen2.5-32b-instruct-q4_k_m.gguf
サイズ: 20GB
メモリ: 約24GB
速度: 8-12 t/s
特徴:
  ✓ 非常に高い理解力
  ✓ 複雑な推論が可能
  ✓ 創造的な文章生成
推奨用途: プロフェッショナルな文章作成、高度な質問応答
```

#### 上級者向け（70B+）

**Llama 3.1 70B Instruct**
```
モデル名: meta-llama/Meta-Llama-3.1-70B-Instruct-GGUF
ファイル: meta-llama-3.1-70b-instruct-q4_k_m.gguf
サイズ: 42GB
メモリ: 約48GB
速度: 3-5 t/s
特徴:
  ✓ トップクラスの性能
  ✓ GPT-4レベルの能力
  ✓ 複雑な推論・分析
推奨用途: 最高品質が必要な用途、研究、専門的な分析
```

**Qwen2.5 72B Instruct**
```
モデル名: Qwen/Qwen2.5-72B-Instruct-GGUF
ファイル: qwen2.5-72b-instruct-q4_k_m.gguf
サイズ: 44GB
メモリ: 約50GB
速度: 3-5 t/s
特徴:
  ✓ Qwen最大級モデル
  ✓ 日本語で最高クラスの性能
  ✓ 128Kコンテキスト対応
推奨用途: 日本語での最高品質推論、長文処理
```

#### 専門用途

**DeepSeek-Coder-V2 16B**
```
モデル名: deepseek-ai/DeepSeek-Coder-V2-Lite-Instruct-GGUF
ファイル: deepseek-coder-v2-lite-instruct-q4_k_m.gguf
サイズ: 10GB
メモリ: 約13GB
速度: 18-22 t/s
特徴:
  ✓ コーディング特化
  ✓ 多言語プログラミング対応
  ✓ コード生成・デバッグ
推奨用途: プログラミング支援、コードレビュー、デバッグ
```

### 5.2.3 ダウンロード手順

**標準ダウンロード**

1. モデルを選択
2. 量子化レベルを選択（Q4_K_M推奨）
3. 「Download」ボタンをクリック
4. ダウンロード進行状況を確認
   ```
   Downloading... 25% (1.2 GB / 4.8 GB)
   Speed: 15 MB/s
   ETA: 4 minutes
   ```
5. 完了を待つ

**💡 TIP**: ダウンロード中もLM Studioは使用可能です。他のモデルで推論を実行できます。

**複数モデルの同時ダウンロード**

LM Studioは最大3つのモデルを並列ダウンロードできます（デフォルト設定）。

```
Settings → Network → Concurrent Downloads: 3
```

**ダウンロードの一時停止・再開**

```
一時停止: ダウンロードバーの「⏸」ボタン
再開: 「▶」ボタン
キャンセル: 「✕」ボタン
```

### 5.2.4 Hugging Face直接ダウンロード

**Hugging Faceからの手動ダウンロード**

LM Studioに表示されないモデルは、Hugging Faceから直接ダウンロードできます。

**手順:**

1. **ブラウザでHugging Faceにアクセス**
   ```
   https://huggingface.co/
   ```

2. **モデルを検索**
   ```
   検索欄に "qwen2.5 gguf" などと入力
   ```

3. **GGUFファイルをダウンロード**
   ```
   モデルページ → Files and versions → .gguf ファイルをクリック
   ```

4. **LM Studioのモデルディレクトリに配置**
   ```
   Windows: C:\Users\<ユーザー名>\.cache\lm-studio\models\
   Linux: ~/.cache/lm-studio/models/

   または設定したカスタムディレクトリ
   ```

5. **LM Studioで認識確認**
   ```
   My Modelsタブを開く → 手動配置したモデルが表示される
   ```

**Git LFS を使った大容量モデルのダウンロード（Linux）**

```bash
# Git LFSのインストール
sudo apt install git-lfs
git lfs install

# モデルのクローン
cd ~/.cache/lm-studio/models/
git clone https://huggingface.co/Qwen/Qwen2.5-72B-Instruct-GGUF

# 特定のファイルのみダウンロード
git lfs pull --include="qwen2.5-72b-instruct-q4_k_m.gguf"
```

## 5.3 モデルの管理

### 5.3.1 My Modelsタブ

**モデル一覧の表示**

```
LM Studio → 📁 My Models
```

**表示情報:**
```
┌──────────────────────────────────────────────────────────┐
│ Model Name              | Size  | Quantization | Added   │
├──────────────────────────────────────────────────────────┤
│ qwen2.5-7b-instruct     | 4.8GB | Q4_K_M       | 2日前   │
│ llama-3.1-70b-instruct  | 42GB  | Q4_K_M       | 1週間前 │
│ deepseek-coder-v2       | 10GB  | Q4_K_M       | 昨日    │
└──────────────────────────────────────────────────────────┘
```

**ソートとフィルター**

```
並び替え:
  - 名前順（A-Z / Z-A）
  - サイズ順（大 → 小 / 小 → 大）
  - 最近使用（新 → 古 / 古 → 新）
  - 追加日時

フィルター:
  - サイズ範囲（0-10GB / 10-50GB / 50GB+）
  - 量子化レベル（Q4 / Q5 / Q6）
  - モデルファミリー
```

### 5.3.2 モデルの詳細情報

**モデルの情報表示**

モデルを右クリック → 「Model Info」

```
モデル情報:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
名前: qwen2.5-7b-instruct-q4_k_m.gguf
サイズ: 4.8 GB
量子化: Q4_K_M
アーキテクチャ: Llama（Qwen2.5）
パラメータ数: 7.62B
コンテキスト長: 32768 tokens
語彙サイズ: 151,936 tokens
追加日: 2025-10-15 14:23:45
最終使用: 2025-10-28 09:15:32
使用回数: 47回
ファイルパス: ~/.cache/lm-studio/models/qwen2.5-7b-instruct-q4_k_m.gguf
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 5.3.3 モデルの削除

**個別削除**

```
My Models → モデルを右クリック → Delete
確認ダイアログ → 「削除」をクリック
```

**一括削除**

```
My Models → 複数のモデルを Ctrl+クリック（Windows/Linux）で選択
右クリック → Delete Selected
```

**⚠️ 注意**: 削除したモデルは復元できません。再度使用するには再ダウンロードが必要です。

**ストレージ使用量の確認**

```
Settings → Storage → Storage Usage

表示内容:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Models: 156.3 GB（23モデル）
Cache: 4.2 GB
Logs: 128 MB
Total: 160.6 GB

ディスク空き容量: 1.2 TB / 2 TB
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 5.3.4 モデルのエクスポートとインポート

**モデルのエクスポート**

他のマシンやユーザーとモデルを共有する場合：

```bash
# モデルファイルをコピー
# Windows
copy "%USERPROFILE%\.cache\lm-studio\models\qwen2.5-7b-instruct-q4_k_m.gguf" D:\Shared\

# Linux
cp ~/.cache/lm-studio/models/qwen2.5-7b-instruct-q4_k_m.gguf /mnt/shared/
```

**モデルのインポート**

```
方法1: ファイルマネージャーから直接コピー
  → モデルディレクトリに.ggufファイルを配置

方法2: LM Studio内でのインポート
  My Models → ⚙ → Import Model → ファイルを選択
```

### 5.3.5 モデルの整理戦略

**推奨ディレクトリ構成**

```
models/
├── daily_use/          # 日常使用（7B-14B）
│   ├── qwen2.5-7b-instruct-q4_k_m.gguf
│   └── llama-3.2-3b-instruct-q4_k_m.gguf
├── professional/       # プロフェッショナル用途（32B-72B）
│   ├── qwen2.5-32b-instruct-q4_k_m.gguf
│   └── llama-3.1-70b-instruct-q4_k_m.gguf
├── specialized/        # 特殊用途
│   ├── deepseek-coder-v2-lite-q4_k_m.gguf
│   └── mistral-7b-instruct-v0.3-q4_k_m.gguf
└── experimental/       # 実験・テスト用
    └── new-model-test-q4_k_m.gguf
```

**💡 TIP**: LM Studioは現在サブディレクトリをサポートしていませんが、ファイル名にプレフィックスを付けることで整理できます。

```
例:
daily_qwen2.5-7b-instruct-q4_k_m.gguf
pro_llama-3.1-70b-instruct-q4_k_m.gguf
code_deepseek-coder-v2-q4_k_m.gguf
```

## 5.4 モデルの更新と最新版の追跡

### 5.4.1 モデルのバージョン管理

**モデルの更新頻度**

```
頻繁に更新されるモデル:
  - Qwen: 月1-2回（マイナーアップデート）
  - Llama: 四半期ごと（メジャーアップデート）
  - Mistral: 不定期（2-3ヶ月）

更新内容:
  - バグ修正
  - 性能向上
  - 新機能追加
  - 安全性向上
```

**更新通知の設定**

```
Settings → Updates
[✓] モデルの新バージョンをチェック
[✓] 新しいモデルファミリーの通知
```

### 5.4.2 モデルの比較

**異なるバージョンの比較**

```
My Models → 比較したいモデルを2つ選択
右クリック → Compare Models

表示内容:
┌────────────────────────────────────────────────────┐
│ Feature        │ Qwen2.5 7B  │ Llama 3.2 3B    │
├────────────────────────────────────────────────────┤
│ Parameters     │ 7.62B       │ 3.21B           │
│ Context Length │ 32K         │ 128K            │
│ Quantization   │ Q4_K_M      │ Q4_K_M          │
│ Size           │ 4.8GB       │ 2.0GB           │
│ Speed (Est.)   │ 35-45 t/s   │ 50-60 t/s       │
│ Japanese       │ ★★★★★      │ ★★★☆☆         │
│ English        │ ★★★★☆      │ ★★★★★         │
│ Coding         │ ★★★★☆      │ ★★★☆☆         │
└────────────────────────────────────────────────────┘
```

### 5.4.3 推奨モデルアップデート戦略

**MS-S1 Max向け推奨セット（合計80GB）**

```
最小構成（20GB）:
  ✓ Qwen2.5 7B Q4_K_M（4.8GB）- メイン
  ✓ Llama 3.2 3B Q4_K_M（2.0GB）- 高速用
  ✓ DeepSeek-Coder V2 16B Q4_K_M（10GB）- コーディング用
  残り: 108GB

標準構成（50GB）:
  ✓ Qwen2.5 14B Q4_K_M（9GB）- メイン
  ✓ Llama 3.1 8B Q4_K_M（5GB）- 英語用
  ✓ Qwen2.5 32B Q4_K_M（20GB）- 高品質用
  ✓ DeepSeek-Coder V2 16B Q4_K_M（10GB）- コーディング用
  ✓ Llama 3.2 3B Q4_K_M（2GB）- 高速用
  残り: 78GB

プロ構成（100GB）:
  ✓ すべての上記モデル
  ✓ Llama 3.1 70B Q4_K_M（42GB）- 最高品質用
  または
  ✓ Qwen2.5 72B Q4_K_M（44GB）- 日本語最高品質
  残り: 28GB
```

## 5.5 トラブルシューティング

### 5.5.1 ダウンロード失敗

**症状: ダウンロードが途中で止まる**

```
原因1: ネットワーク接続の問題
解決: ダウンロードを一時停止 → 再開

原因2: Hugging Faceサーバーの一時的な問題
解決: 30分後に再試行

原因3: ディスク容量不足
解決: 不要なファイルを削除して空き容量を確保
```

**症状: 破損したファイル**

```
エラーメッセージ: "Corrupted model file" or "Invalid GGUF format"

解決手順:
1. モデルを削除
2. ブラウザキャッシュをクリア
3. 再ダウンロード
```

### 5.5.2 モデルが認識されない

**症状: ダウンロードしたモデルがMy Modelsに表示されない**

```
原因1: ファイルが正しいディレクトリにない
確認: Settings → Storage → Models Directory を確認
解決: 正しいディレクトリに移動

原因2: ファイルが完全にダウンロードされていない
確認: ファイルサイズをHugging Faceの表示と比較
解決: 再ダウンロード

原因3: LM Studioの再読み込みが必要
解決: LM Studioを再起動
```

### 5.5.3 モデルロード失敗

**症状: "Failed to load model"**

```
原因1: メモリ不足
確認: タスクマネージャー/htop でメモリ使用量確認
解決:
  - 他のアプリケーションを終了
  - より小さい量子化レベルを選択
  - GPU Layers を減らす

原因2: 破損したモデルファイル
解決:
  - モデルを削除
  - 再ダウンロード

原因3: 互換性の問題
確認: LM Studio のバージョン
解決: LM Studio を最新版に更新
```

## 5.6 本章のまとめ

本章では、モデルのダウンロードと管理について学習しました。

✅ **LLMモデルの基礎**
- 主要モデルファミリー（Llama, Qwen, Mistral等）
- 量子化レベル（Q4_K_M推奨）
- GGUF形式の理解

✅ **モデルの検索とダウンロード**
- LM Studio内での検索
- MS-S1 Max向け推奨モデルカタログ
- Hugging Faceからの直接ダウンロード

✅ **モデル管理**
- My Modelsタブの活用
- 整理戦略
- エクスポート・インポート

✅ **最適な構成**
- 最小構成（20GB）
- 標準構成（50GB）
- プロ構成（100GB）

次章では、推論設定の詳細を学び、各パラメータが推論結果にどう影響するかを理解します。

---

**前章へ**: [第4章 AMD GPU設定の完全ガイド](chapter04_amd_gpu_settings.md)
**次章へ**: [第6章 推論設定の完全解説](chapter06_inference_settings.md)
