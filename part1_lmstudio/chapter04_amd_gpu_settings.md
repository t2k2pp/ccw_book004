# 第4章：AMD GPU設定の完全ガイド

## 4.1 GPU Offload（GPUオフロード）の基礎

### 4.1.1 GPUオフロードとは

**GPUオフロード**は、LLMの推論計算をCPUからGPUに移すことで、大幅な高速化を実現する技術です。

#### 動作原理

LLMは多数の「レイヤー」から構成されています。各レイヤーは以下の演算を実行します：

```
入力 → 行列乗算 → 活性化関数 → 正規化 → 出力
```

GPUオフロードは、これらのレイヤーをGPU上で実行することで並列処理の恩恵を受けます。

#### CPU実行 vs GPU実行の比較

**CPU実行（GPU Layers: 0）**
```
速度: 2-5 tokens/s（70Bモデル）
メモリ: システムRAM使用
利点: VRAMが少なくても実行可能
欠点: 非常に遅い
```

**フルGPU実行（GPU Layers: 全て）**
```
速度: 15-60 tokens/s（モデルサイズに依存）
メモリ: GPU VRAM使用（MS-S1 Maxでは統合メモリ）
利点: 最高速度
欠点: 十分なVRAMが必要
```

**ハイブリッド実行（GPU Layers: 一部）**
```
速度: 中程度（レイヤー数に比例）
メモリ: RAMとVRAMの両方を使用
利点: メモリ不足時の妥協案
欠点: データ転送オーバーヘッド
```

**💡 TIP**: MS-S1 Maxは128GBの統合メモリを持つため、ほとんどの場合フルGPUオフロードが最適です。

### 4.1.2 GPU Layers設定

LM Studioでは、「GPU Layers」スライダーでGPUに割り当てるレイヤー数を調整します。

#### 設定方法

1. **モデルをロードする前**:
   ```
   Chat画面 → モデル選択 → ⚙️（歯車アイコン）→ GPU Settings
   ```

2. **GPU Layersスライダー**:
   ```
   最小値: 0（CPUのみ）
   最大値: モデルのレイヤー数（例: 35, 80等）
   推奨値: 最大値（全レイヤーをGPUにオフロード）
   ```

3. **自動設定**:
   ```
   [Auto] ボタンをクリック
   → LM Studioが利用可能なVRAMに基づいて自動計算
   ```

#### モデル別レイヤー数

| モデル | レイヤー数 | 推奨GPU Layers（MS-S1 Max） |
|--------|-----------|----------------------------|
| Qwen2.5 7B | 32 | 32（全て） |
| Qwen2.5 14B | 40 | 40（全て） |
| Llama 3.1 8B | 32 | 32（全て） |
| Llama 3.1 70B | 80 | 80（全て） |
| Mistral 7B | 32 | 32（全て） |
| Mixtral 8x7B | 32 | 32（全て） |

**⚠️ 注意**: モデルのバリエーションによってレイヤー数が異なる場合があります。

### 4.1.3 メモリ配分の理解

#### MS-S1 Maxでのメモリ配分

```
総メモリ: 128GB LPDDR5X

配分例（70B Q4モデル実行時）:
┌─────────────────────────────────────────┐
│ OS予約: 4GB                              │
│ システムプロセス: 4GB                     │
│ LM Studio本体: 2GB                       │
│ モデルウェイト: 40GB                      │
│ コンテキストキャッシュ: 8GB（32K context）│
│ KVキャッシュ: 10GB                       │
│ 作業用メモリ: 10GB                       │
│ ─────────────────────────────────────   │
│ 使用合計: 78GB                           │
│ 残り利用可能: 50GB                       │
└─────────────────────────────────────────┘
```

**動的メモリ割り当て**

AMD Variable Graphics Memory技術により、GPUはシステムメモリから動的にVRAMを割り当てます。

```bash
# Linux: 現在のメモリ使用状況確認
rocm-smi --showmeminfo

# 出力例:
# GPU[0]: Memory Total: 96GB (動的割り当て)
# GPU[0]: Memory Used: 48GB
# GPU[0]: Memory Free: 48GB
```

## 4.2 詳細GPU設定（Hardware Settings）

### 4.2.1 Hardware Settings画面を開く

**アクセス方法**

- **キーボードショートカット**: `Ctrl+Shift+H`（Windows/Linux）
- **メニューから**: Settings → Advanced → Hardware Settings

