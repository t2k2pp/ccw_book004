# 第5章: ControlNetによる高度な制御

## 5.1 ControlNetの基礎

### 5.1.1 ControlNetとは

ControlNetは、Stable Diffusionに追加の制御信号を提供し、構図やポーズ、エッジなどを正確に制御できる技術です。

**基本原理**
```
従来のText-to-Image:
テキストプロンプト → 画像生成
↓
制御が曖昧、構図が不安定

ControlNet使用時:
テキストプロンプト + 制御画像 → 画像生成
↓
構図を正確に制御、一貫性が高い
```

**主要な制御タイプ**
```
Canny (エッジ検出):
- 輪郭線を保持
- 線画から着色
- 建築物の形状維持

Depth (深度):
- 3D構造を保持
- 奥行き情報を維持
- 空間配置の制御

OpenPose (骨格):
- 人物のポーズ制御
- キャラクター配置
- アニメーション準備

Scribble (ラフスケッチ):
- 手描きスケッチから生成
- ラフな指示で制御
- コンセプトアート作成

Lineart (線画):
- クリーンな線画から生成
- イラスト制作
- 漫画・アニメ向け

Normal (法線マップ):
- 表面の向きを制御
- 3Dモデルから生成
- リアルな陰影

Seg (セグメンテーション):
- 領域ごとの制御
- 複雑な構図
- マルチオブジェクト
```

### 5.1.2 SDXLとControlNet

**SD 1.5 vs SDXL ControlNet**
```
SD 1.5 ControlNet:
- 公式モデル豊富
- 512x512最適化
- コミュニティ活発

SDXL ControlNet:
- 公式モデルなし
- サードパーティ製
- 1024x1024対応
- Union版が推奨（2025年）
```

**ControlNet Union for SDXL**
```
特徴:
- 複数の制御タイプを1モデルに統合
- Canny, Depth, Openpose等を内包
- メモリ効率的
- MS-S1 Max推奨

対応制御:
✅ Canny
✅ Openpose
✅ Depth
✅ LineArt
✅ MLSD (直線検出)
✅ Scribble
✅ HED (ホリスティックエッジ)
✅ Normal
✅ Segmentation

モデルサイズ: 約2.5GB
```

## 5.2 ControlNetのインストール

### 5.2.1 モデルのダウンロード

**SDXL ControlNet Unionモデル**
```bash
cd ~/ai-tools/ComfyUI/models/controlnet

# ControlNet Union SDXL（推奨）
wget https://huggingface.co/xinsir/controlnet-union-sdxl-1.0/resolve/main/diffusion_pytorch_model_promax.safetensors

# または特定の制御タイプ
# Depth
wget https://huggingface.co/SargeZT/controlnet-sd-xl-1.0-depth-16bit-zoe/resolve/main/depth-zoe-xl-v1.0-controlnet.safetensors

# Canny
wget https://huggingface.co/diffusers/controlnet-canny-sdxl-1.0/resolve/main/diffusion_pytorch_model.safetensors

# OpenPose
wget https://huggingface.co/thibaud/controlnet-openpose-sdxl-1.0/resolve/main/control-lora-openposeXL2-rank256.safetensors
```

**モデル配置確認**
```bash
ls -lh ~/ai-tools/ComfyUI/models/controlnet/

# 出力例:
# diffusion_pytorch_model_promax.safetensors (2.5GB) - Union
# depth-zoe-xl-v1.0-controlnet.safetensors (2.5GB)
# diffusion_pytorch_model.safetensors (2.5GB) - Canny
```

### 5.2.2 プリプロセッサのインストール

**ComfyUI ControlNet Preprocessors**
```bash
cd ~/ai-tools/ComfyUI/custom_nodes

# ControlNet Preprocessorsのインストール
git clone https://github.com/Fannovel16/comfyui_controlnet_aux.git
cd comfyui_controlnet_aux
pip install -r requirements.txt

# ComfyUI再起動
cd ~/ai-tools/ComfyUI
python main.py
```

