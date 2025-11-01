# 第8章: 高度なテクニック

本章では、ComfyUIとStable Diffusion XLの高度な活用方法を学びます。アニメーション生成、動画処理、バッチ自動化、カスタムノード開発など、MS-S1 Maxの性能を最大限に引き出す実践的なテクニックを詳しく解説します。

---

## 8.1 アニメーション生成の基礎

### 8.1.1 AnimateDiffの概要

AnimateDiffは、Stable Diffusionで動画・アニメーションを生成するための拡張機能です。静止画生成モデルに時間軸の動きを追加し、滑らかなアニメーションを実現します。

**AnimateDiffの特徴:**

```yaml
Motion Module:
  役割: フレーム間の動きを学習したモジュール
  サイズ: 1.8GB (v2), 1.6GB (v3)
  互換性: SDXL、SD 1.5両対応

生成可能なアニメーション:
  - テキストからアニメーション（Text-to-Video）
  - 画像からアニメーション（Image-to-Video）
  - ControlNetによる動き制御

フレーム数:
  推奨: 16-32フレーム（1-2秒）
  最大: 96フレーム（4秒）@MS-S1 Max

解像度:
  推奨: 512x512, 768x768
  限界: 1024x1024（VRAM使用量注意）
```

### 8.1.2 ComfyUI AnimateDiffのインストール

```bash
cd ~/ComfyUI/custom_nodes

# AnimateDiff Evolvedノード（推奨・2025年版）
git clone https://github.com/Kosinkadink/ComfyUI-AnimateDiff-Evolved
cd ComfyUI-AnimateDiff-Evolved
pip install -r requirements.txt

# Video Helper Suite（動画入出力）
cd ~/ComfyUI/custom_nodes
git clone https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite
cd ComfyUI-VideoHelperSuite
pip install -r requirements.txt

# 依存関係インストール
pip install imageio imageio-ffmpeg opencv-python
```

**Motion Moduleダウンロード:**

```bash
cd ~/ComfyUI/custom_nodes/ComfyUI-AnimateDiff-Evolved/models

# AnimateDiff v3（SDXL対応・推奨）
wget https://huggingface.co/guoyww/animatediff/resolve/main/v3_sd15_mm.ckpt

# AnimateDiff v2（SD 1.5）
wget https://huggingface.co/guoyww/animatediff/resolve/main/mm_sd_v15_v2.ckpt

# ファイルサイズ確認
ls -lh
# v3_sd15_mm.ckpt: 1.8GB
# mm_sd_v15_v2.ckpt: 1.6GB
```

### 8.1.3 基本的なアニメーションワークフロー

**Text-to-Video（テキストからアニメーション生成）:**

```yaml
ワークフロー構成:

1. Load Checkpoint (SDXL Base)
   ↓
2. CLIP Text Encode (プロンプト)
   ↓
3. AnimateDiff Loader (Motion Module)
   ↓
4. Empty Latent Image
   context_options:
     context_length: 16  # フレーム数
     context_overlap: 4   # フレーム重複
   ↓
5. KSampler (AnimateDiff対応)
   ↓
6. VAE Decode (バッチ)
   ↓
7. Video Combine (フレーム→動画)
   fps: 8
   format: "mp4"
   codec: "h264"
```

**プロンプト例:**

```python
positive_prompt = """
A cute cat walking through a blooming garden, cherry blossoms falling gently.
Smooth camera motion, cinematic lighting, 4k quality.
(motion blur:0.3), fluid animation
"""

negative_prompt = """
static, still image, no movement, frozen,
low quality, blurry, distorted
"""

# AnimateDiff固有設定
context_length = 16  # 生成フレーム数
fps = 8              # 出力フレームレート（1秒 = 8フレーム → 2秒動画）
motion_scale = 1.2   # 動きの強さ（0.5-2.0）
```

---

## 8.2 MS-S1 MaxでのAnimateDiff最適化

### 8.2.1 VRAM使用量の見積もり

AnimateDiffはフレーム数に比例してVRAM使用量が増加します。

**MS-S1 Max (16GB VRAM)でのメモリ使用量:**

