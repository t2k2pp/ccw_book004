# 第1章: ComfyUIとStable Diffusion概要

## 1.1 ComfyUIとは

ComfyUIは、Stable Diffusionをはじめとする拡散モデルのための最も強力でモジュラーなGUI、API、バックエンドです。グラフ/ノードインターフェースを特徴とし、視覚的なプログラミング環境として機能します。

### 1.1.1 ComfyUIの特徴

**ノードベースワークフロー**
- ノード（矩形ブロック）とエッジ（接続線）で構成
- 視覚的なプログラミング環境
- シンプルなワークフローは6個程度のノード
- 高度なワークフローは数百個のノードを含む

**主要な利点**
```
✅ 柔軟性: カスタムワークフローを自由に構築
✅ モジュール性: ノードの組み合わせで複雑な処理
✅ 再現性: ワークフローを保存・共有可能
✅ パフォーマンス: 効率的なメモリ管理
✅ 拡張性: カスタムノードで機能追加
```

### 1.1.2 MS-S1 Maxでの優位性

**AMD Radeon 8060Sの活用**
```
GPU仕様:
- アーキテクチャ: RDNA 3.5
- ストリームプロセッサ: 2560 SP
- AI性能: 60 TOPS
- VRAM: 統合メモリから動的割り当て
- メモリ帯域幅: 256 GB/s (LPDDR5X-8000)

最適な画像生成環境:
- 大容量メモリ (128GB): 複数モデルの同時ロード
- 高速メモリバス: バッチ処理の高速化
- ROCm対応: PyTorchによるGPU加速
```

## 1.2 Stable Diffusionの基礎

### 1.2.1 拡散モデルの仕組み

Stable Diffusionは、潜在拡散モデル（Latent Diffusion Model）の一種です。

**基本原理**
```python
# 拡散プロセスの概念図
ノイズ画像 → デノイジング（複数ステップ） → 生成画像

1. テキストエンコード: プロンプトをCLIP埋め込みに変換
2. ノイズ生成: ランダムな潜在空間のノイズ
3. 段階的デノイジング: U-Netで徐々にノイズ除去
4. デコード: VAEで潜在空間から画像空間へ
```

**主要コンポーネント**
- **CLIP Text Encoder**: テキストを埋め込みベクトルに変換
- **U-Net**: ノイズ除去を行うニューラルネットワーク
- **VAE (Variational Autoencoder)**: 画像と潜在空間の相互変換
- **Scheduler (Sampler)**: デノイジングのステップを制御

### 1.2.2 SDXLとその進化

**SD 1.5からSDXLへ**
```
SD 1.5:
- 解像度: 512x512 ネイティブ
- パラメータ数: 約860M
- VRAM使用量: 4-6GB

SDXL (Stable Diffusion XL):
- 解像度: 1024x1024 ネイティブ
- パラメータ数: 約2.6B (Base) + 2.3B (Refiner)
- VRAM使用量: 8-12GB
- 品質: 大幅な向上（特にテキストレンダリング）
```

**SDXLの2段階アーキテクチャ**
```
Base Model:
- 高解像度生成の基礎
- 1024x1024での学習
- 条件付け強化

Refiner Model (オプション):
- Base生成画像の詳細強化
- 高周波数ディテールの改善
- 最終品質の向上
```

## 1.3 MS-S1 Max環境でのComfyUI

### 1.3.1 ハードウェア要件と最適化

**メモリ配分戦略**
```
128GB総メモリの推奨配分:

GPU VRAM割り当て: 16-24GB
- SDXL Base: 8-10GB
- SDXL Refiner: 6-8GB
- ControlNet/LoRA: 2-4GB
- バッファ: 2-4GB

システムメモリ: 残り104-112GB
- OS/バックグラウンド: 8-16GB
- モデルキャッシュ: 20-30GB
- ワークフロー処理: 10-20GB
- 余裕: 60GB以上
```

