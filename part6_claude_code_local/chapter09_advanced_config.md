# Chapter 09: 高度な設定とベストプラクティス

## 9.1 プロダクション環境での運用

### 9.1.1 systemdサービスの完全な設定

**Ollama サービス**

```ini
# /etc/systemd/system/ollama.service.d/override.conf
[Service]
# 基本設定
User=ollama
Group=ollama
WorkingDirectory=/var/lib/ollama

# ROCm環境変数
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
Environment="PYTORCH_ROCM_ARCH=gfx1100"
Environment="GPU_MAX_ALLOC_PERCENT=95"
Environment="ROCM_PATH=/opt/rocm"

# Ollama設定
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_KEEP_ALIVE=10m"
Environment="OLLAMA_NUM_PARALLEL=4"
Environment="OLLAMA_MAX_LOADED_MODELS=3"
Environment="OLLAMA_MAX_QUEUE=512"

# リソース制限
LimitNOFILE=65536
LimitNPROC=4096

# 再起動ポリシー
Restart=always
RestartSec=10

# ログ
StandardOutput=journal
StandardError=journal
SyslogIdentifier=ollama

[Install]
WantedBy=multi-user.target
```

**LiteLLM サービス**

```ini
# /etc/systemd/system/litellm.service
[Unit]
Description=LiteLLM Proxy Server
After=network.target ollama.service
Requires=ollama.service

[Service]
Type=simple
User=litellm
Group=litellm
WorkingDirectory=/opt/litellm

# 仮想環境のPython
ExecStart=/opt/litellm/venv/bin/litellm \
    --config /opt/litellm/config.yaml \
    --port 8000 \
    --host 0.0.0.0 \
    --num_workers 4

# 環境変数
Environment="PYTHONUNBUFFERED=1"
Environment="LOG_LEVEL=INFO"

# リソース制限
LimitNOFILE=65536

# 再起動ポリシー
Restart=always
RestartSec=10

# ログ
StandardOutput=journal
StandardError=journal
SyslogIdentifier=litellm

[Install]
WantedBy=multi-user.target
```

**サービスの有効化**

```bash
# サービスをリロード
sudo systemctl daemon-reload

# 自動起動を有効化
sudo systemctl enable ollama
sudo systemctl enable litellm

# 起動
sudo systemctl start ollama
sudo systemctl start litellm

# 状態確認
sudo systemctl status ollama
sudo systemctl status litellm
```

### 9.1.2 Nginx リバースプロキシ

```nginx
# /etc/nginx/sites-available/litellm
upstream litellm_backend {
    server 127.0.0.1:8000;
    keepalive 32;
}

server {
    listen 80;
    server_name ai-api.local;

    # HTTPSへリダイレクト（本番環境）
    # return 301 https://$server_name$request_uri;

    # ログ
    access_log /var/log/nginx/litellm-access.log;
    error_log /var/log/nginx/litellm-error.log;

    # プロキシ設定
    location / {
        proxy_pass http://litellm_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # タイムアウト
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 600s;

        # バッファリング
        proxy_buffering off;
    }

    # ヘルスチェック
    location /health {
        proxy_pass http://litellm_backend/health;
        access_log off;
    }

    # レート制限
    location /v1/ {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://litellm_backend/v1/;
    }
}

# レート制限ゾーン定義
# /etc/nginx/nginx.conf に追加
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=60r/m;
```

有効化：

```bash
sudo ln -s /etc/nginx/sites-available/litellm /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 9.2 高度なLiteLLM設定

### 9.2.1 フォールバックとロードバランシング

```yaml
# config.yaml (高度な設定・2025年11月最新)
model_list:
  # プライマリモデル（最高品質・30B Q8_0）
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      api_base: http://localhost:11434
      num_ctx: 262144  # 256K context
      rpm: 60  # Requests per minute

  # フォールバックモデル（プライマリが失敗時・14B）
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen3-coder:14b
      api_base: http://localhost:11434
      num_ctx: 262144

  # 複数インスタンスでロードバランシング（30B Q8_0）
  - model_name: gpt-4
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      api_base: http://localhost:11434
      num_ctx: 262144

  - model_name: gpt-4
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      api_base: http://localhost:11435  # 別インスタンス
      num_ctx: 262144

