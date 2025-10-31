# 第2章: ComfyUIのインストールとROCm設定

## 2.1 システム要件の確認

### 2.1.1 MS-S1 Max環境の前提条件

ComfyUIをMS-S1 Maxで実行するための要件を確認します。

**ハードウェア要件**
```
必須:
- AMD Ryzen AI Max+ 395
- Radeon 8060S (RDNA 3.5)
- メモリ: 128GB LPDDR5X-8000
- ストレージ: 100GB以上の空き容量

推奨:
- SSD/NVMe: モデル読み込み高速化
- 冷却: 適切なエアフロー（TDP 130-150W）
```

**ソフトウェア要件**
```
OS: Ubuntu 22.04 LTS または 24.04 LTS
カーネル: 5.15以降（6.x推奨）
Python: 3.10, 3.11, 3.12
ROCm: 6.2以降（6.4推奨、7.0対応）
Git: バージョン管理用
```

### 2.1.2 対応ROCmバージョン

**ROCmバージョン選択ガイド**
```
ROCm 6.2.0:
- 安定性: ★★★★☆
- 性能: ★★★★☆
- 推奨度: ★★★☆☆
- 備考: 安定版、広くテスト済み

ROCm 6.4.2:
- 安定性: ★★★★★
- 性能: ★★★★★
- 推奨度: ★★★★★
- 備考: 最も推奨（2025年1月時点）

ROCm 7.0:
- 安定性: ★★★☆☆
- 性能: ★★★★★
- 推奨度: ★★★☆☆
- 備考: 最新版、実験的機能あり
```

**MS-S1 Max推奨構成**
```bash
# 推奨バージョン
OS: Ubuntu 24.04 LTS
ROCm: 6.4.2
PyTorch: 2.6.0+rocm6.4
Python: 3.11

# 理由
- Ubuntu 24.04: カーネル6.x、最新ドライバ対応
- ROCm 6.4.2: 安定性と性能のバランス
- PyTorch 2.6.0: ROCm 6.4完全対応
```

## 2.2 ROCmのインストール

### 2.2.1 システム準備

**既存ドライバの削除**
```bash
# AMD GPUドライバの確認
lspci | grep VGA
# 出力例:
# 00:02.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Device 1900

# 既存のROCmパッケージ削除（クリーンインストール時）
sudo apt autoremove --purge rocm-* amdgpu-*
sudo apt autoremove --purge hip-* hsa-*

# システムアップデート
sudo apt update && sudo apt upgrade -y
```

**必要な依存関係のインストール**
```bash
# ビルドツールとユーティリティ
sudo apt install -y \
    build-essential \
    cmake \
    git \
    wget \
    curl \
    software-properties-common \
    python3-pip \
    python3-venv \
    libstdc++-12-dev \
    libnuma-dev

# カーネルヘッダー（ドライバビルド用）
sudo apt install -y linux-headers-$(uname -r)
```

### 2.2.2 ROCm 6.4.2のインストール（推奨）

**AMDリポジトリの追加**
```bash
# Ubuntu 24.04 (Noble)の場合
wget https://repo.radeon.com/amdgpu-install/6.4.2/ubuntu/noble/amdgpu-install_6.4.60402-1_all.deb

# Ubuntu 22.04 (Jammy)の場合
# wget https://repo.radeon.com/amdgpu-install/6.4.2/ubuntu/jammy/amdgpu-install_6.4.60402-1_all.deb

# パッケージのインストール
sudo dpkg -i amdgpu-install_6.4.60402-1_all.deb
sudo apt update
```

**ROCmフルスタックのインストール**
```bash
# ROCm開発環境のインストール
sudo amdgpu-install --usecase=rocm,graphics \
    --vulkan=pro \
    --opencl=rocr,legacy

# インストール確認
rocm-smi --showproductname
# 出力例: GPU[0] : Card series: Radeon Graphics
# 出力例: GPU[0] : Card model: 0x1900

# ROCmバージョン確認
apt list --installed | grep rocm
```

**環境変数の設定**
```bash
# ~/.bashrcに追加
cat >> ~/.bashrc << 'EOF'

# ROCm環境変数
export ROCM_HOME=/opt/rocm
export PATH=$ROCM_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ROCM_HOME/lib:$LD_LIBRARY_PATH

# MS-S1 Max (RDNA 3.5) 専用設定
export HSA_OVERRIDE_GFX_VERSION=11.0.0
export PYTORCH_ROCM_ARCH=gfx1100
export ROC_ENABLE_PRE_VEGA=0
export HSA_ENABLE_SDMA=1

# GPU最適化
export GPU_MAX_ALLOC_PERCENT=95
export AMD_DIRECT_RENDERING=1

# Vulkan最適化（画像生成向け）
export RADV_PERFTEST=gpl,nggc
export ACO_DEBUG=validateir,validatera

EOF

# 設定を反映
source ~/.bashrc
```