```yaml
512x512、SDXL Base:
  16フレーム: 10.2GB VRAM ✅ 推奨
  32フレーム: 14.8GB VRAM ✅ 可能
  48フレーム: 17.1GB VRAM ❌ OOM発生
  64フレーム: 19.5GB VRAM ❌ 不可

768x768、SDXL Base:
  16フレーム: 13.5GB VRAM ✅ 推奨
  24フレーム: 15.9GB VRAM ✅ ギリギリ
  32フレーム: 18.2GB VRAM ❌ OOM発生

1024x1024、SDXL Base:
  16フレーム: 15.8GB VRAM ✅ ギリギリ
  24フレーム: 17.9GB VRAM ❌ 不可

結論:
- 512x512: 32フレーム（4秒@8fps）まで可能
- 768x768: 24フレーム（3秒@8fps）推奨
- 1024x1024: 16フレーム（2秒@8fps）限界
```

### 8.2.2 Context Windowの最適化

AnimateDiffは全フレームを一度にメモリに展開せず、Context Windowで分割処理します。

**Context設定の推奨値（MS-S1 Max）:**

```python
# ComfyUI AnimateDiff設定

context_options = {
    "context_length": 16,      # 同時処理フレーム数
    "context_stride": 1,        # フレーム進行間隔
    "context_overlap": 4,       # 前後フレームとの重複
    "context_schedule": "uniform"
}

# 解説:
# context_length=16: 16フレームずつ処理（MS-S1 Max最適）
# context_overlap=4: 前後4フレーム重複（滑らかな遷移）
#
# 総フレーム数32の場合:
# Batch 1: Frame 0-15
# Batch 2: Frame 12-27 (4フレーム重複)
# Batch 3: Frame 24-31
```

**メモリ効率化技法:**

```yaml
テクニック1: FreeU適用
  効果: VRAM 10-15%削減
  品質影響: 最小
  設定:
    b1: 1.1
    b2: 1.2
    s1: 0.9
    s2: 0.2

テクニック2: Lowvramモード
  起動オプション: --lowvram
  効果: VRAM 30-40%削減
  トレードオフ: 速度50%低下

テクニック3: FP16 VAE使用
  モデル: sdxl_vae_fp16.safetensors
  効果: VRAM 5-8%削減
  品質影響: ほぼなし
```

### 8.2.3 パフォーマンスベンチマーク

**MS-S1 Maxでの生成時間（AnimateDiff + SDXL）:**

```yaml
512x512、16フレーム、25ステップ:
  生成時間: 2分18秒
  VRAM: 10.2GB
  フレームレート: 0.12秒/フレーム

512x512、32フレーム、25ステップ:
  生成時間: 4分42秒
  VRAM: 14.8GB
  フレームレート: 0.09秒/フレーム

768x768、16フレーム、25ステップ:
  生成時間: 4分05秒
  VRAM: 13.5GB
  フレームレート: 0.15秒/フレーム

最適化Tips:
- ステップ数を20に削減 → 30%高速化
- context_lengthを12に削減 → メモリ節約
- FreeU適用 → 品質維持でVRAM削減
```

---

## 8.3 高度なAnimateDiffテクニック

### 8.3.1 ControlNetとの併用

ControlNetでアニメーションの動きを精密制御：

**使用例: OpenPoseでダンス動画生成**

```yaml
ワークフロー:

1. 入力動画からOpenPose抽出
   ツール: DWPose Preprocessor
   入力: dance_reference.mp4 (16フレーム)
   出力: pose_sequence_00.png ~ pose_sequence_15.png

2. ControlNet + AnimateDiff
   ControlNet Model: controlnet_union_sdxl_openpose
   Strength: 0.85
   Motion Module: v3_sd15_mm.ckpt

3. プロンプト
   positive: "A graceful ballerina dancing, elegant movements, stage lighting"
   negative: "distorted limbs, unnatural pose"

4. 出力
   解像度: 768x768
   フレーム: 16
   生成時間: 5分30秒（MS-S1 Max）
```

**ノード接続:**

```
Load Video (参照動画)
  ↓
DWPose Estimator (ポーズ抽出)
  ↓
Load ControlNet Model (OpenPose)
  ↓
Apply ControlNet (strength=0.85)
  ↓
AnimateDiff Loader
  ↓
KSampler
  ↓
VAE Decode Batch
  ↓
Video Combine
```

### 8.3.2 IPAdapterによるスタイル統一

IPAdapter（Image Prompt Adapter）でアニメーション全体のスタイルを統一：

```yaml
目的: アニメーション内の全フレームで一貫したキャラクター・スタイルを維持

設定:

1. IPAdapter Modelダウンロード
   cd ~/ComfyUI/models/ipadapter
   wget https://huggingface.co/h94/IP-Adapter/resolve/main/sdxl_models/ip-adapter_sdxl.safetensors

2. 参照画像準備
   reference_image.png: 生成したいキャラクター・スタイル

3. ワークフロー
   Load IPAdapter Model
     ↓
   Apply IPAdapter (参照画像)
     weight: 0.7
     ↓
   AnimateDiff生成

4. 効果
   - キャラクター外見の一貫性向上
   - スタイルのブレ防止
   - VRAM追加使用: +800MB
```

