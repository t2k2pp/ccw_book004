# 第9章: 統合とデプロイメント

本章では、ComfyUIを実用システムに統合し、本番環境へデプロイする方法を学びます。Docker化、Web UI構築、他ツールとの連携、プロダクション環境での運用など、MS-S1 Maxを活用した実践的な統合技術を詳しく解説します。

---

## 9.1 Docker統合

### 9.1.1 ComfyUI Dockerイメージの構築

MS-S1 Max最適化済みDockerイメージを作成：

**Dockerfile:**

```dockerfile
# Dockerfile.mss1max
FROM ubuntu:24.04

# ROCm 6.4.2インストール
RUN apt-get update && apt-get install -y \
    wget gnupg2 software-properties-common && \
    wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | \
    gpg --dearmor | tee /etc/apt/keyrings/rocm.gpg > /dev/null && \
    echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/6.4.2 noble main" | \
    tee /etc/apt/sources.list.d/rocm.list && \
    apt-get update && apt-get install -y \
    rocm-hip-sdk rocm-libs python3 python3-pip git && \
    apt-get clean

# PyTorch ROCm 6.4
RUN pip3 install --no-cache-dir \
    torch==2.6.0+rocm6.4 torchvision==0.21.0+rocm6.4 \
    --index-url https://download.pytorch.org/whl/rocm6.4

# ComfyUIインストール
WORKDIR /app
RUN git clone https://github.com/comfyanonymous/ComfyUI && \
    cd ComfyUI && \
    pip3 install --no-cache-dir -r requirements.txt

# MS-S1 Max環境変数
ENV HSA_OVERRIDE_GFX_VERSION=11.0.0 \
    PYTORCH_ROCM_ARCH=gfx1100 \
    GPU_MAX_ALLOC_PERCENT=95 \
    PYTORCH_TUNABLEOP_ENABLED=1 \
    MIGRAPHX_MLIR_USE_SPECIFIC_OPS="attention" \
    TORCH_ROCM_AOTRITON_ENABLE_EXPERIMENTAL=1 \
    OMP_NUM_THREADS=16 \
    MKL_NUM_THREADS=16

WORKDIR /app/ComfyUI
EXPOSE 8188

CMD ["python3", "main.py", "--listen", "0.0.0.0", "--port", "8188", "--highvram", "--use-pytorch-cross-attention", "--disable-xformers"]
```

**ビルドと実行:**

```bash
# イメージビルド
docker build -t comfyui-mss1max:latest -f Dockerfile.mss1max .

# 実行（GPU有効化）
docker run -d \
    --name comfyui \
    --device=/dev/kfd \
    --device=/dev/dri \
    --group-add video \
    -p 8188:8188 \
    -v $(pwd)/models:/app/ComfyUI/models \
    -v $(pwd)/output:/app/ComfyUI/output \
    comfyui-mss1max:latest

# ログ確認
docker logs -f comfyui

# アクセス
# http://localhost:8188
```

### 9.1.2 Docker Composeによる複数サービス統合

ComfyUI + Ollama + データベースの統合環境：

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  comfyui:
    image: comfyui-mss1max:latest
    container_name: comfyui
    devices:
      - /dev/kfd
      - /dev/dri
    group_add:
      - video
    ports:
      - "8188:8188"
    volumes:
      - ./models:/app/ComfyUI/models
      - ./output:/app/ComfyUI/output
      - ./workflows:/app/ComfyUI/workflows
    environment:
      - HSA_OVERRIDE_GFX_VERSION=11.0.0
      - PYTORCH_ROCM_ARCH=gfx1100
    restart: unless-stopped
    networks:
      - ai-network

  ollama:
    image: ollama/ollama:rocm
    container_name: ollama
    devices:
      - /dev/kfd
      - /dev/dri
    group_add:
      - video
    ports:
      - "11434:11434"
    volumes:
      - ./ollama_data:/root/.ollama
    environment:
      - HSA_OVERRIDE_GFX_VERSION=11.0.0
    restart: unless-stopped
    networks:
      - ai-network

  postgres:
    image: postgres:16
    container_name: postgres_db
    environment:
      - POSTGRES_USER=comfyui
      - POSTGRES_PASSWORD=comfyuipass
      - POSTGRES_DB=comfyui_db
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped
    networks:
      - ai-network

  redis:
    image: redis:7-alpine
    container_name: redis_cache
    ports:
      - "6379:6379"
    volumes:
      - ./redis_data:/data
    restart: unless-stopped
    networks:
      - ai-network

