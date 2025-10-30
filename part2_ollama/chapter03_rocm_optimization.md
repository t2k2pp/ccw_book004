# 第3章:ROCm設定とAMD GPU最適化

## 3.1 ROCmの基礎知識

### 3.1.1 ROCmとは

**ROCm (Radeon Open Compute)** は、AMDが開発したオープンソースのGPUコンピューティングプラットフォームです。

```
ROCm の構成要素:

┌─────────────────────────────────────┐
│   アプリケーション層                 │
│   (Ollama, PyTorch, TensorFlow)     │
├─────────────────────────────────────┤
│   ライブラリ層                       │
│   (rocBLAS, MIOpen, hipBLAS)        │
├─────────────────────────────────────┤
│   ランタイム層                       │
│   (HIP, HSA)                        │
├─────────────────────────────────────┤
│   ドライバ層                         │
│   (amdgpu, amdkfd)                  │
├─────────────────────────────────────┤
│   ハードウェア                       │
│   (Radeon 8060S - RDNA 3.5)         │
└─────────────────────────────────────┘
```

### 3.1.2 MS-S1 Max での ROCm 要件

```bash
# 推奨バージョン
ROCm: 6.1.0 以降（6.3.0 推奨）
Kernel: 6.5 以降
Ubuntu: 22.04 LTS / 24.04 LTS
```

### 3.1.3 現在の環境確認

```bash
# ROCm バージョン
rocm-smi --version

# カーネルバージョン
uname -r

# GPUドライババージョン
modinfo amdgpu | grep ^version
```

## 3.2 ROCm 完全インストール

### 3.2.1 既存ROCmの削除（クリーンインストール）

```bash
# 既存のROCmを完全削除
sudo apt purge -y rocm-* hip-* miopen-*
sudo apt autoremove -y
sudo apt autoclean

# 設定ファイルも削除
sudo rm -rf /opt/rocm*
sudo rm -rf ~/.cache/hip
```

### 3.2.2 ROCm 6.3 のインストール

```bash
# AMDのGPGキー追加
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | \
    gpg --dearmor | sudo tee /etc/apt/keyrings/rocm.gpg > /dev/null

# ROCmリポジトリ追加（Ubuntu 22.04の場合）
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/6.3 jammy main" \
    | sudo tee /etc/apt/sources.list.d/rocm.list

# Ubuntu 24.04の場合
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/6.3 noble main" \
    | sudo tee /etc/apt/sources.list.d/rocm.list

# 優先度設定
echo -e 'Package: *\nPin: release o=repo.radeon.com\nPin-Priority: 600' \
    | sudo tee /etc/apt/preferences.d/rocm-pin-600

# リポジトリ更新
sudo apt update
```

```bash
# ROCm完全版インストール
sudo apt install -y rocm-hip-sdk rocm-libs

# Ollama用に最低限必要なパッケージ
sudo apt install -y \
    rocm-hip-runtime \
    rocm-smi-lib \
    hip-runtime-amd \
    rocm-core

# 開発ツール（オプション）
sudo apt install -y \
    rocm-dev \
    rocm-utils \
    rocminfo \
    rocm-bandwidth-test
```

### 3.2.3 ユーザー権限設定

```bash
# render と video グループに追加
sudo usermod -a -G render $USER
sudo usermod -a -G video $USER

# 確認
groups $USER

# 出力例
username : username adm cdrom sudo dip plugdev render video
```

**⚠️ 重要:** グループ追加後は、再ログインまたは再起動が必要です。

```bash
# 再ログイン
exit
# 再度ログイン

# または再起動
sudo reboot
```

## 3.3 AMD GPU 設定の最適化

### 3.3.1 環境変数の設定

```bash
# ~/.bashrc に追加
nano ~/.bashrc

# 以下を末尾に追加
# ========== ROCm Configuration ==========
# ROCm Path
export PATH=/opt/rocm/bin:$PATH
export LD_LIBRARY_PATH=/opt/rocm/lib:$LD_LIBRARY_PATH

# GPU Target (RDNA 3.5 → gfx1100)
export HSA_OVERRIDE_GFX_VERSION=11.0.0

# HIP Settings
export HIP_VISIBLE_DEVICES=0
export HIP_LAUNCH_BLOCKING=0

# Performance
export ROCM_HOME=/opt/rocm
export GPU_MAX_HEAP_SIZE=100
export GPU_MAX_ALLOC_PERCENT=100

# Ollama Specific
export OLLAMA_FLASH_ATTENTION=1
export OLLAMA_NUM_PARALLEL=2
# ========================================
```