### 2.2.3 ユーザー権限の設定

**renderグループへの追加**
```bash
# 現在のユーザーをrenderとvideoグループに追加
sudo usermod -a -G render,video $USER

# グループ確認
groups
# 出力例: username adm cdrom sudo dip plugdev render video

# 注意: グループ変更を反映するには再ログインが必要
# または以下のコマンドで一時的に反映
newgrp render
```

**デバイスアクセスの確認**
```bash
# GPUデバイスの確認
ls -l /dev/kfd /dev/dri/render*

# 出力例:
# crw-rw----+ 1 root render 510, 0 Jan  1 09:00 /dev/kfd
# crw-rw----+ 1 root render 226, 128 Jan  1 09:00 /dev/dri/renderD128

# rocm-smiでGPU情報表示
rocm-smi
```

## 2.3 PyTorch (ROCm版) のインストール

### 2.3.1 Python仮想環境の作成

**venv環境の構築**
```bash
# ComfyUI用ディレクトリの作成
mkdir -p ~/ai-tools
cd ~/ai-tools

# Python 3.11仮想環境の作成
python3.11 -m venv comfyui_env

# 仮想環境の有効化
source comfyui_env/bin/activate

# pip, setuptools, wheelの更新
pip install --upgrade pip setuptools wheel
```

### 2.3.2 PyTorch ROCm版のインストール

**ROCm 6.4用PyTorchインストール（推奨）**
```bash
# AMD公式リポジトリからのインストール（推奨）
pip install torch torchvision torchaudio \
    --index-url https://download.pytorch.org/whl/rocm6.4

# バージョン確認
python -c "import torch; print(f'PyTorch: {torch.__version__}')"
# 出力例: PyTorch: 2.6.0+rocm6.4

python -c "import torch; print(f'CUDA Available: {torch.cuda.is_available()}')"
# 出力例: CUDA Available: True (ROCmはCUDA APIエミュレート)

python -c "import torch; print(f'Device: {torch.cuda.get_device_name(0)}')"
# 出力例: Device: AMD Radeon Graphics
```

**PyTorch動作確認スクリプト**
```python
# test_pytorch_rocm.py
import torch

print("=" * 50)
print("PyTorch ROCm環境テスト")
print("=" * 50)

# バージョン情報
print(f"PyTorch Version: {torch.__version__}")
print(f"ROCm Available: {torch.cuda.is_available()}")

if torch.cuda.is_available():
    print(f"GPU Count: {torch.cuda.device_count()}")
    print(f"Current Device: {torch.cuda.current_device()}")
    print(f"Device Name: {torch.cuda.get_device_name(0)}")

    # メモリ情報
    print(f"Total Memory: {torch.cuda.get_device_properties(0).total_memory / 1e9:.2f} GB")

    # 簡単な計算テスト
    x = torch.rand(1000, 1000).cuda()
    y = torch.rand(1000, 1000).cuda()
    z = torch.matmul(x, y)
    print(f"GPU Computation Test: {'Success' if z.shape == (1000, 1000) else 'Failed'}")

    # HIP情報
    print(f"HIP Version: {torch.version.hip}")
else:
    print("⚠️ GPUが検出されません！")
```

**実行とトラブルシューティング**
```bash
# テストスクリプトの実行
python test_pytorch_rocm.py

# 期待される出力:
# ==================================================
# PyTorch ROCm環境テスト
# ==================================================
# PyTorch Version: 2.6.0+rocm6.4
# ROCm Available: True
# GPU Count: 1
# Current Device: 0
# Device Name: AMD Radeon Graphics
# Total Memory: 24.00 GB (統合メモリから割り当て)
# GPU Computation Test: Success
# HIP Version: 6.4.60402
```

### 2.3.3 代替インストール方法

**ROCm 7.0使用時（最新版）**
```bash
# PyTorch Nightly版（ROCm 7.0）
pip install --pre torch torchvision torchaudio \
    --index-url https://download.pytorch.org/whl/nightly/rocm7.0/
```