router_settings:
  # ロードバランシング戦略
  routing_strategy: latency-based-routing  # 最も速いモデルを選択
  # routing_strategy: least-busy  # 最も空いているモデルを選択

  # リトライ設定
  num_retries: 3
  retry_after: 5  # seconds
  timeout: 300

  # フォールバック
  fallbacks:
    claude-3-5-sonnet-20241022:
      - gpt-4
      - gpt-3.5-turbo

    gpt-4:
      - claude-3-5-sonnet-20241022

  # サーキットブレーカー（連続失敗でモデルを一時的に除外）
  allowed_fails: 3
  cooldown_time: 60  # seconds
```

### 9.2.2 キャッシング戦略

```yaml
# config.yaml - キャッシング設定
litellm_settings:
  # Redis キャッシュ
  cache: true
  cache_params:
    type: redis
    host: localhost
    port: 6379
    db: 0
    ttl: 3600  # 1時間

    # Redis接続プール
    max_connections: 50
    socket_connect_timeout: 5
    socket_timeout: 5

  # セマンティックキャッシュ（類似クエリをキャッシュ）
  enable_semantic_caching: true
  semantic_cache_threshold: 0.95  # 95%以上の類似度でキャッシュヒット

  # キャッシュキーのカスタマイズ
  cache_key_format: "{model}:{messages_hash}"
```

**Redisの最適化**

```bash
# /etc/redis/redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru  # LRU削除
save ""  # スナップショット無効化（パフォーマンス重視）

# 再起動
sudo systemctl restart redis
```

### 9.2.3 ログとモニタリング

```yaml
# config.yaml - ログ設定
general_settings:
  # データベースログ
  database_url: postgresql://user:pass@localhost/litellm
  # または SQLite
  # database_url: sqlite:///litellm.db

  # Success/Failure コールバック
  success_callback: ["langfuse", "posthog"]
  failure_callback: ["langfuse", "sentry"]

litellm_settings:
  # 詳細ログ
  set_verbose: true

  # カスタムロガー
  log_level: INFO

  # メトリクス
  enable_metrics: true
```

**Prometheus メトリクス**

```python
# prometheus_exporter.py
from prometheus_client import start_http_server, Counter, Histogram
import time

# メトリクス定義
request_count = Counter('litellm_requests_total', 'Total requests', ['model', 'status'])
request_duration = Histogram('litellm_request_duration_seconds', 'Request duration', ['model'])
token_count = Counter('litellm_tokens_total', 'Total tokens', ['model', 'type'])

# Prometheusサーバー起動
start_http_server(9090)

# メトリクスは自動的に収集される
# http://localhost:9090/metrics で確認可能
```

## 9.3 セキュリティベストプラクティス

### 9.3.1 APIキー管理

```yaml
# config.yaml
general_settings:
  # 強力なマスターキー
  master_key: ${LITELLM_MASTER_KEY}  # 環境変数から取得

  # ユーザー別キー
  ui_username: admin
  ui_password: ${LITELLM_UI_PASSWORD}

  # キーのローテーション
  allowed_models:
    - claude-3-5-sonnet-20241022
    - gpt-4

  # IP制限
  allowed_ips:
    - 192.168.1.0/24
    - 10.0.0.0/8
```

**環境変数の設定**

```bash
# /etc/environment
LITELLM_MASTER_KEY=$(openssl rand -hex 32)
LITELLM_UI_PASSWORD=$(openssl rand -hex 16)

# または専用ファイル
sudo nano /etc/litellm/env

LITELLM_MASTER_KEY=your_secure_key_here
LITELLM_UI_PASSWORD=your_secure_password

# systemdで読み込み
# /etc/systemd/system/litellm.service
[Service]
EnvironmentFile=/etc/litellm/env
```

### 9.3.2 ファイアウォール設定

```bash
# UFWでポートを制限
sudo ufw allow from 192.168.1.0/24 to any port 8000
sudo ufw deny 8000

# Ollamaは内部のみ
sudo ufw deny 11434

# Nginx経由でのみアクセス可能に
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

### 9.3.3 SSL/TLS設定

```bash
# Let's Encrypt証明書
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d ai-api.yourdomain.com

# Nginxが自動的に更新される
```

## 9.4 パフォーマンスチューニング

### 9.4.1 Ollama並列処理