```bash
# 設定を反映
source ~/.bashrc

# 確認
echo $HSA_OVERRIDE_GFX_VERSION  # 11.0.0 と表示されるべき
```

### 3.3.2 システムワイド環境変数

Ollamaサービス用にシステムワイドの環境変数を設定します。

```bash
# /etc/environment に追加
sudo nano /etc/environment

# 追加（既存行は残す）
HSA_OVERRIDE_GFX_VERSION=11.0.0
ROCM_HOME=/opt/rocm
```

### 3.3.3 Ollama サービスの環境変数

```bash
# Ollamaサービス用環境変数
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo nano /etc/systemd/system/ollama.service.d/rocm.conf
```

**rocm.conf の内容:**
```ini
[Service]
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
Environment="ROCM_HOME=/opt/rocm"
Environment="PATH=/opt/rocm/bin:/usr/local/bin:/usr/bin:/bin"
Environment="LD_LIBRARY_PATH=/opt/rocm/lib"
Environment="HIP_VISIBLE_DEVICES=0"
Environment="GPU_MAX_HEAP_SIZE=100"
Environment="GPU_MAX_ALLOC_PERCENT=100"
Environment="OLLAMA_FLASH_ATTENTION=1"
```

```bash
# 設定を反映
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

## 3.4 GPU認識の確認

### 3.4.1 ROCm ツールでの確認

```bash
# ROCm システム情報
rocminfo | grep -A 10 "Agent"

# 期待される出力
Agent 2
  Name:                    gfx1100
  Uuid:                    GPU-XXXXXXXXXXXX
  Marketing Name:          AMD Radeon Graphics
  Vendor Name:             AMD
  Feature:                 KERNEL_DISPATCH
  Max Queue Size:          0(0x0)
  Queue Min Size:          0(0x0)
  Queue Type:              MULTI
```

```bash
# GPU一覧
rocm-smi --showproductname

# 出力例
GPU[0]  : Card series:    AMD Radeon Graphics
GPU[0]  : Card model:     0x1900
GPU[0]  : Card vendor:    Advanced Micro Devices, Inc. [AMD/ATI]
```

### 3.4.2 HIP 動作確認

```bash
# HIP バージョン
hipconfig --version

# HIP プラットフォーム
hipconfig --platform

# 出力: amd
```

**簡易テストプログラム:**
```bash
# テストファイル作成
nano hip_test.cpp
```

```cpp
#include <hip/hip_runtime.h>
#include <iostream>

int main() {
    int deviceCount;
    hipGetDeviceCount(&deviceCount);
    std::cout << "Number of HIP devices: " << deviceCount << std::endl;

    for (int i = 0; i < deviceCount; i++) {
        hipDeviceProp_t prop;
        hipGetDeviceProperties(&prop, i);
        std::cout << "Device " << i << ": " << prop.name << std::endl;
        std::cout << "  Compute Capability: " << prop.major << "." << prop.minor << std::endl;
        std::cout << "  Total Memory: " << prop.totalGlobalMem / (1024*1024*1024) << " GB" << std::endl;
    }
    return 0;
}
```

```bash
# コンパイルと実行
hipcc hip_test.cpp -o hip_test
./hip_test

# 期待される出力
Number of HIP devices: 1
Device 0: AMD Radeon Graphics
  Compute Capability: 11.0
  Total Memory: 96 GB
```

### 3.4.3 Ollama での GPU 認識確認

```bash
# OllamaログでGPU検出を確認
sudo journalctl -u ollama | grep -i gpu

# 期待される出力
Detected GPU: AMD Radeon Graphics (gfx1100)
Using ROCm backend
GPU Memory: 96GB available
```

```bash
# Ollamaの実行時ログ
ollama run qwen2.5:7b --verbose
```

## 3.5 パフォーマンスプロファイリング

### 3.5.1 GPU使用率のリアルタイム監視

```bash
# rocm-smiでの監視
watch -n 1 rocm-smi

# 詳細情報表示
watch -n 1 "rocm-smi --showmeminfo vram --showuse"
```

**推論時の期待値（7Bモデル）:**
```
GPU  Temp   AvgPwr  SCLK     MCLK     Fan   Perf  PwrCap  VRAM%  GPU%
0    65.0c  85.0W   2800Mhz  1000Mhz  55%   auto  120.0W  35%    88%
```

### 3.5.2 メモリ帯域幅テスト

```bash
# ROCm帯域幅テスト
/opt/rocm/bin/rocm_bandwidth_test