**ROCm最適化設定**
```bash
# MS-S1 Max専用環境変数
export HSA_OVERRIDE_GFX_VERSION=11.0.0  # RDNA 3.5対応
export PYTORCH_ROCM_ARCH=gfx1100       # アーキテクチャ指定
export ROC_ENABLE_PRE_VEGA=0           # 古いGPU無効化
export HSA_ENABLE_SDMA=1               # DMA転送有効化

# パフォーマンス最適化
export GPU_MAX_ALLOC_PERCENT=95        # GPU割り当て上限
export AMD_DIRECT_RENDERING=1          # 直接レンダリング
export RADV_PERFTEST=gpl,nggc          # Vulkan最適化
```

### 1.3.2 予想パフォーマンス

**SDXL生成速度（MS-S1 Max）**
```
解像度別生成時間（Steps=25）:

1024x1024 (SDXL標準):
- Base生成: 8-12秒
- Refiner適用: +4-6秒
- 合計: 12-18秒/画像

1536x1536 (高解像度):
- Base生成: 18-25秒
- Refiner適用: +8-12秒
- 合計: 26-37秒/画像

512x512 (SD 1.5互換):
- 生成: 2-4秒/画像
- 高速プロトタイピングに最適

バッチ処理（Batch Size=4）:
- 1024x1024: 35-45秒/4画像
- スループット: 約1.3-1.5秒/画像
```

**他のGPUとの比較**
```
NVIDIA RTX 4090 (24GB):
- SDXL 1024x1024: 6-8秒
- 優位性: 約30-40%高速

AMD RX 7900 XTX (24GB):
- SDXL 1024x1024: 10-14秒
- MS-S1 Maxと同等レベル

NVIDIA RTX 4060 Ti (16GB):
- SDXL 1024x1024: 15-20秒
- MS-S1 Maxの方が20-30%高速
```

## 1.4 ComfyUIのワークフロー基本構造

### 1.4.1 基本ノードの種類

**入力ノード**
```
Load Checkpoint:
- 役割: SDXLモデルの読み込み
- 出力: MODEL, CLIP, VAE
- 設定: モデルファイルの選択

CLIP Text Encode (Prompt):
- 役割: プロンプトのエンコード
- 入力: CLIP, text（テキスト）
- 出力: CONDITIONING
- 用途: Positive/Negative プロンプト

Empty Latent Image:
- 役割: 初期潜在空間の生成
- 設定: width, height, batch_size
- 出力: LATENT
- 用途: Text-to-Imageの開始点

Load Image:
- 役割: 画像ファイルの読み込み
- 出力: IMAGE, MASK
- 用途: Image-to-Image, Inpainting
```

**処理ノード**
```
KSampler:
- 役割: メインのデノイジング処理
- 入力: MODEL, CONDITIONING (pos/neg), LATENT
- 設定:
  * seed: ランダムシード
  * steps: デノイジングステップ数
  * cfg: プロンプト遵守度
  * sampler_name: サンプラーアルゴリズム
  * scheduler: ノイズスケジュール
  * denoise: ノイズ除去強度
- 出力: LATENT

VAE Decode:
- 役割: 潜在空間から画像への変換
- 入力: VAE, LATENT
- 出力: IMAGE
- 処理: 512x512潜在 → 4096x4096画像
```

**出力ノード**
```
Save Image:
- 役割: 生成画像の保存
- 入力: IMAGE
- 設定: filename_prefix
- 保存先: ComfyUI/output/

Preview Image:
- 役割: 画像のプレビュー表示
- 入力: IMAGE
- 用途: デバッグ、中間確認
```

### 1.4.2 最小限のワークフロー例

**Text-to-Image基本構成**
```
[Load Checkpoint] → MODEL → [KSampler]
                 ↓ CLIP               ↓ LATENT
                 ↓                    ↓
[CLIP Text Encode (Positive)] ────→ [KSampler]
                                      ↓
[CLIP Text Encode (Negative)] ────→ [KSampler]
                                      ↓
[Empty Latent Image] ──────────────→ [KSampler]
                                      ↓ LATENT
                                      ↓
                     [VAE Decode] ←──┘
                            ↓ IMAGE
                            ↓
                     [Save Image]
```

