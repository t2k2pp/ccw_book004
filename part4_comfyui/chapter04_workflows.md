# 第4章: ComfyUIワークフロー作成とカスタムノード

## 4.1 ワークフローの基本概念

### 4.1.1 ワークフローとは

ComfyUIにおけるワークフローは、複数のノードが接続されたグラフィカルインターフェースであり、AI画像生成プロセス全体を記述します。

**ワークフローの特徴**
```
視覚的プログラミング:
- ノードとエッジで処理フロー構築
- データの流れを視覚的に確認
- 複雑なパイプラインを管理

再利用性:
- ワークフローを保存・共有
- JSON形式でエクスポート
- 画像にメタデータとして埋め込み

柔軟性:
- カスタムノードで機能拡張
- 条件分岐や反復処理
- 複数モデルの組み合わせ
```

### 4.1.2 ワークフローの保存と読み込み

**保存方法**
```javascript
// 方法1: UIから保存
// 1. 右クリック→Save Workflow
// 2. workflow.json として保存

// 方法2: 生成画像から復元
// 1. ComfyUIで生成した画像をドラッグ&ドロップ
// 2. メタデータからワークフローを自動復元

// 方法3: API形式で保存
// JSONファイルに完全なワークフロー定義を保存
```

**MS-S1 Max推奨ワークフロー管理**
```bash
# ワークフローディレクトリ構造
~/ai-tools/ComfyUI/workflows/
├── basic/
│   ├── text-to-image-simple.json
│   ├── image-to-image-basic.json
│   └── upscale-basic.json
├── advanced/
│   ├── controlnet-depth.json
│   ├── lora-mixing.json
│   └── refiner-workflow.json
└── production/
    ├── batch-generation.json
    ├── multi-model.json
    └── automated-pipeline.json
```

## 4.2 基本ワークフローの構築

### 4.2.1 Text-to-Imageワークフロー

**最小構成（6ノード）**
```
┌─────────────────┐
│Load Checkpoint  │
│ sd_xl_base_1.0  │
└────┬───┬───┬────┘
     │   │   │
     │   │   └──→ MODEL → [KSampler]
     │   │
     │   └──→ CLIP → [CLIP Text Encode (Positive)]
     │                      ↓ CONDITIONING
     │                      ↓
     └──→ VAE         [KSampler] ←─ [Empty Latent Image]
                            ↓
                            ↓ LATENT
                      [VAE Decode]
                            ↓ IMAGE
                      [Save Image]
```

**詳細設定**
```yaml
Load Checkpoint:
  ckpt_name: "sd_xl_base_1.0.safetensors"

CLIP Text Encode (Positive):
  text: "Your detailed prompt here"

CLIP Text Encode (Negative):
  text: "low quality, blurry"

Empty Latent Image:
  width: 1024
  height: 1024
  batch_size: 1

KSampler:
  seed: 42
  steps: 25
  cfg: 7.5
  sampler_name: "dpmpp_2m_karras"
  scheduler: "karras"
  denoise: 1.0

Save Image:
  filename_prefix: "ComfyUI"
```

### 4.2.2 Image-to-Imageワークフロー

**構成（8ノード）**
```
[Load Image] → IMAGE → [VAE Encode]
                              ↓ LATENT
                              ↓
[Load Checkpoint] → MODEL → [KSampler] ← denoise: 0.75
                   ↓ CLIP            ↑
                   ↓                 └─ [CLIP Text Encode]
                   ↓ VAE
                   ↓
            [VAE Decode] ← LATENT ← [KSampler]
                   ↓ IMAGE
            [Save Image]
```

**重要パラメータ: denoise**
```
denoise = 1.0:
- 完全に新規生成
- 元画像の影響最小
- Text-to-Imageと同等

denoise = 0.75:
- バランス良好（推奨）
- 元画像の構図維持
- 詳細を変更

denoise = 0.5:
- 元画像を強く維持
- 微調整に最適
- スタイル変更

denoise = 0.25:
- 最小限の変更
- 色調整レベル
- 構図ほぼ保持
```

### 4.2.3 SDXL Refinerワークフロー

**2段階生成（10ノード）**
```
[Load Checkpoint: Base] → MODEL → [KSampler: Base]
                                      ↓ steps: 25
                                      ↓ denoise: 1.0
                                      ↓ LATENT
                                      ↓
[Load Checkpoint: Refiner] → MODEL → [KSampler: Refiner]
                                      ↓ steps: 15
                                      ↓ denoise: 0.3
                                      ↓ LATENT
                              [VAE Decode]
                                      ↓ IMAGE
                              [Save Image]
```