### 4.2.2 GPU有効化設定

#### Enable GPU Acceleration（GPU アクセラレーションを有効にする）

```
[✓] Enable GPU Acceleration

機能: GPUを使用した推論を有効化
推奨設定: ON（チェック）
無効にする場合: デバッグ、CPU性能テスト時のみ
```

**効果の確認**

```
有効時:
  - ステータスバーに「🎮 GPU: Active」表示
  - 推論速度が大幅に向上
  - GPU使用率が70-95%に上昇

無効時:
  - 「💻 CPU Only」表示
  - 推論速度が1/10以下に低下
  - CPU使用率が100%に
```

### 4.2.3 GPU選択設定（マルチGPU環境）

MS-S1 Maxは統合GPUのみですが、将来PCIeスロットに外付けGPUを追加した場合の設定です。

#### GPU Device Selection

```
利用可能なGPU:
  [✓] GPU 0: AMD Radeon 8060S (統合)
  [✓] GPU 1: NVIDIA RTX 4060 (PCIe) ← 追加した場合

割り当て戦略:
  ◉ Even Distribution（均等分散）
  ○ Priority Order（優先順位順）
  ○ Manual（手動割り当て）
```

**割り当て戦略の説明**

1. **Even Distribution（均等分散）**
   ```
   各GPUに均等にレイヤーを割り当て
   例: 80レイヤーのモデル、2つのGPU
   → GPU 0: 40レイヤー、GPU 1: 40レイヤー
   ```

2. **Priority Order（優先順位順）**
   ```
   優先度の高いGPUから順番に割り当て
   例: GPU 0が満杯になるまで使用、その後GPU 1
   ```

3. **Manual（手動）**
   ```
   各GPUのレイヤー数を手動指定
   高度なチューニング用
   ```

**💡 TIP**: 単一GPU環境（標準MS-S1 Max）では、これらの設定は不要です。

### 4.2.4 メモリ制限設定

#### VRAM Limit（VRAM制限）

```
設定項目: Maximum VRAM Usage
設定値: 4GB 〜 96GB
デフォルト: Auto（自動）
推奨値: 80GB（MS-S1 Max）
```

**設定の意図**

他のアプリケーション（ブラウザ、IDEなど）にメモリを残すための制限です。

```
設定例:
80GB制限 = モデル用に80GB、システム用に48GB確保
```

#### GPU Memory Type

```
◉ Unified Memory（統合メモリ）← MS-S1 Maxのデフォルト
○ Dedicated Only（専用メモリのみ）
```

**Unified Memory（推奨）**
- CPUとGPUがメモリを共有
- MS-S1 Maxの標準構成
- 大規模モデルの実行に最適

**Dedicated Only**
- 専用VRAM（dGPU）のみ使用
- MS-S1 Maxでは通常使用しない

### 4.2.5 Flash Attention設定

**Flash Attention**は、アテンションメカニズムの計算を高速化する技術です。

#### Flash Attention v2

```
[✓] Enable Flash Attention 2

機能: メモリ効率的なアテンション計算
効果:
  - メモリ使用量 20-30%削減
  - 推論速度 10-20%向上
  - 長いコンテキスト（32K+）で特に有効

推奨設定: ON（チェック）
```

**対応モデル**

- Llama 3系
- Qwen2系
- Mistral系
- その他最新モデルのほとんど

**非対応モデル**

- 古いGPT-2ベースモデル
- 一部のカスタムアーキテクチャ

**⚠️ 注意**: 非対応モデルでは自動的にオフになります。

## 4.3 ROCm環境の最適化（Linux）

### 4.3.1 ROCm環境変数

Linuxでは、環境変数でROCmの動作を調整できます。

#### 基本環境変数

```bash
# ~/.bashrcまたは~/.zshrcに追加

# GPUターゲット指定（RDNA 3.5をgfx1100として認識）
export HSA_OVERRIDE_GFX_VERSION=11.0.0

# ROCmパス
export PATH=/opt/rocm/bin:$PATH
export LD_LIBRARY_PATH=/opt/rocm/lib:$LD_LIBRARY_PATH

# HIP設定
export HIP_VISIBLE_DEVICES=0  # 使用するGPU ID

# デバイスメモリ制限（オプション、単位: バイト）
# export HSA_XNACK=1  # ページフォールト処理を有効化
```

