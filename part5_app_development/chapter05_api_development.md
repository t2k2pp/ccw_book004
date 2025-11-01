# Chapter 05: Web API開発

## 5.1 RESTful API設計

### 5.1.1 API設計の基本原則

MS-S1 Maxを活用したローカルAI APIの設計では、以下の原則に従います。

**1. リソース指向設計**

```python
# api_v1.py
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, Field
from typing import List, Optional
from datetime import datetime
import uuid

app = FastAPI(
    title="Local AI API",
    description="MS-S1 Max上で動作するローカルAI API",
    version="1.0.0"
)

# データモデル
class Model(BaseModel):
    id: str
    name: str
    size: str
    family: str
    modified_at: datetime

class ChatMessage(BaseModel):
    role: str = Field(..., pattern="^(system|user|assistant)$")
    content: str

class ChatRequest(BaseModel):
    model: str = "qwen2.5:14b"
    messages: List[ChatMessage]
    temperature: float = Field(0.7, ge=0.0, le=2.0)
    max_tokens: Optional[int] = Field(None, ge=1, le=32768)
    stream: bool = False

class ChatResponse(BaseModel):
    id: str
    model: str
    created: datetime
    message: ChatMessage
    usage: dict

class EmbeddingRequest(BaseModel):
    model: str = "mxbai-embed-large"
    input: str | List[str]

class EmbeddingResponse(BaseModel):
    model: str
    embeddings: List[List[float]]
    usage: dict

# エンドポイント定義
@app.get("/v1/models", response_model=List[Model])
async def list_models():
    """利用可能なモデル一覧を取得"""
    import ollama
    models = ollama.list()

    return [
        Model(
            id=m['name'],
            name=m['name'],
            size=m['size'],
            family=m.get('details', {}).get('family', ''),
            modified_at=m['modified_at']
        )
        for m in models['models']
    ]

@app.get("/v1/models/{model_name}", response_model=Model)
async def get_model(model_name: str):
    """特定モデルの情報を取得"""
    import ollama
    try:
        info = ollama.show(model_name)
        return Model(
            id=model_name,
            name=model_name,
            size=info.get('size', ''),
            family=info.get('details', {}).get('family', ''),
            modified_at=datetime.now()
        )
    except Exception as e:
        raise HTTPException(status_code=404, detail=f"Model {model_name} not found")

@app.post("/v1/chat/completions", response_model=ChatResponse)
async def create_chat_completion(request: ChatRequest):
    """チャット補完を生成"""
    import ollama

    start_time = datetime.now()

    # Ollamaフォーマットに変換
    ollama_messages = [
        {"role": msg.role, "content": msg.content}
        for msg in request.messages
    ]

    try:
        response = ollama.chat(
            model=request.model,
            messages=ollama_messages,
            options={
                "temperature": request.temperature,
                "num_predict": request.max_tokens
            } if request.max_tokens else {"temperature": request.temperature}
        )

        return ChatResponse(
            id=str(uuid.uuid4()),
            model=request.model,
            created=start_time,
            message=ChatMessage(
                role="assistant",
                content=response['message']['content']
            ),
            usage={
                "prompt_tokens": response.get('prompt_eval_count', 0),
                "completion_tokens": response.get('eval_count', 0),
                "total_tokens": response.get('prompt_eval_count', 0) + response.get('eval_count', 0)
            }
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/v1/embeddings", response_model=EmbeddingResponse)
async def create_embeddings(request: EmbeddingRequest):
    """埋め込みベクトルを生成"""
    import ollama

    inputs = [request.input] if isinstance(request.input, str) else request.input
    embeddings = []

    for text in inputs:
        response = ollama.embeddings(model=request.model, prompt=text)
        embeddings.append(response['embedding'])

    return EmbeddingResponse(
        model=request.model,
        embeddings=embeddings,
        usage={
            "total_tokens": sum(len(text.split()) for text in inputs)
        }
    )
```

### 5.1.2 エラーハンドリング

