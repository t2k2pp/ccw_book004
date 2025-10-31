# 第3章: SDXLの基礎とプロンプトエンジニアリング

## 3.1 SDXLアーキテクチャの理解

### 3.1.1 SDXL vs SD 1.5

**アーキテクチャの進化**
```
SD 1.5:
- U-Netパラメータ: 860M
- テキストエンコーダ: CLIP ViT-L/14
- 解像度: 512x512ネイティブ
- 総パラメータ: 約1B

SDXL:
- U-Netパラメータ: 2.6B
- テキストエンコーダ: CLIP ViT-L + OpenCLIP ViT-G
- 解像度: 1024x1024ネイティブ
- 総パラメータ: 約6.6B
- 改善点: テキスト理解、構図、詳細表現
```

### 3.1.2 SDXL の2段階モデル

**Base Model（必須）**
```
役割:
- メイン画像生成
- 1024x1024でのネイティブ生成
- プロンプトの主要解釈

パラメータ: 2.6B
学習データ: 高品質画像で事前学習
用途: すべての生成の基礎
```

**Refiner Model（オプション）**
```
役割:
- Base生成画像の詳細強化
- 高周波数ディテールの改善
- 最終仕上げ

パラメータ: 2.3B
学習データ: 高品質画像で微調整
使用タイミング: Baseの80-85%完了時点から
```

**MS-S1 MaxでのRefiner使用判断**
```python
# Refiner推奨ケース
使用推奨:
✅ 最終出力品質重視
✅ 詳細な質感表現が必要
✅ プロフェッショナル用途
✅ 時間に余裕がある（+40-50%時間）

不要なケース:
❌ プロトタイピング
❌ 大量生成
❌ リアルタイム性重視
❌ メモリ制約がある

# 生成時間比較（MS-S1 Max, 1024x1024）
Base only: 10-12秒
Base + Refiner: 16-20秒
```

## 3.2 プロンプトエンジニアリング基礎

### 3.2.1 SDXLのプロンプト特性

**SD 1.5との違い**
```
SD 1.5:
- キーワード中心
- カンマ区切りタグ重視
- 厳密な順序依存
- ネガティブプロンプト必須

SDXL:
- 自然言語理解向上
- 文章形式でも認識
- 文脈理解が可能
- ネガティブプロンプト最小限で可
- より高い意味理解
```

**SDXLプロンプトの強み**
```python
# SD 1.5スタイル（動作するが最適ではない）
prompt_sd15 = "cat, sitting, window, sunlight, detailed fur, 4k"

# SDXLスタイル（推奨）
prompt_sdxl = "A fluffy orange cat sitting on a windowsill, bathed in warm sunlight. The cat's fur is detailed and realistic, with individual strands visible. High quality, 4k resolution."

# どちらも動作するが、SDXL形式の方が意図を正確に反映
```

### 3.2.2 プロンプト構造のベストプラクティス

**推奨プロンプト構造**
```
1. 主題 (Subject)
   - 何を生成するか明確に

2. 詳細な描写 (Detailed Imagery)
   - 色、質感、材質

3. 環境・背景 (Environment)
   - シーン、前景・背景

4. ムード・雰囲気 (Mood/Atmosphere)
   - 照明、感情、スタイル

5. 技術指定 (Technical)
   - 解像度、品質、アーティスト
```

**実例: 風景画像**
```
# 基本プロンプト
"mountain landscape"
→ 生成されるが曖昧

# 構造化プロンプト（推奨）
"A majestic snow-capped mountain range during golden hour.
The peaks are illuminated by warm sunset light, creating
long shadows across alpine meadows in the foreground.
Dramatic clouds gather around the summits. Crystal clear
lake reflects the scene. Professional landscape photography,
high detail, 4k quality, cinematic composition."

構成要素:
1. 主題: mountain range
2. 詳細: snow-capped, golden hour, warm sunset light
3. 環境: alpine meadows, crystal clear lake
4. ムード: dramatic clouds, majestic
5. 技術: professional photography, 4k, cinematic
```