**MS-S1 Max最適設定**
```python
# Base生成
base_steps = 25
base_denoise = 1.0
estimated_time = 11  # 秒

# Refiner適用
refiner_steps = 15
refiner_denoise = 0.3  # Baseの30%を再生成
estimated_time_refiner = 5  # 秒

# 合計
total_time = 16  # 秒

# メモリ使用量
base_model_mem = 8   # GB
refiner_model_mem = 8  # GB
vram_usage = 16  # GB（両方ロード時）
```

## 4.3 カスタムノードの導入

### 4.3.1 ComfyUI Manager（必須）

ComfyUI Managerは、すべてのカスタムノード管理の基盤です。

**インストール**
```bash
cd ~/ai-tools/ComfyUI/custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
cd ComfyUI-Manager
pip install -r requirements.txt

# ComfyUI再起動
cd ~/ai-tools/ComfyUI
python main.py
```

**主要機能**
```
ノード管理:
✅ カスタムノードの検索とインストール
✅ 依存関係の自動解決
✅ ワンクリック更新
✅ 不足ノードの自動検出

ワークフロー管理:
✅ ワークフロー共有サイトからインポート
✅ 不足ノードの一括インストール
✅ "Red Box Hell"（エラー）の解消

モデル管理:
✅ Hugging Faceからのダウンロード
✅ CivitAIモデル検索
✅ モデルの自動配置
```

### 4.3.2 必須カスタムノード（2025年版）

**1. Efficiency Nodes for ComfyUI**
```bash
# インストール
git clone https://github.com/jags111/efficiency-nodes-comfyui.git

# 機能
- 複数ノードを1つに統合
- KSampler + VAE Decode + Save を統合
- プロンプト管理を簡略化
- バッチ処理の効率化

# MS-S1 Maxメリット
- ワークフロー可読性向上
- ノード数削減でメモリ効率改善
- 設定変更が容易
```

**2. ComfyUI Impact Pack**
```bash
# インストール
git clone https://github.com/ltdrdata/ComfyUI-Impact-Pack.git
cd ComfyUI-Impact-Pack
pip install -r requirements.txt

# 機能
- 高度な後処理
- 顔検出とDetailer
- セグメンテーション
- 自動マスク生成

# 用途
- ポートレート品質向上
- 顔の詳細強化
- 自動修正
```

**3. WAS Node Suite**
```bash
# インストール
git clone https://github.com/WASasquatch/was-node-suite-comfyui.git
cd was-node-suite-comfyui
pip install -r requirements.txt

# 機能
- 画像処理ユーティリティ
- テキスト操作
- 数学演算
- ファイルシステム操作

# 用途
- 複雑な画像処理
- バッチ処理自動化
- カスタムパイプライン
```

**4. ComfyUI Ultimate SD Upscale**
```bash
# インストール
git clone https://github.com/ssitu/ComfyUI_UltimateSDUpscale.git

# 機能
- タイル式アップスケール
- メモリ効率的な拡大
- 4倍、8倍対応

# MS-S1 Maxでの利点
- 128GBメモリで大規模画像処理
- 2048x2048 → 8192x8192可能
- GPUとRAMの効率的活用
```

**5. rgthree's ComfyUI Nodes**
```bash
# インストール
git clone https://github.com/rgthree/rgthree-comfy.git
cd rgthree-comfy
pip install -r requirements.txt

# 機能
- Power Prompt: 高度なプロンプト管理
- Quick Nodes: ノード追加効率化
- Context Switch: 条件分岐
- Display Any: デバッグ表示

# 用途
- ワークフロー開発効率化
- デバッグ作業
- 複雑な条件処理
```

## 4.4 高度なワークフロー構築

### 4.4.1 バッチ生成ワークフロー

**複数シード生成**
```
目的: 同一プロンプトで複数バリエーション

構成:
[Load Checkpoint]
         ↓
[CLIP Text Encode (Positive)]
[CLIP Text Encode (Negative)]
         ↓
[Empty Latent Image]
  batch_size: 4  ← 重要
         ↓
[KSampler]
  seed: random  ← 毎回変更
  control_after_generate: "randomize"
         ↓
[VAE Decode]
         ↓
[Save Image]

結果: 1回の実行で4枚生成
MS-S1 Max時間: 35-40秒（@ 1024x1024）
```

**グリッド生成**
```python
# 異なるパラメータで一括生成

# プロンプトバリエーション
prompts = [
    "photo of a cat, outdoor",
    "photo of a cat, indoor",
    "photo of a cat, studio lighting",
    "photo of a cat, natural lighting"
]

# CFGバリエーション
cfg_values = [6.0, 7.0, 7.5, 8.0]

# 組み合わせ: 4 prompts × 4 CFGs = 16画像
# MS-S1 Max時間: 約3分
```

