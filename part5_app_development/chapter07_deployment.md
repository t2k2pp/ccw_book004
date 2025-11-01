# Chapter 07: デプロイメント

## 7.1 Dockerコンテナ化

### 7.1.1 基本的なDockerfile

```dockerfile
# Dockerfile
FROM ubuntu:24.04

# 環境変数
ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1

# ROCm 6.4.2のインストール
RUN apt-get update && apt-get install -y \
    wget \
    gnupg2 \
    && wget -q -O - https://repo.radeon.com/rocm/rocm.gpg.key | apt-key add - \
    && echo 'deb [arch=amd64] https://repo.radeon.com/rocm/apt/6.4.2 jammy main' \
    > /etc/apt/sources.list.d/rocm.list \
    && apt-get update \
    && apt-get install -y \
    rocm-dev \
    rocm-libs \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Python環境
RUN apt-get update && apt-get install -y \
    python3.11 \
    python3-pip \
    python3.11-venv \
    git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 作業ディレクトリ
WORKDIR /app

# 依存関係をコピー
COPY requirements.txt .

# PyTorch + ROCmをインストール
RUN pip3 install --no-cache-dir \
    torch==2.6.0+rocm6.2 \
    torchvision==0.20.0+rocm6.2 \
    --index-url https://download.pytorch.org/whl/rocm6.2

# その他の依存関係
RUN pip3 install --no-cache-dir -r requirements.txt

# アプリケーションコードをコピー
COPY . .

# ポートを公開
EXPOSE 8000

# 起動コマンド
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**requirements.txt**

```txt
fastapi==0.115.0
uvicorn[standard]==0.32.0
pydantic==2.9.2
ollama==0.4.4
chromadb==0.5.23
python-multipart==0.0.12
aiofiles==24.1.0
redis==5.2.0
celery==5.4.0
psycopg2-binary==2.9.10
sqlalchemy==2.0.36
alembic==1.14.0
faster-whisper==1.1.0
pillow==11.0.0
numpy==2.1.3
pandas==2.2.3
```

### 7.1.2 マルチステージビルド（最適化版）

```dockerfile
# Dockerfile.optimized
# ビルドステージ
FROM ubuntu:24.04 AS builder

ENV DEBIAN_FRONTEND=noninteractive

# ビルドツールのインストール
RUN apt-get update && apt-get install -y \
    python3.11 \
    python3-pip \
    python3.11-venv \
    git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# 依存関係のインストール
COPY requirements.txt .
RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir \
    torch==2.6.0+rocm6.2 \
    torchvision==0.20.0+rocm6.2 \
    --index-url https://download.pytorch.org/whl/rocm6.2 && \
    pip install --no-cache-dir -r requirements.txt

# 実行ステージ
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1
ENV PATH="/opt/venv/bin:$PATH"

# ROCmランタイムのみインストール
RUN apt-get update && apt-get install -y \
    wget \
    gnupg2 \
    && wget -q -O - https://repo.radeon.com/rocm/rocm.gpg.key | apt-key add - \
    && echo 'deb [arch=amd64] https://repo.radeon.com/rocm/apt/6.4.2 jammy main' \
    > /etc/apt/sources.list.d/rocm.list \
    && apt-get update \
    && apt-get install -y \
    rocm-libs \
    python3.11 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 非rootユーザーを作成
RUN useradd -m -u 1000 appuser

WORKDIR /app

# ビルドステージから仮想環境をコピー
COPY --from=builder /opt/venv /opt/venv

# アプリケーションコードをコピー
COPY --chown=appuser:appuser . .

# ユーザーを切り替え
USER appuser

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### 7.1.3 Docker Compose設定