```python
# error_handling.py
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import BaseModel
import logging
from datetime import datetime

app = FastAPI()

# エラーレスポンスモデル
class ErrorResponse(BaseModel):
    error: dict

class ErrorDetail(BaseModel):
    message: str
    type: str
    code: str
    timestamp: str

# カスタム例外
class ModelNotFoundError(Exception):
    def __init__(self, model_name: str):
        self.model_name = model_name

class RateLimitError(Exception):
    def __init__(self, retry_after: int):
        self.retry_after = retry_after

class InsufficientResourcesError(Exception):
    def __init__(self, message: str):
        self.message = message

# 例外ハンドラー
@app.exception_handler(ModelNotFoundError)
async def model_not_found_handler(request: Request, exc: ModelNotFoundError):
    return JSONResponse(
        status_code=status.HTTP_404_NOT_FOUND,
        content={
            "error": {
                "message": f"Model '{exc.model_name}' not found",
                "type": "model_not_found_error",
                "code": "model_not_found",
                "timestamp": datetime.now().isoformat()
            }
        }
    )

@app.exception_handler(RateLimitError)
async def rate_limit_handler(request: Request, exc: RateLimitError):
    return JSONResponse(
        status_code=status.HTTP_429_TOO_MANY_REQUESTS,
        content={
            "error": {
                "message": "Rate limit exceeded",
                "type": "rate_limit_error",
                "code": "rate_limit_exceeded",
                "timestamp": datetime.now().isoformat()
            }
        },
        headers={"Retry-After": str(exc.retry_after)}
    )

@app.exception_handler(InsufficientResourcesError)
async def insufficient_resources_handler(request: Request, exc: InsufficientResourcesError):
    return JSONResponse(
        status_code=status.HTTP_507_INSUFFICIENT_STORAGE,
        content={
            "error": {
                "message": exc.message,
                "type": "insufficient_resources_error",
                "code": "insufficient_resources",
                "timestamp": datetime.now().isoformat()
            }
        }
    )

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "error": {
                "message": "Validation error",
                "type": "invalid_request_error",
                "code": "validation_error",
                "details": exc.errors(),
                "timestamp": datetime.now().isoformat()
            }
        }
    )

@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception):
    logging.error(f"Unhandled exception: {exc}", exc_info=True)
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "error": {
                "message": "Internal server error",
                "type": "server_error",
                "code": "internal_error",
                "timestamp": datetime.now().isoformat()
            }
        }
    )
```

## 5.2 認証と認可

### 5.2.1 APIキー認証

