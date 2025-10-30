# 第7章：MS-S1 Max向け最適化設定

## 7.1 MS-S1 Maxの特性を活かす

### 7.1.1 128GBメモリの戦略的活用

MS-S1 Maxの最大の武器は、128GBの大容量統合メモリです。この章では、このリソースを最大限に活用する方法を学びます。

#### メモリ活用の4つの戦略

**戦略1: 大規模モデルの実行**
```
従来の16GB GPU: 7B Q4まで
MS-S1 Max: 70B Q4以上が快適

具体例:
  Llama 3.1 70B Q4_K_M
  メモリ使用: 約48GB
  生成速度: 3-5 t/s
  残りメモリ: 80GB
  → ブラウザ、IDE、その他アプリを同時使用可能
```

**戦略2: 高品質量子化の使用**
```
従来: Q4が限界
MS-S1 Max: Q6、Q8も選択肢

例: Qwen2.5 32B
  Q4_K_M: 20GB → 速度 8-12 t/s
  Q5_K_M: 24GB → 速度 7-10 t/s（品質向上）
  Q6_K: 28GB → 速度 6-9 t/s（さらに高品質）

128GBあれば、Q6_Kでも余裕
```

**戦略3: 超長コンテキストの活用**
```
従来: 4K-8Kが限界
MS-S1 Max: 32K-64Kが実用的

例: Qwen2.5 14B + 64Kコンテキスト
  モデル: 9GB
  コンテキストキャッシュ: 約70GB
  合計: 約79GB
  → 書籍一冊分の分析が可能
```

**戦略4: マルチモデル同時実行**
```
複数のモデルを同時にメモリ上にロード

構成例:
  1. Qwen2.5 7B Q4（5GB）: 高速チャット
  2. DeepSeek-Coder V2 16B Q4（10GB）: コーディング
  3. Llama 3.1 8B Q4（5GB）: 英語専用
  合計: 20GB
  残り: 108GB

利点:
  ✓ モデル切り替えが瞬時
  ✓ 用途に応じた最適モデル使用
  ✓ ロード時間ゼロ
```

### 7.1.2 AMD Radeon 8060Sの最適化

#### GPU特性の理解

```
AMD Radeon 8060S (RDNA 3.5)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
アーキテクチャ: RDNA 3.5 (gfx1100)
コンピュートユニット: 40 CU
ストリームプロセッサ: 2560 SP
FP16性能: 29.6 TFLOPS
INT8性能: 59.2 TOPS
メモリ帯域幅: 212 GB/s（実測）
Infinity Cache: 64MB
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

最適な用途:
  ✓ 7B-14Bモデル: 完全にGPU上で高速実行
  ✓ 34Bモデル: GPUアクセラレーション有効
  ✓ 70Bモデル: メモリ帯域幅がボトルネック
```

#### GPU Layers設定の最適化

**推奨設定（モデル別）:**

```
3B-7Bモデル:
  GPU Layers: 最大値（全レイヤー）
  VRAM割り当て: 10GB
  期待速度: 35-60 t/s
  GPU使用率: 85-95%

13B-14Bモデル:
  GPU Layers: 最大値（全レイヤー）
  VRAM割り当て: 20GB
  期待速度: 18-25 t/s
  GPU使用率: 80-90%

32B-34Bモデル:
  GPU Layers: 最大値（全レイヤー）
  VRAM割り当て: 35GB
  期待速度: 8-12 t/s
  GPU使用率: 75-85%

70Bモデル:
  GPU Layers: 最大値（全レイヤー）
  VRAM割り当て: 60GB
  期待速度: 3-5 t/s
  GPU使用率: 65-75%
  注意: メモリ帯域幅が主なボトルネック
```

**💡 TIP**: MS-S1 Maxでは、ほぼすべてのモデルで全レイヤーをGPUにオフロードするのが最適です。

## 7.2 性能モード別の推奨設定

### 7.2.1 BIOS性能モードの選択

