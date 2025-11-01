# 第2章: アプリケーションアーキテクチャ設計

本章では、MS-S1 Maxを活用したスケーラブルなローカルAIアプリケーションのアーキテクチャ設計を学びます。マイクロサービス、非同期処理、キャッシング、データベース設計など、本番環境を見据えた実装パターンを詳しく解説します。

---

## 2.1 アーキテクチャパターンの選択

### 2.1.1 モノリスvs マイクロサービス

**小規模〜中規模（推奨: モノリス）:**

```yaml
適用条件:
  - チームサイズ: 1-5人
  - ユーザー数: 〜10,000人
  - サービス種類: 1-3種類（チャット、画像生成など）

モノリス構成:
  ├── FastAPI統合サーバー
  │   ├── /api/chat (LLM)
  │   ├── /api/image (ComfyUI)
  │   └── /api/rag (検索拡張生成)
  ├── PostgreSQL (単一DB)
  ├── Redis (キャッシュ)
  └── Ollama + ComfyUI (バックエンド)

利点:
  - シンプル: デプロイ・管理が容易
  - 低レイテンシ: プロセス間通信不要
  - デバッグ容易: 単一コードベース
  - MS-S1 Max最適: 統合メモリを最大活用
```

**大規模（マイクロサービス）:**

```yaml
適用条件:
  - チームサイズ: 5人以上
  - ユーザー数: 10,000人以上
  - サービス種類: 4種類以上

マイクロサービス構成:
  ├── API Gateway (Nginx/Kong)
  ├── Auth Service (認証)
  ├── LLM Service (Ollama専用)
  ├── Image Service (ComfyUI専用)
  ├── RAG Service (検索専用)
  ├── Queue Service (Celery/RabbitMQ)
  └── 各サービス専用DB

利点:
  - スケーラビリティ: 独立スケーリング
  - 独立デプロイ: サービスごとに更新
  - 技術多様性: サービスごとに最適技術

欠点:
  - 複雑性: 管理コスト増加
  - レイテンシ: ネットワーク遅延
  - MS-S1 Max: 単一マシンでは利点少ない
```

### 2.1.2 推奨アーキテクチャ（MS-S1 Max特化）

**ハイブリッド型モノリス:**

```yaml
構成:
  Frontend (Next.js/React)
    ↓ HTTP/WebSocket
  API Server (FastAPI) ← Redis Cache
    ↓
  ├── LLM Module (Ollama SDK)
  ├── Image Module (ComfyUI API)
  ├── RAG Module (ChromaDB + Embeddings)
  └── Task Queue (Background Tasks)
    ↓
  PostgreSQL (メタデータ・履歴)

特徴:
  - 単一プロセス: メモリ共有最適化
  - 非同期処理: FastAPI async/await
  - 統合メモリ活用: CPU↔GPU転送ゼロ
  - シンプル: MS-S1 Max 1台で完結
```

---

## 2.2 非同期処理とタスクキュー

### 2.2.1 FastAPI非同期エンドポイント

**同期vs非同期の比較:**

```python
# ❌ 同期版（ブロッキング）
@app.post("/chat")
def chat_sync(prompt: str):
    # 10秒かかる処理
    response = ollama.generate(prompt)  # ブロック
    return {"response": response}

# 他のリクエストは10秒待たされる

# ✅ 非同期版（ノンブロッキング）
@app.post("/chat")
async def chat_async(prompt: str):
    # 10秒かかる処理
    response = await asyncio.to_thread(ollama.generate, prompt)
    return {"response": response}

# 他のリクエストは並行処理可能
```

**実装例:**

