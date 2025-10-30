# 第3章：LM Studioのインストールと初期設定

## 3.1 インストール前の準備

### 3.1.1 システム要件の確認

LM Studioをインストールする前に、システムが要件を満たしているか確認しましょう。

#### Windows 11での確認手順

**1. システム情報の確認**

```powershell
# PowerShellを管理者権限で起動して実行
systeminfo
```

確認項目：
- OS バージョン: Windows 10/11 64ビット
- 物理メモリ: 128GB（MS-S1 Max）
- プロセッサ: AMD Ryzen AI Max+ 395

**2. AMD GPUの確認**

```powershell
# デバイスマネージャーでGPUを確認
devmgmt.msc
```

「ディスプレイアダプター」に「AMD Radeon Graphics」または「Radeon 8060S」が表示されていることを確認します。

**3. ストレージ空き容量の確認**

```powershell
# ドライブの空き容量を確認
Get-PSDrive -PSProvider FileSystem
```

最低10GB、推奨100GB以上の空き容量を確保してください（モデル保存用）。

#### Linuxでの確認手順

**1. システム情報の確認**

```bash
# ディストリビューション情報
cat /etc/os-release

# CPUの確認
lscpu | grep "Model name"

# メモリの確認
free -h

# ストレージの確認
df -h
```

**2. AMD GPUの確認**

```bash
# PCIデバイスの確認
lspci | grep -i vga
lspci | grep -i amd

# 出力例:
# 01:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Radeon Graphics
```

### 3.1.2 AMD GPUドライバのインストール

LM StudioでAMD GPUを使用するには、適切なドライバが必要です。

#### Windows 11の場合

**方法1: AMD Software（推奨）**

1. **AMD公式サイトからダウンロード**
   - URL: https://www.amd.com/ja/support
   - 「グラフィックス」→「AMD Radeon Graphics」→「Ryzen with Radeon Graphics」を選択
   - 最新の「AMD Software: Adrenalin Edition」をダウンロード

2. **インストール手順**
   ```
   1. ダウンロードしたインストーラーを実行
   2. 「エクスプレスインストール」を選択
   3. インストール完了後、再起動
   ```

3. **ドライババージョンの確認**
   - デスクトップを右クリック→「AMD Software: Adrenalin Edition」
   - 右上の歯車アイコン→「システム」タブ
   - 「ドライババージョン」を確認（24.x.x以降を推奨）

**方法2: Windows Update**

```
1. 設定→Windows Update
2. 「更新プログラムのチェック」
3. オプションの更新プログラムに「AMD Graphics」があればインストール
```

**⚠️ 注意**: Windows版LM Studioは通常のAMDグラフィックスドライバで動作します。ROCmは不要です。

#### Ubuntu 24.04の場合

**ROCmのインストール（必須）**

```bash
# システムを最新に更新
sudo apt update && sudo apt upgrade -y

# 必要なパッケージのインストール
sudo apt install -y wget gnupg2

# ROCm APTリポジトリの追加（ROCm 6.2）
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | \
  gpg --dearmor | sudo tee /etc/apt/keyrings/rocm.gpg > /dev/null

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/6.2 noble main" \
  | sudo tee /etc/apt/sources.list.d/rocm.list

echo -e 'Package: *\nPin: release o=repo.radeon.com\nPin-Priority: 600' \
  | sudo tee /etc/apt/preferences.d/rocm-pin-600

# パッケージリストの更新
sudo apt update

# ROCmのインストール
sudo apt install -y rocm-hip-sdk rocm-libs

# ユーザーをrenderグループに追加
sudo usermod -a -G render,video $USER

# 再起動（重要）
sudo reboot
```

**インストール確認**

再起動後、以下のコマンドで確認します。

```bash
# ROCmのバージョン確認
rocminfo

# GPUの認識確認
rocm-smi

# 期待される出力例:
# ========================ROCm System Management Interface========================
# GPU  Temp   AvgPwr  SCLK    MCLK    Fan     Perf  PwrCap  VRAM%  GPU%
# 0    45.0c  15.0W   800Mhz  1000Mhz 0.0%    auto  120.0W  0%     0%
```

**環境変数の設定**

```bash
# ~/.bashrcに追加
echo 'export PATH=/opt/rocm/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/opt/rocm/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export HSA_OVERRIDE_GFX_VERSION=11.0.0' >> ~/.bashrc
source ~/.bashrc
```

**💡 TIP**: `HSA_OVERRIDE_GFX_VERSION=11.0.0` は、Radeon 8060S（RDNA 3.5）をgfx1100として認識させるための設定です。

## 3.2 LM Studioのダウンロードとインストール

### 3.2.1 ダウンロード

**公式サイト**: https://lmstudio.ai/