**ROCm 6.2使用時（安定版）**
```bash
# PyTorch ROCm 6.2版
pip install torch torchvision torchaudio \
    --index-url https://download.pytorch.org/whl/rocm6.2
```

## 2.4 ComfyUIのインストール

### 2.4.1 ComfyUIのクローン

**GitHubからのクローン**
```bash
# 仮想環境が有効化されていることを確認
source ~/ai-tools/comfyui_env/bin/activate

# ComfyUIのクローン
cd ~/ai-tools
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI

# 最新の安定版タグを確認（オプション）
git tag --list
git checkout tags/v0.2.8  # 例: 特定バージョンを使用する場合
```

### 2.4.2 依存関係のインストール

**requirements.txtの編集**
```bash
# 元のrequirements.txtをバックアップ
cp requirements.txt requirements.txt.bak

# PyTorch関連パッケージをコメントアウト
# (既にROCm版PyTorchをインストール済みのため)
sed -i 's/^torch/#torch/g' requirements.txt
sed -i 's/^torchvision/#torchvision/g' requirements.txt
sed -i 's/^torchaudio/#torchaudio/g' requirements.txt
sed -i 's/^torchsde/#torchsde/g' requirements.txt

# 編集後のrequirements.txtを確認
cat requirements.txt
```

**依存パッケージのインストール**
```bash
# 残りの依存関係をインストール
pip install -r requirements.txt

# torchsdeを個別にインストール（ROCm互換版）
pip install torchsde

# 追加の有用なパッケージ
pip install \
    opencv-python \
    opencv-contrib-python \
    scikit-image \
    scipy \
    numba \
    matplotlib
```

### 2.4.3 ディレクトリ構造の確認

**ComfyUIディレクトリ構成**
```
ComfyUI/
├── comfy/              # コアライブラリ
├── custom_nodes/       # カスタムノード
├── input/              # 入力画像
├── models/             # モデル格納ディレクトリ
│   ├── checkpoints/    # SDXLモデル（.safetensors）
│   ├── clip/           # CLIPモデル
│   ├── clip_vision/    # CLIP Visionモデル
│   ├── controlnet/     # ControlNetモデル
│   ├── embeddings/     # テキスト埋め込み
│   ├── loras/          # LoRAモデル
│   ├── upscale_models/ # アップスケールモデル
│   └── vae/            # VAEモデル
├── output/             # 生成画像出力先
├── web/                # Webインターフェース
├── main.py             # メインスクリプト
└── requirements.txt    # 依存関係リスト
```

**モデルディレクトリの準備**
```bash
cd ~/ai-tools/ComfyUI

# 必要なディレクトリが存在することを確認
ls -la models/

# 出力ディレクトリのパーミッション確認
chmod 755 output/
```

## 2.5 SDXLモデルのダウンロード

### 2.5.1 Hugging Faceからのダウンロード

**SDXL Base 1.0モデル**
```bash
cd ~/ai-tools/ComfyUI/models/checkpoints

# Hugging Face CLIのインストール
pip install huggingface_hub

# SDXL Base 1.0のダウンロード（約6.6GB）
huggingface-cli download \
    stabilityai/stable-diffusion-xl-base-1.0 \
    sd_xl_base_1.0.safetensors \
    --local-dir . \
    --local-dir-use-symlinks False

# または直接wgetでダウンロード
wget https://huggingface.co/stabilityai/stable-diffusion-xl-base-1.0/resolve/main/sd_xl_base_1.0.safetensors
```

**SDXL Refiner 1.0モデル（オプション）**
```bash
# SDXL Refiner（約6.1GB）
huggingface-cli download \
    stabilityai/stable-diffusion-xl-refiner-1.0 \
    sd_xl_refiner_1.0.safetensors \
    --local-dir . \
    --local-dir-use-symlinks False
```

### 2.5.2 推奨モデルリスト

**必須モデル**
```
checkpoints/sd_xl_base_1.0.safetensors (6.6GB)
- SDXL標準モデル
- 1024x1024ネイティブ生成

vae/sdxl_vae.safetensors (335MB)（オプション）
- SDXL専用VAE
- 色味改善
- URL: https://huggingface.co/stabilityai/sdxl-vae
```

**推奨追加モデル**
```
upscale_models/RealESRGAN_x4plus_anime_6B.pth
- アニメ系アップスケール
- 4倍拡大

upscale_models/RealESRGAN_x4plus.pth
- 写実系アップスケール
- 4倍拡大
```