```python
# auth.py
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
from typing import Optional
import secrets
import hashlib
from datetime import datetime, timedelta
import json
from pathlib import Path

app = FastAPI()
security = HTTPBearer()

# APIキー管理
class APIKey(BaseModel):
    key: str
    name: str
    created_at: datetime
    expires_at: Optional[datetime] = None
    is_active: bool = True
    rate_limit: int = 100  # リクエスト/時間
    allowed_endpoints: Optional[list] = None

class APIKeyManager:
    def __init__(self, keys_file: str = "./api_keys.json"):
        self.keys_file = Path(keys_file)
        self.keys: dict = self._load_keys()

    def _load_keys(self) -> dict:
        if self.keys_file.exists():
            with open(self.keys_file, 'r') as f:
                return json.load(f)
        return {}

    def _save_keys(self):
        with open(self.keys_file, 'w') as f:
            json.dump(self.keys, f, default=str, indent=2)

    def create_key(
        self,
        name: str,
        expires_days: Optional[int] = None,
        rate_limit: int = 100,
        allowed_endpoints: Optional[list] = None
    ) -> str:
        """新しいAPIキーを生成"""
        # セキュアなランダムキーを生成
        raw_key = secrets.token_urlsafe(32)
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()

        expires_at = None
        if expires_days:
            expires_at = datetime.now() + timedelta(days=expires_days)

        self.keys[key_hash] = {
            "name": name,
            "created_at": datetime.now().isoformat(),
            "expires_at": expires_at.isoformat() if expires_at else None,
            "is_active": True,
            "rate_limit": rate_limit,
            "allowed_endpoints": allowed_endpoints,
            "usage_count": 0
        }

        self._save_keys()
        return raw_key  # これを一度だけユーザーに返す

    def validate_key(self, raw_key: str) -> Optional[dict]:
        """APIキーを検証"""
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()

        if key_hash not in self.keys:
            return None

        key_data = self.keys[key_hash]

        # アクティブチェック
        if not key_data['is_active']:
            return None

        # 有効期限チェック
        if key_data['expires_at']:
            expires_at = datetime.fromisoformat(key_data['expires_at'])
            if datetime.now() > expires_at:
                return None

        return key_data

    def revoke_key(self, raw_key: str):
        """APIキーを無効化"""
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        if key_hash in self.keys:
            self.keys[key_hash]['is_active'] = False
            self._save_keys()

    def increment_usage(self, raw_key: str):
        """使用回数をインクリメント"""
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        if key_hash in self.keys:
            self.keys[key_hash]['usage_count'] += 1
            self._save_keys()

# グローバルインスタンス
api_key_manager = APIKeyManager()

# 依存関数
async def verify_api_key(credentials: HTTPAuthorizationCredentials = Depends(security)) -> dict:
    """APIキーを検証する依存関数"""
    token = credentials.credentials

    key_data = api_key_manager.validate_key(token)

    if not key_data:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired API key"
        )

    api_key_manager.increment_usage(token)
    return key_data

# 保護されたエンドポイント
@app.post("/v1/chat/completions")
async def protected_chat(
    request: dict,
    key_data: dict = Depends(verify_api_key)
):
    """認証が必要なチャットエンドポイント"""
    # エンドポイント制限をチェック
    if key_data.get('allowed_endpoints'):
        if "/v1/chat/completions" not in key_data['allowed_endpoints']:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Access to this endpoint is not allowed"
            )

    # 通常の処理...
    return {"message": "Success", "key_name": key_data['name']}

# 管理エンドポイント（別の認証が必要）
@app.post("/admin/api-keys")
async def create_api_key(
    name: str,
    expires_days: Optional[int] = None,
    rate_limit: int = 100
):
    """新しいAPIキーを生成（管理者用）"""
    # 実際には管理者認証が必要
    api_key = api_key_manager.create_key(name, expires_days, rate_limit)
    return {"api_key": api_key, "note": "このキーは一度しか表示されません"}
```

### 5.2.2 レート制限

```python
# rate_limiter.py
from fastapi import FastAPI, Request, HTTPException, status
from typing import Dict
from datetime import datetime, timedelta
from collections import defaultdict
import asyncio

app = FastAPI()

class RateLimiter:
    def __init__(self):
        self.requests: Dict[str, list] = defaultdict(list)
        self.lock = asyncio.Lock()

    async def is_allowed(
        self,
        key: str,
        max_requests: int,
        window_seconds: int
    ) -> tuple[bool, int]:
        """
        レート制限をチェック
        Returns: (許可されるか, リトライまでの秒数)
        """
        async with self.lock:
            now = datetime.now()
            window_start = now - timedelta(seconds=window_seconds)

            # 古いリクエストを削除
            self.requests[key] = [
                req_time for req_time in self.requests[key]
                if req_time > window_start
            ]

            # 制限チェック
            if len(self.requests[key]) >= max_requests:
                oldest_request = min(self.requests[key])
                retry_after = int((oldest_request + timedelta(seconds=window_seconds) - now).total_seconds())
                return False, max(retry_after, 1)

            # リクエストを記録
            self.requests[key].append(now)
            return True, 0

    async def cleanup_old_entries(self, max_age_hours: int = 24):
        """古いエントリをクリーンアップ"""
        cutoff = datetime.now() - timedelta(hours=max_age_hours)
        async with self.lock:
            keys_to_delete = []
            for key, timestamps in self.requests.items():
                self.requests[key] = [t for t in timestamps if t > cutoff]
                if not self.requests[key]:
                    keys_to_delete.append(key)

            for key in keys_to_delete:
                del self.requests[key]

# グローバルインスタンス
rate_limiter = RateLimiter()

# ミドルウェアとして使用
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    # APIキーまたはIPアドレスをキーとして使用
    client_key = request.headers.get("Authorization", request.client.host)

    # エンドポイントごとに異なる制限を適用
    if request.url.path.startswith("/v1/chat"):
        max_requests, window = 60, 60  # 60リクエスト/分
    elif request.url.path.startswith("/v1/embeddings"):
        max_requests, window = 120, 60  # 120リクエスト/分
    else:
        max_requests, window = 100, 60  # デフォルト

    allowed, retry_after = await rate_limiter.is_allowed(
        client_key,
        max_requests,
        window
    )

    if not allowed:
        return JSONResponse(
            status_code=status.HTTP_429_TOO_MANY_REQUESTS,
            content={
                "error": {
                    "message": "Rate limit exceeded",
                    "type": "rate_limit_error"
                }
            },
            headers={"Retry-After": str(retry_after)}
        )

    response = await call_next(request)
    return response

# バックグラウンドタスクでクリーンアップ
@app.on_event("startup")
async def startup_event():
    async def cleanup_task():
        while True:
            await asyncio.sleep(3600)  # 1時間ごと
            await rate_limiter.cleanup_old_entries()

    asyncio.create_task(cleanup_task())
```

