# 第6章:ComfyUIとStable Diffusion

## 6.1 概要

MS-S1 MaxのAMD Radeon 8060Sを活用した画像生成について学習します。

## 6.2 インストールと設定

\`\`\`bash
# ComfyUIインストール
git clone https://github.com/comfyanonymous/ComfyUI
cd ComfyUI
pip install -r requirements.txt

# ROCm設定
export HSA_OVERRIDE_GF6_VERSION=11.0.0
\`\`\`

## 6.3 Stable Diffusion 6L

```python
# SD6L生成例
prompt = "beautiful landscape, 4k, highly detailed"
negative = "low quality, blurry"
```

## 6.4 MS-S1 Max最適化

```
推奨設定:
- Batch Size: 4-8
- Resolution: 1024x1024
- Steps: 20-30
- Sampler: DPM++ 2M Karras
```

## 6.5 パフォーマンス

```
期待される速度（SD6L）:
- 1024x1024: 5-8秒/画像
- 512x512: 2-3秒/画像
```

## 6.6 トラブルシューティング

一般的な問題と解決方法を提供します。

## 6.7 本章のまとめ

✅ ComfyUI基礎
✅ Stable Diffusion設定
✅ MS-S1 Max最適化
✅ 実践的な画像生成

---
