# 第6章: LoRAモデルの使用と作成

## 6.1 LoRAの基礎

### 6.1.1 LoRAとは

LoRA (Low-Rank Adaptation) は、大規模モデルを効率的にファインチューニングする技術です。

**従来のファインチューニングとの違い**
```
フルファインチューニング:
- 全パラメータを学習
- モデルサイズ: 6.6GB (SDXL)
- 学習時間: 長い
- VRAM要件: 24GB以上
- 保存容量: 6.6GB per model

LoRA:
- 少数のパラメータのみ学習
- LoRAサイズ: 50-200MB
- 学習時間: 短い
- VRAM要件: 12-16GB
- 保存容量: 50-200MB per model
- 複数LoRAの組み合わせ可能
```

**LoRAの仕組み**
```python
# LoRAの数学的概念
W' = W + ∆W
∆W = B × A

# W: 元のモデルパラメータ
# B: Low-rank行列B (rank × out_features)
# A: Low-rank行列A (in_features × rank)
# rank: 通常8, 16, 32, 64, 128
# ∆W: 学習する変化分

# メモリ効率
Full fine-tune: in_features × out_features
LoRA: (in_features + out_features) × rank
# Rank 32の場合、約100分の1のパラメータ
```

### 6.1.2 LoRAの種類

**用途別分類**
```
Character LoRA:
- 特定キャラクターの外見
- アニメ、ゲームキャラ
- 一貫性のあるキャラ生成

Style LoRA:
- 特定のアートスタイル
- アーティスト模倣
- 画風の適用

Concept LoRA:
- 特定のコンセプト
- ポーズ、構図
- 特殊効果

Object LoRA:
- 特定のオブジェクト
- 商品、アイテム
- 建築様式
```

**パラメータサイズによる分類**
```
Rank 8-16:
- サイズ: 20-50MB
- 学習速度: 最速
- 表現力: 基本的
- 用途: シンプルなスタイル

Rank 32-64:
- サイズ: 50-150MB
- 学習速度: 標準
- 表現力: 良好（推奨）
- 用途: 一般的な用途

Rank 128+:
- サイズ: 150-300MB
- 学習速度: やや遅い
- 表現力: 最高
- 用途: 複雑なスタイル、詳細制御
```

## 6.2 LoRAの使用方法

### 6.2.1 基本的なワークフロー

**単一LoRA適用**
```
[Load Checkpoint: SDXL Base]
         ↓ MODEL, CLIP
         ↓
[Load LoRA]
  lora_name: "anime_style_v2.safetensors"
  strength_model: 0.8    # モデルへの影響
  strength_clip: 1.0     # プロンプト理解への影響
         ↓ MODEL, CLIP
         ↓
[CLIP Text Encode]
  prompt: "anime character, magical girl"
         ↓
[KSampler] → [VAE Decode] → [Save Image]
```

**strength_model と strength_clip**
```yaml
strength_model:
  範囲: 0.0 - 2.0（通常0.5-1.0）
  0.0: LoRA無効
  0.5: 控えめな適用
  0.8: バランス（推奨）
  1.0: 完全適用
  1.5: 強い適用（オーバーフィット注意）

strength_clip:
  範囲: 0.0 - 1.0
  0.0: プロンプト理解に影響なし
  0.5: 部分的な影響
  1.0: 完全な影響（推奨）
```

**MS-S1 Max性能**
```
LoRA読み込み時間:
- 50MB LoRA: 0.3秒
- 150MB LoRA: 0.8秒

生成時間（SDXL + LoRA）:
- 1024x1024: 11-12秒
- LoRAによるオーバーヘッド: 約+5%

メモリ使用:
- SDXL Base: 8GB
- LoRA: 0.5-1GB
- 合計: 8.5-9GB VRAM
```

### 6.2.2 複数LoRAの組み合わせ

**マルチLoRAワークフロー**
```
[Load Checkpoint: SDXL Base]
         ↓
[Load LoRA 1]
  lora: "anime_style.safetensors"
  strength_model: 0.7
  strength_clip: 1.0
         ↓
[Load LoRA 2]
  lora: "character_a.safetensors"
  strength_model: 0.8
  strength_clip: 1.0
         ↓
[Load LoRA 3]
  lora: "lighting_style.safetensors"
  strength_model: 0.5
  strength_clip: 0.5
         ↓
[KSampler]

結果:
- アニメスタイル
- 特定キャラクター
- 特殊ライティング
- 3つの効果が統合
```