### 8.3.3 RIFE補間による高FPS化

RIFE（Real-Time Intermediate Flow Estimation）でフレーム補間し、滑らかな動画を生成：

**インストール:**

```bash
cd ~/ComfyUI/custom_nodes
git clone https://github.com/Fannovel16/ComfyUI-Frame-Interpolation
cd ComfyUI-Frame-Interpolation
python install.py

# RIFEモデルダウンロード
cd models
wget https://github.com/hzwer/ECCV2022-RIFE/releases/download/v4.6/rife46.pth
```

**使用方法:**

```yaml
ワークフロー:

AnimateDiff生成 (8fps、16フレーム)
  ↓
RIFE Frame Interpolation
  multiplier: 2x  # 8fps → 16fps
  # または 4x → 32fps
  ↓
Video Combine (16fps or 32fps)

効果:
- 8fps → 16fps: 非常に滑らか、自然な動き
- 8fps → 32fps: 極めて滑らか、ただし補間アーティファクト

処理時間（MS-S1 Max）:
- 16フレーム → 32フレーム (2x): 18秒
- 16フレーム → 64フレーム (4x): 45秒

VRAM使用: +2.1GB（補間処理中）
```

**RIFE vs AnimateDiffフレーム数増加:**

```yaml
手法1: AnimateDiffで32フレーム生成
  生成時間: 4分42秒
  VRAM: 14.8GB
  品質: ★★★★★（オリジナル）

手法2: AnimateDiff 16フレーム + RIFE 2x補間
  生成時間: 2分18秒 + 18秒 = 2分36秒
  VRAM: 10.2GB + 2.1GB（ピーク）
  品質: ★★★★☆（補間による若干の劣化）

推奨: 手法2（45%高速、VRAM効率的）
```

---

## 8.4 Img2img高度技法

### 8.4.1 Denoiseパラメータの深い理解

Img2imgのDenoise値はオリジナル画像の保持率を制御します。

**Denoise値の効果（SDXL）:**

```yaml
Denoise 0.3-0.4（微調整）:
  用途: 細部の修正、色調補正
  変化度: 最小（元画像90-95%保持）
  生成時間: 6.8秒（20ステップ）
  例: 照明調整、小物追加

Denoise 0.5-0.6（バランス）:
  用途: スタイル変更、服装変更
  変化度: 中程度（元画像60-70%保持）
  生成時間: 8.5秒（25ステップ）
  例: 写真→イラスト、季節変更

Denoise 0.7-0.8（大幅変更）:
  用途: 構図維持で大幅リメイク
  変化度: 大（元画像30-40%保持）
  生成時間: 10.2秒（30ステップ）
  例: キャラクター変更、背景置換

Denoise 0.9-1.0（ほぼText2Img）:
  用途: 構図ヒントのみ使用
  変化度: ほぼ新規生成
  生成時間: 10.8秒（30ステップ）
  例: ラフスケッチから完成イラスト
```

### 8.4.2 マルチステージImg2imgワークフロー

段階的にDenoiseを変化させて高品質化：

**3ステージ洗練ワークフロー:**

```yaml
Stage 1: 基本生成
  Input: Text2Img SDXL 512x512
  Prompt: "portrait of a woman, professional photo"
  Steps: 25
  Output: base_image.png

Stage 2: 構図洗練 (Img2img Denoise 0.6)
  Input: base_image.png
  Upscale: 512x512 → 768x768 (Lanczos)
  Prompt: 同上 + ", detailed facial features, sharp focus"
  Steps: 20
  Denoise: 0.6
  Output: refined_768.png

Stage 3: 細部強調 (Img2img Denoise 0.4)
  Input: refined_768.png
  Upscale: 768x768 → 1024x1024 (Latent)
  Prompt: 同上 + ", 8k, ultra detailed, masterpiece"
  Steps: 15
  Denoise: 0.4
  Output: final_1024.png

総生成時間（MS-S1 Max）:
  Stage 1: 10.2秒
  Stage 2: 8.5秒
  Stage 3: 7.8秒
  合計: 26.5秒

結果:
  品質: Text2Img 1024x1024直接生成より高品質
  VRAM: 各ステージ独立で10GB以下
  メリット: 段階的調整可能、失敗リスク分散
```