MS-S1 MaxのBIOSでは、4つの性能モードが選択できます。用途に応じて使い分けましょう。

#### Performance モード（160W）

```
TDP: 160W（ピーク）
CPU最大クロック: 5.1 GHz
GPU最大クロック: 2.9 GHz

温度:
  アイドル: 40-45℃
  7B推論: 65-75℃
  70B推論: 80-90℃

ファン騒音: 45-50 dBA（高め）

推奨用途:
  ✓ 最大性能が必要な短時間タスク
  ✓ ベンチマーク
  ✓ デモンストレーション
  ✓ 高負荷推論（70B Q8等）

推奨LM Studio設定:
  GPU Layers: 最大
  Context: 32K-64K
  Flash Attention: ON

パフォーマンス向上: 基準比+15-20%
```

#### Balance モード（130W）← 最推奨

```
TDP: 130W（持続可能）
CPU最大クロック: 4.8 GHz
GPU最大クロック: 2.7 GHz

温度:
  アイドル: 35-40℃
  7B推論: 55-65℃
  70B推論: 70-80℃

ファン騒音: 38-42 dBA（許容範囲）

推奨用途:
  ✓ 日常的な推論作業 ← 最適
  ✓ 長時間の使用
  ✓ ほとんどの用途
  ✓ 最高のバランス

推奨LM Studio設定:
  GPU Layers: 最大
  Context: 16K-32K
  Flash Attention: ON

パフォーマンス: 基準（100%）
```

#### Quiet モード（110W）

```
TDP: 110W
CPU最大クロック: 4.5 GHz
GPU最大クロック: 2.5 GHz

温度:
  アイドル: 30-35℃
  7B推論: 45-55℃
  70B推論: 60-70℃

ファン騒音: 32-36 dBA（静か）

推奨用途:
  ✓ 静音環境（図書館、オフィス）
  ✓ 夜間作業
  ✓ 軽量モデル（3B-14B）
  ✓ 音声・動画録画中

推奨LM Studio設定:
  GPU Layers: 最大
  Context: 8K-16K
  モデル: 3B-14B推奨

パフォーマンス: 基準比-10-15%
```

#### Rack モード（140W）

```
TDP: 140W
CPU最大クロック: 4.9 GHz
GPU最大クロック: 2.8 GHz

温度: Balanceと同様
ファン騒音: 42-46 dBA（固定高速回転）

推奨用途:
  ✓ サーバーラック環境
  ✓ 24時間稼働
  ✓ 予測可能な騒音レベルが必要な場合

推奨LM Studio設定:
  Performance モードと同様

パフォーマンス: 基準比+8-12%
```

### 7.2.2 OSごとの最適化

#### Windows 11最適化

**電源プラン設定**

```powershell
# 高パフォーマンス電源プランを作成
powercfg -duplicatescheme 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c

# アクティブに設定
powercfg -setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c

# または、GUIで設定
コントロールパネル → 電源オプション
→ 高パフォーマンス を選択
```

**仮想メモリの最適化**

```
システムプロパティ → 詳細設定 → パフォーマンス設定
→ 詳細設定タブ → 仮想メモリ

推奨設定:
  初期サイズ: 16384 MB (16GB)
  最大サイズ: 32768 MB (32GB)

理由: 128GB物理メモリがあるため、仮想メモリは最小限でOK
```

**不要なバックグラウンドアプリの無効化**

```
設定 → プライバシーとセキュリティ
→ バックグラウンドアプリ
→ 不要なアプリをオフ

特に無効化推奨:
  ✓ OneDrive（必要でなければ）
  ✓ Cortana
  ✓ Windows Search（インデックス作成）
  ✓ Sysmain（旧Superfetch）
```

**AMD Radeon設定**