**組み合わせ戦略**
```yaml
# バランス型（推奨）
lora_1:
  type: style
  strength: 0.7

lora_2:
  type: character
  strength: 0.8

lora_3:
  type: concept
  strength: 0.5

# 強調型
main_lora:
  type: style
  strength: 1.0

support_lora_1:
  type: concept
  strength: 0.3

support_lora_2:
  type: lighting
  strength: 0.2
```

**注意点**
```
過度な組み合わせを避ける:
✅ 2-3個のLoRA: 効果的
⚠️ 4-5個のLoRA: 競合の可能性
❌ 6個以上: 品質低下リスク

strength合計ガイドライン:
- 全LoRAのstrength合計: 2.5以下推奨
- 主要LoRA: 0.7-1.0
- 補助LoRA: 0.3-0.5
```

## 6.3 LoRAの入手とインストール

### 6.3.1 主要な配布サイト

**CivitAI**
```
URL: https://civitai.com

特徴:
✅ 最大のLoRAコミュニティ
✅ 豊富なプレビュー画像
✅ ユーザーレビュー
✅ プロンプト例付き
✅ バージョン管理

カテゴリ:
- Character (キャラクター)
- Style (スタイル)
- Concept (コンセプト)
- Poses (ポーズ)
- Objects (オブジェクト)

ダウンロード:
1. モデルページを開く
2. SDXL対応を確認
3. ダウンロードボタンクリック
4. .safetensorsファイル取得
```

**Hugging Face**
```
URL: https://huggingface.co/models

特徴:
✅ 公式/研究向けLoRA
✅ API経由ダウンロード
✅ git cloneで管理可能

検索:
filter: "lora" AND "sdxl"

ダウンロード:
# CLIツール使用
huggingface-cli download \
    username/lora-name \
    lora_model.safetensors \
    --local-dir ~/ai-tools/ComfyUI/models/loras
```

### 6.3.2 LoRAの配置

**ディレクトリ構造**
```bash
~/ai-tools/ComfyUI/models/loras/
├── characters/
│   ├── character_a_v2.safetensors
│   ├── character_b_sdxl.safetensors
│   └── game_character_pack.safetensors
├── styles/
│   ├── anime_style_v3.safetensors
│   ├── watercolor_sdxl.safetensors
│   └── pixel_art_lora.safetensors
├── concepts/
│   ├── dynamic_poses.safetensors
│   ├── cinematic_lighting.safetensors
│   └── fantasy_effects.safetensors
└── test/
    └── experimental_loras/
```

**権限設定**
```bash
# LoRAディレクトリのパーミッション確認
ls -la ~/ai-tools/ComfyUI/models/loras/

# 必要に応じて設定
chmod 755 ~/ai-tools/ComfyUI/models/loras/
chmod 644 ~/ai-tools/ComfyUI/models/loras/*.safetensors
```

## 6.4 LoRAの作成と学習

### 6.4.1 学習の準備

**データセット準備**
```
画像要件:
- 枚数: 20-100枚（理想は50枚）
- 解像度: 1024x1024推奨
- フォーマット: JPG, PNG
- 品質: 高品質、一貫性
- 多様性: 様々な角度、ポーズ

推奨枚数（用途別）:
キャラクター: 40-60枚
スタイル: 30-50枚
オブジェクト: 20-40枚
コンセプト: 50-100枚
```

**キャプション作成**
```bash
# 自動キャプション生成

# 方法1: BLIP2使用
cd ~/ai-tools
git clone https://github.com/pharmapsychotic/clip-interrogator.git
cd clip-interrogator
pip install -r requirements.txt

python caption_images.py \
    --input_dir ./training_images \
    --output_dir ./captions

# 方法2: 手動キャプション
# 各画像に対応する.txtファイル作成
image_001.jpg → image_001.txt
内容: "anime character, blue hair, red eyes,  school uniform, smiling"

# ディレクトリ構造
training_data/
├── image_001.jpg
├── image_001.txt
├── image_002.jpg
├── image_002.txt
└── ...
```