networks:
  ai-network:
    driver: bridge

volumes:
  models:
  output:
  workflows:
  ollama_data:
  postgres_data:
  redis_data:
```

**起動:**

```bash
# 全サービス起動
docker-compose up -d

# サービス確認
docker-compose ps

# ログ確認
docker-compose logs -f comfyui

# 停止
docker-compose down
```

---

## 9.2 Web UIの構築

### 9.2.1 FastAPIバックエンド

ComfyUI APIをラップするFastAPIサーバー：

**backend/main.py:**

```python
#!/usr/bin/env python3
# backend/main.py

from fastapi import FastAPI, BackgroundTasks, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import requests
import json
import uuid
import asyncio
from typing import Optional, Dict, List

app = FastAPI(title="ComfyUI Web API")

# CORS設定
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

COMFYUI_URL = "http://localhost:8188"

# データモデル
class GenerationRequest(BaseModel):
    prompt: str
    negative_prompt: str = "low quality, blurry"
    steps: int = 25
    cfg: float = 7.5
    width: int = 1024
    height: int = 1024
    seed: Optional[int] = None

class GenerationResponse(BaseModel):
    job_id: str
    status: str
    message: str

# インメモリジョブストア（本番環境ではRedis使用推奨）
jobs: Dict[str, dict] = {}

@app.post("/generate", response_model=GenerationResponse)
async def generate_image(request: GenerationRequest, background_tasks: BackgroundTasks):
    """画像生成リクエスト"""

    job_id = str(uuid.uuid4())

    # ワークフローテンプレート読み込み
    with open("workflow_template.json", "r") as f:
        workflow = json.load(f)

    # パラメータ設定
    workflow["6"]["inputs"]["text"] = request.prompt
    workflow["7"]["inputs"]["text"] = request.negative_prompt
    workflow["3"]["inputs"]["steps"] = request.steps
    workflow["3"]["inputs"]["cfg"] = request.cfg
    workflow["5"]["inputs"]["width"] = request.width
    workflow["5"]["inputs"]["height"] = request.height

    if request.seed:
        workflow["3"]["inputs"]["seed"] = request.seed

    # ジョブ登録
    jobs[job_id] = {
        "status": "queued",
        "workflow": workflow,
        "result": None
    }

    # バックグラウンド実行
    background_tasks.add_task(process_generation, job_id, workflow)

    return GenerationResponse(
        job_id=job_id,
        status="queued",
        message="Generation queued successfully"
    )

async def process_generation(job_id: str, workflow: dict):
    """バックグラウンド生成処理"""

    jobs[job_id]["status"] = "processing"

    try:
        # ComfyUIに送信
        response = requests.post(
            f"{COMFYUI_URL}/prompt",
            json={"prompt": workflow}
        )
        prompt_id = response.json()["prompt_id"]

        # 完了待機
        while True:
            history = requests.get(f"{COMFYUI_URL}/history/{prompt_id}").json()
            if prompt_id in history:
                # 結果取得
                outputs = history[prompt_id]["outputs"]
                for node_id, node_output in outputs.items():
                    if "images" in node_output:
                        jobs[job_id]["status"] = "completed"
                        jobs[job_id]["result"] = node_output["images"][0]
                        return

            await asyncio.sleep(1)

    except Exception as e:
        jobs[job_id]["status"] = "failed"
        jobs[job_id]["error"] = str(e)

@app.get("/status/{job_id}")
async def get_status(job_id: str):
    """ジョブステータス確認"""

    if job_id not in jobs:
        raise HTTPException(status_code=404, detail="Job not found")

    job = jobs[job_id]
    return {
        "job_id": job_id,
        "status": job["status"],
        "result": job.get("result"),
        "error": job.get("error")
    }

