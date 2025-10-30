# 第5章:モデルローダーとフォーマット

## 5.1 ローダーの種類

### 対応ローダー

```python
loaders = {
    'Transformers': '標準HuggingFace',
    'llama.cpp': 'GGUF形式',
    'ExLlamaV2': 'GPTQ/EXL2（最速）',
    'AutoGPTQ': 'GPTQ量子化',
    'AutoAWQ': 'AWQ量子化',
    'HQQ': '高品質量子化'
}
```

## 5.2 モデルフォーマット選択

### MS-S1 Max推奨

```
7B-13Bモデル:
├── ExLlamaV2 (EXL2) - 最速
└── llama.cpp (GGUF Q5) - バランス

32B-70Bモデル:
├── ExLlamaV2 (EXL2 4.0bpw) - 推奨
└── llama.cpp (GGUF Q4_K_M) - 代替
```

## 5.3 量子化レベル

### GGUF量子化

```
Q8_0:  8bit - 最高品質
Q6_K:  6bit - 高品質
Q5_K_M: 5bit - 推奨
Q4_K_M: 4bit - バランス
Q3_K_M: 3bit - コンパクト
```

### EXL2量子化

```
8.0 bpw: 最高品質
6.5 bpw: 高品質
5.0 bpw: 推奨（70B可能）
4.0 bpw: コンパクト（70B推奨）
3.0 bpw: 超コンパクト
```

## 5.4 モデルダウンロード

### 推奨モデル

```bash
# 7B 汎用（日本語）
python download-model.py elyza/ELYZA-japanese-Llama-2-7b-fast-instruct

# 13B バランス
python download-model.py TheBloke/Nous-Hermes-2-SOLAR-10.7B-GGUF

# 32B 高性能
python download-model.py TheBloke/Yi-34B-200K-GGUF

# 70B 最高性能
python download-model.py TheBloke/Llama-2-70B-chat-GGUF
```

## 5.5 モデルの切り替え

### Web UI

```
1. "Model" タブ
2. ドロップダウンからモデル選択
3. "Load" ボタンクリック
4. ローダー選択（自動検出）
5. ロード完了を待つ
```

### コマンドライン

```bash
# 起動時に指定
python server.py --model llama-2-70b-chat.Q4_K_M.gguf --loader llama.cpp
```

## 5.6 ローダー別最適化

### ExLlamaV2設定

```yaml
loader: exllamav2
max_seq_len: 32768
cache_8bit: False
flash_attn: True
```

### llama.cpp設定

```yaml
loader: llama.cpp
n_ctx: 8192
n_batch: 512
n_gpu_layers: -1  # 全レイヤーGPU
```

## 5.7 本章のまとめ

✅ ローダーの種類と特徴
✅ 量子化レベルの選択
✅ MS-S1 Max推奨設定
✅ モデルダウンロードと管理

---

**前章へ**: [第4章 基本操作とインターフェース](chapter04_basic_usage.md)
**次章へ**: [第6章 高度なパラメータ設定](chapter06_advanced_parameters.md)