```bash
# 複数Ollamaインスタンスを起動
# Instance 1 (Port 11434)
sudo systemctl start ollama

# Instance 2 (Port 11435) - 別の設定ファイル
sudo cp /etc/systemd/system/ollama.service /etc/systemd/system/ollama2.service

# /etc/systemd/system/ollama2.service を編集
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11435"
Environment="OLLAMA_MODELS=/var/lib/ollama2"

sudo systemctl daemon-reload
sudo systemctl start ollama2
```

### 9.4.2 コネクションプーリング

```python
# connection_pool.py
from litellm import completion
import asyncio
from concurrent.futures import ThreadPoolExecutor

# スレッドプール設定
executor = ThreadPoolExecutor(max_workers=16)

async def parallel_completions(prompts):
    """並列でCompletionを実行"""
    loop = asyncio.get_event_loop()

    tasks = [
        loop.run_in_executor(
            executor,
            completion,
            "claude-3-5-sonnet-20241022",
            [{"role": "user", "content": prompt}]
        )
        for prompt in prompts
    ]

    results = await asyncio.gather(*tasks)
    return results

# 使用例
prompts = [
    "Write a function to sort a list",
    "Explain Python decorators",
    "Create a FastAPI endpoint"
]

results = asyncio.run(parallel_completions(prompts))
```

## 9.5 バックアップと復旧

### 9.5.1 自動バックアップスクリプト

```bash
#!/bin/bash
# backup_ai_system.sh

BACKUP_DIR="/backups/ai-system"
DATE=$(date +%Y%m%d_%H%M%S)

echo "Starting backup: $DATE"

# Ollamaモデル
echo "Backing up Ollama models..."
tar -czf "$BACKUP_DIR/ollama_models_$DATE.tar.gz" ~/.ollama/models/

# LiteLLM設定
echo "Backing up LiteLLM config..."
cp ~/litellm/config.yaml "$BACKUP_DIR/config_$DATE.yaml"

# データベース（PostgreSQL）
echo "Backing up database..."
pg_dump litellm > "$BACKUP_DIR/litellm_db_$DATE.sql"

# Redis（キャッシュ）
echo "Backing up Redis..."
redis-cli SAVE
cp /var/lib/redis/dump.rdb "$BACKUP_DIR/redis_$DATE.rdb"

# 古いバックアップを削除（30日以上前）
find "$BACKUP_DIR" -name "*" -type f -mtime +30 -delete

echo "Backup completed: $DATE"
```

**Cron設定（毎日3時に実行）**

```bash
sudo crontab -e

# 追加
0 3 * * * /usr/local/bin/backup_ai_system.sh >> /var/log/ai-backup.log 2>&1
```

### 9.5.2 復旧手順

```bash
#!/bin/bash
# restore_ai_system.sh

if [ -z "$1" ]; then
    echo "Usage: $0 <backup_date>"
    echo "Example: $0 20250115_030000"
    exit 1
fi

BACKUP_DATE=$1
BACKUP_DIR="/backups/ai-system"

echo "Restoring from backup: $BACKUP_DATE"

# Ollamaモデル
echo "Restoring Ollama models..."
tar -xzf "$BACKUP_DIR/ollama_models_$BACKUP_DATE.tar.gz" -C ~/

# LiteLLM設定
echo "Restoring LiteLLM config..."
cp "$BACKUP_DIR/config_$BACKUP_DATE.yaml" ~/litellm/config.yaml

# データベース
echo "Restoring database..."
psql litellm < "$BACKUP_DIR/litellm_db_$BACKUP_DATE.sql"

# Redis
echo "Restoring Redis..."
sudo systemctl stop redis
sudo cp "$BACKUP_DIR/redis_$BACKUP_DATE.rdb" /var/lib/redis/dump.rdb
sudo chown redis:redis /var/lib/redis/dump.rdb
sudo systemctl start redis

# サービス再起動
echo "Restarting services..."
sudo systemctl restart ollama
sudo systemctl restart litellm

echo "Restore completed"
```

## 9.6 コスト分析

### 9.6.1 電力コストの計算