1. ブラウザでLM Studio公式サイトにアクセス
2. 「Download」ボタンをクリック
3. OSに応じたインストーラーを選択
   - Windows: `LM-Studio-Setup-x.x.x.exe`
   - Linux: `LM_Studio-x.x.x.AppImage`
   - macOS: `LM-Studio-x.x.x.dmg`

**💡 TIP**: 2025年10月現在、最新版は0.3.19以降です。AMD Ryzen AI Max+ 395を完全サポートするため、必ず最新版をダウンロードしてください。

### 3.2.2 Windowsでのインストール

**標準インストール手順**

1. **インストーラーの実行**
   ```
   ダウンロードした LM-Studio-Setup-x.x.x.exe をダブルクリック
   ```

2. **セキュリティ警告**
   - 「Windows によってPCが保護されました」と表示された場合
   - 「詳細情報」→「実行」をクリック

3. **インストール先の選択**
   ```
   推奨: C:\Users\<ユーザー名>\AppData\Local\Programs\LM Studio
   ```

4. **インストールオプション**
   - [✓] デスクトップにショートカットを作成
   - [✓] スタートメニューに追加
   - [✓] 起動時に自動更新を確認

5. **インストール完了**
   - 「LM Studio を起動」にチェックを入れて「完了」

**カスタムインストール（上級者向け）**

```powershell
# PowerShellで実行（サイレントインストール）
Start-Process -FilePath "LM-Studio-Setup-x.x.x.exe" -ArgumentList "/S" -Wait
```

### 3.2.3 Linuxでのインストール

**AppImageの使用（推奨）**

```bash
# ダウンロードディレクトリに移動
cd ~/Downloads

# 実行権限を付与
chmod +x LM_Studio-x.x.x.AppImage

# 起動
./LM_Studio-x.x.x.AppImage

# オプション: /usr/local/binに配置（システムワイド）
sudo mv LM_Studio-x.x.x.AppImage /usr/local/bin/lmstudio
```

**デスクトップエントリの作成**

```bash
# ~/.local/share/applications/lmstudio.desktop を作成
cat > ~/.local/share/applications/lmstudio.desktop << 'EOF'
[Desktop Entry]
Name=LM Studio
Comment=Run LLMs locally
Exec=/usr/local/bin/lmstudio
Icon=lmstudio
Terminal=false
Type=Application
Categories=Development;
EOF

# 権限設定
chmod +x ~/.local/share/applications/lmstudio.desktop
```

**⚠️ 注意**: AppImageを移動した場合は、Execパスを適切に修正してください。

### 3.2.4 インストールの確認

#### Windows

```powershell
# インストールディレクトリの確認
dir "$env:LOCALAPPDATA\Programs\LM Studio"

# バージョン確認（LM Studio起動後）
# ヘルプ→バージョン情報
```

#### Linux

```bash
# 実行可能か確認
lmstudio --version  # または ./LM_Studio-x.x.x.AppImage --version

# プロセス確認
ps aux | grep lmstudio
```

## 3.3 初回起動と基本設定

### 3.3.1 初回起動

**Windows/Linux共通**

1. LM Studioを起動
2. 「Welcome to LM Studio」画面が表示される
3. 利用規約を確認し、「同意する」をクリック

**初回起動時のダイアログ**

```
□ 匿名の使用統計を送信する（オプション）
□ 起動時に更新を確認する（推奨）
```

**💡 TIP**: 使用統計の送信は任意です。プライバシーを重視する場合はチェックを外してください。

### 3.3.2 インターフェースの概要

LM Studioのメインウィンドウは以下の構成になっています。

```
┌─────────────────────────────────────────────────────┐
│  [検索]  [💬]  [⚙️]  [📚]  [🌐]                   │  ← タブバー
├─────────────────────────────────────────────────────┤
│                                                     │
│              メインコンテンツエリア                  │
│                                                     │
│                                                     │
├─────────────────────────────────────────────────────┤
│  ステータスバー: GPU情報、メモリ使用量など            │
└─────────────────────────────────────────────────────┘
```

**タブの説明**

| アイコン | タブ名 | 説明 |
|---------|--------|------|
| 🔍 | Search | モデルの検索とダウンロード |
| 💬 | Chat | チャットインターフェース（推論実行） |
| 📁 | My Models | ダウンロード済みモデルの管理 |
| 🌐 | Local Server | APIサーバーモード |
| ⚙️ | Settings | アプリケーション設定 |

### 3.3.3 日本語化の設定

LM Studioは日本語インターフェースをサポートしています。

**日本語化手順**

1. 右下の歯車アイコン（⚙️）をクリック→「Settings」
2. 左側メニューから「General」を選択
3. 「Language」セクションを見つける
4. ドロップダウンメニューから「日本語（Japanese (Beta)）」を選択
5. 「再起動が必要です」というダイアログが表示される
6. 「今すぐ再起動」をクリック