## 5.3 WebSocketによるリアルタイム通信

### 5.3.1 WebSocket基本実装

```python
# websocket_api.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, Set
import json
import asyncio
import ollama

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        self.active_connections: Dict[str, WebSocket] = {}
        self.user_sessions: Dict[str, list] = {}

    async def connect(self, session_id: str, websocket: WebSocket):
        await websocket.accept()
        self.active_connections[session_id] = websocket
        self.user_sessions[session_id] = []

    def disconnect(self, session_id: str):
        if session_id in self.active_connections:
            del self.active_connections[session_id]
        if session_id in self.user_sessions:
            del self.user_sessions[session_id]

    async def send_message(self, session_id: str, message: dict):
        if session_id in self.active_connections:
            await self.active_connections[session_id].send_json(message)

    async def broadcast(self, message: dict, exclude: Set[str] = None):
        exclude = exclude or set()
        for session_id, connection in self.active_connections.items():
            if session_id not in exclude:
                await connection.send_json(message)

manager = ConnectionManager()

@app.websocket("/ws/chat/{session_id}")
async def websocket_chat(websocket: WebSocket, session_id: str):
    await manager.connect(session_id, websocket)

    try:
        while True:
            # クライアントからのメッセージを受信
            data = await websocket.receive_json()

            message_type = data.get('type')

            if message_type == 'chat':
                # チャットメッセージを処理
                user_message = data.get('message')
                model = data.get('model', 'qwen2.5:14b')

                # 会話履歴に追加
                manager.user_sessions[session_id].append({
                    "role": "user",
                    "content": user_message
                })

                # ユーザーメッセージを確認
                await manager.send_message(session_id, {
                    "type": "user_message",
                    "content": user_message
                })

                # ストリーミングレスポンスを生成
                await manager.send_message(session_id, {
                    "type": "assistant_start"
                })

                full_response = []
                stream = ollama.chat(
                    model=model,
                    messages=manager.user_sessions[session_id],
                    stream=True
                )

                for chunk in stream:
                    content = chunk['message']['content']
                    full_response.append(content)

                    await manager.send_message(session_id, {
                        "type": "assistant_chunk",
                        "content": content
                    })

                    # 少し遅延を入れてスムーズに表示
                    await asyncio.sleep(0.01)

                # 完全な応答を履歴に追加
                complete_response = ''.join(full_response)
                manager.user_sessions[session_id].append({
                    "role": "assistant",
                    "content": complete_response
                })

                await manager.send_message(session_id, {
                    "type": "assistant_end",
                    "full_response": complete_response
                })

            elif message_type == 'clear':
                # 会話履歴をクリア
                manager.user_sessions[session_id] = []
                await manager.send_message(session_id, {
                    "type": "cleared"
                })

            elif message_type == 'ping':
                # 接続確認
                await manager.send_message(session_id, {
                    "type": "pong"
                })

    except WebSocketDisconnect:
        manager.disconnect(session_id)
    except Exception as e:
        await manager.send_message(session_id, {
            "type": "error",
            "message": str(e)
        })
        manager.disconnect(session_id)
```

### 5.3.2 WebSocketクライアント（JavaScript）