**利用可能なプリプロセッサ**
```
画像 → Canny Edge:
- エッジ検出
- 閾値調整可能

画像 → Depth Map:
- MiDaS, ZoeDepth
- 単眼深度推定

画像 → OpenPose:
- 人物骨格検出
- 手、顔の詳細対応

画像 → LineArt:
- 線画抽出
- アニメ、リアル両対応

画像 → Normal Map:
- 法線マップ生成
- BAE Normalizer使用

画像 → Segmentation:
- OneFormer使用
- 領域分割
```

## 5.3 基本的なControlNetワークフロー

### 5.3.1 Canny Edge制御ワークフロー

**構成（10ノード）**
```
[Load Image] → IMAGE
         ↓
[Canny Edge Preprocessor]
  low_threshold: 100
  high_threshold: 200
         ↓ IMAGE (エッジ画像)
         ↓
[Load ControlNet Model]
  control_net_name: "diffusion_pytorch_model_promax.safetensors"
         ↓ CONTROL_NET
         ↓
[Apply ControlNet]
  strength: 0.8
         ↓ CONDITIONING
         ↓
[Load Checkpoint: SDXL Base]
         ↓
[CLIP Text Encode] → CONDITIONING
         ↓
[KSampler]
         ↓
[VAE Decode] → [Save Image]
```

**設定詳細**
```yaml
Canny Edge Preprocessor:
  low_threshold: 100    # エッジ検出下限
  high_threshold: 200   # エッジ検出上限
  # 値が低い→より多くのエッジ
  # 値が高い→主要なエッジのみ

Apply ControlNet:
  strength: 0.8         # 制御の強さ
  # 0.0 = 制御なし
  # 0.5 = 中程度の制御
  # 1.0 = 最大制御

KSampler:
  steps: 30            # ControlNet使用時は多めに
  cfg: 7.5
  sampler: dpmpp_2m_karras
```

**MS-S1 Max性能**
```
処理時間:
- Preprocessor (Canny): 0.5秒
- 生成 (1024x1024): 11秒
- 合計: 11.5秒

メモリ使用:
- SDXL Base: 8GB
- ControlNet: 2.5GB
- 合計: 10.5GB VRAM
```

### 5.3.2 Depth制御ワークフロー

**深度マップ生成**
```
[Load Image (参照画像)]
         ↓
[MiDaS Depth Preprocessor]
  model_type: "DPT_Large"
  # または ZoeDepth（より高精度）
         ↓ IMAGE (深度マップ)
         ↓
[Apply ControlNet]
  control_net_name: "depth-zoe-xl-v1.0"
  strength: 0.7
         ↓
[SDXL Base] → [KSampler]
         ↓
[VAE Decode] → [Save Image]
```

**用途と設定**
```yaml
風景写真の構図保持:
  strength: 0.6-0.7
  prompt: "fantasy landscape, magical atmosphere"
  効果: 奥行きを維持しつつスタイル変更

建築物の再構築:
  strength: 0.8-0.9
  prompt: "futuristic building, cyberpunk style"
  効果: 3D構造を正確に維持

人物配置の制御:
  strength: 0.5-0.7
  prompt: "portrait in different lighting"
  効果: 空間配置を保ちながら変更
```

### 5.3.3 OpenPose骨格制御

**人物ポーズ制御**
```
[Load Image (参照人物)]
         ↓
[OpenPose Preprocessor]
  detect_hand: True     # 手の検出
  detect_face: True     # 顔の検出
  detect_body: True     # 体の検出
         ↓ IMAGE (骨格図)
         ↓
[Apply ControlNet]
  control_net_name: "control-lora-openposeXL2"
  strength: 0.8
         ↓
[SDXL Base + Prompt]
  prompt: "anime character, magical girl outfit"
         ↓
[KSampler] → [VAE Decode]
```

