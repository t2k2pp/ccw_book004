# LMStudioを皮切りにローカルでAIで使い倒す

## 書籍概要

本書は、AMD Ryzen AI Max+ 395（128GBメモリ）搭載のMinisforum MS-S1 Maxで、LM Studioを使ってローカル環境で大規模言語モデル（LLM）を最大限に活用するための完全ガイドです。

### 対象読者

- ローカルAI環境の構築に興味がある方
- プライバシーを重視し、自分のマシンでAIを動かしたい方
- AMD Ryzen AI Max+ 395 / Minisforum MS-S1 Maxのユーザー
- LM Studioの使い方を網羅的に学びたい方
- 70Bパラメータ以上の大規模モデルを実行したい方

### 本書の特徴

✅ **AMD Ryzen AI Max+ 395に完全対応**
- 128GBメモリの効果的な活用法
- ROCm環境の構築と最適化
- 性能モード別の推奨設定

✅ **網羅的な設定解説**
- すべての推論パラメータを詳細に解説
- 用途別の最適な設定を提示
- トラブルシューティングガイド

✅ **実践的なユースケース**
- 文章作成・編集
- コード生成とレビュー
- APIサーバーモードの活用
- RAG（検索拡張生成）の実装

✅ **日本語に特化**
- 日本語対応モデルの推奨
- 日本語での効果的なプロンプト作成
- 日本語環境での最適化

## 書籍構成

### 第一部：LM Studio完全ガイド

#### [第1章：はじめに - LM StudioとローカルAIの世界](part1_lmstudio/chapter01_introduction.md)

- ローカルAIのメリットと重要性
- LM Studioの特徴と機能
- AMD Ryzen AI Max+ 395の驚異的な性能
- Minisforum MS-S1 Maxのハードウェア仕様
- 本書の構成と活用方法

**主なポイント:**
- クラウドAI vs ローカルAI
- 128GBメモリで実現できること
- 70Bモデルも快適に実行可能

#### [第2章：ハードウェア仕様とシステム要件](part1_lmstudio/chapter02_hardware_specs.md)

- AMD Ryzen AI Max+ 395の詳細仕様
  - Zen5アーキテクチャ（16コア/32スレッド）
  - RDNA 3.5 GPU（2560 SP、60 TOPS）
  - 128GB LPDDR5X-8000メモリ（256GB/s帯域幅）
- Minisforum MS-S1 Maxの完全仕様
- LM Studioのシステム要件
- モデルサイズとメモリ要件
- 性能予測とベンチマーク

**主なポイント:**
- 理論性能の計算
- モデル別メモリ要件表
- 競合環境との比較

#### [第3章：LM Studioのインストールと初期設定](part1_lmstudio/chapter03_installation.md)

- インストール前の準備
- AMD GPUドライバのインストール
  - Windows: AMD Software Adrenalin Edition
  - Linux: ROCm 6.2のセットアップ
- LM Studioのダウンロードとインストール
- 初回起動と基本設定
- 日本語化の設定
- AMD GPU認識の確認
- パフォーマンステストの実施

**主なポイント:**
- ROCm環境変数の設定
- GPU認識のトラブルシューティング
- 3Bモデルでの動作確認

#### [第4章：AMD GPU設定の完全ガイド](part1_lmstudio/chapter04_amd_gpu_settings.md)

- GPU Offload（GPUオフロード）の基礎
- GPU Layers設定の最適化
- 詳細GPU設定（Hardware Settings）
  - GPU Acceleration有効化
  - Flash Attention 2
  - メモリ制限設定
- ROCm環境の最適化（Linux）
- 温度管理とサーマルスロットリング
- ベンチマークとパフォーマンス測定

**主なポイント:**
- 全レイヤーGPUオフロードの重要性
- ROCm環境変数の詳細解説
- 性能モード別の温度特性

#### [第5章：モデルのダウンロードと管理](part1_lmstudio/chapter05_model_management.md)

- LLMモデルの基礎知識
  - 主要モデルファミリー（Llama, Qwen, Mistral等）
  - 量子化（Q4_K_M推奨）
  - GGUF形式
- モデルの検索とダウンロード
- MS-S1 Max向け推奨モデルカタログ
  - 3B-7B（初心者向け）
  - 13B-34B（中級者向け）
  - 70B+（上級者向け）
  - 専門用途（コーディング等）
- モデルの管理と整理
- 更新と最新版の追跡

**主なポイント:**
- 日本語に強いQwenシリーズ
- MS-S1 Max向け推奨構成（20GB/50GB/100GB）
- モデルのエクスポート・インポート

#### [第6章：推論設定の完全解説](part1_lmstudio/chapter06_inference_settings.md)