**実例: ポートレート**
```
"A portrait of a young woman with long flowing auburn hair,
piercing green eyes, and a gentle smile. Soft natural lighting
from a window creates a warm glow on her face. She's wearing
a cream-colored sweater. Shallow depth of field blurs the
cozy interior background. Professional portrait photography,
85mm lens, f/1.8, soft focus, high quality."

構成要素:
1. 主題: young woman
2. 詳細: auburn hair, green eyes, cream sweater
3. 環境: cozy interior, window light
4. ムード: gentle smile, warm glow
5. 技術: 85mm, f/1.8, soft focus
```

### 3.2.3 キーワードウェイト

**ウェイト構文**
```python
# ComfyUIでのウェイト指定
基本: (keyword)          # 1.1倍
強調: ((keyword))        # 1.21倍 (1.1^2)
さらに: (((keyword)))    # 1.331倍 (1.1^3)

# 数値指定（より正確）
(keyword:1.2)   # 1.2倍
(keyword:1.5)   # 1.5倍
(keyword:0.8)   # 0.8倍（弱める）

# SDXLの注意点
⚠️ SDXLは重みに敏感
⚠️ 1.4以上は通常不要
⚠️ やりすぎると崩壊
```

**適切なウェイト使用例**
```
# 適切な使用
"A (detailed:1.2) portrait of a woman with (flowing hair:1.1),
natural lighting, high quality"

# 過度な使用（非推奨）
"A (((detailed:1.5))) portrait of a (((woman:1.8))) with
(((flowing hair:2.0))), (((natural lighting:1.7)))"
→ アーティファクト発生のリスク
```

**MS-S1 Max推奨ウェイト戦略**
```yaml
標準強調: 1.1 - 1.2
中程度強調: 1.2 - 1.3
強い強調: 1.3 - 1.4
最大: 1.5（慎重に使用）

弱める: 0.7 - 0.9
大幅に弱める: 0.5 - 0.7
```

## 3.3 ネガティブプロンプト戦略

### 3.3.1 SDXLでのネガティブプロンプト

**SD 1.5 vs SDXL**
```
SD 1.5:
- 大量のネガティブキーワード必須
- "ugly, bad, deformed, extra fingers, ..." (50+ words)
- ネガティブなしでは品質低下

SDXL:
- 最小限で効果的
- 避けたい具体的要素のみ
- 過度なネガティブは逆効果
```

**SDXL最適化ネガティブプロンプト**
```python
# 最小限（推奨）
negative_minimal = "low quality, blurry"

# 標準（一般用途）
negative_standard = "low quality, blurry, distorted, watermark"

# 詳細（特定用途）
negative_detailed = """
low quality, blurry, out of focus,
distorted, watermark, text,
oversaturated, underexposed
"""

# 過度（非推奨）
negative_excessive = """
ugly, bad, deformed, extra fingers,
mutated hands, poorly drawn, bad anatomy,
wrong anatomy, extra limbs, missing limbs,
floating limbs, disconnected limbs, mutation,
mutated, ugly, disgusting, blurry, amputation,
JPEG artifacts, signature, watermark, username,
sketch, cartoon, drawing, anime, text, cropped,
out of frame, worst quality, low quality,
jpeg artifacts, ugly, duplicate, morbid,
mutilated, extra digits, fewer digits, ...
"""
→ SDXLでは不要、むしろ悪影響
```

**用途別ネガティブプロンプト**
```
写実的写真:
"cartoon, illustration, 3d render, painting"

イラスト生成:
"photograph, realistic, photorealistic"

ポートレート:
"multiple people, crowd, extra arms, extra legs"

風景:
"人物, buildings (避けたい場合), vehicles"

アニメスタイル:
"realistic, photograph, 3d"
```

### 3.3.2 MS-S1 Max推奨設定

**品質レベル別設定**
```yaml
# 高速プロトタイピング
positive: "simple subject description"
negative: "low quality"
steps: 20
cfg: 7.0

# 標準品質
positive: "detailed subject with environment and mood"
negative: "low quality, blurry, distorted"
steps: 25
cfg: 7.5

# 最高品質
positive: "comprehensive description with all elements"
negative: "low quality, blurry, distorted, watermark, oversaturated"
steps: 30-35
cfg: 7.0-8.0
use_refiner: true
```