```yaml
# docker-compose.yml
version: '3.8'

services:
  # FastAPI アプリケーション
  api:
    build:
      context: .
      dockerfile: Dockerfile.optimized
    container_name: local-ai-api
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/aidb
      - REDIS_URL=redis://redis:6379/0
      - OLLAMA_HOST=http://ollama:11434
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    depends_on:
      - postgres
      - redis
      - ollama
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    group_add:
      - video
      - render

  # Ollama サービス
  ollama:
    image: ollama/ollama:rocm
    container_name: ollama-service
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    group_add:
      - video
      - render
    environment:
      - HSA_OVERRIDE_GFX_VERSION=11.0.0
      - PYTORCH_ROCM_ARCH=gfx1100

  # PostgreSQL データベース
  postgres:
    image: postgres:16-alpine
    container_name: postgres-db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=aidb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  # Redis キャッシュ
  redis:
    image: redis:7-alpine
    container_name: redis-cache
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

  # Celery ワーカー
  celery-worker:
    build:
      context: .
      dockerfile: Dockerfile.optimized
    container_name: celery-worker
    restart: unless-stopped
    command: celery -A tasks worker --loglevel=info --concurrency=4
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/aidb
      - REDIS_URL=redis://redis:6379/0
      - OLLAMA_HOST=http://ollama:11434
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    depends_on:
      - postgres
      - redis
      - ollama
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    group_add:
      - video
      - render

  # Nginx リバースプロキシ
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./nginx/logs:/var/log/nginx
    depends_on:
      - api

volumes:
  ollama-data:
  postgres-data:
  redis-data:
```

### 7.1.4 Nginx設定

```nginx
# nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # gzip圧縮
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript
               application/json application/javascript application/xml+rss;

    # レート制限
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=60r/m;

    upstream api_backend {
        server api:8000;
    }

    # HTTPサーバー（HTTPSへリダイレクト）
    server {
        listen 80;
        server_name localhost;

        location / {
            return 301 https://$server_name$request_uri;
        }
    }

    # HTTPSサーバー
    server {
        listen 443 ssl http2;
        server_name localhost;

        ssl_certificate /etc/nginx/ssl/server.crt;
        ssl_certificate_key /etc/nginx/ssl/server.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        # セキュリティヘッダー
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # APIエンドポイント
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;

            proxy_pass http://api_backend/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # WebSocketサポート
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            # タイムアウト設定
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 300s;
        }

        # ストリーミングエンドポイント
        location /stream/ {
            proxy_pass http://api_backend/stream/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_buffering off;
            proxy_cache off;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            chunked_transfer_encoding on;
        }

        # ヘルスチェック
        location /health {
            proxy_pass http://api_backend/health;
            access_log off;
        }

        # 静的ファイル
        location /static/ {
            alias /app/static/;
            expires 30d;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

## 7.2 環境管理

### 7.2.1 環境変数管理

```bash
# .env.example
# アプリケーション設定
APP_NAME=LocalAIService
APP_ENV=production
DEBUG=false
LOG_LEVEL=INFO

# データベース
DATABASE_URL=postgresql://postgres:password@postgres:5432/aidb
DATABASE_POOL_SIZE=10
DATABASE_MAX_OVERFLOW=20

# Redis
REDIS_URL=redis://redis:6379/0
REDIS_MAX_CONNECTIONS=50

# Ollama
OLLAMA_HOST=http://ollama:11434
OLLAMA_TIMEOUT=300

# セキュリティ
SECRET_KEY=your-secret-key-here-change-in-production
API_KEY_SALT=your-api-key-salt-here
ALLOWED_ORIGINS=http://localhost:3000,https://yourdomain.com

# ROCm設定
HSA_OVERRIDE_GFX_VERSION=11.0.0
PYTORCH_ROCM_ARCH=gfx1100
GPU_MAX_ALLOC_PERCENT=95

# Celery
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/1
CELERY_TASK_SERIALIZER=json
CELERY_RESULT_SERIALIZER=json

# モニタリング
ENABLE_METRICS=true
METRICS_PORT=9090
```

### 7.2.2 設定管理

```python
# config.py
from pydantic_settings import BaseSettings
from typing import Optional, List
from functools import lru_cache

class Settings(BaseSettings):
    """アプリケーション設定"""

    # アプリケーション
    app_name: str = "LocalAIService"
    app_env: str = "development"
    debug: bool = False
    log_level: str = "INFO"

    # データベース
    database_url: str
    database_pool_size: int = 10
    database_max_overflow: int = 20

    # Redis
    redis_url: str
    redis_max_connections: int = 50

    # Ollama
    ollama_host: str = "http://localhost:11434"
    ollama_timeout: int = 300

    # セキュリティ
    secret_key: str
    api_key_salt: str
    allowed_origins: List[str] = ["http://localhost:3000"]

    # ROCm
    hsa_override_gfx_version: str = "11.0.0"
    pytorch_rocm_arch: str = "gfx1100"
    gpu_max_alloc_percent: int = 95

    # Celery
    celery_broker_url: str
    celery_result_backend: str

    # モニタリング
    enable_metrics: bool = True
    metrics_port: int = 9090

    class Config:
        env_file = ".env"
        case_sensitive = False

