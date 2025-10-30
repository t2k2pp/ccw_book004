# 第8章:マルチモデル運用と同時実行

## 8.1 マルチモデル戦略

### 8.1.1 用途別モデル配置

MS-S1 Maxの128GBメモリを活用し、複数のモデルを常駐させます。

```bash
# 推奨構成1: 汎用+専門
qwen2.5:32b     # メイン（汎用）- 20GB
codellama:13b   # コーディング - 8GB
qwen2.5:7b      # 高速応答 - 5GB
# 合計: 33GB（95GB余裕あり）

# 推奨構成2: 大型+サポート
llama3.1:70b    # 高精度タスク - 42GB
qwen2.5:14b     # バランス - 9GB
qwen2.5:7b      # 軽量タスク - 5GB
# 合計: 56GB（72GB余裕あり）

# 推奨構成3: マルチタスク
qwen2.5:14b × 3  # 並行処理 - 27GB
codellama:7b     # コード - 4GB
# 合計: 31GB（97GB余裕あり）
```

### 8.1.2 自動ルーティング

```python
# model_router.py
import ollama

class ModelRouter:
    def __init__(self):
        self.models = {
            'code': 'codellama:13b',
            'chat': 'qwen2.5:14b',
            'fast': 'qwen2.5:7b',
            'powerful': 'qwen2.5:32b'
        }

    def route(self, prompt, task_type='auto'):
        if task_type == 'auto':
            task_type = self.detect_task(prompt)

        model = self.models.get(task_type, 'qwen2.5:14b')
        return ollama.generate(model=model, prompt=prompt)

    def detect_task(self, prompt):
        """タスク種別を自動検出"""
        code_keywords = ['function', 'code', 'program', 'script', 'def']
        if any(kw in prompt.lower() for kw in code_keywords):
            return 'code'

        if len(prompt) < 50:
            return 'fast'

        if 'detail' in prompt.lower() or 'explain' in prompt.lower():
            return 'powerful'

        return 'chat'

# 使用例
router = ModelRouter()

# 自動ルーティング
response1 = router.route("Write a Python function")  # → codellama
response2 = router.route("What is 2+2?")             # → qwen2.5:7b
response3 = router.route("Explain quantum physics in detail")  # → qwen2.5:32b
```

## 8.2 並行実行の実装

### 8.2.1 マルチスレッド処理

```python
# concurrent_requests.py
from concurrent.futures import ThreadPoolExecutor, as_completed
import ollama
import time

def process_request(request_id, model, prompt):
    start = time.time()

    response = ollama.generate(model=model, prompt=prompt)

    elapsed = time.time() - start

    return {
        'id': request_id,
        'model': model,
        'elapsed': elapsed,
        'response': response['response'][:100]
    }

# 複数リクエストを並行処理
requests = [
    (1, 'qwen2.5:7b', '日本の首都は?'),
    (2, 'qwen2.5:7b', 'フランスの首都は?'),
    (3, 'qwen2.5:7b', 'ドイツの首都は?'),
    (4, 'qwen2.5:7b', 'イタリアの首都は?'),
]

print(f"Processing {len(requests)} requests in parallel...")
start_total = time.time()

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = [
        executor.submit(process_request, req_id, model, prompt)
        for req_id, model, prompt in requests
    ]

    for future in as_completed(futures):
        result = future.result()
        print(f"Request {result['id']}: {result['elapsed']:.2f}s - {result['response']}")

total_elapsed = time.time() - start_total
print(f"\nTotal time: {total_elapsed:.2f}s")
print(f"Average: {total_elapsed/len(requests):.2f}s per request")
```

### 8.2.2 キューベースの処理