## 3.4 解像度とアスペクト比

### 3.4.1 SDXLネイティブ解像度

**ピクセル総数の重要性**
```
SDXLの学習解像度: 1024x1024 = 1,048,576ピクセル

推奨解像度（同じピクセル総数）:
✅ 1024x1024 (1:1)   - 1,048,576
✅ 1152x896  (9:7)   - 1,032,192
✅ 896x1152  (7:9)   - 1,032,192
✅ 1216x832  (3:2)   - 1,011,712
✅ 832x1216  (2:3)   - 1,011,712
✅ 1344x768  (16:9)  - 1,032,192
✅ 768x1344  (9:16)  - 1,032,192
✅ 1536x640  (21:9)  - 983,040

⚠️ 避けるべき:
❌ 512x512   - ぼやける
❌ 2048x2048 - メモリ不足、アーティファクト
```

### 3.4.2 MS-S1 Maxメモリ別推奨解像度

**VRAM割り当て別**
```python
# 16GB VRAM割り当て
resolutions_16gb = {
    "standard": (1024, 1024),      # 10-12秒
    "portrait": (832, 1216),       # 10-12秒
    "landscape": (1216, 832),      # 10-12秒
    "wide": (1344, 768),           # 11-13秒
    "max_safe": (1152, 896),       # 10-12秒
}

# 20GB VRAM割り当て
resolutions_20gb = {
    "standard": (1024, 1024),      # 10-12秒
    "high_res": (1536, 1536),      # 26-30秒
    "ultrawide": (1728, 768),      # 18-22秒
    "batch_2": (1024, 1024, 2),    # 18-22秒
}

# 24GB VRAM割り当て（MS-S1 Max最大）
resolutions_24gb = {
    "standard": (1024, 1024),      # 10-12秒
    "high_res": (1536, 1536),      # 26-30秒
    "ultra_res": (2048, 2048),     # 55-65秒（注意）
    "batch_4": (1024, 1024, 4),    # 35-40秒
    "batch_2_hires": (1536, 1536, 2), # 50-60秒
}
```

## 3.5 サンプラーとスケジューラの詳細

### 3.5.1 SDXL推奨サンプラー

**サンプラーパフォーマンス比較（MS-S1 Max）**
```
┌──────────────────┬────────┬────────┬──────────┬─────────┐
│ Sampler          │ Speed  │ Quality│ Steps    │ Use Case│
├──────────────────┼────────┼────────┼──────────┼─────────┤
│ Euler a          │ ★★★★★ │ ★★★☆☆ │ 20-40    │ 実験    │
│ Euler            │ ★★★★★ │ ★★★☆☆ │ 25-50    │ 高速    │
│ DPM++ 2M Karras  │ ★★★★☆ │ ★★★★★ │ 20-30    │ 推奨    │
│ DPM++ SDE Karras │ ★★★☆☆ │ ★★★★★ │ 25-35    │ 高品質  │
│ DPM++ 2M SDE     │ ★★★☆☆ │ ★★★★☆ │ 20-30    │ バランス│
│ DDIM             │ ★★★★☆ │ ★★★★☆ │ 30-50    │ 再現性  │
│ UniPC            │ ★★★★☆ │ ★★★☆☆ │ 15-25    │ 超高速  │
└──────────────────┴────────┴────────┴──────────┴─────────┘
```

**用途別サンプラー選択**
```yaml
# プロトタイピング（速度重視）
sampler: euler_a
steps: 20
time: ~8秒 @ 1024x1024

# 一般用途（バランス）
sampler: dpmpp_2m_karras
steps: 25
time: ~11秒 @ 1024x1024

# 高品質出力
sampler: dpmpp_sde_karras
steps: 30
time: ~15秒 @ 1024x1024

# アニメーション（再現性）
sampler: ddim
steps: 40
time: ~20秒 @ 1024x1024
```