### 4.4.2 自動Img2Imgパイプライン

**反復改善ワークフロー**
```
[Load Image (元画像)]
         ↓
[VAE Encode] → LATENT
         ↓
  ┌──→ [KSampler] denoise: 0.5
  │        ↓ LATENT
  │   [VAE Decode]
  │        ↓ IMAGE
  │   [Preview Image]
  │        ↓
  │   [VAE Encode]
  │        ↓
  └────── LATENT （ループ）

用途:
- 段階的な品質向上
- スタイル調整の微調整
- 試行錯誤の自動化
```

### 4.4.3 条件分岐ワークフロー

**解像度別処理**
```python
# rgthree Context Switch使用

if resolution == "1024x1024":
    steps = 25
    cfg = 7.5
    use_refiner = False
elif resolution == "1536x1536":
    steps = 30
    cfg = 7.0
    use_refiner = True
elif resolution == "2048x2048":
    steps = 35
    cfg = 6.5
    use_refiner = True
    # タイルアップスケール使用
```

## 4.5 MS-S1 Max最適化ワークフロー

### 4.5.1 メモリ効率的な大規模生成

**8K生成ワークフロー**
```
戦略: タイル式生成 + アップスケール

Step 1: Base生成（1024x1024）
[KSampler] → IMAGE (1024x1024)
         ↓
時間: 11秒
VRAM: 8GB

Step 2: Ultimate SD Upscale (2x)
[Ultimate SD Upscale] → IMAGE (2048x2048)
  tile_size: 512
  overlap: 64
         ↓
時間: 35秒
VRAM: 12GB（ピーク）

Step 3: Ultimate SD Upscale (2x again)
[Ultimate SD Upscale] → IMAGE (4096x4096)
  tile_size: 512
  overlap: 64
         ↓
時間: 140秒
VRAM: 12GB（ピーク）

合計時間: 約3分
最終解像度: 4096x4096
総VRAM: 12GB（タイル処理で効率化）
```

### 4.5.2 並列モデル実行

**128GBメモリ活用**
```
同時ロードモデル:

1. SDXL Base (8GB)
2. SDXL Refiner (8GB)
3. SD 1.5 Anime Model (4GB)
4. ControlNet Models (6GB)
5. LoRA Models (2GB)
6. VAE Models (1GB)
7. Upscale Models (1GB)

合計: 30GB
残りメモリ: 98GB（システム用）

利点:
- モデル切り替え時間ゼロ
- 複数ワークフロー同時実行
- バックグラウンド処理
```

## 4.6 実践的ワークフロー例

### 4.6.1 プロダクション品質ポートレート

**完全ワークフロー（15ノード）**
```yaml
フェーズ1: Base生成
- Load Checkpoint: SDXL Base
- CLIP Text Encode: 詳細プロンプト
- Empty Latent Image: 832x1216
- KSampler: steps=25, cfg=7.5
- 時間: 11秒

フェーズ2: Refiner適用
- Load Checkpoint: SDXL Refiner
- KSampler: steps=15, denoise=0.3
- 時間: 5秒

フェーズ3: 顔検出と強化
- Impact Pack Face Detailer
- Upscale: 1.5x
- 詳細強化
- 時間: 8秒

フェーズ4: 最終調整
- Color Correction
- Sharpen
- Save Image
- 時間: 2秒

合計時間: 26秒
最終品質: プロフェッショナル
```

### 4.6.2 バッチコンセプトアート

**大量生成ワークフロー**
```python
設定:
- Batch Size: 8
- Resolution: 1024x1024
- Steps: 20（速度優先）
- Sampler: euler_a
- CFG: 7.0
- Seeds: ランダム

処理フロー:
1. プロンプト準備（10バリエーション）
2. 各プロンプトで8画像生成
3. 合計80画像
4. 総時間: 約15分（MS-S1 Max）

出力:
- 80枚のコンセプトアート
- 自動的にフォルダ分け
- メタデータ付き
```

### 4.6.3 スタイル転送ワークフロー

**Image-to-Image + LoRA**
```
[Load Image (元画像)]
         ↓
[VAE Encode]
         ↓
[Load Checkpoint + LoRA]
  LoRA: anime_style.safetensors
  strength: 0.8
         ↓
[KSampler]
  denoise: 0.7
  prompt: "anime style, {original description}"
         ↓
[VAE Decode]
         ↓
[Save Image]

用途:
- 写真をアニメ化
- 実写→イラスト
- スタイル一括変換
```