```javascript
// websocket_client.js
class ChatWebSocketClient {
  constructor(sessionId, url = 'ws://localhost:8000') {
    this.sessionId = sessionId;
    this.url = `${url}/ws/chat/${sessionId}`;
    this.ws = null;
    this.messageHandlers = new Map();
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = 5;
  }

  connect() {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(this.url);

      this.ws.onopen = () => {
        console.log('WebSocket connected');
        this.reconnectAttempts = 0;
        resolve();
      };

      this.ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        this.handleMessage(data);
      };

      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        reject(error);
      };

      this.ws.onclose = () => {
        console.log('WebSocket disconnected');
        this.attemptReconnect();
      };
    });
  }

  handleMessage(data) {
    const { type } = data;

    if (this.messageHandlers.has(type)) {
      this.messageHandlers.get(type)(data);
    }
  }

  on(messageType, handler) {
    this.messageHandlers.set(messageType, handler);
  }

  sendChat(message, model = 'qwen2.5:14b') {
    this.send({
      type: 'chat',
      message: message,
      model: model
    });
  }

  clearHistory() {
    this.send({
      type: 'clear'
    });
  }

  ping() {
    this.send({
      type: 'ping'
    });
  }

  send(data) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    } else {
      console.error('WebSocket is not connected');
    }
  }

  attemptReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);

      console.log(`Reconnecting in ${delay}ms... (attempt ${this.reconnectAttempts})`);

      setTimeout(() => {
        this.connect().catch(err => {
          console.error('Reconnection failed:', err);
        });
      }, delay);
    } else {
      console.error('Max reconnection attempts reached');
    }
  }

  disconnect() {
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }
}

// 使用例
const client = new ChatWebSocketClient('session-123');

client.on('user_message', (data) => {
  console.log('User:', data.content);
});

client.on('assistant_start', () => {
  console.log('Assistant is typing...');
});

client.on('assistant_chunk', (data) => {
  process.stdout.write(data.content);
});

client.on('assistant_end', (data) => {
  console.log('\nComplete response:', data.full_response);
});

client.on('error', (data) => {
  console.error('Error:', data.message);
});

// 接続してチャット
client.connect().then(() => {
  client.sendChat('Pythonのリスト内包表記について教えてください');
});
```

## 5.4 API ドキュメント

### 5.4.1 OpenAPI/Swagger自動生成