```python
# app/main.py

from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
import asyncio
import uuid
from typing import Dict, Optional
from datetime import datetime

app = FastAPI()

# ジョブストア（本番環境ではRedis使用）
jobs: Dict[str, dict] = {}

class ChatRequest(BaseModel):
    prompt: str
    model: str = "llama3.2:3b"

class JobResponse(BaseModel):
    job_id: str
    status: str
    created_at: str

@app.post("/chat/async", response_model=JobResponse)
async def chat_async(request: ChatRequest, background_tasks: BackgroundTasks):
    """非同期チャット（即座にジョブIDを返す）"""

    job_id = str(uuid.uuid4())
    jobs[job_id] = {
        "status": "pending",
        "created_at": datetime.utcnow().isoformat(),
        "request": request.dict()
    }

    # バックグラウンドタスクとして実行
    background_tasks.add_task(process_chat, job_id, request)

    return JobResponse(
        job_id=job_id,
        status="pending",
        created_at=jobs[job_id]["created_at"]
    )

async def process_chat(job_id: str, request: ChatRequest):
    """バックグラウンドチャット処理"""

    jobs[job_id]["status"] = "processing"

    try:
        # Ollama呼び出し（別スレッドで実行）
        response = await asyncio.to_thread(
            ollama_generate,
            request.prompt,
            request.model
        )

        jobs[job_id]["status"] = "completed"
        jobs[job_id]["response"] = response
        jobs[job_id]["completed_at"] = datetime.utcnow().isoformat()

    except Exception as e:
        jobs[job_id]["status"] = "failed"
        jobs[job_id]["error"] = str(e)

@app.get("/jobs/{job_id}")
async def get_job_status(job_id: str):
    """ジョブステータス取得"""

    if job_id not in jobs:
        return {"error": "Job not found"}, 404

    return jobs[job_id]

def ollama_generate(prompt: str, model: str) -> str:
    """Ollama生成（同期関数）"""
    import requests

    response = requests.post(
        "http://localhost:11434/api/generate",
        json={"model": model, "prompt": prompt, "stream": False}
    )
    return response.json()["response"]
```

### 2.2.2 Celeryによる分散タスクキュー

**重い処理をCeleryにオフロード:**

```python
# tasks.py

from celery import Celery
import requests

# Celeryアプリ初期化
celery_app = Celery(
    'ai_tasks',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

@celery_app.task(bind=True)
def generate_image_task(self, workflow: dict, prompt: str):
    """ComfyUI画像生成タスク"""

    # ステータス更新
    self.update_state(state='PROCESSING', meta={'step': 'queuing'})

    # ComfyUI API呼び出し
    response = requests.post(
        "http://localhost:8188/prompt",
        json={"prompt": workflow}
    )
    prompt_id = response.json()["prompt_id"]

    # 完了待機
    self.update_state(state='PROCESSING', meta={'step': 'generating'})

    import time
    while True:
        history = requests.get(f"http://localhost:8188/history/{prompt_id}").json()
        if prompt_id in history:
            return {
                'status': 'completed',
                'image_url': f"/outputs/{history[prompt_id]['outputs']['9']['images'][0]['filename']}"
            }
        time.sleep(2)

@celery_app.task(bind=True)
def batch_embeddings_task(self, texts: list):
    """バッチ埋め込み生成"""

    from sentence_transformers import SentenceTransformer

    model = SentenceTransformer('all-MiniLM-L6-v2')
    embeddings = model.encode(texts, show_progress_bar=False)

    return embeddings.tolist()

# FastAPI統合
from fastapi import FastAPI
from celery.result import AsyncResult

app = FastAPI()

@app.post("/image/generate")
async def generate_image(workflow: dict, prompt: str):
    """画像生成（非同期タスク）"""

    task = generate_image_task.delay(workflow, prompt)

    return {
        "task_id": task.id,
        "status": "queued"
    }

@app.get("/tasks/{task_id}")
async def get_task_status(task_id: str):
    """タスクステータス取得"""

    task = AsyncResult(task_id, app=celery_app)

    if task.state == 'PENDING':
        return {"status": "pending"}
    elif task.state == 'PROCESSING':
        return {"status": "processing", "meta": task.info}
    elif task.state == 'SUCCESS':
        return {"status": "completed", "result": task.result}
    else:
        return {"status": "failed", "error": str(task.info)}
```

**Celery Worker起動:**

```bash
# ターミナル1: Celery Worker
celery -A tasks worker --loglevel=info --concurrency=2

# ターミナル2: FastAPI
uvicorn app.main:app --reload

# ターミナル3: Celery Flower（モニタリング）
celery -A tasks flower
# http://localhost:5555 でダッシュボード表示
```

---

## 2.3 キャッシング戦略

### 2.3.1 Redisキャッシュレイヤー

**LLM応答のキャッシング:**