**設定の反映**

```bash
source ~/.bashrc  # または ~/.zshrc
```

### 4.3.2 ROCmパフォーマンスプロファイリング

#### GPUアクティビティの監視

```bash
# リアルタイムGPU使用率監視
watch -n 1 rocm-smi

# 出力例:
# ========================ROCm System Management Interface========================
# GPU  Temp   AvgPwr  SCLK     MCLK     Fan   Perf  PwrCap  VRAM%  GPU%
# 0    68.0c  95.0W   2900Mhz  1000Mhz  65%   auto  120.0W  45%    92%
```

#### メモリ帯域幅テスト

```bash
# ROCmのメモリ帯域幅テストツール
/opt/rocm/bin/rocm_bandwidth_test

# 期待される結果（MS-S1 Max）:
# Unidirectional copy peak bandwidth GB/s: ~212 GB/s
```

#### HIP プロファイリング

```bash
# LM Studio起動時にプロファイリングを有効化
ROCM_PROFILE=1 /usr/local/bin/lmstudio

# プロファイルデータは~/.rocm_profile/に保存される
```

### 4.3.3 カーネルパラメータの最適化

#### AMDGPUカーネルオプション

```bash
# /etc/modprobe.d/amdgpu.confを編集
sudo nano /etc/modprobe.d/amdgpu.conf

# 以下を追加:
options amdgpu ppfeaturemask=0xffffffff
options amdgpu gpu_recovery=1
options amdgpu noretry=0
```

**パラメータの説明**

- `ppfeaturemask=0xffffffff`: すべての電力管理機能を有効化
- `gpu_recovery=1`: GPU hang時の自動回復を有効化
- `noretry=0`: メモリリトライを有効化（大規模モデル用）

**適用方法**

```bash
# カーネルモジュール再ロード
sudo update-initramfs -u
sudo reboot
```

### 4.3.4 パフォーマンスモードの設定

#### GPUクロックプロファイル

```bash
# 現在のパフォーマンスモード確認
sudo rocm-smi --showprofile

# 高性能モードに設定（推論時推奨）
sudo rocm-smi --setperflevel high

# 自動モードに戻す
sudo rocm-smi --setperflevel auto
```

#### 電力制限の調整

```bash
# 現在の電力制限確認
sudo rocm-smi --showpower

# 電力制限を120Wに設定（デフォルト）
sudo rocm-smi --setpoweroverdrive 120

# ⚠️ 注意: MS-S1 MaxのTDP設定（BIOS）を超えないように
```

## 4.4 温度管理とサーマルスロットリング

### 4.4.1 温度監視

#### リアルタイム温度モニタリング

**Linux:**
```bash
# 温度とファン速度の監視
watch -n 1 "rocm-smi | grep -E 'Temp|Fan'"

# sensors コマンド（lm-sensorsパッケージ）
sensors | grep -A 5 amdgpu
```

**Windows:**
```
AMD Software: Adrenalin Edition
→ パフォーマンス → メトリクス
```

#### 温度閾値

```
MS-S1 Max（Balance モード）の温度特性:

アイドル時: 35-40℃
軽負荷（7B推論）: 50-60℃
中負荷（34B推論）: 60-75℃
高負荷（70B推論）: 75-85℃

⚠️ 警告温度: 90℃
🚨 クリティカル: 95℃（サーマルスロットリング開始）
🛑 シャットダウン: 105℃
```

### 4.4.2 冷却の最適化

#### BIOSでの性能モード選択

MS-S1 MaxのBIOS設定（起動時にDELまたはF2キーを押下）

```
Advanced → Power Management → Performance Mode

選択肢と特性:

1. Performance（160W）
   温度: 高め（75-85℃常用）
   速度: 最高
   ファン音: 大きい
   推奨用途: 短時間の最高性能推論

2. Balance（130W）← 推奨
   温度: 適度（65-75℃常用）
   速度: 高い
   ファン音: 許容範囲
   推奨用途: 通常の推論作業

3. Quiet（110W）
   温度: 低め（55-65℃常用）
   速度: やや低い
   ファン音: 静か
   推奨用途: 静音環境での作業

4. Rack（140W）
   温度: 高め（70-80℃常用）
   速度: 高い
   ファン音: 一定（高め）
   推奨用途: サーバーラック環境
```

#### ファンカーブのカスタマイズ（Linux）