```python
# documented_api.py
from fastapi import FastAPI, Query, Path, Body
from pydantic import BaseModel, Field
from typing import List, Optional
from enum import Enum

# タグの定義
tags_metadata = [
    {
        "name": "models",
        "description": "モデル管理のエンドポイント",
    },
    {
        "name": "chat",
        "description": "チャット補完のエンドポイント",
    },
    {
        "name": "embeddings",
        "description": "埋め込みベクトル生成のエンドポイント",
    },
]

app = FastAPI(
    title="Local AI API",
    description="""
    ## MS-S1 Max ローカルAI API

    このAPIは、MS-S1 Max（AMD Ryzen AI Max+ 395）上で動作するローカルAIサービスを提供します。

    ### 主な機能

    * **モデル管理**: 利用可能なモデルの一覧取得と詳細情報
    * **チャット補完**: 会話型AIによるテキスト生成
    * **埋め込み生成**: テキストのベクトル表現を生成
    * **ストリーミング**: リアルタイムレスポンス生成

    ### 認証

    すべてのエンドポイントはAPIキー認証が必要です。
    `Authorization: Bearer YOUR_API_KEY` ヘッダーを含めてください。

    ### レート制限

    - チャットエンドポイント: 60リクエスト/分
    - 埋め込みエンドポイント: 120リクエスト/分
    - その他: 100リクエスト/分
    """,
    version="1.0.0",
    openapi_tags=tags_metadata,
    docs_url="/docs",
    redoc_url="/redoc"
)

# モデル定義
class ModelFamily(str, Enum):
    LLAMA = "llama"
    QWEN = "qwen"
    MISTRAL = "mistral"
    EMBEDDING = "embedding"

class Model(BaseModel):
    """AIモデル情報"""
    id: str = Field(..., description="モデルの一意な識別子", example="qwen2.5:14b")
    name: str = Field(..., description="モデル名", example="Qwen 2.5 14B")
    size: str = Field(..., description="モデルサイズ", example="8.5GB")
    family: ModelFamily = Field(..., description="モデルファミリー")
    modified_at: str = Field(..., description="最終更新日時")

    class Config:
        json_schema_extra = {
            "example": {
                "id": "qwen2.5:14b",
                "name": "Qwen 2.5 14B",
                "size": "8.5GB",
                "family": "qwen",
                "modified_at": "2025-01-15T10:30:00Z"
            }
        }

class ChatMessage(BaseModel):
    """チャットメッセージ"""
    role: str = Field(
        ...,
        description="メッセージの役割",
        pattern="^(system|user|assistant)$",
        example="user"
    )
    content: str = Field(
        ...,
        description="メッセージ内容",
        example="Pythonのリスト内包表記について教えてください"
    )

class ChatRequest(BaseModel):
    """チャットリクエスト"""
    model: str = Field(
        default="qwen2.5:14b",
        description="使用するモデル名",
        example="qwen2.5:14b"
    )
    messages: List[ChatMessage] = Field(
        ...,
        description="会話履歴",
        min_items=1
    )
    temperature: float = Field(
        default=0.7,
        ge=0.0,
        le=2.0,
        description="生成のランダム性（0.0-2.0）",
        example=0.7
    )
    max_tokens: Optional[int] = Field(
        default=None,
        ge=1,
        le=32768,
        description="最大生成トークン数",
        example=2048
    )
    stream: bool = Field(
        default=False,
        description="ストリーミングレスポンスを有効化"
    )

    class Config:
        json_schema_extra = {
            "example": {
                "model": "qwen2.5:14b",
                "messages": [
                    {"role": "user", "content": "Pythonのリスト内包表記とは？"}
                ],
                "temperature": 0.7,
                "max_tokens": 2048,
                "stream": False
            }
        }

# エンドポイント
@app.get(
    "/v1/models",
    response_model=List[Model],
    tags=["models"],
    summary="モデル一覧を取得",
    description="利用可能なAIモデルの一覧を取得します。",
    responses={
        200: {
            "description": "モデル一覧の取得に成功",
            "content": {
                "application/json": {
                    "example": [
                        {
                            "id": "qwen2.5:14b",
                            "name": "Qwen 2.5 14B",
                            "size": "8.5GB",
                            "family": "qwen",
                            "modified_at": "2025-01-15T10:30:00Z"
                        }
                    ]
                }
            }
        }
    }
)
async def list_models():
    """利用可能なモデル一覧を返します"""
    # 実装は省略
    pass

@app.get(
    "/v1/models/{model_id}",
    response_model=Model,
    tags=["models"],
    summary="モデル詳細を取得",
    description="指定したモデルの詳細情報を取得します。",
    responses={
        200: {"description": "モデル詳細の取得に成功"},
        404: {"description": "モデルが見つかりません"}
    }
)
async def get_model(
    model_id: str = Path(
        ...,
        description="モデルID",
        example="qwen2.5:14b"
    )
):
    """特定モデルの詳細情報を返します"""
    # 実装は省略
    pass

@app.post(
    "/v1/chat/completions",
    tags=["chat"],
    summary="チャット補完を生成",
    description="""
    会話型AIを使用してテキストを生成します。

    ### MS-S1 Maxでのパフォーマンス

    | モデル | 速度 | メモリ |
    |--------|------|--------|
    | qwen2.5:3b | 45 tokens/s | 2.5GB |
    | qwen2.5:14b | 18 tokens/s | 11.2GB |

    ### ストリーミング

    `stream: true` を指定すると、Server-Sent Events形式でリアルタイムレスポンスを受け取れます。
    """,
    responses={
        200: {"description": "チャット補完の生成に成功"},
        400: {"description": "リクエストが不正です"},
        401: {"description": "認証が必要です"},
        429: {"description": "レート制限を超過しました"}
    }
)
async def create_chat_completion(request: ChatRequest = Body(...)):
    """チャット補完を生成します"""
    # 実装は省略
    pass
```

### 5.4.2 カスタムドキュメンテーション