### 2.5.3 モデルダウンロードスクリプト

**自動ダウンロードスクリプト**
```bash
#!/bin/bash
# download_models.sh

set -e

COMFYUI_DIR=~/ai-tools/ComfyUI
CHECKPOINTS_DIR=$COMFYUI_DIR/models/checkpoints
VAE_DIR=$COMFYUI_DIR/models/vae
UPSCALE_DIR=$COMFYUI_DIR/models/upscale_models

echo "ComfyUI基本モデルのダウンロード"
echo "================================"

# Checkpointsディレクトリ
cd $CHECKPOINTS_DIR
echo "SDXL Base 1.0をダウンロード中..."
if [ ! -f "sd_xl_base_1.0.safetensors" ]; then
    wget -q --show-progress \
        https://huggingface.co/stabilityai/stable-diffusion-xl-base-1.0/resolve/main/sd_xl_base_1.0.safetensors
    echo "✓ SDXL Base完了"
else
    echo "✓ SDXL Base既存"
fi

# VAEディレクトリ
cd $VAE_DIR
echo "SDXL VAEをダウンロード中..."
if [ ! -f "sdxl_vae.safetensors" ]; then
    wget -q --show-progress \
        https://huggingface.co/stabilityai/sdxl-vae/resolve/main/sdxl_vae.safetensors
    echo "✓ SDXL VAE完了"
else
    echo "✓ SDXL VAE既存"
fi

# Upscaleモデル
cd $UPSCALE_DIR
echo "RealESRGAN x4をダウンロード中..."
if [ ! -f "RealESRGAN_x4plus.pth" ]; then
    wget -q --show-progress \
        https://github.com/xinntao/Real-ESRGAN/releases/download/v0.1.0/RealESRGAN_x4plus.pth
    echo "✓ RealESRGAN完了"
else
    echo "✓ RealESRGAN既存"
fi

echo ""
echo "ダウンロード完了！"
echo "総容量: 約7GB"
```

**実行**
```bash
chmod +x download_models.sh
./download_models.sh
```

## 2.6 初回起動と動作確認

### 2.6.1 ComfyUIの起動

**基本起動コマンド**
```bash
# 仮想環境の有効化
source ~/ai-tools/comfyui_env/bin/activate

# ComfyUIディレクトリへ移動
cd ~/ai-tools/ComfyUI

# 起動
python main.py

# 期待される出力:
# Total VRAM 24576 MB, total RAM 131072 MB
# Set vram state to: NORMAL_VRAM
# Device: cuda:0 AMD Radeon Graphics : native
# VAE dtype: torch.float16
# Using pytorch cross attention
# Starting server
# To see the GUI go to: http://127.0.0.1:8188
```

**起動オプション**
```bash
# VRAM最適化（低メモリ時）
python main.py --lowvram

# 高精度モード（128GBメモリを活用）
python main.py --normalvram --highvram

# ポート変更
python main.py --port 8189

# LAN内アクセス許可
python main.py --listen 0.0.0.0

# MS-S1 Max推奨起動コマンド
python main.py --normalvram --listen 127.0.0.1 --port 8188
```

### 2.6.2 Webインターフェースへのアクセス

**ブラウザでアクセス**
```
URL: http://127.0.0.1:8188

推奨ブラウザ:
- Chrome/Chromium（最も安定）
- Firefox
- Edge

初回アクセス時:
1. デフォルトワークフローが表示される
2. Load Checkpointノードでsd_xl_base_1.0を選択
3. Queueボタンで生成開始
```

### 2.6.3 初回生成テスト

**シンプルなText-to-Image**
```
1. デフォルトワークフローを使用
2. CLIP Text Encode (Positive)に入力:
   "a beautiful mountain landscape, 4k, highly detailed"
3. CLIP Text Encode (Negative)に入力:
   "low quality, blurry, distorted"
4. Empty Latent Image:
   - width: 1024
   - height: 1024
   - batch_size: 1
5. KSampler設定:
   - seed: 42
   - steps: 25
   - cfg: 7.5
   - sampler_name: dpmpp_2m_karras
   - scheduler: karras
6. "Queue Prompt"ボタンをクリック
7. 10-15秒で生成完了
```

## 2.7 MS-S1 Max最適化設定

### 2.7.1 環境変数の最適化