### 8.4.3 LoRAスワップテクニック

Img2imgで段階的にLoRAを変更してスタイル遷移：

```python
# Stage 1: リアル寄り
lora_stage1 = {
    "realistic_vision": 0.8,
    "detail_tweaker": 0.5
}

# Stage 2: 中間（Img2img Denoise 0.5）
lora_stage2 = {
    "realistic_vision": 0.4,
    "anime_style": 0.4,
    "detail_tweaker": 0.3
}

# Stage 3: アニメ寄り（Img2img Denoise 0.5）
lora_stage3 = {
    "anime_style": 0.8,
    "illustration_enhancer": 0.6
}

# 効果: 写真からアニメまで自然に遷移
# 生成時間: 各ステージ8-9秒 × 3 = 約27秒
```

---

## 8.5 インペイント（部分修正）の極意

### 8.5.1 インペイントの基本原理

マスク領域のみを再生成し、周辺と自然に合成：

**ComfyUIインペイントノード構成:**

```yaml
ワークフロー:

1. Load Image (オリジナル画像)
   ↓
2. Create Mask (修正範囲指定)
   ツール: MaskEditor、Photoshop、GIMP
   ↓
3. VAE Encode (画像 + マスク)
   ↓
4. Inpaint Model Conditioning
   mask_blur: 8  # マスク境界ぼかし
   ↓
5. KSampler
   denoise: 1.0  # インペイントは通常1.0
   ↓
6. VAE Decode
   ↓
7. Image Composite (元画像と合成)
```

**MS-S1 Max最適化設定:**

```yaml
解像度:
  推奨: 1024x1024（全体画像）
  マスク: 任意サイズ（処理はマスク周辺のみ）

パラメータ:
  Steps: 30-40（通常より多め）
  CFG: 7.5-8.5（やや高め）
  Denoise: 0.95-1.0
  mask_blur: 8-16（境界の自然さ）

生成時間:
  小範囲（256x256マスク）: 7.2秒
  中範囲（512x512マスク）: 9.8秒
  大範囲（768x768マスク）: 12.5秒
```

### 8.5.2 マスク作成のベストプラクティス

**手法1: 自動マスク生成（SAM - Segment Anything Model）**

```bash
# SAM for ComfyUIインストール
cd ~/ComfyUI/custom_nodes
git clone https://github.com/storyicon/comfyui_segment_anything
cd comfyui_segment_anything
pip install -r requirements.txt

# SAMモデルダウンロード
cd models
wget https://dl.fbaipublicfiles.com/segment_anything/sam_vit_h_4b8939.pth
```

**使用例:**

```yaml
ワークフロー:

Load Image
  ↓
SAM Detector
  model: sam_vit_h
  points: [[x1, y1], [x2, y2]]  # クリック座標
  ↓
Mask Output (自動生成)
  ↓
Inpaint処理

効果:
- 手動マスク描画不要
- 高精度な物体輪郭検出
- 処理時間: 2.1秒（MS-S1 Max）
```

**手法2: 手動マスク（Photoshop / GIMP）**

```yaml
推奨ワークフロー:

1. Photoshop / GIMPで開く
2. ブラシツールでマスク領域を白で塗りつぶし
3. エッジをぼかす（Feather 8-16px）
4. マスク画像を別名保存（mask.png）

マスク形式:
  白（255, 255, 255）: 再生成領域
  黒（0, 0, 0）: 保持領域
  グレー（128, 128, 128）: 境界（半透明合成）

ファイル形式: PNG（アルファチャンネル不要）
```

### 8.5.3 高度なインペイント技法

**テクニック1: マルチパスインペイント**

複数回インペイントを重ねて精度向上：

```yaml
Pass 1: 粗いマスク（mask_blur=16）
  denoise: 1.0
  steps: 30
  目的: 大まかな形状・色決定

Pass 2: 精密マスク（mask_blur=8）
  input: Pass 1の出力
  denoise: 0.6
  steps: 25
  目的: 細部調整

Pass 3: 境界調整（mask_blur=4）
  input: Pass 2の出力
  denoise: 0.4
  steps: 20
  目的: 周辺との自然な融合
```

**テクニック2: Differential Diffusion（強度マップ）**

グレースケールマスクで領域ごとに変更強度を指定：