@lru_cache()
def get_settings() -> Settings:
    """設定のシングルトンインスタンスを取得"""
    return Settings()

# 使用例
settings = get_settings()
```

## 7.3 データベースマイグレーション

### 7.3.1 Alembic設定

```python
# alembic/env.py
from logging.config import fileConfig
from sqlalchemy import engine_from_config
from sqlalchemy import pool
from alembic import context
from config import get_settings

# モデルをインポート
from models import Base

config = context.config
settings = get_settings()

# データベースURLを設定
config.set_main_option("sqlalchemy.url", settings.database_url)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata

def run_migrations_offline() -> None:
    """オフラインモードでマイグレーションを実行"""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online() -> None:
    """オンラインモードでマイグレーションを実行"""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata
        )

        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### 7.3.2 マイグレーションスクリプト例

```python
# alembic/versions/001_initial_schema.py
"""Initial schema

Revision ID: 001
Revises:
Create Date: 2025-01-01 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa

revision = '001'
down_revision = None
branch_labels = None
depends_on = None

def upgrade() -> None:
    # ユーザーテーブル
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('username', sa.String(50), nullable=False),
        sa.Column('email', sa.String(100), nullable=False),
        sa.Column('api_key_hash', sa.String(256), nullable=True),
        sa.Column('created_at', sa.DateTime(), nullable=False),
        sa.Column('updated_at', sa.DateTime(), nullable=False),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('username'),
        sa.UniqueConstraint('email')
    )
    op.create_index('ix_users_email', 'users', ['email'])

    # セッションテーブル
    op.create_table(
        'sessions',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('user_id', sa.Integer(), nullable=False),
        sa.Column('created_at', sa.DateTime(), nullable=False),
        sa.Column('last_activity', sa.DateTime(), nullable=False),
        sa.Column('metadata', sa.JSON(), nullable=True),
        sa.ForeignKeyConstraint(['user_id'], ['users.id'], ondelete='CASCADE'),
        sa.PrimaryKeyConstraint('id')
    )
    op.create_index('ix_sessions_user_id', 'sessions', ['user_id'])

    # メッセージテーブル
    op.create_table(
        'messages',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('session_id', sa.String(36), nullable=False),
        sa.Column('role', sa.String(20), nullable=False),
        sa.Column('content', sa.Text(), nullable=False),
        sa.Column('timestamp', sa.DateTime(), nullable=False),
        sa.Column('metadata', sa.JSON(), nullable=True),
        sa.ForeignKeyConstraint(['session_id'], ['sessions.id'], ondelete='CASCADE'),
        sa.PrimaryKeyConstraint('id')
    )
    op.create_index('ix_messages_session_id', 'messages', ['session_id'])

def downgrade() -> None:
    op.drop_index('ix_messages_session_id', 'messages')
    op.drop_table('messages')
    op.drop_index('ix_sessions_user_id', 'sessions')
    op.drop_table('sessions')
    op.drop_index('ix_users_email', 'users')
    op.drop_table('users')
```

## 7.4 ヘルスチェックとモニタリング

### 7.4.1 ヘルスチェックエンドポイント