# 重要な出力
Unidirectional copy peak bandwidth GB/s:
Host to Device: 212.45
Device to Host: 215.32
Device to Device: 1024.78
```

**MS-S1 Maxでの期待値:**
- Host↔Device: 200-220 GB/s（LPDDR5X-8000の理論値256GB/sに近い）
- Device内部: 1000+ GB/s

### 3.5.3 推論ベンチマーク

```python
# benchmark_detailed.py
import ollama
import time
import statistics

def benchmark_model(model_name, prompt, num_runs=5):
    print(f"\n{'='*60}")
    print(f"Benchmarking: {model_name}")
    print(f"{'='*60}")

    speeds = []

    for i in range(num_runs):
        print(f"Run {i+1}/{num_runs}...", end=" ", flush=True)

        start = time.time()
        response = ollama.generate(
            model=model_name,
            prompt=prompt,
            options={"num_predict": 100}
        )
        elapsed = time.time() - start

        speed = 100 / elapsed
        speeds.append(speed)
        print(f"{speed:.1f} tokens/s")

    avg_speed = statistics.mean(speeds)
    std_dev = statistics.stdev(speeds) if len(speeds) > 1 else 0

    print(f"\nResults:")
    print(f"  Average: {avg_speed:.1f} tokens/s")
    print(f"  Std Dev: {std_dev:.1f} tokens/s")
    print(f"  Min: {min(speeds):.1f} tokens/s")
    print(f"  Max: {max(speeds):.1f} tokens/s")

if __name__ == "__main__":
    models = ["qwen2.5:7b", "qwen2.5:14b", "qwen2.5:32b"]
    prompt = "Explain quantum computing in simple terms."

    for model in models:
        try:
            benchmark_model(model, prompt)
        except Exception as e:
            print(f"Error with {model}: {e}")
```

```bash
# 実行
python3 benchmark_detailed.py
```

**MS-S1 Max での期待値:**
```
Benchmarking: qwen2.5:7b
Run 1/5... 42.3 tokens/s
Run 2/5... 43.1 tokens/s
Run 3/5... 42.8 tokens/s
Run 4/5... 42.5 tokens/s
Run 5/5... 43.0 tokens/s

Results:
  Average: 42.7 tokens/s
  Std Dev: 0.3 tokens/s
```

## 3.6 ROCm 最適化設定

### 3.6.1 カーネルパラメータ

```bash
# amdgpu カーネルモジュール設定
sudo nano /etc/modprobe.d/amdgpu.conf
```

**amdgpu.conf の推奨設定（MS-S1 Max）:**
```conf
# 基本設定
options amdgpu ppfeaturemask=0xffffffff
options amdgpu dpm=1
options amdgpu gpu_recovery=1

# メモリ設定
options amdgpu noretry=0
options amdgpu tmz=0

# パフォーマンス
options amdgpu aspm=0
options amdgpu runpm=0
```

**パラメータの説明:**

| パラメータ | 説明 | 推奨値 |
|-----------|------|--------|
| ppfeaturemask | PowerPlay機能マスク | 0xffffffff（全機能） |
| dpm | 動的電力管理 | 1（有効） |
| gpu_recovery | GPU hang時の回復 | 1（有効） |
| noretry | メモリアクセスリトライ | 0（リトライ有効） |
| aspm | Active State Power Management | 0（無効・安定性優先） |

```bash
# 設定を反映（initramfs再構築）
sudo update-initramfs -u -k all

# 再起動
sudo reboot
```

### 3.6.2 GPU電力・クロック設定

```bash
# 現在の電力設定確認
sudo rocm-smi --showpower
sudo rocm-smi --showclocks

# パフォーマンスレベル設定
sudo rocm-smi --setperflevel high

# 電力制限設定（デフォルト120W）
sudo rocm-smi --setpoweroverdrive 120
```

**MS-S1 Max の電力モード:**
```
Performance (160W): 最高性能、高温
Balance (130W):     推奨、バランス型
Quiet (110W):       静音、やや低速
```

### 3.6.3 メモリ管理の最適化

```bash
# システムスワップ設定（大容量メモリなので最小化）
sudo sysctl vm.swappiness=10