**⚠️ 注意**: Beta版のため、一部英語のままの箇所がある場合があります。

### 3.3.4 基本設定の推奨値

#### 一般設定（General）

```
言語（Language）: 日本語（Japanese (Beta)）
テーマ（Theme）: Auto（システム設定に従う）
起動時の動作:
  □ システム起動時にLM Studioを起動
  ✓ 起動時に更新を確認
  □ バックグラウンドで実行を継続
```

#### モデルストレージ設定

**デフォルトのモデル保存場所**

- **Windows**: `C:\Users\<ユーザー名>\.cache\lm-studio\models`
- **Linux**: `~/.cache/lm-studio/models`

**カスタムパスの設定（推奨）**

大容量モデルを多数ダウンロードする場合は、専用のディレクトリを設定します。

1. Settings → Storage
2. 「Models Directory」の「変更」をクリック
3. 保存先を選択（例: `D:\LM_Studio\models` または `/data/lmstudio/models`）
4. 「保存」をクリック

**💡 TIP**: MS-S1 Maxのデュアルストレージ構成の場合、大容量ドライブをモデル保存先に指定すると便利です。

#### ネットワーク設定

```
ダウンロード設定:
  並列ダウンロード数: 3（デフォルト）
  最大ダウンロード速度: 無制限
  プロキシ: 未設定（必要に応じて設定）

Hugging Face設定:
  □ Hugging Faceトークンを使用（プライベートモデル用）
```

## 3.4 AMD GPU認識の確認

### 3.4.1 GPU検出状態の確認

LM StudioがAMD GPUを正しく認識しているか確認します。

**確認手順**

1. LM Studioを起動
2. 画面下部のステータスバーを確認
3. 以下のような表示があることを確認

**Windows:**
```
🎮 GPU: AMD Radeon Graphics (RDNA 3.5) | VRAM: 利用可能
```

**Linux:**
```
🎮 GPU: AMD ROCm (gfx1100) | VRAM: 利用可能
```

### 3.4.2 GPU詳細情報の確認

**高度なGPU情報の表示**

1. キーボードショートカット `Ctrl+Shift+H`（Windows/Linux）を押す
2. 「Hardware Settings」ダイアログが開く
3. 以下の情報を確認

```
GPU情報:
  名前: AMD Radeon Graphics / AMD ROCm
  アーキテクチャ: RDNA 3.5 / gfx1100
  利用可能メモリ: 約100GB（128GBから予約分を引いた値）
  コンピュートユニット: 40
  ROCmバージョン: 6.2.x（Linux）

状態:
  ✓ GPU が検出されました
  ✓ GPU Offload が利用可能です
```

### 3.4.3 トラブルシューティング

#### WindowsでGPUが認識されない場合

**原因1: ドライバが古い**

```powershell
# デバイスマネージャーを開く
devmgmt.msc

# 「ディスプレイアダプター」→「AMD Radeon Graphics」を右クリック
# → 「ドライバーの更新」
```

**原因2: LM Studioのバージョンが古い**

```
1. LM Studioを完全に終了
2. 公式サイトから最新版をダウンロード
3. 再インストール
```

#### LinuxでGPUが認識されない場合

**原因1: ROCmがインストールされていない**

```bash
# ROCmのインストール確認
dpkg -l | grep rocm

# インストールされていない場合、3.1.2の手順でインストール
```

**原因2: 環境変数の設定漏れ**

```bash
# ~/.bashrcに以下が設定されているか確認
echo $PATH | grep rocm
echo $LD_LIBRARY_PATH | grep rocm
echo $HSA_OVERRIDE_GFX_VERSION

# 設定されていない場合
echo 'export PATH=/opt/rocm/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/opt/rocm/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export HSA_OVERRIDE_GFX_VERSION=11.0.0' >> ~/.bashrc
source ~/.bashrc

# LM Studioを再起動
```

**原因3: ユーザー権限の問題**

```bash
# renderグループに所属しているか確認
groups

# renderが含まれていない場合
sudo usermod -a -G render,video $USER

# ログアウト→ログインまたは再起動
```

**原因4: GPUが正しく認識されていない**

```bash
# GPUの認識確認
rocm-smi

# エラーが出る場合
sudo dmesg | grep amdgpu
sudo dmesg | grep -i error

# カーネルモジュールの再ロード
sudo modprobe -r amdgpu
sudo modprobe amdgpu
```

## 3.5 パフォーマンステストの実施

### 3.5.1 簡易テストモデルのダウンロード

初期設定が完了したら、小さなモデルでテストを行います。

**推奨テストモデル: Qwen2.5 3B**