```
AMD Software: Adrenalin Edition を開く

Graphics → 詳細設定:
  ✓ Radeon Anti-Lag: オフ（LM Studioでは不要）
  ✓ Radeon Boost: オフ
  ✓ Radeon Image Sharpening: オフ
  ✓ GPU Scaling: オフ

Performance → Tuning:
  ◉ Default (自動)
  または
  ◉ Manual → Power Limit: +10% (必要に応じて)
```

#### Ubuntu 24.04最適化

**カーネルパラメータの追加**

```bash
# /etc/default/grubを編集
sudo nano /etc/default/grub

# GRUB_CMDLINE_LINUX_DEFAULT行に追加:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amdgpu.ppfeaturemask=0xffffffff amd_iommu=on iommu=pt"

# 設定を適用
sudo update-grub
sudo reboot
```

**スワップの最適化**

```bash
# 現在のスワップ使用傾向を確認
cat /proc/sys/vm/swappiness
# デフォルト: 60

# スワップ使用を最小化（128GBメモリがあるため）
sudo sysctl vm.swappiness=10

# 永続化
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
```

**CPUガバナーの設定**

```bash
# パフォーマンスガバナーに設定
echo "performance" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 起動時に自動設定
sudo apt install cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
sudo systemctl restart cpufrequtils
```

**ROCmの最適化**

```bash
# ~/.bashrcに追加
cat >> ~/.bashrc << 'EOF'
# ROCm最適化
export HSA_OVERRIDE_GFX_VERSION=11.0.0
export GPU_MAX_HEAP_SIZE=100
export GPU_MAX_ALLOC_PERCENT=100
export HSA_ENABLE_SDMA=0
EOF

source ~/.bashrc
```

## 7.3 ワークフロー別最適構成

### 7.3.1 日常チャット環境

**目的**: 高速で快適なチャット体験

```yaml
構成名: Daily Chat Setup

ハードウェア:
  性能モード: Balance (130W)
  OS: Windows 11 または Ubuntu 24.04

モデル構成:
  メインモデル: Qwen2.5 7B Q4_K_M
  サブモデル: Llama 3.2 3B Q4_K_M（超高速用）

LM Studio設定:
  GPU Layers: 最大
  Context Length: 16384
  Temperature: 0.7
  Top P: 0.95
  Max Tokens: 2048

期待パフォーマンス:
  レスポンス開始: 即座（< 0.5秒）
  生成速度: 35-45 t/s
  メモリ使用: 約8GB
  残りメモリ: 120GB（他の作業に使用可能）

推奨用途:
  ✓ 日常的な質問応答
  ✓ メール作成
  ✓ アイデア出し
  ✓ 簡単な文章生成
```

### 7.3.2 プロフェッショナル執筆環境

**目的**: 高品質な長文生成

```yaml
構成名: Professional Writing Setup

ハードウェア:
  性能モード: Balance (130W)
  OS: Windows 11 または Ubuntu 24.04

モデル構成:
  メインモデル: Qwen2.5 32B Q5_K_M
  サブモデル: Qwen2.5 14B Q4_K_M（ドラフト用）

LM Studio設定:
  GPU Layers: 最大
  Context Length: 32768
  Temperature: 0.75
  Top P: 0.92
  Repeat Penalty: 1.15
  Max Tokens: 8192

期待パフォーマンス:
  生成速度: 7-10 t/s (32B Q5)
  メモリ使用: 約60GB（32Kコンテキスト込み）
  残りメモリ: 68GB

推奨用途:
  ✓ ブログ記事執筆
  ✓ 技術文書作成
  ✓ レポート作成
  ✓ 書籍執筆
```

### 7.3.3 開発・コーディング環境

**目的**: 効率的なコード生成とレビュー