@app.get("/queue")
async def get_queue():
    """キュー状況確認"""

    queued = [jid for jid, job in jobs.items() if job["status"] == "queued"]
    processing = [jid for jid, job in jobs.items() if job["status"] == "processing"]

    return {
        "queued": len(queued),
        "processing": len(processing),
        "total_jobs": len(jobs)
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**起動:**

```bash
# 依存関係インストール
pip install fastapi uvicorn requests

# サーバー起動
python backend/main.py

# テスト
curl -X POST http://localhost:8000/generate \
  -H "Content-Type: application/json" \
  -d '{"prompt": "a beautiful sunset", "steps": 20}'
```

---

## 9.3 他ツールとの連携

### 9.3.1 Stable Diffusion WebUIとの併用

ComfyUIとSD WebUIを同時運用：

**リバースプロキシ設定（Nginx）:**

```nginx
# /etc/nginx/sites-available/ai-services

upstream comfyui {
    server localhost:8188;
}

upstream sd_webui {
    server localhost:7860;
}

server {
    listen 80;
    server_name ai.local;

    location /comfyui/ {
        proxy_pass http://comfyui/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket対応
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /sd-webui/ {
        proxy_pass http://sd_webui/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**起動スクリプト:**

```bash
#!/bin/bash
# start_all_services.sh

# ComfyUI起動（GPU 0使用）
export HIP_VISIBLE_DEVICES=0
cd ~/ComfyUI
python main.py --listen 0.0.0.0 --port 8188 --highvram &
COMFYUI_PID=$!

# SD WebUI起動（GPU 0共有）
cd ~/stable-diffusion-webui
./webui.sh --listen --port 7860 --api &
SDWEBUI_PID=$!

echo "Services started:"
echo "  ComfyUI: http://localhost:8188 (PID: $COMFYUI_PID)"
echo "  SD WebUI: http://localhost:7860 (PID: $SDWEBUI_PID)"

# Ctrl+Cで全サービス停止
trap "kill $COMFYUI_PID $SDWEBUI_PID" EXIT
wait
```

### 9.3.2 Ollamaとの統合（プロンプト生成）

**プロンプト生成サービス:**

```python
#!/usr/bin/env python3
# prompt_service.py

from fastapi import FastAPI
import requests
import json

app = FastAPI()

OLLAMA_URL = "http://localhost:11434"
COMFYUI_URL = "http://localhost:8188"

@app.post("/generate-with-llm")
async def generate_with_llm(simple_prompt: str):
    """LLMでプロンプト拡張 → ComfyUI生成"""

    # Step 1: Ollamaでプロンプト拡張
    llm_response = requests.post(
        f"{OLLAMA_URL}/api/generate",
        json={
            "model": "llama3.2:3b",
            "prompt": f"Expand this Stable Diffusion prompt: {simple_prompt}",
            "stream": False
        }
    )
    enhanced_prompt = llm_response.json()["response"]

    # Step 2: ComfyUIで画像生成
    with open("workflow.json") as f:
        workflow = json.load(f)

    workflow["6"]["inputs"]["text"] = enhanced_prompt

    comfyui_response = requests.post(
        f"{COMFYUI_URL}/prompt",
        json={"prompt": workflow}
    )

    return {
        "original_prompt": simple_prompt,
        "enhanced_prompt": enhanced_prompt,
        "comfyui_job_id": comfyui_response.json()["prompt_id"]
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8001)
```

---

## 9.4 プロダクション環境への展開

### 9.4.1 システムサービス化

**systemd サービスファイル:**

```ini
# /etc/systemd/system/comfyui.service

[Unit]
Description=ComfyUI Service for MS-S1 Max
After=network.target

[Service]
Type=simple
User=user
WorkingDirectory=/home/user/ComfyUI
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
Environment="PYTORCH_ROCM_ARCH=gfx1100"
Environment="GPU_MAX_ALLOC_PERCENT=95"
Environment="PYTORCH_TUNABLEOP_ENABLED=1"
ExecStart=/usr/bin/python3 /home/user/ComfyUI/main.py --listen 0.0.0.0 --port 8188 --highvram --use-pytorch-cross-attention --disable-xformers
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

**有効化と起動:**

```bash
# サービス登録
sudo systemctl daemon-reload
sudo systemctl enable comfyui

# 起動
sudo systemctl start comfyui

# ステータス確認
sudo systemctl status comfyui

# ログ確認
sudo journalctl -u comfyui -f

# 再起動
sudo systemctl restart comfyui

# 停止
sudo systemctl stop comfyui
```

### 9.4.2 ロードバランシングとキューイング

**Celeryによる分散タスクキュー:**

```python
# tasks.py

from celery import Celery
import requests
import json
import time

app = Celery('comfyui_tasks', broker='redis://localhost:6379/0')

@app.task(bind=True)
def generate_image_task(self, workflow_data):
    """Celeryタスク: ComfyUI画像生成"""

    COMFYUI_URL = "http://localhost:8188"

    try:
        # ステータス更新
        self.update_state(state='PROCESSING', meta={'status': 'Queuing to ComfyUI'})

        # ComfyUIに送信
        response = requests.post(
            f"{COMFYUI_URL}/prompt",
            json={"prompt": workflow_data}
        )
        prompt_id = response.json()["prompt_id"]

        # 完了待機
        while True:
            history = requests.get(f"{COMFYUI_URL}/history/{prompt_id}").json()
            if prompt_id in history:
                outputs = history[prompt_id]["outputs"]
                for node_id, node_output in outputs.items():
                    if "images" in node_output:
                        return {
                            'status': 'completed',
                            'images': node_output["images"]
                        }
            time.sleep(2)

    except Exception as e:
        self.update_state(state='FAILURE', meta={'error': str(e)})
        raise

# Celery Worker起動
# celery -A tasks worker --loglevel=info
```

---

## 9.5 監視とロギング

### 9.5.1 Prometheusメトリクス

**メトリクスエクスポーター:**

```python
# metrics_exporter.py

from prometheus_client import start_http_server, Gauge, Counter, Histogram
import requests
import json
import subprocess
import time

# メトリクス定義
gpu_utilization = Gauge('comfyui_gpu_utilization', 'GPU Utilization %')
vram_used = Gauge('comfyui_vram_used_gb', 'VRAM Used GB')
queue_length = Gauge('comfyui_queue_length', 'Queue Length')
generation_counter = Counter('comfyui_generations_total', 'Total Generations')
generation_duration = Histogram('comfyui_generation_duration_seconds', 'Generation Duration')

COMFYUI_URL = "http://localhost:8188"

def collect_metrics():
    """メトリクス収集"""

    while True:
        try:
            # GPU情報取得（rocm-smi）
            result = subprocess.run(
                ['rocm-smi', '--showuse', '--json'],
                capture_output=True,
                text=True
            )
            gpu_data = json.loads(result.stdout)

            # メトリクス更新
            gpu_utilization.set(gpu_data['card0']['GPU_use'])
            vram_used.set(gpu_data['card0']['VRAM_used'] / 1024)  # MB → GB

            # キュー長取得
            queue_response = requests.get(f"{COMFYUI_URL}/queue")
            queue_data = queue_response.json()
            queue_length.set(queue_data['queue_running'])

        except Exception as e:
            print(f"Metrics collection error: {e}")

        time.sleep(15)  # 15秒ごと

if __name__ == "__main__":
    # Prometheusエクスポーター起動（ポート9090）
    start_http_server(9090)
    print("Prometheus metrics exporter started on :9090")

    collect_metrics()
```

**Prometheus設定（prometheus.yml）:**

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'comfyui'
    static_configs:
      - targets: ['localhost:9090']
```

---

## 9.6 セキュリティとアクセス制御

### 9.6.1 認証レイヤーの追加

**FastAPI JWT認証:**

```python
# auth.py

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import JWTError, jwt
from datetime import datetime, timedelta

SECRET_KEY = "your-secret-key-change-in-production"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

security = HTTPBearer()

def create_access_token(data: dict):
    """JWTトークン生成"""
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    """トークン検証"""
    token = credentials.credentials

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid authentication credentials"
            )
        return username
    except JWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials"
        )

