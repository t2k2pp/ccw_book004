# 第2章:インストールと環境構築

## 2.1 システム要件

### MS-S1 Max 推奨構成
```
OS: Ubuntu 22.04 LTS / 24.04 LTS
Python: 3.10 / 3.11
CUDA/ROCm: ROCm 6.1+
Memory: 128GB (MS-S1 Max)
Storage: 100GB+ 空き容量
```

## 2.2 依存関係のインストール

```bash
# システムパッケージ
sudo apt update
sudo apt install -y python3-pip python3-venv git build-essential

# ROCm（第2部を参照）
# 既にインストール済みの場合はスキップ
```

## 2.3 Text Generation WebUIのインストール

### 推奨方法（自動インストール）

```bash
# リポジトリをクローン
cd ~
git clone https://github.com/oobabooga/text-generation-webui
cd text-generation-webui

# AMD GPU用インストールスクリプト実行
./start_linux.sh

# 初回実行時、依存関係を自動インストール
# ROCmサポートを選択
```

### 手動インストール（推奨・MS-S1 Max最適化）

```bash
cd ~/text-generation-webui

# 仮想環境作成
python3 -m venv venv
source venv/bin/activate

# PyTorch (ROCm版)
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm6.2

# 基本依存関係
pip install -r requirements_amd.txt

# ExLlamaV2 (ROCm版)
pip install exllamav2 --no-build-isolation

# Gradio and dependencies
pip install gradio==3.50.2
```

## 2.4 環境変数設定

```bash
nano ~/.bashrc

# 追加
export HSA_OVERRIDE_GFX_VERSION=11.0.0
export ROCM_HOME=/opt/rocm
export PYTORCH_ROCM_ARCH="gfx1100"

source ~/.bashrc
```

## 2.5 初回起動

```bash
cd ~/text-generation-webui
source venv/bin/activate

# WebUI起動
python server.py --listen --api

# ブラウザでアクセス
# http://localhost:7860
```

## 2.6 モデルのダウンロード

### Web UI経由

```
1. ブラウザでhttp://localhost:7860を開く
2. "Model" タブをクリック
3. "Download model or LoRA" セクション
4. モデル名を入力（例: TheBloke/Llama-2-7B-GGUF）
5. ファイルを選択してダウンロード
```

### コマンドライン

```bash
cd ~/text-generation-webui
python download-model.py TheBloke/Llama-2-7B-Chat-GGUF llama-2-7b-chat.Q4_K_M.gguf
```

## 2.7 起動オプション

```bash
# 基本起動
python server.py

# リモートアクセス許可
python server.py --listen

# API有効化
python server.py --api

# 特定のモデルで起動
python server.py --model llama-2-7b-chat.Q4_K_M

# ExLlamaV2ローダー指定
python server.py --loader exllamav2

# 複数オプション
python server.py --listen --api --loader exllamav2 --gpu-memory 96
```

## 2.8 systemdサービス化

```bash
sudo nano /etc/systemd/system/textgen.service
```

```ini
[Unit]
Description=Text Generation WebUI
After=network.target

[Service]
Type=simple
User=username
WorkingDirectory=/home/username/text-generation-webui
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
ExecStart=/home/username/text-generation-webui/venv/bin/python server.py --listen --api
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable textgen
sudo systemctl start textgen
```

## 2.9 本章のまとめ

✅ システム要件確認
✅ 依存関係インストール
✅ Text Generation WebUI セットアップ
✅ 環境変数設定
✅ 初回起動とモデルダウンロード

---

**前章へ**: [第1章 はじめに](chapter01_introduction.md)
**次章へ**: [第3章 ROCm設定とExLlamaV2最適化](chapter03_rocm_exllama.md)