```python
# queue_processor.py
import queue
import threading
import ollama

class OllamaQueue:
    def __init__(self, num_workers=4):
        self.task_queue = queue.Queue()
        self.result_queue = queue.Queue()
        self.workers = []

        for i in range(num_workers):
            worker = threading.Thread(target=self._worker, daemon=True)
            worker.start()
            self.workers.append(worker)

    def _worker(self):
        while True:
            task = self.task_queue.get()
            if task is None:
                break

            task_id, model, prompt = task

            try:
                response = ollama.generate(model=model, prompt=prompt)
                self.result_queue.put((task_id, 'success', response))
            except Exception as e:
                self.result_queue.put((task_id, 'error', str(e)))

            self.task_queue.task_done()

    def submit(self, task_id, model, prompt):
        self.task_queue.put((task_id, model, prompt))

    def get_result(self, timeout=None):
        return self.result_queue.get(timeout=timeout)

    def wait(self):
        self.task_queue.join()

    def shutdown(self):
        for _ in self.workers:
            self.task_queue.put(None)
        for worker in self.workers:
            worker.join()

# 使用例
queue_processor = OllamaQueue(num_workers=4)

# タスク投入
for i in range(10):
    queue_processor.submit(i, 'qwen2.5:7b', f'Tell me about topic {i}')

# 結果取得
for i in range(10):
    task_id, status, result = queue_processor.get_result()
    print(f"Task {task_id}: {status}")

queue_processor.shutdown()
```

### 8.2.3 負荷分散

```python
# load_balancer.py
import ollama
from collections import defaultdict
import threading
import time

class LoadBalancer:
    def __init__(self, models):
        self.models = models  # 同じモデルを複数インスタンス想定
        self.stats = defaultdict(lambda: {'requests': 0, 'total_time': 0})
        self.locks = {model: threading.Lock() for model in models}

    def get_least_loaded_model(self):
        """負荷が最も低いモデルを選択"""
        min_load = float('inf')
        selected = self.models[0]

        for model in self.models:
            stats = self.stats[model]
            if stats['requests'] > 0:
                avg_time = stats['total_time'] / stats['requests']
                load = stats['requests'] * avg_time
            else:
                load = 0

            if load < min_load:
                min_load = load
                selected = model

        return selected

    def generate(self, prompt, **kwargs):
        model = self.get_least_loaded_model()

        with self.locks[model]:
            start = time.time()
            response = ollama.generate(model=model, prompt=prompt, **kwargs)
            elapsed = time.time() - start

            self.stats[model]['requests'] += 1
            self.stats[model]['total_time'] += elapsed

        return response

    def get_stats(self):
        return dict(self.stats)

# 使用例（複数の同じモデルがある場合）
lb = LoadBalancer(['qwen2.5:7b', 'qwen2.5:7b', 'qwen2.5:7b'])

# 並行リクエスト
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=10) as executor:
    futures = [
        executor.submit(lb.generate, f"Question {i}")
        for i in range(20)
    ]

    for future in futures:
        future.result()

print("Load balancer stats:", lb.get_stats())
```

## 8.3 ストリーミング並行処理

### 8.3.1 複数ストリームの同時処理

```python
# multi_stream.py
import ollama
import threading

def stream_response(stream_id, model, prompt):
    print(f"\n[Stream {stream_id}] Starting...")

    for chunk in ollama.generate(model=model, prompt=prompt, stream=True):
        if chunk.get('done'):
            print(f"\n[Stream {stream_id}] Complete")
            break
        print(f"[Stream {stream_id}] {chunk['response']}", end='', flush=True)

# 複数のストリームを並行実行
threads = []
prompts = [
    ('A', 'qwen2.5:7b', 'AIについて説明'),
    ('B', 'qwen2.5:7b', 'MLについて説明'),
    ('C', 'qwen2.5:7b', 'DLについて説明'),
]

for stream_id, model, prompt in prompts:
    thread = threading.Thread(
        target=stream_response,
        args=(stream_id, model, prompt)
    )
    thread.start()
    threads.append(thread)

for thread in threads:
    thread.join()

print("\n\nAll streams completed")
```

### 8.3.2 WebSocket マルチセッション