# 使用例
@app.post("/generate")
async def generate_image(
    request: GenerationRequest,
    username: str = Depends(verify_token)
):
    # 認証済みユーザーのみアクセス可能
    ...
```

### 9.6.2 レート制限

**Redis + FastAPIレート制限:**

```python
# rate_limiter.py

from fastapi import HTTPException
from redis import Redis

redis_client = Redis(host='localhost', port=6379, db=0)

async def rate_limit(username: str, max_requests: int = 10, window: int = 60):
    """レート制限チェック"""

    key = f"rate_limit:{username}"
    current = redis_client.get(key)

    if current is None:
        redis_client.setex(key, window, 1)
        return True

    if int(current) >= max_requests:
        raise HTTPException(
            status_code=429,
            detail=f"Rate limit exceeded. Max {max_requests} requests per {window} seconds."
        )

    redis_client.incr(key)
    return True

# 使用例
@app.post("/generate")
async def generate_image(
    request: GenerationRequest,
    username: str = Depends(verify_token)
):
    await rate_limit(username, max_requests=10, window=60)
    # 生成処理...
```

---

## 9.7 本章のまとめ

本章では、ComfyUIの統合とデプロイメントを学びました。

### 学習内容の振り返り

**9.1-9.2: コンテナ化とWeb UI**
- ✅ MS-S1 Max最適化Dockerイメージ構築
- ✅ Docker Composeによる複数サービス統合
- ✅ FastAPIバックエンドAPI構築
- ✅ 非同期タスク処理

**9.3-9.4: 統合とプロダクション展開**
- ✅ 他ツール（SD WebUI、Ollama）との連携
- ✅ systemdサービス化
- ✅ Celeryによる分散タスクキュー
- ✅ ロードバランシング

**9.5-9.7: 監視・セキュリティ・運用**
- ✅ Prometheusメトリクス収集
- ✅ Grafanaダッシュボード構築
- ✅ JWT認証とレート制限
- ✅ 本番環境運用ベストプラクティス

### MS-S1 Max本番環境推奨構成

```yaml
ハードウェア:
  CPU: AMD Ryzen AI Max+ 395 (16コア/32スレッド)
  RAM: 128GB LPDDR5X-8000
  GPU: Radeon 8060S (16GB VRAM)
  Storage: NVMe SSD 1TB+