**実用例**
```python
# ケース1: 写真→アニメキャラ変換
input_image = "photo_of_person_pose.jpg"
preprocessor = "OpenPose"
strength = 0.85
prompt = "anime style, character design, colorful outfit"

# ケース2: 複数人物の配置
input_image = "group_photo.jpg"
preprocessor = "OpenPose"
strength = 0.75
prompt = "fantasy characters, RPG party"

# ケース3: アニメーションフレーム
input_images = ["frame_001.jpg", "frame_002.jpg", ...]
# 各フレームで一貫したキャラクター生成
```

## 5.4 高度なControlNet技術

### 5.4.1 マルチControlNet

**複数の制御を同時使用**
```
[Load Image]
    ↓
    ├→ [Canny Edge] → ControlNet 1 (strength: 0.6)
    ↓
    ├→ [Depth Map] → ControlNet 2 (strength: 0.5)
    ↓
    └→ [両方を組み合わせ] → [KSampler]

効果:
- エッジと深度の両方を制御
- より正確な構図再現
- 複雑なシーンに有効
```

**strength調整戦略**
```yaml
優先度高い制御:
  controlnet_1:
    type: Depth
    strength: 0.8     # 主要制御
  controlnet_2:
    type: Canny
    strength: 0.4     # 補助制御

バランス型:
  controlnet_1:
    type: OpenPose
    strength: 0.6
  controlnet_2:
    type: LineArt
    strength: 0.6

微調整型:
  controlnet_1:
    type: Depth
    strength: 0.9
  controlnet_2:
    type: Normal
    strength: 0.3     # 細部調整のみ
```

### 5.4.2 ControlNetとLoRAの組み合わせ

**スタイル+構図制御**
```
[Load Checkpoint: SDXL Base]
         ↓
[Load LoRA]
  lora_name: "anime_style_v2.safetensors"
  strength_model: 0.8
         ↓
[Apply ControlNet (Depth)]
  strength: 0.7
         ↓
[KSampler]
  prompt: "anime landscape, studio ghibli style"
         ↓
[VAE Decode]

結果:
- LoRAによるスタイル適用
- ControlNetによる構図制御
- 両方の利点を活用
```

### 5.4.3 IP-Adapter + ControlNet

**スタイル参照+構図制御**
```
[Load Image: Style Reference]
         ↓
[IP-Adapter]
  weight: 0.7
         ↓
[Load Image: Composition Reference]
         ↓
[ControlNet Depth]
  strength: 0.8
         ↓
[KSampler]

用途:
- スタイル転送+構図維持
- アート作品の再解釈
- 一貫性のあるシリーズ作成
```

## 5.5 用途別ControlNetワークフロー

### 5.5.1 建築パース作成

**3Dモデル→フォトリアル**
```yaml
Step 1: 3Dモデルからレンダリング
- Blender/SketchUpで基本形状
- シンプルなマテリアル
- カメラアングル確定

Step 2: Depth + Normal抽出
- Depth Map: Z-Buffer
- Normal Map: Render pass

Step 3: ControlNet適用
preprocessor: なし（既に深度画像）
controlnet_1: Depth (strength: 0.9)
controlnet_2: Normal (strength: 0.5)
prompt: "modern architecture, glass and concrete,
         professional photography, golden hour"
steps: 35
cfg: 8.0

生成時間 (MS-S1 Max): 18秒
品質: プロフェッショナルパース
```

### 5.5.2 キャラクターデザイン

**ポーズバリエーション作成**
```python
# ベースポーズ準備
base_pose_images = [
    "standing_pose.jpg",
    "action_pose.jpg",
    "sitting_pose.jpg"
]

# 各ポーズでキャラクター生成
for pose_img in base_pose_images:
    workflow = {
        "preprocessor": "OpenPose",
        "controlnet": "openpose-sdxl",
        "strength": 0.85,
        "prompt": "anime character, warrior outfit,
                   detailed armor, fantasy style",
        "steps": 30,
        "cfg": 7.0,
        "batch_size": 4  # 4バリエーション
    }

# 結果: 3ポーズ × 4バリエーション = 12画像
# MS-S1 Max時間: 約2分
```

### 5.5.3 商品写真の背景差し替え