**実際のノード数**
```
最小構成: 6ノード
1. Load Checkpoint
2. CLIP Text Encode (Positive)
3. CLIP Text Encode (Negative)
4. Empty Latent Image
5. KSampler
6. VAE Decode → Save Image

実用構成: 10-15ノード
- Upscaling追加
- LoRA適用
- 複数パラメータ調整
- プレビュー表示

高度な構成: 50-200ノード
- ControlNet統合
- マルチパス生成
- 条件分岐
- カスタムロジック
```

## 1.5 主要なサンプラーとスケジューラ

### 1.5.1 サンプラーアルゴリズム

**推奨サンプラー（MS-S1 Max）**
```
DPM++ 2M Karras:
- 速度: ★★★★☆
- 品質: ★★★★★
- 特徴: バランスが良く、最も汎用的
- Steps推奨: 20-30
- 用途: ほとんどのケースで推奨

DPM++ SDE Karras:
- 速度: ★★★☆☆
- 品質: ★★★★★
- 特徴: 高品質、やや遅い
- Steps推奨: 25-35
- 用途: 最終出力、品質優先

Euler a:
- 速度: ★★★★★
- 品質: ★★★☆☆
- 特徴: 高速、多様性高い
- Steps推奨: 20-40
- 用途: プロトタイピング、実験

DDIM:
- 速度: ★★★★☆
- 品質: ★★★★☆
- 特徴: 決定論的、再現性高い
- Steps推奨: 25-50
- 用途: 一貫性が必要な場合
```

**MS-S1 Max最適化サンプラー設定**
```python
# 速度優先（プロトタイピング）
sampler_name = "euler_a"
steps = 20
scheduler = "normal"
# 生成時間: 6-8秒 @ 1024x1024

# バランス（推奨）
sampler_name = "dpmpp_2m_karras"
steps = 25
scheduler = "karras"
# 生成時間: 10-12秒 @ 1024x1024

# 品質優先（最終出力）
sampler_name = "dpmpp_sde_karras"
steps = 30
scheduler = "karras"
# 生成時間: 15-18秒 @ 1024x1024
```

### 1.5.2 スケジューラの種類

**スケジューラ比較**
```
normal:
- 線形的なノイズスケジュール
- 標準的な動作
- 予測可能な結果

karras:
- Karrasらの論文に基づく
- 初期ステップでノイズ除去を重点化
- 多くのケースで品質向上
- 推奨度: ★★★★★

exponential:
- 指数関数的なスケジュール
- 特定のモデルで有効
- 実験的

sgm_uniform:
- Stability AI SGMのデフォルト
- SDXLでの互換性高い
```

## 1.6 CFG (Classifier Free Guidance)

### 1.6.1 CFGの役割

CFGは、プロンプトへの遵守度を制御するパラメータです。

**CFG値の影響**
```
CFG = 1.0:
- プロンプト無視
- ほぼランダムな生成
- 用途: 実験的

CFG = 3.0-5.0:
- プロンプト緩く適用
- 創造的、多様性高い
- 用途: 芸術的表現

CFG = 7.0-8.0:
- バランスが良い（推奨）
- プロンプト適切に反映
- 用途: 一般的な生成

CFG = 10.0-12.0:
- プロンプト強く適用
- 詳細な指示に有効
- 用途: 特定の要求

CFG = 15.0以上:
- 過度な適用
- 色の飽和、アーティファクト
- 通常は非推奨
```

**MS-S1 Max推奨CFG設定**
```
SDXL Base:
- CFG: 7.0-8.0（標準）
- CFG: 6.0-7.0（創造的）
- CFG: 8.0-10.0（詳細指示）

SDXL Refiner:
- CFG: 6.0-7.0（Baseより低め推奨）
- Baseで生成した内容の微調整のため
```

## 1.7 MS-S1 Maxでの実践的設定

### 1.7.1 解像度別推奨設定

**SDXL 1024x1024（標準）**
```yaml
resolution: 1024x1024
batch_size: 1-2
steps: 25
sampler: dpmpp_2m_karras
scheduler: karras
cfg: 7.5
denoise: 1.0

メモリ使用量: 8-10GB
生成時間: 10-12秒/画像
品質: 高品質、バランス良好
```