### 3.5.2 CFG（Classifier Free Guidance）の最適化

**CFG値の影響**
```python
cfg_effects = {
    1.0: {
        "adherence": "なし",
        "creativity": "最大",
        "result": "プロンプト無視、ランダム",
        "use": "実験的"
    },
    3.0: {
        "adherence": "低",
        "creativity": "高",
        "result": "自由な解釈、多様性",
        "use": "芸術的表現"
    },
    5.0: {
        "adherence": "中",
        "creativity": "中",
        "result": "バランス良好",
        "use": "探索的生成"
    },
    7.0: {
        "adherence": "高",
        "creativity": "中",
        "result": "プロンプト正確に反映",
        "use": "標準推奨"
    },
    10.0: {
        "adherence": "非常に高",
        "creativity": "低",
        "result": "詳細指示に従う",
        "use": "特定要求"
    },
    15.0: {
        "adherence": "過度",
        "creativity": "ほぼなし",
        "result": "色飽和、アーティファクト",
        "use": "通常非推奨"
    }
}
```

**用途別CFG推奨値**
```yaml
写実的写真:
  cfg: 7.0-8.0
  reason: 正確な描写が重要

イラスト:
  cfg: 6.0-7.5
  reason: 芸術的自由度とのバランス

アニメスタイル:
  cfg: 6.5-8.0
  reason: スタイル一貫性重視

コンセプトアート:
  cfg: 5.0-7.0
  reason: 創造性重視

技術図面:
  cfg: 9.0-11.0
  reason: 正確性最優先

抽象芸術:
  cfg: 3.0-6.0
  reason: 自由な表現
```

## 3.6 実践的プロンプト例

### 3.6.1 風景写真

**基本**
```
Prompt:
A serene mountain lake at dawn, perfectly still water
reflecting snow-capped peaks. Mist rising from the
surface, soft pink and orange sunrise colors. Pine
trees frame the foreground. Professional landscape
photography, high detail.

Negative:
low quality, people, buildings

Settings:
- Resolution: 1344x768 (16:9)
- Steps: 25
- CFG: 7.5
- Sampler: dpmpp_2m_karras
```

**高度**
```
Prompt:
Dramatic alpine landscape during the golden hour,
viewed from an elevated vantage point. Jagged peaks
pierce through layers of clouds, creating a sea of
mist below. Warm sunset light bathes the mountain
faces in golden and amber tones, while shadows define
deep valleys. In the foreground, weathered rocks and
hardy alpine flowers add depth. A winding hiking trail
disappears into the distance. Professional outdoor
photography with wide-angle lens, f/11 for maximum
depth of field, graduated ND filter for balanced
exposure, tack sharp details, National Geographic
quality.

Negative:
low quality, blurry, people, man-made structures,
watermark, oversaturated

Settings:
- Resolution: 1216x832 (3:2)
- Steps: 30
- CFG: 7.0
- Sampler: dpmpp_sde_karras
- Refiner: Yes (0.85 denoise start)
```

### 3.6.2 ポートレート

**基本**
```
Prompt:
Portrait of a young woman with wavy brown hair and
blue eyes, natural smile. Soft window light from the
left creates a gentle glow. Wearing a simple white
shirt. Blurred background. Professional headshot,
85mm lens.

Negative:
multiple people, cartoon, low quality

Settings:
- Resolution: 832x1216 (2:3 portrait)
- Steps: 25
- CFG: 7.5
- Sampler: dpmpp_2m_karras
```

**高度**
```
Prompt:
Cinematic portrait of an elderly craftsman in his
workshop, captured in dramatic Rembrandt lighting.
Deep wrinkles and weathered hands tell stories of
decades of skilled work. A single window provides
directional light from the left, creating strong
chiaroscuro effect with half the face illuminated.
Warm amber tones from workshop lamps add depth.
Background shows blurred tools and wooden surfaces.
He's wearing a worn leather apron, hands holding a
handcrafted item. Eyes reflect wisdom and pride. Shot
with 50mm f/1.4 lens at f/2 for shallow depth of
field, professional color grading, film grain texture,
award-winning photography.

Negative:
low quality, blurry, multiple people, cartoon,
oversaturated, modern clothing

Settings:
- Resolution: 832x1216 (2:3 portrait)
- Steps: 32
- CFG: 7.5
- Sampler: dpmpp_sde_karras
- Refiner: Yes (0.80 denoise start)
```