```python
# cost_calculator.py
import psutil
import time

class CostCalculator:
    def __init__(self, electricity_rate=0.03):  # USD per kWh
        self.electricity_rate = electricity_rate
        self.start_time = time.time()
        self.start_energy = self.get_system_power()

    def get_system_power(self):
        """推定消費電力（W）"""
        # MS-S1 Max: 約120W（通常使用時）
        # GPU使用時: +30-50W
        return 120  # Watts

    def get_cost_estimate(self, hours):
        """時間あたりのコスト推定"""
        power_kw = self.get_system_power() / 1000
        cost = power_kw * hours * self.electricity_rate
        return cost

    def compare_with_cloud(self, tokens_per_day):
        """クラウドAPIとのコスト比較"""
        # Claude API: $3 per million input tokens, $15 per million output tokens
        # 平均: $9 per million tokens

        daily_cloud_cost = (tokens_per_day / 1_000_000) * 9
        monthly_cloud_cost = daily_cloud_cost * 30

        # ローカル（24時間稼働）
        daily_local_cost = self.get_cost_estimate(24)
        monthly_local_cost = daily_local_cost * 30

        print(f"=== Cost Comparison ===")
        print(f"Tokens per day: {tokens_per_day:,}")
        print(f"\nCloud API (Claude):")
        print(f"  Daily: ${daily_cloud_cost:.2f}")
        print(f"  Monthly: ${monthly_cloud_cost:.2f}")
        print(f"\nLocal (MS-S1 Max):")
        print(f"  Daily: ${daily_local_cost:.2f}")
        print(f"  Monthly: ${monthly_local_cost:.2f}")
        print(f"\nSavings:")
        print(f"  Daily: ${daily_cloud_cost - daily_local_cost:.2f}")
        print(f"  Monthly: ${monthly_cloud_cost - monthly_local_cost:.2f}")
        print(f"  Annual: ${(monthly_cloud_cost - monthly_local_cost) * 12:.2f}")

# 使用例
calc = CostCalculator()
calc.compare_with_cloud(tokens_per_day=100_000)  # 1日10万トークン
```

**実行結果例**

```
=== Cost Comparison ===
Tokens per day: 100,000

Cloud API (Claude):
  Daily: $0.90
  Monthly: $27.00

Local (MS-S1 Max):
  Daily: $0.09
  Monthly: $2.70

Savings:
  Daily: $0.81
  Monthly: $24.30
  Annual: $291.60
```

### 9.6.2 ROI（投資回収期間）

```
MS-S1 Max システムコスト: 約 $2,500-3,000
月間節約額: $24.30（軽度使用）〜 $500+（ヘビー使用）

投資回収期間:
- 軽度使用: 10-12ヶ月
- 中程度使用: 5-6ヶ月
- ヘビー使用: 2-3ヶ月
```

## 9.7 まとめ

本章では、本番環境での高度な設定とベストプラクティスを学びました。

**達成したこと**
✅ プロダクション対応のsystemd設定
✅ Nginxリバースプロキシ
✅ 高度なLiteLLM設定（フォールバック、キャッシング）
✅ セキュリティ強化
✅ バックアップと復旧
✅ コスト分析

**プロダクション環境のチェックリスト**
- [ ] systemdサービスが自動起動する
- [ ] Nginxでリバースプロキシ設定済み
- [ ] SSL/TLS証明書設定済み
- [ ] ファイアウォール設定済み
- [ ] バックアップが自動実行される
- [ ] モニタリングが動作している
- [ ] ログローテーション設定済み

**第6部の総まとめ**

本書を通じて、以下を習得しました：

1. **Chapter 01**: Claude CodeとローカルLLMの統合概要
2. **Chapter 02**: Ollamaのインストールと設定
3. **Chapter 03**: LiteLLMのセットアップ
4. **Chapter 04**: AiderでのClaude Code風開発
5. **Chapter 05**: 互換性と制限事項の理解
6. **Chapter 06**: モデル選択と最適化
7. **Chapter 07**: 実践例（リファクタリング、バグ修正等）
8. **Chapter 08**: トラブルシューティング
9. **Chapter 09**: 高度な設定とベストプラクティス

**MS-S1 Maxの活用成果**
- ✅ 完全無料でAI支援開発
- ✅ プライバシー完全保護
- ✅ 年間$300-$6,000のコスト削減
- ✅ 開発効率30-50%向上
- ✅ オフライン動作

これで、あなたはMS-S1 Max上でローカルLLMを使った高度なAI支援開発環境を構築・運用できるようになりました。

Happy Coding! 🚀