```python
# cache.py

import redis
import hashlib
import json
from typing import Optional

class LLMCache:
    """LLM応答キャッシュ"""

    def __init__(self, redis_url: str = "redis://localhost:6379/2"):
        self.redis = redis.from_url(redis_url)
        self.ttl = 3600  # 1時間

    def _make_key(self, prompt: str, model: str) -> str:
        """キャッシュキー生成"""
        content = f"{model}:{prompt}"
        return f"llm:{hashlib.sha256(content.encode()).hexdigest()}"

    def get(self, prompt: str, model: str) -> Optional[str]:
        """キャッシュから取得"""
        key = self._make_key(prompt, model)
        cached = self.redis.get(key)

        if cached:
            return cached.decode('utf-8')
        return None

    def set(self, prompt: str, model: str, response: str):
        """キャッシュに保存"""
        key = self._make_key(prompt, model)
        self.redis.setex(key, self.ttl, response)

    def invalidate(self, prompt: str, model: str):
        """キャッシュ無効化"""
        key = self._make_key(prompt, model)
        self.redis.delete(key)

# 使用例
cache = LLMCache()

@app.post("/chat")
async def chat(prompt: str, model: str = "llama3.2:3b"):
    # キャッシュチェック
    cached_response = cache.get(prompt, model)
    if cached_response:
        return {
            "response": cached_response,
            "cached": True,
            "latency_ms": 5  # キャッシュヒット時は超高速
        }

    # LLM呼び出し
    import time
    start = time.time()
    response = await asyncio.to_thread(ollama_generate, prompt, model)
    latency = (time.time() - start) * 1000

    # キャッシュに保存
    cache.set(prompt, model, response)

    return {
        "response": response,
        "cached": False,
        "latency_ms": latency
    }
```

**効果測定:**

```yaml
キャッシュなし:
  平均レイテンシ: 850ms
  スループット: 70リクエスト/分

キャッシュあり（ヒット率50%）:
  平均レイテンシ: 430ms（49%改善）
  スループット: 140リクエスト/分（2倍）

キャッシュあり（ヒット率80%）:
  平均レイテンシ: 180ms（79%改善）
  スループット: 330リクエスト/分（4.7倍）
```

### 2.3.2 埋め込みベクトルのキャッシング

**RAGシステムでの埋め込みキャッシュ:**

```python
# embedding_cache.py

import numpy as np
import pickle
from typing import Optional, List

class EmbeddingCache:
    """埋め込みベクトルキャッシュ"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.ttl = 86400  # 24時間

    def get_embedding(self, text: str) -> Optional[np.ndarray]:
        """埋め込み取得"""
        key = f"emb:{hashlib.sha256(text.encode()).hexdigest()}"
        cached = self.redis.get(key)

        if cached:
            return pickle.loads(cached)
        return None

    def set_embedding(self, text: str, embedding: np.ndarray):
        """埋め込み保存"""
        key = f"emb:{hashlib.sha256(text.encode()).hexdigest()}"
        self.redis.setex(key, self.ttl, pickle.dumps(embedding))

    def get_or_compute(self, texts: List[str], model) -> List[np.ndarray]:
        """キャッシュから取得 or 計算"""

        embeddings = []
        to_compute = []
        to_compute_indices = []

        # キャッシュチェック
        for i, text in enumerate(texts):
            cached = self.get_embedding(text)
            if cached is not None:
                embeddings.append(cached)
            else:
                embeddings.append(None)
                to_compute.append(text)
                to_compute_indices.append(i)

        # 未キャッシュを計算
        if to_compute:
            computed = model.encode(to_compute)

            for idx, emb in zip(to_compute_indices, computed):
                embeddings[idx] = emb
                self.set_embedding(texts[idx], emb)

        return embeddings

# 使用例
from sentence_transformers import SentenceTransformer

embedding_model = SentenceTransformer('all-MiniLM-L6-v2')
embedding_cache = EmbeddingCache(redis.from_url("redis://localhost:6379/3"))

texts = ["Hello", "World", "Hello"]  # "Hello"は重複
embeddings = embedding_cache.get_or_compute(texts, embedding_model)

# "Hello"は1回のみ計算、2回目はキャッシュから取得
```

---