**~/ai-tools/comfyui_env.sh作成**
```bash
#!/bin/bash
# comfyui_env.sh - MS-S1 Max最適化環境変数

# ROCm基本設定
export ROCM_HOME=/opt/rocm
export PATH=$ROCM_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ROCM_HOME/lib:$LD_LIBRARY_PATH

# RDNA 3.5 (gfx1100) 設定
export HSA_OVERRIDE_GFX_VERSION=11.0.0
export PYTORCH_ROCM_ARCH=gfx1100

# GPU割り当て最適化（128GBメモリを活用）
export GPU_MAX_ALLOC_PERCENT=95
export GPU_SINGLE_ALLOC_PERCENT=90

# HIP/ROCm最適化
export HSA_ENABLE_SDMA=1
export ROC_ENABLE_PRE_VEGA=0
export AMD_DIRECT_RENDERING=1

# PyTorch最適化
export PYTORCH_HIP_ALLOC_CONF=expandable_segments:True
export TORCH_ALLOW_TF32_CUBLAS_OVERRIDE=1

# ComfyUI専用最適化
export COMFYUI_VRAM_MODE=normalvram
export COMFYUI_FORCE_FP16=1

# Vulkan最適化
export RADV_PERFTEST=gpl,nggc
export ACO_DEBUG=validateir,validatera

# ログレベル（デバッグ時）
# export ROCM_LOG_LEVEL=3
# export AMD_LOG_LEVEL=3

echo "MS-S1 Max環境変数設定完了"
```

**使用方法**
```bash
# スクリプトに実行権限
chmod +x ~/ai-tools/comfyui_env.sh

# ComfyUI起動前に読み込み
source ~/ai-tools/comfyui_env.sh
source ~/ai-tools/comfyui_env/bin/activate
cd ~/ai-tools/ComfyUI
python main.py
```

### 2.7.2 起動スクリプトの作成

**~/ai-tools/start_comfyui.sh**
```bash
#!/bin/bash
# start_comfyui.sh - ComfyUI起動スクリプト

set -e

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
COMFYUI_DIR="$SCRIPT_DIR/ComfyUI"
VENV_DIR="$SCRIPT_DIR/comfyui_env"

echo "ComfyUI起動スクリプト (MS-S1 Max最適化)"
echo "========================================"

# 環境変数の読み込み
if [ -f "$SCRIPT_DIR/comfyui_env.sh" ]; then
    source "$SCRIPT_DIR/comfyui_env.sh"
else
    echo "警告: comfyui_env.shが見つかりません"
fi

# 仮想環境の有効化
if [ -d "$VENV_DIR" ]; then
    source "$VENV_DIR/bin/activate"
    echo "✓ 仮想環境を有効化"
else
    echo "エラー: 仮想環境が見つかりません"
    exit 1
fi

# ComfyUIディレクトリへ移動
cd "$COMFYUI_DIR"

# GPU確認
echo ""
echo "GPU情報:"
rocm-smi --showproductname 2>/dev/null || echo "  rocm-smi未検出"

# PyTorch確認
python -c "import torch; print(f'  PyTorch: {torch.__version__}')"
python -c "import torch; print(f'  GPU Available: {torch.cuda.is_available()}')"

echo ""
echo "ComfyUIを起動します..."
echo "URL: http://127.0.0.1:8188"
echo ""

# ComfyUI起動（MS-S1 Max最適化設定）
exec python main.py \
    --normalvram \
    --listen 127.0.0.1 \
    --port 8188 \
    "$@"
```

**使用方法**
```bash
chmod +x ~/ai-tools/start_comfyui.sh

# 起動
~/ai-tools/start_comfyui.sh

# オプション付き起動例
~/ai-tools/start_comfyui.sh --preview-method auto
```

## 2.8 カスタムノードのインストール

### 2.8.1 ComfyUI Managerのインストール

ComfyUI Managerは、カスタムノードの管理を簡単にするツールです。

```bash
cd ~/ai-tools/ComfyUI/custom_nodes

# ComfyUI-Managerのクローン
git clone https://github.com/ltdrdata/ComfyUI-Manager.git

# 依存関係のインストール
cd ComfyUI-Manager
pip install -r requirements.txt

# ComfyUI再起動後、WebUIに"Manager"ボタンが表示される
```

### 2.8.2 推奨カスタムノード

**画質向上系**
```bash
cd ~/ai-tools/ComfyUI/custom_nodes

# ComfyUI-Impact-Pack（高度な後処理）
git clone https://github.com/ltdrdata/ComfyUI-Impact-Pack.git
cd ComfyUI-Impact-Pack && pip install -r requirements.txt && cd ..

# Ultimate SD Upscale（タイル式アップスケール）
git clone https://github.com/ssitu/ComfyUI_UltimateSDUpscale.git
```