```python
# health.py
from fastapi import APIRouter, status
from typing import Dict
import psutil
import time
from datetime import datetime
from sqlalchemy import text
from database import get_db
import redis
import ollama

router = APIRouter()

@router.get("/health")
async def health_check() -> Dict:
    """基本的なヘルスチェック"""
    return {
        "status": "healthy",
        "timestamp": datetime.now().isoformat()
    }

@router.get("/health/detailed")
async def detailed_health_check() -> Dict:
    """詳細なヘルスチェック"""
    checks = {}

    # データベース接続
    try:
        db = next(get_db())
        db.execute(text("SELECT 1"))
        checks["database"] = {"status": "healthy"}
    except Exception as e:
        checks["database"] = {"status": "unhealthy", "error": str(e)}

    # Redis接続
    try:
        r = redis.from_url(settings.redis_url)
        r.ping()
        checks["redis"] = {"status": "healthy"}
    except Exception as e:
        checks["redis"] = {"status": "unhealthy", "error": str(e)}

    # Ollama接続
    try:
        models = ollama.list()
        checks["ollama"] = {
            "status": "healthy",
            "models_count": len(models.get('models', []))
        }
    except Exception as e:
        checks["ollama"] = {"status": "unhealthy", "error": str(e)}

    # システムリソース
    checks["system"] = {
        "cpu_percent": psutil.cpu_percent(interval=1),
        "memory_percent": psutil.virtual_memory().percent,
        "disk_percent": psutil.disk_usage('/').percent
    }

    # 全体のステータス
    overall_status = "healthy" if all(
        check.get("status") == "healthy"
        for check in [checks["database"], checks["redis"], checks["ollama"]]
    ) else "unhealthy"

    return {
        "status": overall_status,
        "timestamp": datetime.now().isoformat(),
        "checks": checks
    }

@router.get("/health/ready")
async def readiness_check() -> Dict:
    """準備状態チェック（Kubernetes用）"""
    try:
        # 必須サービスの確認
        db = next(get_db())
        db.execute(text("SELECT 1"))

        r = redis.from_url(settings.redis_url)
        r.ping()

        ollama.list()

        return {
            "status": "ready",
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        return {
            "status": "not_ready",
            "error": str(e),
            "timestamp": datetime.now().isoformat()
        }

@router.get("/health/live")
async def liveness_check() -> Dict:
    """生存確認（Kubernetes用）"""
    return {
        "status": "alive",
        "timestamp": datetime.now().isoformat()
    }
```

### 7.4.2 Prometheusメトリクス

```python
# metrics.py
from prometheus_client import Counter, Histogram, Gauge, generate_latest
from fastapi import APIRouter, Response
import time
from functools import wraps

router = APIRouter()

# カウンター
request_count = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

# ヒストグラム
request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint']
)

# ゲージ
active_requests = Gauge(
    'http_active_requests',
    'Number of active requests'
)

# AI関連メトリクス
ai_request_count = Counter(
    'ai_requests_total',
    'Total AI requests',
    ['model', 'endpoint']
)

ai_request_duration = Histogram(
    'ai_request_duration_seconds',
    'AI request duration',
    ['model', 'endpoint']
)

ai_tokens_generated = Counter(
    'ai_tokens_generated_total',
    'Total tokens generated',
    ['model']
)

def track_metrics(endpoint: str):
    """メトリクス追跡デコレーター"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            active_requests.inc()
            start_time = time.time()

            try:
                result = await func(*args, **kwargs)
                status = 200
                return result
            except Exception as e:
                status = 500
                raise
            finally:
                duration = time.time() - start_time
                request_duration.labels(
                    method='POST',
                    endpoint=endpoint
                ).observe(duration)
                request_count.labels(
                    method='POST',
                    endpoint=endpoint,
                    status=status
                ).inc()
                active_requests.dec()

        return wrapper
    return decorator

@router.get("/metrics")
async def metrics():
    """Prometheusメトリクスを公開"""
    return Response(content=generate_latest(), media_type="text/plain")
```

## 7.5 バックアップと復旧

### 7.5.1 自動バックアップスクリプト

```bash
#!/bin/bash
# backup.sh

set -e

BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${DATE}"

echo "Starting backup: ${BACKUP_NAME}"

# データベースバックアップ
echo "Backing up PostgreSQL..."
docker exec postgres-db pg_dump -U postgres aidb | gzip > "${BACKUP_DIR}/${BACKUP_NAME}_db.sql.gz"

# Redisバックアップ
echo "Backing up Redis..."
docker exec redis-cache redis-cli SAVE
docker cp redis-cache:/data/dump.rdb "${BACKUP_DIR}/${BACKUP_NAME}_redis.rdb"

# ChromaDBバックアップ
echo "Backing up ChromaDB..."
tar -czf "${BACKUP_DIR}/${BACKUP_NAME}_chroma.tar.gz" -C /app/data chroma_db/

# Ollamaモデルバックアップ
echo "Backing up Ollama models..."
docker exec ollama-service tar -czf - /root/.ollama > "${BACKUP_DIR}/${BACKUP_NAME}_ollama.tar.gz"

# 古いバックアップを削除（30日以上前）
echo "Cleaning up old backups..."
find ${BACKUP_DIR} -name "backup_*" -type f -mtime +30 -delete

echo "Backup completed: ${BACKUP_NAME}"
```

### 7.5.2 復旧スクリプト