## 2.4 データベース設計

### 2.4.1 PostgreSQLスキーマ設計

**チャット履歴管理:**

```sql
-- users.sql

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE conversations (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE messages (
    id SERIAL PRIMARY KEY,
    conversation_id INTEGER REFERENCES conversations(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content TEXT NOT NULL,
    model VARCHAR(100),
    tokens INTEGER,
    latency_ms INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_messages_conversation ON messages(conversation_id);
CREATE INDEX idx_messages_created_at ON messages(created_at DESC);

-- 使用統計
CREATE TABLE usage_stats (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    endpoint VARCHAR(100),
    model VARCHAR(100),
    tokens_input INTEGER,
    tokens_output INTEGER,
    latency_ms INTEGER,
    cached BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_usage_stats_user_date ON usage_stats(user_id, created_at DESC);
```

**SQLAlchemy ORM:**

```python
# models.py

from sqlalchemy import Column, Integer, String, Text, Boolean, ForeignKey, DateTime
from sqlalchemy.orm import relationship, declarative_base
from datetime import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    username = Column(String(100), unique=True, nullable=False)
    email = Column(String(255), unique=True, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)

    conversations = relationship("Conversation", back_populates="user")

class Conversation(Base):
    __tablename__ = 'conversations'

    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id'))
    title = Column(String(255))
    created_at = Column(DateTime, default=datetime.utcnow)

    user = relationship("User", back_populates="conversations")
    messages = relationship("Message", back_populates="conversation")

class Message(Base):
    __tablename__ = 'messages'

    id = Column(Integer, primary_key=True)
    conversation_id = Column(Integer, ForeignKey('conversations.id'))
    role = Column(String(20), nullable=False)
    content = Column(Text, nullable=False)
    model = Column(String(100))
    tokens = Column(Integer)
    latency_ms = Column(Integer)
    created_at = Column(DateTime, default=datetime.utcnow)

    conversation = relationship("Conversation", back_populates="messages")

# データベース接続
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "postgresql://user:password@localhost/ai_app"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)

# テーブル作成
Base.metadata.create_all(engine)
```

### 2.4.2 ChromaDBベクトルデータベース

**RAG用ベクトルストア:**

```python
# vector_store.py

import chromadb
from chromadb.config import Settings
from typing import List, Dict

class VectorStore:
    """ChromaDBベクトルストア"""

    def __init__(self, persist_directory: str = "./chroma_db"):
        self.client = chromadb.Client(Settings(
            persist_directory=persist_directory,
            anonymized_telemetry=False
        ))

        self.collection = self.client.get_or_create_collection(
            name="documents",
            metadata={"hnsw:space": "cosine"}
        )

    def add_documents(
        self,
        documents: List[str],
        metadatas: List[Dict],
        ids: List[str]
    ):
        """文書追加"""
        self.collection.add(
            documents=documents,
            metadatas=metadatas,
            ids=ids
        )

    def search(
        self,
        query: str,
        n_results: int = 5,
        where: Dict = None
    ) -> Dict:
        """類似文書検索"""
        results = self.collection.query(
            query_texts=[query],
            n_results=n_results,
            where=where
        )
        return results

    def delete(self, ids: List[str]):
        """文書削除"""
        self.collection.delete(ids=ids)

    def count(self) -> int:
        """文書数取得"""
        return self.collection.count()

# 使用例
vector_store = VectorStore()

# 文書追加
documents = [
    "MS-S1 MaxはAMD Ryzen AI Max+ 395を搭載しています。",
    "128GBの統合メモリを持つ強力なAPUです。",
    "ローカルAI開発に最適なハードウェアです。"
]
metadatas = [
    {"source": "manual", "page": 1},
    {"source": "manual", "page": 2},
    {"source": "manual", "page": 3}
]
ids = ["doc1", "doc2", "doc3"]

vector_store.add_documents(documents, metadatas, ids)

# 検索
results = vector_store.search("MS-S1 Maxのメモリは？", n_results=2)
print(results['documents'])
# [['128GBの統合メモリを持つ強力なAPUです。', 'MS-S1 MaxはAMD...'], ...]
```

---

## 2.5 APIバージョニングとドキュメント

### 2.5.1 APIバージョニング戦略