### 3.6.3 アーキテクチャ

**基本**
```
Prompt:
Modern minimalist house with large glass windows,
clean white walls, and flat roof. Set in a green
landscape with trees. Blue sky. Architectural
photography, clear details.

Negative:
low quality, blurry, people, cars

Settings:
- Resolution: 1216x832 (3:2)
- Steps: 25
- CFG: 8.0
- Sampler: dpmpp_2m_karras
```

**高度**
```
Prompt:
Stunning contemporary architectural masterpiece
featuring cantilevered volumes and floor-to-ceiling
glass curtain walls. The structure seamlessly blends
geometric concrete forms with natural wood cladding.
Set on a dramatic hillside overlooking a valley,
captured during blue hour when interior lights create
warm glows against the deepening sky. A reflective
infinity pool in the foreground mirrors both building
and sky. Carefully designed landscape with native
plants frames the composition. Shot with tilt-shift
lens to correct perspective, long exposure for smooth
water, professional architectural photography,
published in Architectural Digest, hyperdetailed,
perfect symmetry where intended.

Negative:
low quality, distorted perspective, people,
vehicles, watermark, oversaturated

Settings:
- Resolution: 1344x768 (16:9)
- Steps: 35
- CFG: 8.5
- Sampler: dpmpp_sde_karras
- Refiner: Yes (0.85 denoise start)
```

### 3.6.4 キャラクターデザイン

**アニメスタイル**
```
Prompt:
Anime-style character portrait of a female mage with
long silver hair and purple eyes, wearing an ornate
blue and gold robe with intricate magical patterns.
She's holding a glowing staff with a crystal at the
top. Confident expression. Fantasy setting with soft
magical particles floating around. High quality anime
artwork, detailed shading, vibrant colors.

Negative:
photograph, realistic, 3d, low quality, blurry

Settings:
- Resolution: 832x1216 (2:3)
- Steps: 28
- CFG: 7.0
- Sampler: dpmpp_2m_karras
```

**ファンタジーリアリズム**
```
Prompt:
Highly detailed character concept art of a battle-worn
female warrior in dark fantasy setting. She wears
weathered leather and chainmail armor with visible
scratches and battle damage. Long dark hair tied back
practically, intense green eyes show determination.
Holding a notched longsword, stance ready for combat.
Background shows a misty battlefield at dawn. Painted
in the style of high-end game concept art with
dramatic lighting, rich textures, and careful
attention to material properties. Photorealistic
rendering with painterly touches, trending on ArtStation.

Negative:
low quality, cartoon, anime, oversexualized,
impractical armor, blurry

Settings:
- Resolution: 832x1216 (2:3)
- Steps: 32
- CFG: 7.5
- Sampler: dpmpp_sde_karras
- Refiner: Yes (0.80 denoise start)
```

## 3.7 イテレーティブプロンプト改善

### 3.7.1 段階的アプローチ

**ステップ1: 基本プロンプト**
```
"a cat"
→ 生成される、が一般的すぎる
```

**ステップ2: 詳細追加**
```
"a fluffy orange tabby cat sitting on a windowsill"
→ より具体的、しかしムードが不足
```

**ステップ3: 環境とムード**
```
"a fluffy orange tabby cat sitting on a windowsill,
warm sunlight streaming through the window, cozy
indoor atmosphere"
→ 良好、さらに品質指定可能
```

**ステップ4: 技術仕様**
```
"a fluffy orange tabby cat sitting on a windowsill,
warm sunlight streaming through the window creating
soft shadows, cozy indoor atmosphere with blurred
background. Professional pet photography, shallow
depth of field, 85mm lens, high detail, 4k quality"
→ 最適化完了
```

