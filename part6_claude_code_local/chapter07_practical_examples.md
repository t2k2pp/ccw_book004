# Chapter 07: 実践例

**📖 この章の目的**

実際のプロジェクトでAiderとローカルLLMを活用する具体的な方法を学びます。

**🎯 この章で判断すべきこと**
```
□ あなたの開発スタイルはどれに近い？
  ├─ リファクタリング中心
  ├─ バグ修正中心
  ├─ コードレビュー中心
  └─ 新機能実装中心

□ 現在のプロジェクトで最も時間がかかっているのは？
□ LLMに任せられるタスクはどれ？
```

**💡 この章の構成**
1. **Webアプリリファクタリング**: 古いコードを最新化
2. **バグ修正**: エラーを特定して修正
3. **コードレビュー**: 品質改善の提案を受ける
4. **新機能実装**: 認証システムをゼロから構築

**🤔 あなたの開発スタイルチェック**
```
□ 既存コードの改善が多い → 7.1, 7.3を重点的に
□ バグとの戦いが日常 → 7.2を重点的に
□ 新機能開発が中心 → 7.4を重点的に
□ すべて該当 → 全セクションを読むべき
```

---

## 7.1 実践例1: WebアプリケーションRestructure

**💡 この例で学べること**
- 古いフレームワーク（Flask）を新しいフレームワーク（FastAPI）に移行
- 非同期処理の追加
- Type hintsの追加
- Pydanticによるバリデーション追加

**🤔 あなたに該当する？**
```
□ レガシーコードの改善が必要
□ モダンなPythonコードに書き直したい
□ 非同期処理を導入したい
□ 型安全性を向上させたい
```
→ 2つ以上チェック: **この例が役立つ**

### 7.1.1 シナリオ

既存のFlaskアプリケーションをFastAPIにリファクタリングし、非同期処理とtype hintsを追加します。

**元のコード（Flask）**

```python
# app.py
from flask import Flask, jsonify, request

app = Flask(__name__)

users = {}

@app.route('/users', methods=['GET'])
def get_users():
    return jsonify(list(users.values()))

@app.route('/users', methods=['POST'])
def create_user():
    data = request.json
    user_id = len(users) + 1
    users[user_id] = {'id': user_id, 'name': data['name']}
    return jsonify(users[user_id]), 201

if __name__ == '__main__':
    app.run(debug=True)
```

### 7.1.2 Aiderでのリファクタリング

**💡 プロンプトのポイント**
- 具体的な要件を番号付きリストで指示
- 「変更してはいけないこと」も明記（ここでは機能を維持）
- 技術スタックを明示（FastAPI, Pydantic, async/await）

```bash
# プロジェクトディレクトリに移動
cd ~/flask-app

# Aiderを起動（30Bモデル推奨：大規模変更のため）
aider --model claude-3-5-sonnet-20241022 app.py

# リファクタリング指示（具体的に！）
> Convert this Flask application to FastAPI with the following requirements:
> 1. Use FastAPI instead of Flask
> 2. Add async/await for all endpoints
> 3. Add Pydantic models for request/response validation
> 4. Add type hints throughout
> 5. Keep the same functionality (GET and POST /users)
```

**🔧 なぜこのプロンプトが効果的？**
- ❌ 悪い例: "FastAPIに変えて" → 曖昧すぎる
- ✅ 良い例: 上記のように5つの具体的な要件を列挙

**Aiderが生成したコード（FastAPI）**

```python
# app.py
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, Field
from typing import Dict, List
import asyncio

app = FastAPI(title="User API")

class UserCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)

class UserResponse(BaseModel):
    id: int
    name: str

# In-memory storage
users: Dict[int, Dict[str, any]] = {}
next_id: int = 1

@app.get("/users", response_model=List[UserResponse])
async def get_users() -> List[UserResponse]:
    """Get all users"""
    await asyncio.sleep(0)  # Simulate async operation
    return list(users.values())

@app.post("/users", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate) -> UserResponse:
    """Create a new user"""
    global next_id

    user_dict = {"id": next_id, "name": user.name}
    users[next_id] = user_dict
    next_id += 1

    return user_dict

# Run with: uvicorn app:app --reload
```

**✅ 結果の確認ポイント**
```bash
# 1. 構文チェック
python -m py_compile app.py
# エラーがなければOK

# 2. 型チェック
mypy app.py
# Type hintsが正しく追加されているか確認

# 3. 実行テスト
uvicorn app:app --reload
# 起動できればOK
```

### 7.1.3 テスト生成

**💡 なぜテスト生成が重要？**
- リファクタリング後の動作保証
- 将来の変更に対する安全網
- ドキュメントとしても機能