```bash
# fancontrol の設定
sudo apt install lm-sensors fancontrol

# センサー検出
sudo sensors-detect

# ファンカーブ設定
sudo pwmconfig

# fancontrol 起動
sudo systemctl enable fancontrol
sudo systemctl start fancontrol
```

**カスタムファンカーブ例:**
```
30℃以下: 30%（最小回転数）
30-50℃: 30-45%（緩やかに上昇）
50-70℃: 45-70%（中程度）
70-85℃: 70-90%（高速回転）
85℃以上: 100%（最大回転）
```

### 4.4.3 サーマルスロットリング対策

#### サーマルスロットリングの検出

**Linux:**
```bash
# dmesgでスロットリングイベントを確認
sudo dmesg | grep -i "thermal"
sudo dmesg | grep -i "throttle"

# 出力例（スロットリング発生時）:
# [12345.678] amdgpu 0000:01:00.0: GPU thermal throttling activated
```

**Windows:**
```
イベントビューアー → Windowsログ → システム
フィルター: ソース "amdgpu" または "thermal"
```

#### 対策方法

**1. 環境改善**
```
- 本体周辺の通気確保（前後左右10cm以上）
- 室温を下げる（エアコン、25℃以下推奨）
- 本体を高い位置に設置（冷気は下に溜まる）
```

**2. 性能モード変更**
```
BIOS設定でPerformance → Balance に変更
```

**3. GPU Layers削減（最終手段）**
```
全レイヤーをGPUにオフロードしている場合:
GPU Layers を 75-80% に減らす
例: 80レイヤー → 60レイヤー

効果:
- 温度 5-10℃低下
- 速度 10-20%低下（トレードオフ）
```

## 4.5 ベンチマークとパフォーマンス測定

### 4.5.1 LM Studio内蔵ベンチマーク

**ベンチマークの実行**

1. モデルをロード
2. Chat画面右上の「⋮」メニュー
3. 「Run Benchmark」を選択
4. テストパラメータ設定:
   ```
   Prompt length: 512 tokens
   Generation length: 128 tokens
   Runs: 3
   ```
5. 「Start Benchmark」

**結果の見方**

```
ベンチマーク結果:

Prompt Processing (プロンプト処理):
  - Speed: 1250 tokens/s
  - Time: 0.41s

Text Generation (テキスト生成):
  - Speed: 15.3 tokens/s
  - Time: 8.37s

Total Time: 8.78s
Peak VRAM: 42.5 GB
```

### 4.5.2 モデル別性能テスト

#### テストモデルと期待値（MS-S1 Max、Balance モード）

| モデル | 量子化 | プロンプト速度 | 生成速度 | VRAM使用 |
|--------|--------|---------------|---------|----------|
| Qwen2.5 3B | Q4_K_M | 2000+ t/s | 50-60 t/s | 2.5GB |
| Qwen2.5 7B | Q4_K_M | 1500+ t/s | 35-45 t/s | 4.8GB |
| Qwen2.5 14B | Q4_K_M | 1000+ t/s | 20-25 t/s | 9GB |
| Qwen2.5 32B | Q4_K_M | 600+ t/s | 8-12 t/s | 20GB |
| Llama 3.1 70B | Q4_K_M | 300+ t/s | 3-5 t/s | 42GB |

**💡 TIP**: プロンプト処理速度は、コンテキスト理解の速度を示します。生成速度は、実際の応答生成速度です。

### 4.5.3 最適化のチェックリスト

設定が最適化されているか、以下をチェックしましょう。

```
✅ GPU Settings
  [✓] GPU Acceleration: 有効
  [✓] GPU Layers: 最大値（全レイヤー）
  [✓] Flash Attention 2: 有効
  [✓] VRAM Limit: 80GB以上

✅ System Settings
  [✓] 性能モード: Balance（または必要に応じてPerformance）
  [✓] 温度: 85℃以下を維持
  [✓] バックグラウンドアプリ: 最小化

✅ Driver & Runtime（Linux）
  [✓] ROCm 6.2以降
  [✓] HSA_OVERRIDE_GFX_VERSION=11.0.0 設定済み
  [✓] render/videoグループに所属

✅ Performance Indicators
  [✓] GPU使用率: 70-95%（推論中）
  [✓] 生成速度: 期待値の±20%以内
  [✓] スロットリング: 発生していない
```

## 4.6 トラブルシューティング