**SDXL 1536x1536（高解像度）**
```yaml
resolution: 1536x1536
batch_size: 1
steps: 30
sampler: dpmpp_sde_karras
scheduler: karras
cfg: 7.0
denoise: 1.0

メモリ使用量: 14-18GB
生成時間: 26-30秒/画像
品質: 最高品質、詳細表現
```

**バッチ処理最適化**
```yaml
resolution: 1024x1024
batch_size: 4
steps: 20
sampler: euler_a
scheduler: normal
cfg: 7.0

メモリ使用量: 18-22GB
生成時間: 35-40秒/4画像
スループット: 約1.3秒/画像
用途: 大量生成、バリエーション作成
```

### 1.7.2 電力モードとパフォーマンス

MS-S1 MaxのBIOS設定によるパフォーマンス変化：

**Performance Mode (150W TDP)**
```
GPU性能: 100%
生成時間: 基準
発熱: 高（要冷却）
推奨: 連続作業時
ベンチマーク: 10秒 @ 1024x1024
```

**Balance Mode (130W TDP、推奨）**
```
GPU性能: 約90%
生成時間: +10%
発熱: 中程度
推奨: 通常使用
ベンチマーク: 11秒 @ 1024x1024
コストパフォーマンス: 最高
```

**Quiet Mode (100W TDP)**
```
GPU性能: 約70%
生成時間: +40%
発熱: 低
推奨: 静音重視、バックグラウンド生成
ベンチマーク: 14秒 @ 1024x1024
```

## 1.8 ComfyUIの利点と他ツールとの比較

### 1.8.1 AUTOMATIC1111 WebUIとの比較

```
AUTOMATIC1111 WebUI:
✅ ユーザーフレンドリー
✅ 豊富な拡張機能
✅ 大規模コミュニティ
❌ 柔軟性に制限
❌ メモリ効率やや劣る
❌ 複雑なワークフロー困難

ComfyUI:
✅ 極めて柔軟なワークフロー
✅ 優れたメモリ管理
✅ 高度な制御が可能
✅ カスタムノード開発容易
❌ 学習曲線やや急
❌ UIがやや技術的
```

### 1.8.2 用途別推奨

```
ComfyUI推奨ケース:
- 複雑なワークフロー構築
- カスタム処理パイプライン
- メモリ効率重視
- 再現性が重要
- 研究・開発用途

AUTOMATIC1111推奨ケース:
- 初めてのStable Diffusion
- シンプルな画像生成
- 豊富な拡張機能利用
- コミュニティレシピ活用
```

## 1.9 次章への準備

次章では、MS-S1 MaxへのComfyUI実際のインストール手順を詳しく解説します。

**学習ポイント**
```
✅ ROCm 6.2+のインストール
✅ PyTorch ROCm版のセットアップ
✅ ComfyUIのクローンと依存関係
✅ SDXLモデルのダウンロード
✅ 初回起動と動作確認
✅ トラブルシューティング
```

## 1.10 本章のまとめ

本章で学んだ内容：

**ComfyUI基礎**
- ノードベースのワークフローシステム
- 視覚的プログラミング環境
- 柔軟性とモジュール性

**Stable Diffusion技術**
- 潜在拡散モデルの仕組み
- SDXLアーキテクチャ
- 主要コンポーネント（CLIP、U-Net、VAE）

**MS-S1 Max最適化**
- 128GBメモリの効果的活用
- RDNA 3.5 GPUのパフォーマンス
- ROCm設定とチューニング

**実践的パラメータ**
- サンプラーとスケジューラの選択
- CFG値の調整
- 解像度別推奨設定

次章では実際のインストール手順を進めていきます。

---

**推奨リソース**
- ComfyUI公式GitHub: https://github.com/comfyanonymous/ComfyUI
- ComfyUI Wiki: https://comfyui-wiki.com/
- AMD ROCm公式: https://rocm.docs.amd.com/
- SDXL論文: https://arxiv.org/abs/2307.01952