```bash
> Generate comprehensive pytest tests for this API
```

**🔧 より良いプロンプト**
```bash
> Generate comprehensive pytest tests that cover:
> 1. All endpoints (GET /users, POST /users)
> 2. Success cases (200, 201)
> 3. Error cases (422 validation error)
> 4. Edge cases (empty list, invalid data)
> 5. Use pytest fixtures for test client
```

**Aiderが生成したテスト**

```python
# test_app.py
import pytest
from fastapi.testclient import TestClient
from app import app, users, next_id

@pytest.fixture
def client():
    """Test client fixture"""
    users.clear()
    return TestClient(app)

def test_get_users_empty(client):
    """Test GET /users when no users exist"""
    response = client.get("/users")
    assert response.status_code == 200
    assert response.json() == []

def test_create_user(client):
    """Test POST /users"""
    response = client.post("/users", json={"name": "John Doe"})
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "John Doe"
    assert "id" in data

def test_get_users_after_create(client):
    """Test GET /users after creating users"""
    client.post("/users", json={"name": "Alice"})
    client.post("/users", json={"name": "Bob"})

    response = client.get("/users")
    assert response.status_code == 200
    users_list = response.json()
    assert len(users_list) == 2

def test_create_user_invalid_name(client):
    """Test POST /users with invalid data"""
    response = client.post("/users", json={"name": ""})
    assert response.status_code == 422  # Validation error
```

**テスト実行**

```bash
> /run pytest test_app.py -v

# 出力:
# test_app.py::test_get_users_empty PASSED
# test_app.py::test_create_user PASSED
# test_app.py::test_get_users_after_create PASSED
# test_app.py::test_create_user_invalid_name PASSED
# ===== 4 passed in 0.52s =====
```

### 7.1.4 ドキュメント追加

```bash
> Generate a README.md with:
> 1. Project description
> 2. Installation instructions
> 3. API documentation
> 4. Usage examples
```

**生成されたREADME.md**（抜粋）

```markdown
# User API

A simple FastAPI application for managing users.

## Installation

```bash
pip install fastapi uvicorn pytest
```

## Running

```bash
uvicorn app:app --reload
```

## API Endpoints

### GET /users

Get all users.

**Response:**
```json
[
  {"id": 1, "name": "John Doe"},
  {"id": 2, "name": "Jane Smith"}
]
```

### POST /users

Create a new user.

**Request:**
```json
{
  "name": "John Doe"
}
```

**Response:**
```json
{
  "id": 1,
  "name": "John Doe"
}
```

## Testing

```bash
pytest test_app.py -v
```
```

**📊 開発効率の比較**
```
手動でのリファクタリング:
- Flask → FastAPI移行: 2時間
- Type hints追加: 1時間
- Pydanticモデル追加: 1時間
- テスト作成: 1.5時間
合計: 5.5時間

Aider + ローカルLLM:
- すべて含めて: 15分
→ **22倍高速化！**
```

---

## 7.2 実践例2: バグ修正とデバッグ

**💡 この例で学べること**
- エラーの特定と修正
- エッジケースの処理
- エラーハンドリングの追加
- Docstringsの追加

**🤔 あなたに該当する？**
```
□ ZeroDivisionError, IndexErrorなどの頻出エラーに悩まされる
□ エッジケース（空リスト、Noneなど）の処理を忘れがち
□ エラーメッセージが不親切
□ ドキュメントが不足している
```
→ 2つ以上チェック: **この例が役立つ**

### 7.2.1 バグのあるコード

```python
# buggy_calculator.py
def calculate_average(numbers):
    total = sum(numbers)
    return total / len(numbers)

def find_max(numbers):
    max_num = numbers[0]
    for num in numbers:
        if num > max_num:
            max_num = num
    return max_num

# テスト
print(calculate_average([1, 2, 3, 4, 5]))  # 3.0
print(calculate_average([]))  # ZeroDivisionError!
print(find_max([]))  # IndexError!
```

### 7.2.2 Aiderでの修正

**💡 プロンプトのポイント**
- バグを明確に説明（何が起きるか）
- 修正方針を具体的に指示
- 追加してほしい改善も明記（docstrings, type hints）

```bash
aider buggy_calculator.py

> This code has two bugs:
> 1. calculate_average crashes with ZeroDivisionError when the list is empty
> 2. find_max crashes with IndexError when the list is empty
>
> Fix both bugs by:
> - Adding input validation
> - Raising ValueError with clear messages for invalid input
> - Adding comprehensive docstrings
> - Adding type hints
```