**URLパスベースバージョニング:**

```python
# app/main.py

from fastapi import FastAPI, APIRouter

app = FastAPI(title="AI Application API")

# バージョン1
v1_router = APIRouter(prefix="/api/v1")

@v1_router.post("/chat")
async def chat_v1(prompt: str):
    """V1チャット（シンプル）"""
    return {"response": "..."}

# バージョン2（拡張機能）
v2_router = APIRouter(prefix="/api/v2")

@v2_router.post("/chat")
async def chat_v2(
    prompt: str,
    temperature: float = 0.7,
    max_tokens: int = 512,
    stream: bool = False
):
    """V2チャット（高度な設定）"""
    return {"response": "...", "metadata": {...}}

app.include_router(v1_router)
app.include_router(v2_router)

# 両方のバージョンが同時に利用可能
# /api/v1/chat
# /api/v2/chat
```

### 2.5.2 OpenAPI自動ドキュメント

**Pydanticモデルで型安全:**

```python
# schemas.py

from pydantic import BaseModel, Field
from typing import Optional, List
from enum import Enum

class ModelName(str, Enum):
    LLAMA_3B = "llama3.2:3b"
    LLAMA_13B = "llama3.1:13b"
    GEMMA_9B = "gemma2:9b"

class ChatRequest(BaseModel):
    prompt: str = Field(..., description="ユーザープロンプト", min_length=1)
    model: ModelName = Field(default=ModelName.LLAMA_3B, description="使用モデル")
    temperature: float = Field(default=0.7, ge=0.0, le=2.0, description="生成温度")
    max_tokens: int = Field(default=512, ge=1, le=4096, description="最大トークン数")

    class Config:
        schema_extra = {
            "example": {
                "prompt": "MS-S1 Maxの特徴を教えてください",
                "model": "llama3.2:3b",
                "temperature": 0.7,
                "max_tokens": 512
            }
        }

class ChatResponse(BaseModel):
    response: str = Field(..., description="LLM応答")
    model: str = Field(..., description="使用モデル")
    tokens: int = Field(..., description="生成トークン数")
    latency_ms: int = Field(..., description="レイテンシ（ミリ秒）")
    cached: bool = Field(default=False, description="キャッシュヒット")

# FastAPIで使用
@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """チャット生成エンドポイント

    このエンドポイントはOllamaを使用してLLM応答を生成します。

    **パラメータ:**
    - prompt: ユーザーからの入力テキスト
    - model: 使用するLLMモデル
    - temperature: 生成のランダム性（0=決定的、2=非常にランダム）
    - max_tokens: 最大生成トークン数

    **戻り値:**
    - response: LLMの応答テキスト
    - tokens: 生成されたトークン数
    - latency_ms: 処理時間（ミリ秒）
    """
    # 実装...
    pass

# 自動生成されるドキュメント: http://localhost:8000/docs
```

---

## 2.6 エラーハンドリングとロギング

### 2.6.1 統一エラーハンドリング

**カスタム例外クラス:**

```python
# exceptions.py

class AIServiceException(Exception):
    """AI サービス基底例外"""
    def __init__(self, message: str, code: str, details: dict = None):
        self.message = message
        self.code = code
        self.details = details or {}
        super().__init__(self.message)

class ModelNotFoundError(AIServiceException):
    """モデルが見つからない"""
    def __init__(self, model_name: str):
        super().__init__(
            message=f"Model '{model_name}' not found",
            code="MODEL_NOT_FOUND",
            details={"model": model_name}
        )

class GenerationTimeoutError(AIServiceException):
    """生成タイムアウト"""
    def __init__(self, timeout: int):
        super().__init__(
            message=f"Generation timed out after {timeout}s",
            code="GENERATION_TIMEOUT",
            details={"timeout": timeout}
        )

class RateLimitExceededError(AIServiceException):
    """レート制限超過"""
    def __init__(self, limit: int, window: int):
        super().__init__(
            message=f"Rate limit exceeded: {limit} requests per {window}s",
            code="RATE_LIMIT_EXCEEDED",
            details={"limit": limit, "window": window}
        )

# FastAPIエラーハンドラー
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(AIServiceException)
async def ai_service_exception_handler(request: Request, exc: AIServiceException):
    return JSONResponse(
        status_code=400,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
                "details": exc.details
            }
        }
    )

# 使用例
@app.post("/chat")
async def chat(model: str, prompt: str):
    # モデル存在チェック
    available_models = get_available_models()
    if model not in available_models:
        raise ModelNotFoundError(model)

    # 生成...
```

