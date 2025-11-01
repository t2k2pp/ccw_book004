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