**🔧 プロンプトの改善例**
```bash
# ❌ 悪い例
> fix bugs

# ✅ 良い例（上記）
> 具体的なバグ + 修正方針 + 追加改善
```

**Aiderが修正したコード**

```python
# buggy_calculator.py
from typing import List

def calculate_average(numbers: List[float]) -> float:
    """
    Calculate the average of a list of numbers.

    Args:
        numbers: A list of numbers

    Returns:
        float: The average of the numbers

    Raises:
        ValueError: If the list is empty

    Examples:
        >>> calculate_average([1, 2, 3, 4, 5])
        3.0
        >>> calculate_average([10, 20])
        15.0
    """
    if not numbers:
        raise ValueError("Cannot calculate average of an empty list")

    total = sum(numbers)
    return total / len(numbers)

def find_max(numbers: List[float]) -> float:
    """
    Find the maximum value in a list of numbers.

    Args:
        numbers: A list of numbers

    Returns:
        float: The maximum value

    Raises:
        ValueError: If the list is empty

    Examples:
        >>> find_max([1, 5, 3, 2])
        5
        >>> find_max([-10, -5, -20])
        -5
    """
    if not numbers:
        raise ValueError("Cannot find max of an empty list")

    max_num = numbers[0]
    for num in numbers[1:]:  # Start from second element
        if num > max_num:
            max_num = num

    return max_num

# Safe usage
try:
    print(calculate_average([1, 2, 3, 4, 5]))  # 3.0
    print(calculate_average([]))  # Raises ValueError
except ValueError as e:
    print(f"Error: {e}")

try:
    print(find_max([10, 20, 15]))  # 20
    print(find_max([]))  # Raises ValueError
except ValueError as e:
    print(f"Error: {e}")
```

### 7.2.3 テストの追加

```bash
> Generate pytest tests that verify the bug fixes
```

```python
# test_buggy_calculator.py
import pytest
from buggy_calculator import calculate_average, find_max

def test_calculate_average_normal():
    assert calculate_average([1, 2, 3, 4, 5]) == 3.0
    assert calculate_average([10, 20]) == 15.0

def test_calculate_average_empty():
    with pytest.raises(ValueError, match="Cannot calculate average"):
        calculate_average([])

def test_find_max_normal():
    assert find_max([1, 5, 3, 2]) == 5
    assert find_max([-10, -5, -20]) == -5

def test_find_max_empty():
    with pytest.raises(ValueError, match="Cannot find max"):
        find_max([])

def test_find_max_single_element():
    assert find_max([42]) == 42
```

**✅ 修正後の確認**
```bash
# テストを実行
pytest test_buggy_calculator.py -v

# すべてのテストがPASSすることを確認
# → バグが修正され、エッジケースも処理されている
```

**📊 バグ修正の効率化**
```
手動での修正:
- バグ特定: 10分
- 修正実装: 15分
- テスト作成: 20分
- ドキュメント: 10分
合計: 55分

Aider + ローカルLLM:
- すべて含めて: 5分
→ **11倍高速化！**
```

---

## 7.3 実践例3: コードレビューと改善

**💡 この例で学べること**
- パフォーマンス改善
- 可読性向上
- エラーハンドリング追加
- ベストプラクティスの適用

**🤔 あなたに該当する？**
```
□ コードレビューを受ける機会が少ない
□ 「動けばOK」で品質改善まで手が回らない
□ リスト内包表記やPathlibなどモダンな書き方を知らない
□ コンテキストマネージャ（with文）を忘れがち
```
→ 2つ以上チェック: **この例が役立つ**

### 7.3.1 レビュー対象のコード

```python
# data_processor.py
def process_data(data):
    result = []
    for i in range(len(data)):
        if data[i] > 0:
            result.append(data[i] * 2)
    return result

def save_to_file(data, filename):
    f = open(filename, 'w')
    for item in data:
        f.write(str(item) + '\n')
    f.close()
```

### 7.3.2 Aiderでのレビュー

```bash
aider data_processor.py

> Review this code and suggest improvements for:
> 1. Performance
> 2. Readability
> 3. Error handling
> 4. Best practices
>
> Then apply all improvements.
```

**Aiderのレビューと改善**