```python
# websocket_multi_session.py
from fastapi import FastAPI, WebSocket
from fastapi.responses import HTMLResponse
import ollama
import asyncio
import json

app = FastAPI()

# アクティブな接続を管理
active_connections = {}

@app.websocket("/ws/{session_id}")
async def websocket_endpoint(websocket: WebSocket, session_id: str):
    await websocket.accept()
    active_connections[session_id] = websocket

    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            # ストリーミングレスポンス
            response_content = ""
            for chunk in ollama.generate(
                model=message.get('model', 'qwen2.5:7b'),
                prompt=message['prompt'],
                stream=True
            ):
                if chunk.get('done'):
                    break

                content = chunk['response']
                response_content += content

                await websocket.send_text(json.dumps({
                    'type': 'chunk',
                    'session_id': session_id,
                    'content': content
                }))

            await websocket.send_text(json.dumps({
                'type': 'complete',
                'session_id': session_id,
                'full_response': response_content
            }))

    except Exception as e:
        print(f"Session {session_id} error: {e}")
    finally:
        del active_connections[session_id]

@app.get("/")
async def get():
    return HTMLResponse("""
    <html>
        <head><title>Multi-Session Chat</title></head>
        <body>
            <h1>WebSocket Multi-Session Demo</h1>
            <p>Active sessions: <span id="count">0</span></p>
            <script>
                let ws = new WebSocket(`ws://localhost:8000/ws/session_${Date.now()}`);
                ws.onmessage = (event) => {
                    console.log('Received:', event.data);
                };
            </script>
        </body>
    </html>
    """)

# 実行: uvicorn websocket_multi_session:app --host 0.0.0.0 --port 8000
```

## 8.4 リソース管理

### 8.4.1 動的メモリ割り当て

```python
# dynamic_memory.py
import ollama
import psutil

class MemoryManager:
    def __init__(self, max_memory_gb=100):
        self.max_memory = max_memory_gb * 1024**3  # バイトに変換
        self.loaded_models = {}

    def get_available_memory(self):
        """利用可能メモリを取得"""
        vm = psutil.virtual_memory()
        return vm.available

    def estimate_model_size(self, model):
        """モデルサイズを推定"""
        size_map = {
            '7b': 5 * 1024**3,
            '14b': 9 * 1024**3,
            '32b': 20 * 1024**3,
            '70b': 42 * 1024**3
        }

        for key, size in size_map.items():
            if key in model.lower():
                return size

        return 5 * 1024**3  # デフォルト

    def can_load_model(self, model):
        """モデルをロード可能か判定"""
        available = self.get_available_memory()
        estimated_size = self.estimate_model_size(model)

        return available > estimated_size * 1.2  # 20%バッファ

    def load_model_safe(self, model, prompt):
        """メモリを確認してから実行"""
        if not self.can_load_model(model):
            # より小さいモデルにフォールバック
            fallback = self.get_fallback_model(model)
            print(f"Memory insufficient. Falling back to {fallback}")
            model = fallback

        return ollama.generate(model=model, prompt=prompt)

    def get_fallback_model(self, model):
        """フォールバックモデルを取得"""
        if '70b' in model:
            return model.replace('70b', '32b')
        elif '32b' in model:
            return model.replace('32b', '14b')
        elif '14b' in model:
            return model.replace('14b', '7b')
        return model

# 使用
mm = MemoryManager(max_memory_gb=100)
response = mm.load_model_safe('qwen2.5:70b', 'Hello')
```

### 8.4.2 自動アンロード

```python
# auto_unload.py
import ollama
import time
from datetime import datetime, timedelta

class ModelCache:
    def __init__(self, ttl_minutes=30):
        self.cache = {}
        self.ttl = timedelta(minutes=ttl_minutes)

    def generate(self, model, prompt, **kwargs):
        # 古いモデルをチェック
        self.cleanup()

        # モデル使用記録
        self.cache[model] = datetime.now()

        return ollama.generate(model=model, prompt=prompt, **kwargs)

    def cleanup(self):
        """TTL超過モデルをアンロード"""
        now = datetime.now()
        expired = []

        for model, last_used in self.cache.items():
            if now - last_used > self.ttl:
                expired.append(model)

        for model in expired:
            print(f"Unloading expired model: {model}")
            # Ollamaは自動的にアンロードするため、キャッシュから削除のみ
            del self.cache[model]

# 使用
cache = ModelCache(ttl_minutes=30)
response = cache.generate('qwen2.5:7b', 'Hello')
```

## 8.5 フェイルオーバーと冗長性

### 8.5.1 自動リトライ

```python
# retry_logic.py
import ollama
import time
from functools import wraps

def retry_on_failure(max_retries=3, delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt < max_retries - 1:
                        print(f"Attempt {attempt + 1} failed: {e}. Retrying...")
                        time.sleep(delay * (attempt + 1))
                    else:
                        print(f"All {max_retries} attempts failed")
                        raise
        return wrapper
    return decorator

@retry_on_failure(max_retries=3, delay=2)
def robust_generate(model, prompt):
    return ollama.generate(model=model, prompt=prompt)

# 使用
try:
    response = robust_generate('qwen2.5:7b', 'Hello')
    print(response['response'])
except Exception as e:
    print(f"Failed after retries: {e}")
```

### 8.5.2 モデルフォールバックチェーン

```python
# fallback_chain.py
import ollama

class FallbackChain:
    def __init__(self, models):
        self.models = models  # 優先順位順

    def generate(self, prompt, **kwargs):
        for model in self.models:
            try:
                print(f"Trying model: {model}")
                response = ollama.generate(model=model, prompt=prompt, **kwargs)
                return {
                    'model_used': model,
                    'response': response
                }
            except Exception as e:
                print(f"Model {model} failed: {e}")
                continue

        raise Exception("All models in chain failed")

# 使用
chain = FallbackChain([
    'qwen2.5:32b',  # 第一選択
    'qwen2.5:14b',  # フォールバック1
    'qwen2.5:7b'    # フォールバック2
])

result = chain.generate('Explain AI')
print(f"Used model: {result['model_used']}")
```

## 8.6 実践例:マルチタスクシステム

### 8.6.1 統合システム

```python
# multi_task_system.py
import ollama
from concurrent.futures import ThreadPoolExecutor
import queue
import threading

class MultiTaskSystem:
    def __init__(self):
        self.executor = ThreadPoolExecutor(max_workers=4)
        self.result_queue = queue.Queue()

        # 用途別モデル
        self.models = {
            'translation': 'qwen2.5:14b',
            'summarization': 'qwen2.5:14b',
            'code_generation': 'codellama:13b',
            'qa': 'qwen2.5:7b'
        }

    def translate(self, text, target_lang='Japanese'):
        future = self.executor.submit(
            self._process,
            'translation',
            f"Translate to {target_lang}: {text}"
        )
        return future

    def summarize(self, text):
        future = self.executor.submit(
            self._process,
            'summarization',
            f"Summarize this text: {text}"
        )
        return future

    def generate_code(self, description):
        future = self.executor.submit(
            self._process,
            'code_generation',
            f"Write code: {description}"
        )
        return future

    def answer_question(self, question):
        future = self.executor.submit(
            self._process,
            'qa',
            question
        )
        return future

    def _process(self, task_type, prompt):
        model = self.models[task_type]
        response = ollama.generate(model=model, prompt=prompt)
        return {
            'task_type': task_type,
            'model': model,
            'response': response['response']
        }

# 使用例
system = MultiTaskSystem()

# 複数タスクを同時実行
future1 = system.translate("Hello World")
future2 = system.summarize("Long article text...")
future3 = system.generate_code("Sort a list in Python")
future4 = system.answer_question("What is AI?")

# 結果取得
print("Translation:", future1.result()['response'])
print("Summary:", future2.result()['response'])
print("Code:", future3.result()['response'])
print("Answer:", future4.result()['response'])
```

## 8.7 本章のまとめ

本章では、以下の内容を学習しました。

✅ **マルチモデル戦略**
- 用途別配置
- 自動ルーティング

✅ **並行実行**
- マルチスレッド
- キューベース処理

✅ **ストリーミング**
- 複数ストリーム同時処理
- WebSocketマルチセッション

✅ **リソース管理**
- 動的メモリ割り当て
- 自動アンロード

✅ **フェイルオーバー**
- 自動リトライ
- モデルフォールバック

次章では、高度なテクニックとトラブルシューティングの総合ガイドを学びます。

---

**前章へ**: [第7章 MS-S1 Max向けパフォーマンス最適化](chapter07_performance_optimization.md)
**次章へ**: [第9章 高度なテクニックとトラブルシューティング](chapter09_advanced_techniques.md)