```python
# custom_docs.py
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi
from fastapi.staticfiles import StaticFiles

app = FastAPI()

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema

    openapi_schema = get_openapi(
        title="Local AI API",
        version="1.0.0",
        description="MS-S1 Max ローカルAI API",
        routes=app.routes,
    )

    # カスタム情報を追加
    openapi_schema["info"]["x-logo"] = {
        "url": "https://example.com/logo.png"
    }

    # セキュリティスキームを追加
    openapi_schema["components"]["securitySchemes"] = {
        "BearerAuth": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "API Key"
        }
    }

    # すべてのエンドポイントに認証を要求
    for path in openapi_schema["paths"].values():
        for operation in path.values():
            operation["security"] = [{"BearerAuth": []}]

    # サーバー情報を追加
    openapi_schema["servers"] = [
        {
            "url": "http://localhost:8000",
            "description": "開発環境"
        },
        {
            "url": "http://ms-s1-max.local:8000",
            "description": "MS-S1 Max ローカルサーバー"
        }
    ]

    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi
```

## 5.5 APIテスト

### 5.5.1 pytest による単体テスト

```python
# test_api.py
import pytest
from fastapi.testclient import TestClient
from main import app
import json

client = TestClient(app)

def test_list_models():
    """モデル一覧取得のテスト"""
    response = client.get("/v1/models")

    assert response.status_code == 200
    models = response.json()
    assert isinstance(models, list)
    assert len(models) > 0

    # 各モデルの構造を確認
    for model in models:
        assert "id" in model
        assert "name" in model
        assert "size" in model

def test_get_model():
    """特定モデル取得のテスト"""
    response = client.get("/v1/models/qwen2.5:14b")

    assert response.status_code == 200
    model = response.json()
    assert model["id"] == "qwen2.5:14b"

def test_get_nonexistent_model():
    """存在しないモデル取得のテスト"""
    response = client.get("/v1/models/nonexistent-model")

    assert response.status_code == 404
    error = response.json()
    assert "error" in error

def test_chat_completion():
    """チャット補完のテスト"""
    request_data = {
        "model": "qwen2.5:14b",
        "messages": [
            {"role": "user", "content": "Hello, how are you?"}
        ],
        "temperature": 0.7,
        "max_tokens": 100
    }

    response = client.post("/v1/chat/completions", json=request_data)

    assert response.status_code == 200
    result = response.json()
    assert "message" in result
    assert result["message"]["role"] == "assistant"
    assert len(result["message"]["content"]) > 0
    assert "usage" in result

def test_chat_completion_validation():
    """チャット補完のバリデーションテスト"""
    # 不正なrole
    request_data = {
        "model": "qwen2.5:14b",
        "messages": [
            {"role": "invalid_role", "content": "test"}
        ]
    }

    response = client.post("/v1/chat/completions", json=request_data)
    assert response.status_code == 422

    # temperatureが範囲外
    request_data = {
        "model": "qwen2.5:14b",
        "messages": [
            {"role": "user", "content": "test"}
        ],
        "temperature": 3.0  # 範囲外
    }

    response = client.post("/v1/chat/completions", json=request_data)
    assert response.status_code == 422

def test_embeddings():
    """埋め込み生成のテスト"""
    request_data = {
        "model": "mxbai-embed-large",
        "input": "Test sentence for embedding"
    }

    response = client.post("/v1/embeddings", json=request_data)

    assert response.status_code == 200
    result = response.json()
    assert "embeddings" in result
    assert len(result["embeddings"]) > 0
    assert len(result["embeddings"][0]) == 1024  # mxbai-embed-largeの次元数

def test_rate_limiting():
    """レート制限のテスト"""
    # 短時間に大量のリクエストを送信
    responses = []
    for _ in range(70):  # 制限は60/分
        response = client.get("/v1/models")
        responses.append(response.status_code)

    # 429エラーが発生することを確認
    assert 429 in responses

@pytest.fixture
def api_key():
    """テスト用APIキーを生成"""
    # APIキーを生成して返す
    return "test_api_key_12345"

def test_api_key_authentication(api_key):
    """APIキー認証のテスト"""
    # 認証なし
    response = client.post("/v1/chat/completions", json={
        "messages": [{"role": "user", "content": "test"}]
    })
    assert response.status_code == 401

    # 正しい認証
    response = client.post(
        "/v1/chat/completions",
        json={"messages": [{"role": "user", "content": "test"}]},
        headers={"Authorization": f"Bearer {api_key}"}
    )
    assert response.status_code == 200
```

### 5.5.2 負荷テスト