### 4.6.1 一般的な問題と解決法

#### 問題1: GPUが認識されない

**症状:**
```
ステータスバーに「CPU Only」表示
GPU Layers スライダーがグレーアウト
```

**解決法（Windows）:**
```
1. AMD Software を最新版に更新
2. LM Studio を最新版に更新
3. デバイスマネージャーでGPUドライバを再インストール
4. Windows を再起動
```

**解決法（Linux）:**
```bash
# ROCmの再インストール
sudo apt remove --purge rocm-*
sudo apt autoremove
# 第3章の手順でROCmを再インストール

# 環境変数の確認
echo $HSA_OVERRIDE_GFX_VERSION  # 11.0.0 と表示されるべき

# ユーザー権限の確認
groups | grep render  # renderが含まれているべき
```

#### 問題2: 推論速度が異常に遅い

**症状:**
```
7Bモデルで 5 tokens/s 以下
GPU使用率が 10% 以下
```

**原因と解決法:**

**原因A: GPU Layersが少ない**
```
確認: GPU Layers の値を確認
解決: スライダーを最大値に設定
```

**原因B: CPUモードで動作している**
```
確認: ステータスバーを確認
解決: Hardware Settings で GPU Acceleration を有効化
```

**原因C: サーマルスロットリング**
```
確認: GPU温度が90℃以上
解決:
  - 環境温度を下げる
  - 性能モードをBalanceに変更
  - 本体の通気を確保
```

**原因D: バックグラウンドプロセス**
```
確認:
  # Windows
  タスクマネージャーでメモリ使用量確認

  # Linux
  htop または top でメモリ確認

解決: 不要なアプリケーションを終了
```

#### 問題3: メモリ不足エラー

**症状:**
```
エラーメッセージ: "Out of memory"
モデルのロードに失敗
```

**解決法:**

```
1. より小さい量子化レベルを選択
   Q8 → Q6 → Q5 → Q4

2. コンテキスト長を削減
   128K → 32K → 8K

3. 他のアプリケーションを終了
   ブラウザ、IDEなど

4. モデルサイズを下げる
   70B → 34B → 13B
```

#### 問題4: 異常終了・クラッシュ

**症状:**
```
推論中にLM Studioが突然終了
応答が途中で止まる
```

**解決法:**

**Windows:**
```
1. イベントビューアーでエラーログ確認
2. AMD Software で GPU診断実行
3. LM Studio のログ確認:
   %APPDATA%\LM Studio\logs\
4. クリーンインストール
```

**Linux:**
```bash
# システムログ確認
sudo dmesg | tail -50
journalctl -xe | grep lmstudio

# GPU hang の確認
sudo dmesg | grep "GPU hang"

# LM Studio ログ確認
~/.config/LM Studio/logs/

# GPU リセット
sudo systemctl restart display-manager
```

### 4.6.2 詳細ログの有効化

**デバッグ情報の取得**

```bash
# Linux: 詳細ログモードでLM Studio起動
LMSTUDIO_LOG_LEVEL=debug /usr/local/bin/lmstudio

# ログ出力先
tail -f ~/.config/"LM Studio"/logs/main.log
```

**Windows:**
```
LM Studio設定 → Advanced → Enable Debug Logging
再起動後、ログは以下に保存:
%APPDATA%\LM Studio\logs\debug.log
```

## 4.7 本章のまとめ

本章では、AMD GPU設定の詳細について学習しました。

✅ **GPU Offloadの基礎**
- レイヤーベースのオフロード機構
- MS-S1 Maxでは全レイヤーオフロードが最適

✅ **詳細GPU設定**
- GPU Acceleration有効化
- Flash Attention 2の活用
- メモリ制限設定

✅ **ROCm最適化（Linux）**
- 環境変数の設定
- パフォーマンスプロファイリング
- カーネルパラメータ調整

✅ **温度管理**
- 性能モード選択（Balance推奨）
- サーマルスロットリング対策
- ファンカーブのカスタマイズ

✅ **ベンチマークとトラブルシューティング**
- 性能測定方法
- 一般的な問題の解決法

次章では、実際にモデルをダウンロードして管理する方法を学びます。

---

**前章へ**: [第3章 LM Studioのインストールと初期設定](chapter03_installation.md)
**次章へ**: [第5章 モデルのダウンロードと管理](chapter05_model_management.md)