```yaml
構成名: Coding Assistant Setup

ハードウェア:
  性能モード: Balance (130W)
  OS: Ubuntu 24.04 推奨（開発環境）

モデル構成:
  メインモデル: DeepSeek-Coder-V2 16B Q4_K_M
  サブモデル: Qwen2.5 7B Q4_K_M（ドキュメント生成用）

LM Studio設定:
  GPU Layers: 最大
  Context Length: 16384
  Temperature: 0.2（正確性重視）
  Top P: 0.90
  Max Tokens: 4096

統合:
  VS Code + Continue拡張機能
  LM Studio Local Server モード使用

期待パフォーマンス:
  生成速度: 18-22 t/s
  コード補完レイテンシ: < 1秒
  メモリ使用: 約15GB
  残りメモリ: 113GB

推奨用途:
  ✓ コード生成
  ✓ コードレビュー
  ✓ バグ修正提案
  ✓ リファクタリング
  ✓ ドキュメント自動生成
```

### 7.3.4 研究・分析環境

**目的**: 最高品質の推論と長文分析

```yaml
構成名: Research & Analysis Setup

ハードウェア:
  性能モード: Performance (160W)
  冷却: 最適化（室温25℃以下推奨）
  OS: Ubuntu 24.04 推奨

モデル構成:
  メインモデル: Llama 3.1 70B Q4_K_M
  またはQwen2.5 72B Q4_K_M（日本語重視）

LM Studio設定:
  GPU Layers: 最大
  Context Length: 65536（64K）
  Temperature: 0.5（バランス）
  Top P: 0.92
  Max Tokens: 8192

期待パフォーマンス:
  生成速度: 3-5 t/s
  メモリ使用: 約100GB（64Kコンテキスト込み）
  残りメモリ: 28GB

推奨用途:
  ✓ 学術論文の分析
  ✓ 書籍全体の要約
  ✓ 複雑な推論タスク
  ✓ 多段階の問題解決
```

## 7.4 メモリ管理の高度なテクニック

### 7.4.1 コンテキストキャッシュの理解

LM Studioは、会話履歴をコンテキストキャッシュとして保持します。

**キャッシュの仕組み:**

```
会話の流れ:

ユーザー: "こんにちは"（5トークン）
AI: "こんにちは！...」（50トークン）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
キャッシュ: 55トークン

ユーザー: "今日の天気は?"（8トークン）
AI: "申し訳ありませんが...」（45トークン）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
キャッシュ: 108トークン

...会話が続くと...

キャッシュが Context Length に到達
→ 古い会話から削除される
```

**メモリへの影響:**

```
例: Qwen2.5 32B、32Kコンテキスト

会話開始時:
  モデル: 20GB
  キャッシュ: 1GB（ほぼ空）
  合計: 21GB

15分後（15K トークンの会話）:
  モデル: 20GB
  キャッシュ: 35GB
  合計: 55GB

30分後（32K トークン、上限到達）:
  モデル: 20GB
  キャッシュ: 72GB（最大）
  合計: 92GB
```

**最適化テクニック:**

```
テクニック1: 会話のリセット
  長時間使用後、「New Chat」で会話をリセット
  → キャッシュがクリアされ、メモリ解放

テクニック2: コンテキスト長の動的調整
  短い会話: 8K設定
  長い会話が必要: 32K設定に変更

テクニック3: 複数チャットの管理
  用途別にチャットを分ける
  不要なチャットは削除
```

### 7.4.2 複数モデルの効率的な切り替え

**方法1: モデルのアンロード**

```
現在のモデル: Llama 3.1 70B（48GB使用中）
↓
Chat画面 → モデル選択 → "Unload Model"
↓
メモリ解放: 48GB
↓
新しいモデルをロード: Qwen2.5 7B（5GB）
```

**方法2: 事前ロード（MS-S1 Max推奨）**

```
メモリに余裕があれば、複数モデルを同時ロード可能

例:
  モデル1: Qwen2.5 7B（5GB）- Chat A
  モデル2: DeepSeek-Coder 16B（10GB）- Chat B
  モデル3: Llama 3.2 3B（2GB）- Chat C
  合計: 17GB
  残り: 111GB

切り替え:
  Chat A、B、C のタブを切り替えるだけ
  ロード時間ゼロ
```