**構図維持+背景変更**
```
[Load Image (商品写真)]
         ↓
[Remove Background]  # カスタムノード
         ↓ 商品のみ
         ↓
[Canny Edge Preprocessor]
         ↓
[ControlNet Canny (strength: 0.9)]
         ↓
[KSampler]
  prompt: "product photography, luxury background,
           marble surface, studio lighting"
         ↓
[VAE Decode]

用途:
- EC サイト用画像
- カタログ作成
- バリエーション生成

MS-S1 Max効率:
- バッチサイズ8で複数背景
- 生成時間: 35秒/8画像
```

## 5.6 プリプロセッサの詳細設定

### 5.6.1 Cannyエッジ調整

**パラメータの影響**
```python
# シンプルな輪郭のみ
canny_simple = {
    "low_threshold": 150,
    "high_threshold": 250,
    "result": "主要なエッジのみ検出"
}

# 詳細なエッジ
canny_detailed = {
    "low_threshold": 50,
    "high_threshold": 150,
    "result": "細かいディテールまで検出"
}

# バランス（推奨）
canny_balanced = {
    "low_threshold": 100,
    "high_threshold": 200,
    "result": "適度なディテール"
}
```

**用途別設定**
```yaml
建築物:
  low: 120
  high: 220
  理由: クリーンなラインが重要

人物:
  low: 80
  high: 180
  理由: 柔らかな輪郭が必要

風景:
  low: 100
  high: 200
  理由: バランス重視

線画アート:
  low: 150
  high: 250
  理由: クリーンなラインのみ
```

### 5.6.2 Depth精度調整

**深度推定モデル比較**
```
MiDaS DPT_Large:
- 精度: ★★★★☆
- 速度: ★★★☆☆
- 用途: 一般的な深度推定
- 処理時間: 1.5秒 @ 1024x1024 (MS-S1 Max)

ZoeDepth:
- 精度: ★★★★★
- 速度: ★★☆☆☆
- 用途: 高精度が必要なケース
- 処理時間: 3.0秒 @ 1024x1024 (MS-S1 Max)

MiDaS Small:
- 精度: ★★★☆☆
- 速度: ★★★★★
- 用途: プロトタイピング
- 処理時間: 0.8秒 @ 1024x1024 (MS-S1 Max)
```

**MS-S1 Max推奨**
```
通常作業: MiDaS DPT_Large
- バランスが良い
- 十分な精度

最終出力: ZoeDepth
- 最高品質
- 時間がかかっても可

大量生成: MiDaS Small
- 速度優先
- バッチ処理向け
```

### 5.6.3 OpenPose精度設定

**検出オプション**
```python
openpose_config = {
    "detect_body": True,     # 必須
    "detect_hand": True,     # 手のディテール
    "detect_face": True,     # 顔の向き
    "resolution": 512,       # 検出解像度
}

# 用途別設定

# 全身ポートレート
config_fullbody = {
    "detect_body": True,
    "detect_hand": True,   # 重要
    "detect_face": True,
    "resolution": 512,
}

# 顔中心
config_face = {
    "detect_body": True,
    "detect_hand": False,  # 不要
    "detect_face": True,   # 最重要
    "resolution": 768,     # 高解像度
}

# アクションポーズ
config_action = {
    "detect_body": True,   # 最重要
    "detect_hand": True,
    "detect_face": False,  # 優先度低
    "resolution": 512,
}
```

## 5.7 トラブルシューティング

### 5.7.1 一般的な問題

**問題1: 制御が効かない**
```
症状:
- ControlNet適用しても構図が変わらない
- プロンプトのみで生成されている

原因と解決:
1. strengthが低すぎる
   解決: 0.7-0.9に上げる

2. CFGが高すぎる
   解決: CFG 7.5以下に調整

3. Steps不足
   解決: 30-35 stepsに増やす

4. ControlNetモデル未ロード
   解決: Apply ControlNetノード確認
```