### 6.4.2 Lora-Training-in-Comfy使用

**カスタムノードのインストール**
```bash
cd ~/ai-tools/ComfyUI/custom_nodes
git clone https://github.com/LarryJane491/Lora-Training-in-Comfy.git
cd Lora-Training-in-Comfy
pip install -r requirements.txt

# ComfyUI再起動
cd ~/ai-tools/ComfyUI
python main.py
```

**学習ワークフロー構築**
```
[Init SDXL LoRA Training]
  model_path: "sd_xl_base_1.0.safetensors"
  dataset_path: "./training_data"
  output_name: "my_lora_v1"
  rank: 32                # LoRAのRank
  alpha: 32               # 学習率調整
  batch_size: 2           # MS-S1 Max推奨
  max_train_steps: 1000   # 学習ステップ数
  learning_rate: 1e-4     # 学習率
  save_every_n_steps: 250 # 保存間隔
         ↓
[Train LoRA]
         ↓
[Save LoRA Model]
  output_dir: "~/ai-tools/ComfyUI/models/loras/"
```

**MS-S1 Max最適設定**
```yaml
# 学習設定（MS-S1 Max向け）

ハードウェア活用:
  batch_size: 4          # 128GBメモリ活用
  gradient_accumulation: 2
  mixed_precision: "fp16"
  use_8bit_adam: True

学習パラメータ:
  rank: 32               # バランス良好
  alpha: 32              # rank と同じ推奨
  learning_rate: 1e-4    # 標準
  max_train_steps: 1000-2000

データ拡張:
  random_crop: True
  random_flip: True
  color_jitter: 0.1

学習時間（MS-S1 Max）:
- 50画像, 1000 steps: 約20-30分
- 100画像, 2000 steps: 約50-70分
- VRAM使用: 12-16GB
```

### 6.4.3 Kohya ss-sdxl-lora-trainer

**より高度な学習環境**
```bash
# Kohya Trainerのインストール
cd ~/ai-tools
git clone https://github.com/kohya-ss/sd-scripts.git
cd sd-scripts

# 依存関係インストール
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# PyTorch ROCm版確認
pip install torch torchvision torchaudio \
    --index-url https://download.pytorch.org/whl/rocm6.4
```

**学習スクリプト**
```bash
#!/bin/bash
# train_lora.sh

# ROCm環境変数
export HSA_OVERRIDE_GFX_VERSION=11.0.0
export PYTORCH_ROCM_ARCH=gfx1100

# 学習実行
accelerate launch --num_cpu_threads_per_process=8 \
    sdxl_train_network.py \
    --pretrained_model_name_or_path="sd_xl_base_1.0.safetensors" \
    --train_data_dir="./training_data" \
    --output_dir="./output_loras" \
    --output_name="my_character_lora" \
    --network_module="networks.lora" \
    --network_dim=32 \
    --network_alpha=32 \
    --learning_rate=1e-4 \
    --max_train_steps=2000 \
    --train_batch_size=4 \
    --gradient_accumulation_steps=2 \
    --mixed_precision="fp16" \
    --save_every_n_steps=250 \
    --save_model_as="safetensors" \
    --clip_skip=2 \
    --seed=42 \
    --enable_bucket \
    --bucket_reso_steps=64 \
    --bucket_no_upscale \
    --min_bucket_reso=512 \
    --max_bucket_reso=2048

# MS-S1 Max: 約30分で完了（50画像, 2000 steps）
```

**高度なパラメータ**
```yaml
データ処理:
  enable_bucket: True       # 解像度バケット
  bucket_reso_steps: 64     # 解像度ステップ
  min_bucket_reso: 512
  max_bucket_reso: 2048
  random_crop: True

最適化:
  optimizer_type: "AdamW8bit"  # メモリ効率的
  lr_scheduler: "cosine"       # 学習率スケジュール
  lr_warmup_steps: 100
  gradient_checkpointing: True

正則化:
  noise_offset: 0.05       # ノイズオフセット
  adaptive_noise_scale: 0.00357
  clip_skip: 2             # CLIP層スキップ
```