```bash
#!/bin/bash
# restore.sh

set -e

if [ -z "$1" ]; then
    echo "Usage: $0 <backup_name>"
    echo "Available backups:"
    ls -1 /backups/ | grep "backup_" | cut -d'_' -f2-3 | sort -u
    exit 1
fi

BACKUP_NAME="$1"
BACKUP_DIR="/backups"

echo "Starting restore from: ${BACKUP_NAME}"

# PostgreSQL復旧
echo "Restoring PostgreSQL..."
gunzip < "${BACKUP_DIR}/backup_${BACKUP_NAME}_db.sql.gz" | \
    docker exec -i postgres-db psql -U postgres aidb

# Redis復旧
echo "Restoring Redis..."
docker stop redis-cache
docker cp "${BACKUP_DIR}/backup_${BACKUP_NAME}_redis.rdb" redis-cache:/data/dump.rdb
docker start redis-cache

# ChromaDB復旧
echo "Restoring ChromaDB..."
tar -xzf "${BACKUP_DIR}/backup_${BACKUP_NAME}_chroma.tar.gz" -C /app/data/

# Ollama復旧
echo "Restoring Ollama models..."
docker exec -i ollama-service tar -xzf - -C / < "${BACKUP_DIR}/backup_${BACKUP_NAME}_ollama.tar.gz"

echo "Restore completed"
```

## 7.6 CI/CDパイプライン

### 7.6.1 GitHub Actions設定

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest pytest-cov

    - name: Run tests
      run: |
        pytest tests/ --cov=. --cov-report=xml

    - name: Upload coverage
      uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        file: ./Dockerfile.optimized
        push: true
        tags: yourusername/local-ai-api:latest
        cache-from: type=registry,ref=yourusername/local-ai-api:buildcache
        cache-to: type=registry,ref=yourusername/local-ai-api:buildcache,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
    - name: Deploy to server
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        script: |
          cd /opt/local-ai
          docker-compose pull
          docker-compose up -d
          docker system prune -f
```

## 7.7 本番環境運用

### 7.7.1 起動・停止スクリプト

```bash
#!/bin/bash
# manage.sh

set -e

COMPOSE_FILE="docker-compose.yml"

case "$1" in
    start)
        echo "Starting services..."
        docker-compose -f ${COMPOSE_FILE} up -d
        echo "Services started"
        ;;

    stop)
        echo "Stopping services..."
        docker-compose -f ${COMPOSE_FILE} down
        echo "Services stopped"
        ;;

    restart)
        echo "Restarting services..."
        docker-compose -f ${COMPOSE_FILE} restart
        echo "Services restarted"
        ;;

    logs)
        docker-compose -f ${COMPOSE_FILE} logs -f --tail=100
        ;;

    status)
        docker-compose -f ${COMPOSE_FILE} ps
        ;;

    update)
        echo "Updating services..."
        docker-compose -f ${COMPOSE_FILE} pull
        docker-compose -f ${COMPOSE_FILE} up -d
        echo "Services updated"
        ;;

    backup)
        ./backup.sh
        ;;

    restore)
        if [ -z "$2" ]; then
            echo "Usage: $0 restore <backup_name>"
            exit 1
        fi
        ./restore.sh "$2"
        ;;

    *)
        echo "Usage: $0 {start|stop|restart|logs|status|update|backup|restore}"
        exit 1
        ;;
esac
```

### 7.7.2 systemdサービス設定

```ini
# /etc/systemd/system/local-ai.service
[Unit]
Description=Local AI Service
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/local-ai
ExecStart=/usr/bin/docker-compose up -d
ExecStop=/usr/bin/docker-compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

## 7.8 まとめ

本章では、MS-S1 Maxで動作するAIアプリケーションのデプロイメントについて学びました。

**主要なポイント**

1. **Dockerコンテナ化**
   - ROCm対応のDockerfile作成
   - マルチステージビルドによる最適化
   - Docker Composeによるマルチコンテナ管理

2. **インフラ構成**
   - Nginxリバースプロキシ
   - PostgreSQL、Redis統合
   - SSL/TLS設定

3. **運用管理**
   - 環境変数管理
   - データベースマイグレーション
   - ヘルスチェックとモニタリング

4. **バックアップと復旧**
   - 自動バックアップスクリプト
   - 迅速な復旧手順
   - データ保護戦略

5. **CI/CD**
   - GitHub Actionsによる自動化
   - テスト、ビルド、デプロイのパイプライン
   - 継続的インテグレーション

次章では、本番環境でのモニタリングとトラブルシューティングについて学びます。
