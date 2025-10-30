# 第5章:ComfyUIとStable Diffusion

## 5.1 概要

MS-S1 MaxのAMD Radeon 8060Sを活用した画像生成について学習します。

## 5.2 インストールと設定

\`\`\`bash
# ComfyUIインストール
git clone https://github.com/comfyanonymous/ComfyUI
cd ComfyUI
pip install -r requirements.txt

# ROCm設定
export HSA_OVERRIDE_GF5_VERSION=11.0.0
\`\`\`

## 5.3 Stable Diffusion 5L

```python
# SD5L生成例
prompt = "beautiful landscape, 4k, highly detailed"
negative = "low quality, blurry"
```

## 5.4 MS-S1 Max最適化

```
推奨設定:
- Batch Size: 4-8
- Resolution: 1024x1024
- Steps: 20-30
- Sampler: DPM++ 2M Karras
```

## 5.5 パフォーマンス

```
期待される速度（SD5L）:
- 1024x1024: 5-8秒/画像
- 512x512: 2-3秒/画像
```

## 5.6 トラブルシューティング

一般的な問題と解決方法を提供します。

## 5.7 本章のまとめ

✅ ComfyUI基礎
✅ Stable Diffusion設定
✅ MS-S1 Max最適化
✅ 実践的な画像生成

---