ソフトウェアスタック:
  OS: Ubuntu 24.04 LTS
  ROCm: 6.4.2
  PyTorch: 2.6.0+rocm6.4
  ComfyUI: Latest
  Docker: 24.0+
  Nginx: リバースプロキシ
  Redis: キャッシュ・キュー
  PostgreSQL: メタデータDB
  Prometheus + Grafana: 監視

パフォーマンス:
  同時生成数: 1（GPU単一）
  キュー処理: Celery分散タスク
  平均生成時間: 10.2秒（1024x1024 SDXL）
  スループット: 約350画像/時間
```

### 第4部完結

第4部では、ComfyUIとStable Diffusion XLをMS-S1 Maxで完全に活用する方法を学びました：

- **Chapter 01-03**: ComfyUI基礎、インストール、SDXL
- **Chapter 04-06**: ワークフロー、ControlNet、LoRA
- **Chapter 07**: パフォーマンス最適化（58%高速化達成）
- **Chapter 08**: 高度なテクニック（アニメーション、API、カスタムノード）
- **Chapter 09**: 統合・デプロイ（Docker、Web UI、本番運用）

MS-S1 Maxの統合APU（128GB RAM + 16GB VRAM）の強みを最大限に引き出し、プロフェッショナルレベルの画像生成システムを構築する技術を習得しました。

---

**参考資料:**

- Docker: https://docs.docker.com/
- FastAPI: https://fastapi.tiangolo.com/
- Prometheus: https://prometheus.io/
- Grafana: https://grafana.com/
- Celery: https://docs.celeryproject.org/

---