## 4.7 ワークフローのデバッグ

### 4.7.1 一般的なエラーと解決

**エラー1: Red Node（ノードエラー）**
```
原因:
- カスタムノードが未インストール
- 依存関係が不足
- モデルファイルが見つからない

解決策:
1. ComfyUI Manager起動
2. "Install Missing Custom Nodes"クリック
3. 自動インストール実行
4. ComfyUI再起動
```

**エラー2: 接続の型不一致**
```
症状:
- ノード間を接続できない
- 線が赤くなる

原因:
- 出力と入力の型が不一致
  例: IMAGE → LATENT (不可)

解決策:
- 正しい変換ノード挿入
  IMAGE → [VAE Encode] → LATENT
  LATENT → [VAE Decode] → IMAGE
```

**エラー3: メモリ不足**
```
症状:
- "HIP out of memory"
- 生成が途中で失敗

解決策（MS-S1 Max）:
1. Batch Sizeを削減
2. 解像度を下げる
3. --lowvramフラグで起動
4. タイル処理を使用

# 通常128GBでは発生しない
# 発生時はGPU割り当て確認
export GPU_MAX_ALLOC_PERCENT=90
```

### 4.7.2 パフォーマンス最適化

**ワークフロー最適化チェックリスト**
```yaml
□ 不要なPreview Imageノード削除
  - デバッグ後は削除
  - 各Preview: +0.5秒

□ VAE Decodeの最小化
  - 最終出力のみVAE Decode
  - 中間はLATENTのまま

□ モデルロードの削減
  - 同じモデルを複数回ロードしない
  - 1つのLoad Checkpointを共有

□ Batch処理の活用
  - 単一生成より効率的
  - バッチサイズ4推奨

□ 適切なSteps数
  - 20-25で十分な場合多い
  - 50+は通常不要
```

**MS-S1 Maxベンチマーク**
```python
# ワークフロー効率測定

# 非最適化ワークフロー
nodes = 25
preview_nodes = 5
multiple_vae_decodes = 4
time = 35  # 秒

# 最適化ワークフロー
nodes = 12  # 統合ノード使用
preview_nodes = 1
vae_decodes = 1
time = 18  # 秒

# 改善率: 48%高速化
```

## 4.8 ワークフロー共有とコミュニティ

### 4.8.1 ワークフロー共有サイト

**主要プラットフォーム**
```
OpenArt.ai/workflows:
- 大規模ワークフローコレクション
- カテゴリ別検索
- ダウンロード数表示
- コメント機能

ComfyWorkflows.com:
- 専門ワークフローサイト
- タグベース検索
- 難易度表示

GitHub repositories:
- comfyanonymous/ComfyUI_examples
- 公式例集
- 定期更新
```

### 4.8.2 ワークフローのエクスポート/インポート

**エクスポート手順**
```
方法1: JSON保存
1. ワークフロー完成
2. 右クリック → Save Workflow
3. descriptive_name.json として保存
4. GitHubやドライブで共有

方法2: 画像埋め込み
1. 生成画像を保存
2. 画像にワークフロー自動埋め込み
3. 画像を共有
4. 受け取った側はドラッグ&ドロップで復元
```

**インポート手順**
```
方法1: JSONインポート
1. Load Workflowボタンクリック
2. JSONファイル選択
3. 不足ノードをManagerでインストール

方法2: 画像ドラッグ
1. ComfyUI UIに画像ドラッグ
2. 自動的にワークフロー復元
3. "Install Missing Nodes"で依存解決
```

## 4.9 本章のまとめ

本章で学んだ内容：

**ワークフロー基礎**
- ノードベースの視覚的プログラミング
- 基本構成（Text-to-Image, Image-to-Image）
- 保存と共有方法

**カスタムノード**
- ComfyUI Manager（必須）
- 2025年版必須ノード5選
- 効率化ノードの活用

**高度な構築**
- バッチ生成ワークフロー
- 条件分岐と自動化
- 大規模画像生成

**MS-S1 Max最適化**
- 128GBメモリの活用
- 並列モデルロード
- パフォーマンスチューニング

**実践とデバッグ**
- プロダクション品質ワークフロー
- 一般的エラーと解決
- コミュニティリソース

次章では、ControlNetを使用した詳細な制御について学びます。

---

**参考リソース**
- ComfyUI Manager: https://github.com/ltdrdata/ComfyUI-Manager
- OpenArt Workflows: https://openart.ai/workflows
- ComfyUI Wiki: https://comfyui-wiki.com/
- Awesome ComfyUI: https://github.com/ComfyUI-Workflow/awesome-comfyui