## 6.5 LoRAの評価とテスト

### 6.5.1 品質評価

**チェックポイントテスト**
```python
# 学習中の各チェックポイントをテスト

checkpoints = [
    "my_lora_step_250.safetensors",
    "my_lora_step_500.safetensors",
    "my_lora_step_750.safetensors",
    "my_lora_step_1000.safetensors"
]

test_prompts = [
    "anime character, standing pose, white background",
    "anime character, action pose, outdoor scene",
    "anime character, close-up portrait, detailed face"
]

for checkpoint in checkpoints:
    for prompt in test_prompts:
        # ワークフロー実行
        generate_image(
            lora=checkpoint,
            prompt=prompt,
            strength=0.8,
            seed=42  # 一貫性のため固定
        )

# 結果を比較して最適なチェックポイント選択
```

**過学習検出**
```
症状:
- 学習画像に酷似しすぎる
- 新しいプロンプトで破綻
- 多様性の欠如

確認方法:
1. 学習に使っていないプロンプトでテスト
2. 様々なstrength値で生成
3. 他のLoRAと組み合わせテスト

対策:
- max_train_stepsを減らす
- learning_rateを下げる
- データセットを増やす
- 正則化を強化
```

### 6.5.2 LoRAのマージ

**複数LoRAの統合**
```
用途:
- 複数LoRAの効果を1つに統合
- ストレージ節約
- 推論時の効率化

方法:
[Load LoRA 1]
  strength: 0.7
     ↓
[Load LoRA 2]
  strength: 0.5
     ↓
[Merge LoRAs]
     ↓
[Save Merged LoRA]
  output: "merged_lora.safetensors"

注意:
- 互換性のあるLoRAのみ
- 効果の予測が難しい
- テストが必須
```

## 6.6 実践的なLoRA活用

### 6.6.1 キャラクター一貫性維持

**シリーズ作品作成**
```yaml
目的: 同じキャラクターで複数シーン作成

ワークフロー:
Step 1: Character LoRA適用
  lora: "my_character.safetensors"
  strength: 0.9

Step 2: 各シーンのプロンプト
  scene_1: "{character trigger}, in forest, daytime"
  scene_2: "{character trigger}, in city, night"
  scene_3: "{character trigger}, at beach, sunset"

Step 3: 固定パラメータ
  seed: 毎回変更（バリエーション）
  steps: 30
  cfg: 7.5
  sampler: dpmpp_2m_karras

結果:
- 一貫したキャラクター外見
- 多様な背景とポーズ
- シリーズ作品完成

MS-S1 Max:
- 10シーン生成: 約2分
- バッチサイズ4: さらに効率化
```

### 6.6.2 スタイルミキシング

**複数スタイルの融合**
```python
# アニメ + 水彩 + ファンタジー

workflow = {
    "base_model": "sd_xl_base_1.0.safetensors",
    "loras": [
        {
            "name": "anime_style.safetensors",
            "strength_model": 0.6,
            "strength_clip": 1.0
        },
        {
            "name": "watercolor_painting.safetensors",
            "strength_model": 0.5,
            "strength_clip": 0.8
        },
        {
            "name": "fantasy_concept.safetensors",
            "strength_model": 0.4,
            "strength_clip": 0.6
        }
    ],
    "prompt": "magical forest, glowing mushrooms, fairy lights",
    "negative": "low quality, blurry",
    "steps": 30,
    "cfg": 7.5
}

# 結果: アニメ風水彩ファンタジーアート
```

### 6.6.3 商用利用の注意点

**ライセンス確認**
```
確認事項:
□ LoRAのライセンス（CivitAIで確認）
□ 元モデルのライセンス
□ 学習データの権利
□ 商用利用の可否
□ クレジット表記の要否

推奨:
- 公式LoRAを優先
- ライセンスを記録
- 不明な場合は使用を避ける
```

## 6.7 トラブルシューティング

### 6.7.1 学習時の問題