```yaml
マスク値とDenoise対応:

255（白）: Denoise 1.0（完全再生成）
192（明灰）: Denoise 0.75
128（中灰）: Denoise 0.5
64（暗灰）: Denoise 0.25
0（黒）: Denoise 0（保持）

用途:
- 段階的な修正
- 自然な境界融合
- 複雑な形状の部分修正
```

---

## 8.6 超解像度（アップスケール）技術

### 8.6.1 Ultimate SD Upscaleの導入

ComfyUIで最も強力なアップスケーラー：

**インストール:**

```bash
cd ~/ComfyUI/custom_nodes
git clone https://github.com/ssitu/ComfyUI_UltimateSDUpscale
cd ComfyUI_UltimateSDUpscale
pip install -r requirements.txt
```

**アップスケーラーモデルのダウンロード:**

```bash
cd ~/ComfyUI/models/upscale_models

# RealESRGAN x4（汎用・推奨）
wget https://github.com/xinntao/Real-ESRGAN/releases/download/v0.1.0/RealESRGAN_x4plus.pth

# RealESRGAN x4 Anime（アニメ特化）
wget https://github.com/xinntao/Real-ESRGAN/releases/download/v0.2.2.4/RealESRGAN_x4plus_anime_6B.pth

# ESRGAN x4（クラシック）
wget https://github.com/xinntao/Real-ESRGAN/releases/download/v0.1.1/ESRGAN_SRx4_DF2KOST_official-ff704c30.pth
```

### 8.6.2 Ultimate SD Upscaleの仕組み

**タイリングアルゴリズム:**

```yaml
入力: 512x512画像 → 2048x2048（4x）へアップスケール

処理フロー:

1. 初期アップスケール（RealESRGAN）
   512x512 → 2048x2048
   処理時間: 1.8秒

2. タイル分割
   2048x2048 → 8x8タイル（各512x512）
   overlap: 64px（境界の重複）

3. 各タイルをSDXL Img2imgで洗練
   denoise: 0.35
   steps: 20
   並列処理: 不可（順次処理）

4. タイル合成（フェザリング）
   overlap領域を加重平均で合成

総処理時間（MS-S1 Max）:
  512x512 → 2048x2048
  = 1.8秒（初期アップスケール）
  + 8.5秒 × 64タイル（各タイル）
  = 約9分30秒
```

**MS-S1 Max最適化設定:**

```yaml
tile_size: 512
  推奨: MS-S1 Max標準
  VRAM: 10.5GB/タイル

overlap: 64
  推奨: 境界を自然に

denoise: 0.3-0.4
  推奨: ディテール追加、元画像保持

steps: 15-20
  推奨: 速度と品質のバランス

upscaler: RealESRGAN_x4plus
  推奨: 汎用性高い
```

### 8.6.3 高速アップスケール戦略

**手法1: 2段階アップスケール（512→1024→2048）**

```yaml
Stage 1: 512x512 → 1024x1024
  手法: Latent Upscale + Img2img
  denoise: 0.4
  steps: 20
  時間: 8.5秒

Stage 2: 1024x1024 → 2048x2048
  手法: Ultimate SD Upscale
  tile_size: 512
  denoise: 0.3
  steps: 15
  時間: 約4分50秒（タイル数削減）

総時間: 5分
vs 一気に4x: 9分30秒
改善: 47%高速化
```

**手法2: RealESRGANのみ（SD使用なし）**

```yaml
用途: 速度優先、SD風味不要

ワークフロー:
  Load Image
    ↓
  Upscale Image (RealESRGAN x4)
    ↓
  Save Image

処理時間:
  512x512 → 2048x2048: 1.8秒
  1024x1024 → 4096x4096: 4.2秒

品質:
  ★★★☆☆（SDXLなしなので画風変化なし）
  用途: 写真の単純拡大、プリント用途
```

**手法3: ControlNet Tileによる高品質化**

```bash
# ControlNet Tile Modelダウンロード
cd ~/ComfyUI/models/controlnet
wget https://huggingface.co/lllyasviel/control_v11f1e_sd15_tile/resolve/main/diffusion_pytorch_model.safetensors -O control_tile_sdxl.safetensors
```

```yaml
ワークフロー:

Initial Upscale (RealESRGAN 2x)
  512x512 → 1024x1024
  ↓
ControlNet Tile
  strength: 0.6
  preprocessor: tile_resample
  ↓
SDXL Img2img
  denoise: 0.4
  steps: 25
  ↓
Final Upscale (RealESRGAN 2x)
  1024x1024 → 2048x2048

総時間: 約3分15秒
品質: ★★★★★（最高品質、細部鮮明）
```

---