### 3.7.2 A/Bテスト戦略

**パラメータ変更テスト**
```python
# ベースライン
baseline = {
    "prompt": "基本プロンプト",
    "negative": "low quality",
    "steps": 25,
    "cfg": 7.5,
    "sampler": "dpmpp_2m_karras",
    "seed": 42
}

# テスト1: CFG変更
test_cfg = baseline.copy()
test_cfg["cfg"] = 6.5
# 結果を比較

# テスト2: Steps変更
test_steps = baseline.copy()
test_steps["steps"] = 30
# 結果を比較

# テスト3: Sampler変更
test_sampler = baseline.copy()
test_sampler["sampler"] = "dpmpp_sde_karras"
# 結果を比較

# 最良の組み合わせを特定
```

## 3.8 MS-S1 Max最適化ワークフロー

### 3.8.1 バッチ生成戦略

**探索フェーズ**
```yaml
目的: 複数バリエーション生成
設定:
  resolution: 1024x1024
  batch_size: 4
  steps: 20
  sampler: euler_a
  cfg: 7.0
  time_per_batch: 35-40秒
  purpose: 4つの異なるseedで迅速にアイデア探索
```

**精錬フェーズ**
```yaml
目的: 選択した1-2枚を高品質化
設定:
  resolution: 1536x1536 または 1024x1024
  batch_size: 1
  steps: 30
  sampler: dpmpp_sde_karras
  cfg: 7.5
  refiner: true
  time_per_image: 26-35秒（Refinerなし）または 40-50秒（Refinerあり）
```

### 3.8.2 メモリ効率的ワークフロー

**128GBメモリ活用**
```
ワークフロー設計:
1. 複数モデル同時ロード
   - SDXL Base: 8GB
   - SDXL Refiner: 8GB
   - VAE: 0.5GB
   - ControlNet（必要時）: 2-4GB
   - LoRA複数: 1-2GB

2. 残りメモリでキャッシュ
   - 生成画像履歴: 10-20GB
   - ワークフローバッファ: 5-10GB
   - システム余裕: 80GB以上

3. 同時実行可能
   - 生成中に次のプロンプト準備
   - バックグラウンドでアップスケール
   - 複数ワークフロー並行
```

## 3.9 トラブルシューティング

### 3.9.1 一般的な問題

**問題: 生成画像が期待と違う**
```
原因1: プロンプトが曖昧
解決策: より具体的な描写を追加

原因2: CFGが不適切
解決策: 7.0-8.0の範囲で調整

原因3: Steps不足
解決策: 25-30に増やす

原因4: Seedが不運
解決策: バッチ生成で複数試す
```

**問題: アーティファクト発生**
```
原因1: 解像度が不適切
解決策: 1024x1024ベースの解像度使用

原因2: CFGが高すぎる
解決策: 7.5以下に下げる

原因3: ウェイトの過度な使用
解決策: 1.4以下に制限

原因4: ネガティブプロンプト過多
解決策: 最小限に削減
```

## 3.10 本章のまとめ

本章で学んだ内容：

**SDXLアーキテクチャ**
- Base + Refinerの2段階モデル
- SD 1.5からの進化点
- MS-S1 Maxでの最適利用

**プロンプトエンジニアリング**
- 構造化プロンプトの重要性
- SDXLの自然言語理解
- キーワードウェイトの適切な使用
- ネガティブプロンプトの最小化

**技術パラメータ**
- 解像度とアスペクト比の最適化
- サンプラーとCFGの選択
- MS-S1 Max向け設定

**実践的アプローチ**
- 用途別プロンプト例
- イテレーティブ改善手法
- バッチ生成戦略

次章では、ComfyUIのワークフロー作成を詳しく学びます。

---

**参考リソース**
- SDXL論文: https://arxiv.org/abs/2307.01952
- Stable Diffusion Art ガイド: https://stable-diffusion-art.com/
- ComfyUI ワークフロー例: https://github.com/comfyanonymous/ComfyUI_examples