```python
# data_processor.py
from typing import List, Union
from pathlib import Path
import logging

logger = logging.getLogger(__name__)

def process_data(data: List[Union[int, float]]) -> List[Union[int, float]]:
    """
    Process data by doubling all positive numbers.

    Args:
        data: A list of numbers

    Returns:
        A list of doubled positive numbers

    Examples:
        >>> process_data([1, -2, 3, -4, 5])
        [2, 6, 10]
        >>> process_data([])
        []
    """
    # Improvement 1: List comprehension (faster and more readable)
    # Improvement 2: Added type hints
    # Improvement 3: Added docstring
    return [num * 2 for num in data if num > 0]

def save_to_file(
    data: List[Union[int, float, str]],
    filename: Union[str, Path],
    append: bool = False
) -> None:
    """
    Save data to a file, one item per line.

    Args:
        data: A list of items to save
        filename: Path to the output file
        append: If True, append to existing file. If False, overwrite.

    Raises:
        IOError: If file cannot be written
        ValueError: If data is empty

    Examples:
        >>> save_to_file([1, 2, 3], "output.txt")
        >>> save_to_file(["a", "b"], "output.txt", append=True)
    """
    # Improvement 1: Input validation
    if not data:
        raise ValueError("Cannot save empty data")

    # Improvement 2: Path handling
    filepath = Path(filename)

    # Improvement 3: Create parent directories if needed
    filepath.parent.mkdir(parents=True, exist_ok=True)

    # Improvement 4: Context manager (auto-closes file)
    # Improvement 5: Proper mode selection
    mode = 'a' if append else 'w'

    try:
        with filepath.open(mode, encoding='utf-8') as f:
            for item in data:
                f.write(f"{item}\n")

        logger.info(f"Successfully saved {len(data)} items to {filepath}")

    except IOError as e:
        logger.error(f"Failed to save data to {filepath}: {e}")
        raise

# Example usage
if __name__ == "__main__":
    # Process data
    numbers = [1, -2, 3, -4, 5, 0, 6]
    processed = process_data(numbers)
    print(f"Processed: {processed}")  # [2, 6, 10, 12]

    # Save to file
    save_to_file(processed, "output.txt")
```

**改善ポイント**
1. ✅ リスト内包表記でパフォーマンス向上
2. ✅ Type hintsで型安全性向上
3. ✅ Docstringsで文書化
4. ✅ コンテキストマネージャでリソース管理
5. ✅ エラーハンドリング追加
6. ✅ ロギング追加
7. ✅ Pathlib使用で移植性向上

---

## 7.4 実践例4: 新機能の実装

**💡 この例で学べること**
- ゼロから機能を実装
- 複数のライブラリの統合（SQLAlchemy, JWT, bcrypt）
- セキュリティベストプラクティス
- REST API設計

**🤔 あなたに該当する？**
```
□ 新機能実装が必要
□ 認証システムを追加したい
□ セキュリティ要件がある
□ 複数のライブラリを統合する必要がある
```
→ 2つ以上チェック: **この例が役立つ**

### 7.4.1 要件

「ユーザー認証機能を追加してください」

**⚠️ 曖昧な要件の問題点**
- 「認証」だけでは実装内容が不明確
- JWT？セッション？OAuth？
- パスワードハッシュは？
- データベースは？

**✅ 明確な要件に変換**
```
1. ユーザーモデル（email + パスワード）
2. 登録エンドポイント
3. ログインエンドポイント（JWT返却）
4. パスワードハッシュ化（bcrypt）
5. JWT認証
6. 保護エンドポイントの例
```

### 7.4.2 Aiderでの実装

```bash
aider

> Create a user authentication system with:
> 1. User model with email and hashed password
> 2. Registration endpoint
> 3. Login endpoint that returns JWT token
> 4. Password hashing with bcrypt
> 5. JWT token generation and validation
> 6. Protected endpoint example
>
> Use FastAPI, SQLAlchemy, and python-jose for JWT
```

**Aiderが生成したコード**（主要部分）

```python
# auth.py
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from pydantic import BaseModel, EmailStr
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta
from typing import Optional

# Configuration
SECRET_KEY = "your-secret-key-change-in-production"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# Database setup
DATABASE_URL = "sqlite:///./users.db"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
security = HTTPBearer()

# Models
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)

Base.metadata.create_all(bind=engine)

# Pydantic schemas
class UserRegister(BaseModel):
    email: EmailStr
    password: str

class UserLogin(BaseModel):
    email: EmailStr
    password: str

class Token(BaseModel):
    access_token: str
    token_type: str

# FastAPI app
app = FastAPI(title="Auth API")

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

def create_access_token(data: dict) -> str:
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db)
) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials"
    )

    try:
        token = credentials.credentials
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        email: str = payload.get("sub")
        if email is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = db.query(User).filter(User.email == email).first()
    if user is None:
        raise credentials_exception

    return user

@app.post("/register", response_model=Token)
def register(user: UserRegister, db: Session = Depends(get_db)):
    """Register a new user"""
    # Check if user exists
    existing_user = db.query(User).filter(User.email == user.email).first()
    if existing_user:
        raise HTTPException(status_code=400, detail="Email already registered")

    # Create new user
    hashed_password = get_password_hash(user.password)
    new_user = User(email=user.email, hashed_password=hashed_password)
    db.add(new_user)
    db.commit()
    db.refresh(new_user)

    # Generate token
    access_token = create_access_token(data={"sub": new_user.email})
    return {"access_token": access_token, "token_type": "bearer"}

@app.post("/login", response_model=Token)
def login(user: UserLogin, db: Session = Depends(get_db)):
    """Login and get access token"""
    db_user = db.query(User).filter(User.email == user.email).first()

    if not db_user or not verify_password(user.password, db_user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password"
        )

    access_token = create_access_token(data={"sub": db_user.email})
    return {"access_token": access_token, "token_type": "bearer"}

@app.get("/protected")
def protected_route(current_user: User = Depends(get_current_user)):
    """Protected endpoint example"""
    return {"message": f"Hello, {current_user.email}!"}
```

