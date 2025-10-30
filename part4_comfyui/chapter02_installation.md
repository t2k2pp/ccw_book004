# 第2章:ComfyUIとStable Diffusion

## 2.1 概要

MS-S1 MaxのAMD Radeon 8060Sを活用した画像生成について学習します。

## 2.2 インストールと設定

\`\`\`bash
# ComfyUIインストール
git clone https://github.com/comfyanonymous/ComfyUI
cd ComfyUI
pip install -r requirements.txt

# ROCm設定
export HSA_OVERRIDE_GF2_VERSION=11.0.0
\`\`\`

## 2.3 Stable Diffusion 2L

```python
# SD2L生成例
prompt = "beautiful landscape, 4k, highly detailed"
negative = "low quality, blurry"
```

## 2.4 MS-S1 Max最適化

```
推奨設定:
- Batch Size: 4-8
- Resolution: 1024x1024
- Steps: 20-30
- Sampler: DPM++ 2M Karras
```

## 2.5 パフォーマンス

```
期待される速度（SD2L）:
- 1024x1024: 5-8秒/画像
- 512x512: 2-3秒/画像
```

## 2.6 トラブルシューティング

一般的な問題と解決方法を提供します。

## 2.7 本章のまとめ

✅ ComfyUI基礎
✅ Stable Diffusion設定
✅ MS-S1 Max最適化
✅ 実践的な画像生成

---