- 推論パラメータの基礎
- 主要パラメータの詳細解説
  - **Temperature（温度）**: 0.7推奨、創造性の制御
  - **Top P（Nucleus Sampling）**: 0.95推奨
  - **Top K**: 40推奨
  - **Repeat Penalty**: 1.1推奨、繰り返し抑制
  - **Context Length**: 16K-32K推奨
  - **Max Tokens**: 用途に応じて調整
- プリセットの活用
  - Precise、Balanced、Creative
  - カスタムプリセットの作成
- 用途別推奨設定
- パフォーマンスとの関係

**主なポイント:**
- 各パラメータの数値と効果の関係
- 用途別最適設定（技術文書、創作、チャット）
- トラブルシューティング

#### [第7章：MS-S1 Max向け最適化設定](part1_lmstudio/chapter07_optimization.md)

- 128GBメモリの戦略的活用
  - 大規模モデルの実行（70B Q4）
  - 高品質量子化の使用（Q6、Q8）
  - 超長コンテキスト（32K-64K）
  - マルチモデル同時実行
- 性能モード別の推奨設定
  - **Balance（130W）**: 最推奨、日常使用
  - Performance（160W）: 最大性能
  - Quiet（110W）: 静音環境
  - Rack（140W）: サーバー用途
- OSごとの最適化
  - Windows 11: 電源プラン、仮想メモリ
  - Ubuntu 24.04: カーネルパラメータ、スワップ
- ワークフロー別最適構成
- メモリ管理の高度なテクニック
- パフォーマンスモニタリング

**主なポイント:**
- Balance モードが最適なバランス
- コンテキストキャッシュの理解
- ボトルネック診断チャート

#### [第8章：実践的な使い方](part1_lmstudio/chapter08_practical_usage.md)

- チャットインターフェースの活用
  - 効果的なプロンプトの書き方
  - システムプロンプトの活用
  - マルチターン対話の管理
- 実践的なユースケース
  - 文章作成・編集（ブログ、メール）
  - コード生成とレビュー
  - データ分析と要約
  - 学習支援
- APIサーバーモードの活用
  - ローカルサーバーの起動
  - VS Code + Continue拡張機能との連携
  - Pythonスクリプトからの利用
  - Webアプリケーションの構築（Streamlit）
- 効率的なワークフロー
- トラブルシューティング

**主なポイント:**
- OpenAI API互換のローカルサーバー
- 実践的なStreamlitアプリ例
- プロンプトテンプレート

#### [第9章：高度な機能とカスタマイズ](part1_lmstudio/chapter09_advanced_features.md)

- RAG（検索拡張生成）の実装
  - LangChainを使った実装
  - MS-S1 Max向け最適化
  - 複数ドキュメントの処理
- プロンプトエンジニアリングの高度なテクニック
  - Few-Shot Learning
  - Chain-of-Thought（思考の連鎖）
  - Self-Consistency
- マルチモーダル対応（将来の拡張）
  - 画像認識モデルとの連携
  - 音声認識との連携
- パフォーマンスの極限最適化
  - KVキャッシュの最適化
  - カスタムGGUFモデルの作成
  - バッチ処理の実装
- セキュリティとプライバシー
- コミュニティとエコシステム
- 今後の展開

**主なポイント:**
- 完全なRAGシステムの構築
- PyTorchモデルからGGUFへの変換
- ローカル実行のプライバシーメリット

## システム要件

### 推奨環境

**ハードウェア:**
- **CPU**: AMD Ryzen AI Max+ 395（16コア/32スレッド）
- **GPU**: Radeon 8060S（RDNA 3.5、統合）
- **メモリ**: 128GB LPDDR5X-8000
- **ストレージ**: 2TB NVMe SSD（デュアルM.2推奨）
- **システム**: Minisforum MS-S1 Max

**ソフトウェア:**
- **OS**: Windows 11 Pro または Ubuntu 24.04 LTS
- **LM Studio**: 0.3.19以降
- **ROCm**（Linux）: 6.2以降
- **AMD Driver**（Windows）: 最新版

### 最小要件

- CPU: AVX2対応プロセッサ
- メモリ: 16GB以上（32GB推奨）
- ストレージ: 100GB以上の空き容量
- GPU: オプション（AMD Radeon RX 5700以上）

## クイックスタートガイド

### 1. インストール

```bash
# Ubuntu 24.04の場合

# ROCmのインストール
sudo apt update && sudo apt upgrade -y
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | \
  gpg --dearmor | sudo tee /etc/apt/keyrings/rocm.gpg > /dev/null
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/6.2 noble main" \
  | sudo tee /etc/apt/sources.list.d/rocm.list
sudo apt update
sudo apt install -y rocm-hip-sdk rocm-libs

# 環境変数の設定
echo 'export HSA_OVERRIDE_GFX_VERSION=11.0.0' >> ~/.bashrc
source ~/.bashrc

# LM StudioのAppImageをダウンロードして実行
chmod +x LM_Studio-x.x.x.AppImage
./LM_Studio-x.x.x.AppImage
```