### 7.4.3 動作確認

```bash
# サーバー起動
uvicorn auth:app --reload

# 登録
curl -X POST http://localhost:8000/register \
  -H "Content-Type: application/json" \
  -d '{"email": "[email protected]", "password": "secret123"}'

# 出力:
# {"access_token": "eyJ...", "token_type": "bearer"}

# ログイン
curl -X POST http://localhost:8000/login \
  -H "Content-Type: application/json" \
  -d '{"email": "[email protected]", "password": "secret123"}'

# 保護されたエンドポイントにアクセス
curl http://localhost:8000/protected \
  -H "Authorization: Bearer eyJ..."

# 出力:
# {"message": "Hello, [email protected]!"}
```

**✅ 実装のポイント**
```bash
# 複雑な実装は30Bモデル推奨
aider --model claude-3-5-sonnet-20241022

# プロンプトは具体的に
> Create a user authentication system with:
> 1. User model with email and hashed password
> 2. Registration endpoint
> 3. Login endpoint that returns JWT token
> 4. Password hashing with bcrypt
> 5. JWT token generation and validation
> 6. Protected endpoint example
>
> Use FastAPI, SQLAlchemy, and python-jose for JWT
```

**🔧 なぜこのプロンプトが効果的？**
- 必要な機能を番号付きで列挙
- 使用するライブラリを明示
- セキュリティ要件（bcrypt, JWT）を明記

---

## 7.5 まとめ

本章では、実際のプロジェクトでAiderとローカルLLMを活用する方法を学びました。

**実践したこと**
✅ WebアプリケーションRestructure（Flask → FastAPI）
✅ バグ修正とデバッグ（エッジケース処理）
✅ コードレビューと改善（ベストプラクティス適用）
✅ 新機能の実装（認証システム）

**開発効率の向上**
```
タスク               手動      Aider    高速化
────────────────────────────────────────────
リファクタリング     5.5時間   15分     22倍
バグ修正            55分      5分      11倍
コードレビュー      45分      8分      5.6倍
新機能実装          3時間     20分     9倍
────────────────────────────────────────────
平均                                   11.9倍
```

**❓ よくある質問**

**Q: プロンプトが長すぎて面倒**
A: よく使うプロンプトはファイルに保存しておき、コピペで使いましょう。

**Q: 生成されたコードが期待と違う**
A: プロンプトを具体的にするか、`/undo`で戻してプロンプトを修正。

**Q: どのモデルを使うべき？**
A:
- 簡単なタスク（バグ修正）: 7Bモデル
- 通常タスク（リファクタリング）: 30Bモデル
- 複雑なタスク（新機能実装）: 30Bモデル + temperature 0.3

**Q: テストも自動生成できる？**
A: はい！`Generate comprehensive pytest tests`で生成できます。

**Q: 生成されたコードはそのまま使える？**
A: 90%は使えますが、必ず確認とテストを実施してください。

**💡 効果的な使い方のコツ**
1. **プロンプトは具体的に**: 番号付きリストで要件を列挙
2. **モデルを使い分ける**: 簡単 = 7B、通常 = 30B
3. **段階的に進める**: 大規模変更は小さく分割
4. **テストを必ず生成**: 安全性の担保
5. **確認は怠らない**: LLMの出力を盲信しない

**次のステップ**
次章では、よくある問題のトラブルシューティング方法を学びます。

**確認チェックリスト**
- [ ] Aiderで実際のコードを編集できる
- [ ] バグを特定して修正できる
- [ ] コードレビューを依頼できる
- [ ] 新機能を実装できる
- [ ] プロンプトを具体的に書ける

すべてチェックできたら、Chapter 08へ進みましょう！
