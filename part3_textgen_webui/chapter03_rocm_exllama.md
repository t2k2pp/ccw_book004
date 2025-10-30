# 第3章:ROCm設定とExLlamaV2最適化

## 3.1 ExLlamaV2とは

ExLlamaV2は、GPTQ/EXL2量子化モデル専用の超高速推論エンジンです。

### MS-S1 Maxでの優位性

```
速度比較（70Bモデル、Q4量子化）:
- Transformers: 2-3 tokens/s
- llama.cpp: 4-6 tokens/s
- ExLlamaV2: 8-12 tokens/s（2-3倍高速！）
```

## 3.2 ROCm最適化

```bash
# 環境変数
export HSA_OVERRIDE_GFX_VERSION=11.0.0
export ROCM_HOME=/opt/rocm
export GPU_MAX_HEAP_SIZE=100
export GPU_MAX_ALLOC_PERCENT=100
export HSA_FORCE_FINE_GRAIN_PCIE=1
```

## 3.3 ExLlamaV2のインストール

```bash
cd ~/text-generation-webui
source venv/bin/activate

pip install exllamav2 --no-build-isolation --extra-index-url https://download.pytorch.org/whl/rocm6.2
```

## 3.4 settings.yamlの最適化

```yaml
# ~/text-generation-webui/settings.yaml
loader: exllamav2
gpu_memory:
  - 96  # MS-S1 Max: 96GB
cpu_memory: 32

# ExLlamaV2固有
max_seq_len: 32768
compress_pos_emb: 1.0
alpha_value: 1.0
rope_freq_base: 0
cache_8bit: False  # 128GBあるので無効化
```

## 3.5 パフォーマンステスト

```python
# test_performance.py
import time

def benchmark():
    # WebUI APIを使用
    import requests
    
    start = time.time()
    response = requests.post('http://localhost:5000/api/v1/generate', json={
        'prompt': 'Write a 200 word essay',
        'max_new_tokens': 200
    })
    elapsed = time.time() - start
    
    print(f"Time: {elapsed:.2f}s")
    print(f"Speed: {200/elapsed:.1f} tokens/s")

benchmark()
```

## 3.6 トラブルシューティング

### GPU未認識
```bash
# ROCm確認
rocm-smi

# 環境変数確認
env | grep HSA
```

### メモリエラー
```yaml
# settings.yaml を調整
gpu_memory:
  - 80  # 96から削減
```

## 3.7 本章のまとめ

✅ ExLlamaV2の特徴と優位性
✅ ROCm最適化設定
✅ パフォーマンステスト
✅ トラブルシューティング

---

**前章へ**: [第2章 インストールと環境構築](chapter02_installation.md)
**次章へ**: [第4章 基本操作とインターフェース](chapter04_basic_usage.md)