### 2. 推奨モデルのダウンロード

```
LM Studio起動 → Search → "qwen2.5 7b q4" と検索
→ qwen2.5-7b-instruct-q4_k_m.gguf をダウンロード
```

### 3. 最初の推論

```
Chat画面 → モデル選択 → qwen2.5-7b-instruct-q4_k_m
→ Load Model
→ "こんにちは！あなたは誰ですか？" と入力
```

## 推奨モデル（MS-S1 Max）

### 日常使用

| モデル | サイズ | 速度 | 用途 |
|--------|--------|------|------|
| Qwen2.5 7B Q4_K_M | 4.8GB | 35-45 t/s | 日常チャット |
| Llama 3.2 3B Q4_K_M | 2.0GB | 50-60 t/s | 高速応答 |

### プロフェッショナル

| モデル | サイズ | 速度 | 用途 |
|--------|--------|------|------|
| Qwen2.5 32B Q4_K_M | 20GB | 8-12 t/s | 高品質文章 |
| DeepSeek-Coder V2 16B | 10GB | 18-22 t/s | コーディング |

### 最高品質

| モデル | サイズ | 速度 | 用途 |
|--------|--------|------|------|
| Llama 3.1 70B Q4_K_M | 42GB | 3-5 t/s | 最高品質推論 |
| Qwen2.5 72B Q4_K_M | 44GB | 3-5 t/s | 日本語最高品質 |

## 推奨設定

### Balance構成（最推奨）

```yaml
ハードウェア:
  性能モード: Balance (130W)
  温度目標: 70-80℃

モデル:
  日常用: Qwen2.5 7B Q4_K_M
  高品質用: Qwen2.5 32B Q4_K_M

LM Studio設定:
  GPU Layers: 最大
  Context Length: 16384
  Temperature: 0.7
  Top P: 0.95
  Repeat Penalty: 1.1
  Flash Attention: ON
```

## トラブルシューティング

### GPUが認識されない

**Windows:**
```
1. AMD Softwareを最新版に更新
2. デバイスマネージャーでドライバ確認
3. LM Studio再起動
```

**Linux:**
```bash
# ROCm確認
rocm-smi

# 環境変数確認
echo $HSA_OVERRIDE_GFX_VERSION  # 11.0.0であるべき

# グループ確認
groups | grep render  # renderが含まれるべき
```

### 推論速度が遅い

```
1. GPU使用率を確認（70%以上であるべき）
2. GPU Layers設定を確認（最大値に設定）
3. 温度を確認（85℃以下であるべき）
4. バックグラウンドアプリを終了
```

## FAQ

**Q: 本書はAMD専用ですか？**
A: 主にAMD Ryzen AI Max+ 395 / MS-S1 Maxを対象としていますが、他のAMD GPU（RX 7000シリーズ等）やNVIDIA GPU、Apple Siliconでも応用できます。

**Q: LM Studioは無料ですか？**
A: はい、LM Studioは完全に無料です。

**Q: どのモデルを最初に試すべきですか？**
A: Qwen2.5 7B Q4_K_Mを推奨します。日本語に強く、バランスが良いモデルです。

**Q: 70Bモデルは実用的ですか？**
A: MS-S1 Maxでは、70B Q4_K_Mモデルを3-5 tokens/sで実行できます。これは読書速度に近く、実用的です。

**Q: インターネット接続は必要ですか？**
A: モデルのダウンロード時のみ必要です。推論実行中は完全にオフラインで動作します。

## 第二部以降の予定

- **第二部**: Ollama完全ガイド
- **第三部**: テキスト生成WebUI（Oobabooga）
- **第四部**: ComfyUIとStable Diffusion
- **第五部**: ローカルAIアプリケーション開発

## リソース

### 公式リンク

- **LM Studio公式**: https://lmstudio.ai/
- **LM Studio Discord**: https://discord.gg/lmstudio
- **Hugging Face**: https://huggingface.co/
- **AMD ROCm**: https://rocm.docs.amd.com/

### コミュニティ

- **Reddit r/LocalLLaMA**: ローカルLLMコミュニティ
- **Reddit r/LMStudio**: LM Studio専用
- **GitHub**: モデルとツールのリポジトリ

## ライセンスと注意事項

### 本書について

本書は情報提供を目的としています。実際の性能は環境やモデルにより異なる場合があります。

### モデルのライセンス

各モデルには独自のライセンスがあります。商用利用前に必ず確認してください。

- Llama 3: Llama 3 Community License
- Qwen: Apache 2.0
- Mistral: Apache 2.0

## 変更履歴

- **v1.0.0** (2025-10-30): 初版リリース

## 著者

- **制作**: Claude（Anthropic）
- **協力**: Claude Code
- **監修**: コミュニティフィードバック

---

**© 2025 - すべての権利を保有**

本書の内容を最大限に活用して、ローカルAIの世界を楽しんでください！