1. 「Search」タブを開く
2. 検索欄に「qwen2.5 3b q4」と入力
3. 「Qwen/Qwen2.5-3B-Instruct-GGUF」を見つける
4. 「qwen2.5-3b-instruct-q4_k_m.gguf」を選択
5. 「Download」をクリック

**ダウンロード容量**: 約2GB
**所要時間**: 1-5分（回線速度に依存）

### 3.5.2 初回推論テスト

**テスト手順**

1. ダウンロード完了後、「Chat」タブに移動
2. 上部の「Select a model」から「qwen2.5-3b-instruct-q4_k_m」を選択
3. 「Load Model」をクリック

**ロード時の確認項目**

```
ステータス表示:
  Loading model... (0%)
  Loading model... (50%)
  Loading model... (100%)
  ✓ Model loaded successfully

GPU情報:
  GPU Layers: 32/32 (全レイヤーをGPUにオフロード)
  VRAM使用量: 約2.5GB
```

**テストプロンプト**

```
こんにちは！あなたは誰ですか?
```

**期待される動作**

- 応答時間: 0.5秒以内に生成開始
- 生成速度: 30-50トークン/秒（3Bモデル）
- スムーズな応答、遅延なし

**⚠️ 注意**: 初回ロード時は、モデルのキャッシュ生成に時間がかかる場合があります。2回目以降は高速化されます。

### 3.5.3 パフォーマンス指標の確認

LM Studioは推論中のパフォーマンス情報を表示します。

**チャットウィンドウ下部の表示**

```
⚡ 45.3 tokens/s | 🎯 Prompt: 12 tokens | 📝 Generated: 89 tokens | 🕐 2.0s
```

| 指標 | 説明 | MS-S1 Maxでの目安（3Bモデル） |
|------|------|------------------------------|
| tokens/s | 生成速度 | 40-60 t/s |
| Prompt tokens | 入力トークン数 | プロンプトに依存 |
| Generated tokens | 生成トークン数 | 応答の長さに依存 |
| Time | 総処理時間 | 応答長/速度 |

**GPU使用状況の確認**

```
ステータスバー右側:
💻 CPU: 15% | 🎮 GPU: 85% | 💾 RAM: 8.5GB/128GB
```

**💡 TIP**: GPUが高い使用率（70%以上）を示していれば、正常にGPUアクセラレーションが機能しています。

## 3.6 設定のバックアップとリストア

### 3.6.1 設定ファイルの場所

LM Studioの設定は以下の場所に保存されます。

**Windows:**
```
C:\Users\<ユーザー名>\AppData\Roaming\LM Studio\
  ├── config.json        # アプリケーション設定
  ├── models.json        # モデル情報
  └── presets/           # カスタムプリセット
```

**Linux:**
```
~/.config/LM Studio/
  ├── config.json
  ├── models.json
  └── presets/
```

### 3.6.2 バックアップの作成

**手動バックアップ**

```bash
# Windows (PowerShell)
Copy-Item "$env:APPDATA\LM Studio" -Destination "D:\Backup\LM_Studio_backup" -Recurse

# Linux
cp -r ~/.config/"LM Studio" ~/backup/lmstudio_config_backup
```

**設定のエクスポート（プリセットのみ）**

1. Settings → Advanced → Export Settings
2. 保存先を選択
3. `lmstudio_settings_YYYYMMDD.json` として保存

### 3.6.3 リストア

**設定の復元**

```bash
# Windows (PowerShell)
Copy-Item "D:\Backup\LM_Studio_backup\*" -Destination "$env:APPDATA\LM Studio" -Recurse -Force

# Linux
cp -r ~/backup/lmstudio_config_backup/* ~/.config/"LM Studio"/
```

LM Studioを再起動すると、復元した設定が適用されます。

## 3.7 本章のまとめ

本章では、以下の内容を実践しました。

✅ システム要件の確認
- AMD Ryzen AI Max+ 395の認識
- メモリとストレージの確認

✅ AMDドライバとROCmのインストール
- Windows: AMD Software Adrenalin Edition
- Linux: ROCm 6.2のセットアップ

✅ LM Studioのインストール
- Windows: インストーラー実行
- Linux: AppImageの設定

✅ 初期設定の実施
- 日本語化
- モデル保存先の設定
- GPU認識の確認

✅ パフォーマンステスト
- 3Bモデルでの動作確認
- GPUアクセラレーションの検証

次章では、AMD GPU設定をさらに詳しく掘り下げ、最適なパフォーマンスを引き出す方法を学びます。

---

**前章へ**: [第2章 ハードウェア仕様とシステム要件](chapter02_hardware_specs.md)
**次章へ**: [第4章 AMD GPU設定の完全ガイド](chapter04_amd_gpu_settings.md)