**問題2: 過度な制御**
```
症状:
- 元画像を単純にトレースしただけ
- 創造性がない

原因と解決:
1. strengthが高すぎる
   解決: 0.5-0.7に下げる

2. denoiseが低すぎる
   解決: denoise 0.9-1.0に設定

3. プロンプトが不十分
   解決: 詳細なプロンプト追加
```

**問題3: メモリ不足（MS-S1 Maxでは稀）**
```
症状:
- "HIP out of memory"
- マルチControlNet使用時

解決策:
1. ControlNet数を削減
   2つまでに制限

2. 解像度を下げる
   1024x1024 → 896x896

3. GPU割り当て確認
   export GPU_MAX_ALLOC_PERCENT=95
```

### 5.7.2 品質最適化

**チェックリスト**
```yaml
□ プリプロセッサ設定を調整
  - Cannyの閾値
  - Depth の精度
  - OpenPoseの検出オプション

□ ControlNet strengthを微調整
  - 0.5から開始
  - 0.1ずつ調整
  - 最適値を見つける

□ 適切なSteps数
  - ControlNet使用時: 30-35
  - 品質優先: 35-40
  - 速度優先: 25-30

□ CFG値の調整
  - ControlNet使用時: 7.0-8.0
  - 複雑な構図: 7.5-8.5
  - シンプルな構図: 6.5-7.5

□ Sampler選択
  - 推奨: dpmpp_2m_karras
  - 高品質: dpmpp_sde_karras
```

## 5.8 MS-S1 Max最適化戦略

### 5.8.1 バッチControlNet処理

**複数画像の一括処理**
```python
# ワークフロー設計
input_images = [
    "ref_001.jpg",
    "ref_002.jpg",
    "ref_003.jpg",
    "ref_004.jpg"
]

# バッチ処理ワークフロー
for image in input_images:
    # Preprocessor適用（並列実行可能）
    depth_map = apply_preprocessor(image, "MiDaS")

    # ControlNet生成
    result = generate_with_controlnet(
        control_image=depth_map,
        prompt="fantasy landscape, dramatic lighting",
        strength=0.75,
        steps=30,
        batch_size=2  # 各画像で2バリエーション
    )

# 合計: 4画像 × 2バリエーション = 8画像
# MS-S1 Max時間: 約1.5分
```

### 5.8.2 並列プリプロセッサ実行

**128GBメモリ活用**
```bash
# 複数プリプロセッサ同時実行

# ターミナル1: Depth処理
python preprocess_batch.py --type depth --input_dir ./images

# ターミナル2: Canny処理（並行）
python preprocess_batch.py --type canny --input_dir ./images

# ターミナル3: OpenPose処理（並行）
python preprocess_batch.py --type openpose --input_dir ./images

# メモリ使用量:
# 各プリプロセッサ: 4-8GB
# 合計: 12-24GB
# 残り: 100GB以上（余裕）
```

## 5.9 本章のまとめ

本章で学んだ内容：

**ControlNet基礎**
- 制御タイプの理解（Canny, Depth, OpenPose等）
- SDXL対応モデル（Union推奨）
- プリプロセッサの役割

**基本ワークフロー**
- Canny Edge制御
- Depth Map制御
- OpenPose骨格制御

**高度な技術**
- マルチControlNet
- LoRA/IP-Adapterとの組み合わせ
- 実用的なワークフロー例

**MS-S1 Max最適化**
- バッチ処理戦略
- 並列プリプロセッサ実行
- メモリ効率的な運用

**トラブルシューティング**
- 一般的な問題と解決
- 品質最適化チェックリスト

次章では、LoRAモデルの使用と作成について詳しく学びます。

---

**参考リソース**
- ControlNet論文: https://arxiv.org/abs/2302.05543
- ControlNet Union SDXL: https://huggingface.co/xinsir/controlnet-union-sdxl-1.0
- ComfyUI ControlNet Aux: https://github.com/Fannovel16/comfyui_controlnet_aux
- Stable Diffusion Art ControlNet Guide: https://stable-diffusion-art.com/controlnet/