# 永続化
sudo nano /etc/sysctl.conf

# 追加
vm.swappiness=10
vm.vfs_cache_pressure=50
```

## 3.7 Flash Attention 最適化

### 3.7.1 Flash Attention とは

**Flash Attention** は、アテンション計算を高速化し、メモリ使用量を削減する技術です。

```
従来のアテンション:
  - メモリ使用量: O(N²)
  - 速度: 遅い

Flash Attention:
  - メモリ使用量: O(N)
  - 速度: 2-4倍高速
  - 長コンテキストで特に有効
```

### 3.7.2 Ollama での Flash Attention 有効化

```bash
# 環境変数で有効化
export OLLAMA_FLASH_ATTENTION=1

# Ollamaサービスに永続設定
sudo nano /etc/systemd/system/ollama.service.d/rocm.conf

# 追加
Environment="OLLAMA_FLASH_ATTENTION=1"
```

```bash
# 再起動
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### 3.7.3 効果の確認

```bash
# Flash Attention無効での実行時間測定
OLLAMA_FLASH_ATTENTION=0 time ollama run llama3.1:70b "Write a 500 word essay."

# Flash Attention有効での実行時間測定
OLLAMA_FLASH_ATTENTION=1 time ollama run llama3.1:70b "Write a 500 word essay."
```

**期待される改善（70Bモデル、32Kコンテキスト）:**
```
無効時: 45秒
有効時: 32秒（約30%高速化）
```

## 3.8 トラブルシューティング

### 3.8.1 GPU が認識されない

**症状:**
```bash
ollama run llama3.1
# CPU only と表示される
```

**診断手順:**

```bash
# 1. ROCm インストール確認
dpkg -l | grep rocm-core

# 2. 環境変数確認
echo $HSA_OVERRIDE_GFX_VERSION  # 11.0.0 であるべき

# 3. GPU デバイス確認
ls -l /dev/kfd /dev/dri/render*

# 4. 権限確認
groups | grep -E 'render|video'

# 5. ROCm 動作確認
rocminfo | grep -i gfx
```

**解決策:**

```bash
# 権限の再設定
sudo usermod -a -G render,video $USER

# 環境変数の永続化
echo 'export HSA_OVERRIDE_GFX_VERSION=11.0.0' | sudo tee -a /etc/environment

# 再起動
sudo reboot
```

### 3.8.2 低いGPU使用率

**症状:**
```
rocm-smi で GPU使用率が 10-20% のみ
```

**原因と解決:**

**原因1: CPU フォールバック**
```bash
# ログ確認
journalctl -u ollama | grep -i "fallback\|cpu"

# 解決: ROCm 再インストール
```

**原因2: 環境変数未設定**
```bash
# Ollamaサービスの環境変数確認
sudo systemctl show ollama | grep Environment

# 未設定の場合、3.3.3を参照して設定
```

**原因3: 小さすぎるモデル**
```bash
# 3Bモデルなど小型モデルはCPU処理が速いため、GPUが未使用になることがある
# 7B以上のモデルで確認
ollama run qwen2.5:14b
```

### 3.8.3 メモリエラー

**症状:**
```
Error: failed to allocate memory
```

**解決策:**

```bash
# 1. 利用可能メモリ確認
free -h

# 2. 他のプロセス確認
htop

# 3. Ollama 再起動
sudo systemctl restart ollama

# 4. より小さい量子化を使用
ollama pull llama3.1:70b-q4_K_M  # Q8ではなくQ4
```

## 3.9 本章のまとめ

本章では、以下の内容を学習しました。

✅ **ROCmの完全インストール**
- ROCm 6.3のセットアップ
- ユーザー権限設定

✅ **環境変数の最適化**
- HSA_OVERRIDE_GFX_VERSION設定
- システムワイド/サービス用設定

✅ **GPU認識確認**
- rocminfo、rocm-smiでの確認
- HIP動作テスト

✅ **パフォーマンス最適化**
- カーネルパラメータ調整
- Flash Attention有効化
- 電力・クロック設定

✅ **トラブルシューティング**
- 一般的な問題の診断と解決

次章では、Ollamaの基本的なコマンドと実践的な使い方を学びます。

---

**前章へ**: [第2章 インストールとセットアップ](chapter02_installation.md)
**次章へ**: [第4章 基本コマンドと使い方](chapter04_basic_commands.md)