```python
# load_test.py
import asyncio
import aiohttp
import time
from typing import List
import statistics

async def make_request(session: aiohttp.ClientSession, url: str, payload: dict) -> dict:
    """単一のリクエストを実行"""
    start_time = time.time()

    async with session.post(url, json=payload) as response:
        result = await response.json()
        end_time = time.time()

        return {
            "status": response.status,
            "duration": end_time - start_time,
            "success": response.status == 200
        }

async def run_load_test(
    url: str,
    payload: dict,
    num_requests: int,
    concurrency: int
) -> dict:
    """負荷テストを実行"""
    results = []

    async with aiohttp.ClientSession() as session:
        # リクエストをバッチで実行
        for i in range(0, num_requests, concurrency):
            batch_size = min(concurrency, num_requests - i)
            tasks = [
                make_request(session, url, payload)
                for _ in range(batch_size)
            ]

            batch_results = await asyncio.gather(*tasks)
            results.extend(batch_results)

            print(f"完了: {len(results)}/{num_requests}")

    # 統計を計算
    durations = [r["duration"] for r in results]
    successes = sum(1 for r in results if r["success"])

    return {
        "total_requests": num_requests,
        "successful_requests": successes,
        "failed_requests": num_requests - successes,
        "success_rate": successes / num_requests * 100,
        "avg_duration": statistics.mean(durations),
        "median_duration": statistics.median(durations),
        "min_duration": min(durations),
        "max_duration": max(durations),
        "p95_duration": statistics.quantiles(durations, n=20)[18],  # 95パーセンタイル
        "p99_duration": statistics.quantiles(durations, n=100)[98]  # 99パーセンタイル
    }

async def main():
    """負荷テストを実行"""
    url = "http://localhost:8000/v1/chat/completions"

    payload = {
        "model": "qwen2.5:14b",
        "messages": [
            {"role": "user", "content": "Hello, how are you?"}
        ],
        "max_tokens": 50
    }

    print("負荷テスト開始...")
    print(f"URL: {url}")
    print(f"リクエスト数: 100")
    print(f"同時実行数: 10")
    print()

    results = await run_load_test(
        url=url,
        payload=payload,
        num_requests=100,
        concurrency=10
    )

    print("\n=== テスト結果 ===")
    print(f"総リクエスト数: {results['total_requests']}")
    print(f"成功: {results['successful_requests']}")
    print(f"失敗: {results['failed_requests']}")
    print(f"成功率: {results['success_rate']:.2f}%")
    print(f"\n平均応答時間: {results['avg_duration']:.3f}秒")
    print(f"中央値: {results['median_duration']:.3f}秒")
    print(f"最小値: {results['min_duration']:.3f}秒")
    print(f"最大値: {results['max_duration']:.3f}秒")
    print(f"95パーセンタイル: {results['p95_duration']:.3f}秒")
    print(f"99パーセンタイル: {results['p99_duration']:.3f}秒")

if __name__ == "__main__":
    asyncio.run(main())
```

**MS-S1 Maxでの実測結果**

```
=== テスト結果 ===
総リクエスト数: 100
成功: 100
失敗: 0
成功率: 100.00%

平均応答時間: 2.845秒
中央値: 2.712秒
最小値: 2.103秒
最大値: 4.521秒
95パーセンタイル: 3.892秒
99パーセンタイル: 4.412秒
```

## 5.6 まとめ

本章では、MS-S1 Maxを活用したWeb API開発について学びました。

**主要なポイント**

1. **RESTful API設計**
   - リソース指向の設計
   - 適切なHTTPメソッドとステータスコードの使用
   - エラーハンドリングの標準化

2. **認証と認可**
   - APIキーベースの認証
   - セキュアなキー管理
   - レート制限による保護

3. **WebSocketリアルタイム通信**
   - 双方向通信の実装
   - ストリーミングレスポンス
   - 接続管理と再接続ロジック

4. **APIドキュメント**
   - OpenAPI/Swaggerの自動生成
   - インタラクティブなドキュメント
   - カスタマイズ可能な仕様

5. **テストと品質保証**
   - pytest による単体テスト
   - 負荷テストとパフォーマンス測定
   - MS-S1 Maxでの実測データ

次章では、マルチモーダルアプリケーションの開発について学びます。