**ワークフロー拡張系**
```bash
# rgthree's ComfyUI Nodes（UI拡張）
git clone https://github.com/rgthree/rgthree-comfy.git
cd rgthree-comfy && pip install -r requirements.txt && cd ..

# Efficiency Nodes（効率化ノード）
git clone https://github.com/jags111/efficiency-nodes-comfyui.git
```

## 2.9 トラブルシューティング

### 2.9.1 一般的な問題と解決策

**問題1: GPUが認識されない**
```bash
# 症状
# "CUDA Available: False" または "Using CPU"

# 確認1: ROCmインストール状態
rocm-smi

# 確認2: ユーザーグループ
groups | grep render

# 解決策: renderグループ追加と再ログイン
sudo usermod -a -G render,video $USER
# その後、ログアウト→ログイン

# 確認3: HSA_OVERRIDE_GFX_VERSION設定
echo $HSA_OVERRIDE_GFX_VERSION
# 出力: 11.0.0 であるべき
```

**問題2: メモリ不足エラー**
```bash
# 症状
# "RuntimeError: HIP out of memory"

# 解決策1: lowvramモード
python main.py --lowvram

# 解決策2: GPU割り当て調整
export GPU_MAX_ALLOC_PERCENT=80
python main.py

# 解決策3: バッチサイズ削減
# ワークフローのbatch_sizeを1に設定
```

**問題3: 生成が非常に遅い**
```bash
# 原因チェック1: CPUで動作していないか確認
# ComfyUI起動時のログを確認
# "Device: cpu" → GPU未使用
# "Device: cuda:0" → GPU使用中（正常）

# 原因チェック2: 環境変数確認
env | grep HSA_OVERRIDE_GFX_VERSION

# 解決策: 環境変数を再設定
source ~/ai-tools/comfyui_env.sh
```

### 2.9.2 パフォーマンスベンチマーク

**ベンチマークスクリプト**
```python
# benchmark.py
import torch
import time

print("ComfyUI パフォーマンステスト")
print("=" * 50)

# GPU情報
print(f"Device: {torch.cuda.get_device_name(0)}")
print(f"Memory: {torch.cuda.get_device_properties(0).total_memory / 1e9:.1f} GB")

# 行列乗算テスト
sizes = [1000, 2000, 4000, 8000]
for size in sizes:
    x = torch.rand(size, size).cuda()
    y = torch.rand(size, size).cuda()

    # ウォームアップ
    _ = torch.matmul(x, y)
    torch.cuda.synchronize()

    # 計測
    start = time.time()
    z = torch.matmul(x, y)
    torch.cuda.synchronize()
    elapsed = time.time() - start

    gflops = (2 * size**3) / elapsed / 1e9
    print(f"{size}x{size}: {elapsed*1000:.1f}ms ({gflops:.1f} GFLOPS)")

print("\nベンチマーク完了")
```

**期待される結果（MS-S1 Max）**
```
Device: AMD Radeon Graphics
Memory: 24.0 GB
1000x1000: 2.3ms (869 GFLOPS)
2000x2000: 8.1ms (1975 GFLOPS)
4000x4000: 45.2ms (2832 GFLOPS)
8000x8000: 312.5ms (3276 GFLOPS)
```

## 2.10 本章のまとめ

本章で実施した内容：

**環境構築**
- Ubuntu 24.04 + ROCm 6.4.2のインストール
- PyTorch 2.6.0 (ROCm版) のセットアップ
- MS-S1 Max専用環境変数の設定

**ComfyUIインストール**
- GitHubからのクローン
- 依存関係のインストール
- SDXL Base/Refinerモデルのダウンロード

**最適化設定**
- HSA_OVERRIDE_GFX_VERSION=11.0.0（RDNA 3.5対応）
- GPU割り当て最適化
- 起動スクリプトの作成

**動作確認**
- 初回起動とWebUI確認
- Text-to-Image生成テスト
- パフォーマンスベンチマーク

次章では、SDXLの基礎とプロンプト技術を詳しく学びます。

---

**参考リソース**
- ROCm公式ドキュメント: https://rocm.docs.amd.com/
- ComfyUI GitHub: https://github.com/comfyanonymous/ComfyUI
- PyTorch ROCm: https://pytorch.org/get-started/locally/
- AMD GPUOpen: https://gpuopen.com/