**問題1: メモリ不足**
```
症状:
- "HIP out of memory"
- 学習が途中で停止

解決策（MS-S1 Max）:
1. batch_sizeを削減
   4 → 2 → 1

2. gradient_accumulationで補完
   batch_size: 2
   gradient_accumulation: 2
   # 実質batch_size 4相当

3. mixed_precisionを使用
   mixed_precision: "fp16"

4. gradient_checkpointing有効化
   gradient_checkpointing: True
```

**問題2: 学習が進まない**
```
症状:
- loss値が下がらない
- 生成結果が改善しない

原因と解決:
1. learning_rateが不適切
   解決: 1e-4 から 5e-5 に調整

2. データセットの質が低い
   解決: 高品質画像に差し替え

3. キャプションが不適切
   解決: 詳細なキャプション作成

4. steps不足
   解決: 2000 steps以上に増やす
```

### 6.7.2 使用時の問題

**問題1: LoRAが効かない**
```
症状:
- LoRA適用しても変化なし

原因と解決:
1. strengthが低すぎる
   解決: 0.8-1.0に上げる

2. プロンプトにトリガーワード不足
   解決: LoRAのトリガーワード確認・追加

3. CFGが不適切
   解決: CFG 7-8に調整

4. モデル互換性
   解決: SDXL用LoRAか確認
```

**問題2: 品質が悪い**
```
症状:
- アーティファクト発生
- 崩壊した画像

原因と解決:
1. strengthが高すぎる
   解決: 0.5-0.7に下げる

2. 複数LoRAの競合
   解決: LoRA数を減らす

3. 過学習したLoRA
   解決: 異なるチェックポイント試行
```

## 6.8 MS-S1 Max最適化戦略

### 6.8.1 効率的な学習

**並列学習**
```bash
# 128GBメモリを活用した並列学習

# ターミナル1: Character LoRA学習
cd ~/ai-tools/sd-scripts
source venv/bin/activate
./train_character.sh

# ターミナル2: Style LoRA学習（並行）
cd ~/ai-tools/sd-scripts
source venv/bin/activate
./train_style.sh

# リソース配分:
# 各学習: 16GB VRAM, 20GB RAM
# 合計: 32GB VRAM, 40GB RAM
# 残り: 88GB（余裕）
```

### 6.8.2 LoRAライブラリ管理

**メタデータ管理**
```bash
# lora_metadata.json
{
    "loras": [
        {
            "name": "anime_style_v3.safetensors",
            "type": "style",
            "rank": 32,
            "trained_on": "SDXL Base 1.0",
            "trigger_words": ["anime style", "cel shaded"],
            "recommended_strength": 0.7,
            "tags": ["anime", "style", "general"],
            "created": "2025-01-15",
            "source": "civitai",
            "license": "CreativeML Open RAIL-M"
        },
        {
            "name": "character_miku.safetensors",
            "type": "character",
            "rank": 64,
            "trained_on": "SDXL Base 1.0",
            "trigger_words": ["miku", "twin tails", "blue hair"],
            "recommended_strength": 0.9,
            "tags": ["character", "vocaloid"],
            "created": "2025-01-20",
            "source": "custom_trained",
            "license": "personal_use_only"
        }
    ]
}
```

## 6.9 本章のまとめ

本章で学んだ内容：

**LoRA基礎**
- LoRAの仕組みと利点
- 種類とパラメータサイズ
- フルファインチューニングとの違い

**LoRA使用**
- 基本的なワークフロー
- 複数LoRAの組み合わせ
- strengthパラメータの調整

**LoRA作成**
- データセット準備
- Lora-Training-in-Comfy使用
- Kohya ss-scriptsでの学習

**MS-S1 Max最適化**
- 128GBメモリ活用
- 並列学習戦略
- 効率的なワークフロー

**実践と管理**
- キャラクター一貫性維持
- スタイルミキシング
- ライブラリ管理

次章では、AMDGPUとROCmの最適化について詳しく学びます。

---

**参考リソース**
- LoRA論文: https://arxiv.org/abs/2106.09685
- Kohya ss-scripts: https://github.com/kohya-ss/sd-scripts
- CivitAI: https://civitai.com/
- Lora-Training-in-Comfy: https://github.com/LarryJane491/Lora-Training-in-Comfy