**方法3: スクリプトによる自動化（上級）**

```bash
# LM Studio API経由で自動切り替え
curl -X POST http://localhost:1234/v1/models/load \
  -H "Content-Type: application/json" \
  -d '{"model": "qwen2.5-7b-instruct-q4_k_m"}'
```

## 7.5 パフォーマンスモニタリング

### 7.5.1 リアルタイムモニタリングツール

#### Windows

**タスクマネージャー拡張版**

```
Ctrl+Shift+Esc → パフォーマンスタブ

確認項目:
  CPU使用率: 推論中は15-30%が目安
  メモリ: 使用量を監視
  GPU: AMD Radeon Graphicsの使用率
```

**HWiNFO64（推奨）**

```
ダウンロード: https://www.hwinfo.com/

モニタリング項目:
  ✓ CPU温度（各コア）
  ✓ GPU温度
  ✓ メモリ使用量
  ✓ GPU使用率
  ✓ 電力消費
  ✓ クロック周波数

設定:
  Sensors → カスタムレイアウト作成
  システムトレイに表示
```

#### Linux

**コマンドラインツール**

```bash
# CPU、メモリ監視
htop

# GPU監視
watch -n 1 rocm-smi

# 統合監視（推奨）
sudo apt install nvtop  # AMD GPUにも対応
nvtop
```

**統合監視スクリプト**

```bash
#!/bin/bash
# ms-s1-max-monitor.sh

while true; do
  clear
  echo "=== MS-S1 Max Performance Monitor ==="
  echo ""
  echo "--- CPU ---"
  top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print "CPU Usage: " 100 - $1"%"}'

  echo ""
  echo "--- Memory ---"
  free -h | awk '/^Mem:/ {print "Used: " $3 " / " $2 " (" $3/$2*100 "%)"}'

  echo ""
  echo "--- GPU ---"
  rocm-smi --showuse | grep "GPU use" || echo "ROCm not available"

  echo ""
  echo "--- Temperature ---"
  sensors | grep -A 0 'edge' | head -1

  sleep 2
done
```

### 7.5.2 ボトルネック診断

**症状別診断チャート:**

```
症状: 推論速度が遅い（期待値の50%以下）

→ GPU使用率を確認

  GPU使用率 < 50%:
    → CPU/メモリボトルネック
    → 解決策:
      1. バックグラウンドアプリを終了
      2. GPU Layers設定を確認
      3. LM Studio再起動

  GPU使用率 > 80%:
    → GPU性能の限界、またはメモリ帯域幅
    → 解決策:
      1. より小さいモデルを試す
      2. より軽い量子化を試す（Q6→Q4）
      3. 性能モードをPerformanceに変更

  温度 > 85℃:
    → サーマルスロットリング
    → 解決策:
      1. 室温を下げる
      2. 本体の通気を確保
      3. 性能モードをBalanceに変更
```

## 7.6 本章のまとめ

本章では、MS-S1 Max固有の最適化について学習しました。

✅ **128GBメモリの活用戦略**
- 大規模モデル実行（70B Q4）
- 高品質量子化（Q6、Q8）
- 超長コンテキスト（32K-64K）
- マルチモデル同時実行

✅ **性能モードの選択**
- Balance (130W): 最推奨、日常使用
- Performance (160W): 最大性能
- Quiet (110W): 静音環境

✅ **ワークフロー別最適構成**
- 日常チャット: Qwen2.5 7B + 16K
- 執筆: Qwen2.5 32B + 32K
- コーディング: DeepSeek-Coder 16B
- 研究: Llama 70B + 64K

✅ **パフォーマンスモニタリング**
- リアルタイム監視ツール
- ボトルネック診断

次章では、実践的な使い方を学びます。

---

**前章へ**: [第6章 推論設定の完全解説](chapter06_inference_settings.md)
**次章へ**: [第8章 実践的な使い方](chapter08_practical_usage.md)