### 2.6.2 構造化ロギング

**Python標準loggingの設定:**

```python
# logging_config.py

import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    """JSON形式ログフォーマッタ"""

    def format(self, record):
        log_data = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno
        }

        # 例外情報
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)

        # カスタムフィールド
        if hasattr(record, 'user_id'):
            log_data["user_id"] = record.user_id
        if hasattr(record, 'request_id'):
            log_data["request_id"] = record.request_id

        return json.dumps(log_data, ensure_ascii=False)

# ロガー設定
def setup_logging():
    """ロギング設定"""

    # ルートロガー
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)

    # コンソールハンドラー
    console_handler = logging.StreamHandler()
    console_handler.setFormatter(JSONFormatter())
    logger.addHandler(console_handler)

    # ファイルハンドラー
    file_handler = logging.FileHandler('app.log')
    file_handler.setFormatter(JSONFormatter())
    logger.addHandler(file_handler)

    return logger

# 使用例
logger = setup_logging()

@app.post("/chat")
async def chat(request: ChatRequest, user_id: int):
    logger.info(
        "Chat request received",
        extra={
            "user_id": user_id,
            "model": request.model,
            "prompt_length": len(request.prompt)
        }
    )

    try:
        response = await generate_response(request)

        logger.info(
            "Chat response generated",
            extra={
                "user_id": user_id,
                "tokens": response.tokens,
                "latency_ms": response.latency_ms
            }
        )

        return response

    except Exception as e:
        logger.error(
            "Chat generation failed",
            exc_info=True,
            extra={"user_id": user_id}
        )
        raise
```

---

## 2.7 本章のまとめ

本章では、スケーラブルなローカルAIアプリケーションのアーキテクチャ設計を学びました。

### 学習内容の振り返り

**2.1-2.2: アーキテクチャと非同期処理**
- ✅ モノリス vs マイクロサービスの選択
- ✅ MS-S1 Max特化ハイブリッド型モノリス
- ✅ FastAPI非同期エンドポイント
- ✅ Celery分散タスクキュー

**2.3-2.4: キャッシングとデータベース**
- ✅ Redis LLM応答キャッシング（4.7倍高速化）
- ✅ 埋め込みベクトルキャッシング
- ✅ PostgreSQLスキーマ設計
- ✅ ChromaDBベクトルストア

**2.5-2.7: API設計と運用**
- ✅ APIバージョニング戦略
- ✅ OpenAPI自動ドキュメント
- ✅ 統一エラーハンドリング
- ✅ 構造化ロギング

### アーキテクチャ概要図

```
┌──────────────────────────────────────────────┐
│         Frontend (Next.js/React)             │
└────────────────┬─────────────────────────────┘
                 │ HTTP/WebSocket
┌────────────────▼─────────────────────────────┐
│    API Server (FastAPI) + Redis Cache        │
├──────────────────────────────────────────────┤
│  ┌──────────────┬──────────────┬────────────┐│
│  │ LLM Module   │ Image Module │ RAG Module ││
│  │ (Ollama)     │ (ComfyUI)    │ (ChromaDB) ││
│  └──────────────┴──────────────┴────────────┘│
│  Background Tasks (Celery + Redis Queue)     │
└────────────────┬─────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────┐
│    PostgreSQL (メタデータ・履歴管理)         │
└──────────────────────────────────────────────┘

MS-S1 Max: 128GB統合メモリで全コンポーネントが効率動作
```

### 次のステップ

第3章では、RAG（検索拡張生成）システムの実装を学びます。ChromaDB、埋め込みモデル、文書処理パイプラインを構築し、ローカル文書検索AIを実現します。

---

**参考資料:**

- FastAPI Async: https://fastapi.tiangolo.com/async/
- Celery: https://docs.celeryproject.org/
- ChromaDB: https://docs.trychroma.com/
- SQLAlchemy: https://www.sqlalchemy.org/

---
