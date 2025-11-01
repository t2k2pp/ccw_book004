# Chapter 05: 機能の互換性と制限

## 5.1 ローカルLLMとClaude APIの違い

### 5.1.1 アーキテクチャの違い

**Claude API（本家）**
```
Claude Code → Anthropic API → Claude 3.5 Sonnet
               ↓
          - 専用プロトコル
          - 関数呼び出し（Function Calling）
          - ビジョン（画像理解）
          - 長いコンテキスト（200K tokens）
          - ストリーミング
```

**ローカルLLM（本書の構成・2025年11月最新）**
```
Aider → LiteLLM Proxy → Ollama → Qwen3 Coder 30B Q8_0
         ↓
    - OpenAI互換プロトコル
    - 限定的な関数呼び出し
    - テキストのみ
    - コンテキスト（256K tokens、最大1M拡張可能）
    - ストリーミング
    - MS-S1 Max: 96GB VRAM対応
```

### 5.1.2 機能比較表

| 機能 | Claude API | ローカルLLM (Qwen3 Coder 30B) | 対応方法 |
|------|-----------|------------------------------|---------|
| チャット補完 | ✅ | ✅ | 完全互換 |
| ストリーミング | ✅ | ✅ | 完全互換 |
| 関数呼び出し | ✅ | ⚠️ 限定的 | プロンプトで代替 |
| ビジョン（画像） | ✅ | ❌ | LLaVA使用（別途） |
| 長いコンテキスト | ✅ 200K | ✅ 256K (1M拡張可) | ローカルが有利 |
| 多言語対応 | ✅ | ✅ | 問題なし |
| コード生成 | ✅ | ✅ | Qwen3で高品質 |
| 日本語 | ✅ | ✅ | 高品質 |
| レスポンス速度 | ⚠️ ネットワーク | ✅ 22 tokens/s | ローカルが高速 |
| プライバシー | ❌ 外部送信 | ✅ 完全ローカル | ローカルが有利 |
| コスト | ❌ 有料 | ✅ 無料 | ローカルが有利 |
| VRAM要件 | ❌ N/A | ✅ 32GB (MS-S1: 96GB可) | MS-S1で余裕 |

## 5.2 動作する機能

### 5.2.1 完全に動作する機能

**1. テキストベースのコード生成**

```
> Create a Python function to calculate prime numbers

# ✅ 完全に動作
```

**動作例**

```python
def is_prime(n):
    """Check if a number is prime."""
    if n < 2:
        return False
    for i in range(2, int(n ** 0.5) + 1):
        if n % i == 0:
            return False
    return True

def get_primes(limit):
    """Generate prime numbers up to limit."""
    return [n for n in range(2, limit + 1) if is_prime(n)]

# Example usage
print(get_primes(100))
```

**2. ファイル編集**

```
> /add main.py
> Add error handling to this function

# ✅ 完全に動作
# Aiderがファイルを読み込み、編集を適用
```

**3. コードレビュー**

```
> /add complex_function.py
> Review this code and suggest improvements

# ✅ 完全に動作
```

**出力例**
```
Here are my suggestions:

1. **Type Hints**: Add type annotations for better code clarity
2. **Error Handling**: Add try-except blocks for file operations
3. **Docstrings**: Add comprehensive documentation
4. **Performance**: Use list comprehension instead of loops
5. **Naming**: Rename variable 'x' to 'result' for clarity

Would you like me to apply these changes?
```

**4. リファクタリング**

```
> Refactor this code to use async/await

# ✅ 動作（モデルの能力に依存）
```

**5. テスト生成**

```
> Generate pytest tests for this module

# ✅ 完全に動作
```

**6. ドキュメント生成**

```
> Create a README.md with usage examples

# ✅ 完全に動作
```

**7. バグ修正**

```
> Fix the IndexError in line 45

# ✅ 動作（精度はモデルに依存）
```

### 5.2.2 部分的に動作する機能

**1. 関数呼び出し（Function Calling）**

Claude APIのFunction Callingは限定的にしか動作しません。

**回避策: プロンプトエンジニアリング**

```
# ❌ Claude APIスタイル（動作しない）
tools = [
    {
        "name": "search_file",
        "description": "Search for a file",
        "parameters": {...}
    }
]

# ✅ プロンプトで代替（動作する）
> I need to search for files containing "TODO".
> Please suggest a bash command to do this.

# モデルが提案:
# grep -r "TODO" ./src/
```

**2. 複数ステップのタスク**

Claude Code本家は複雑なタスクを自動で分解できますが、ローカルLLMは明示的な指示が必要です。

```
# ⚠️ 曖昧な指示（動作は不安定）
> Create a web app with user authentication

# ✅ ステップバイステップ（安定動作）
> Step 1: Create a FastAPI server with a /login endpoint
# ... 完了後 ...
> Step 2: Add JWT token generation
# ... 完了後 ...
> Step 3: Create middleware for authentication
```

**3. 大規模なコードベースの理解**

Qwen3 Coder 30B Q8_0は256Kトークン（ネイティブ）のコンテキストを持ち、最大1Mまで拡張可能です。MS-S1 Maxの96GB VRAMにより、大規模なプロジェクトも処理できます。

**推奨: ファイルを選択的に追加（効率的な処理のため）**

```bash
# ⚠️ プロジェクト全体を追加（可能だが非効率）
> /add src/**/*.py  # 256K以内なら可能

# ✅ 関連ファイルのみ追加（推奨・効率的）
> /add src/auth.py src/models.py src/utils.py

# ✅ MS-S1 Maxの大容量VRAMを活かした使用
> /add src/module1/ src/module2/  # 合計100-200Kトークンまで快適
```

## 5.3 動作しない機能

### 5.3.1 完全に動作しない機能

**1. 画像理解（ビジョン）**

```
# ❌ 動作しない
> Analyze this screenshot and find the bug
> (screenshot.png)

# Qwen3 Coder 30B Q8_0はテキストのみ対応
```

**回避策: LLaVAモデルを使用**

```bash
# 別途LLaVAモデルをダウンロード
ollama pull llava:13b

# Pythonスクリプトで画像を分析
python3 << EOF
import ollama

response = ollama.chat(
    model='llava:13b',
    messages=[{
        'role': 'user',
        'content': 'Analyze this code screenshot',
        'images': ['screenshot.png']
    }]
)
print(response['message']['content'])
EOF
```

**2. Claude特有のプロンプト形式**

```
# ❌ Claude APIの特殊タグ（動作しない）
<thinking>...</thinking>
<answer>...</answer>

# ✅ 標準的なプロンプト
> Think step by step and provide your answer.
```

**3. Web検索統合**

```
# ❌ 動作しない
> Search the web for latest Python best practices

# Ollamaはオフライン動作のみ
```

**回避策: 手動で情報を提供**

```
> Here's the latest info from Python.org: [貼り付け]
> Based on this, suggest improvements to my code
```

### 5.3.2 パフォーマンスが劣る機能

**1. 非常に複雑な推論**

Claude 3.5 Sonnetは高度な推論が得意ですが、ローカルモデルは限界があります。

**例: 数学的証明**

```
# ⚠️ ローカルLLMでは精度が低い
> Prove that the sum of two even numbers is always even

# より簡単なタスクに分解すると改善
> Explain what an even number is
> Show examples of adding two even numbers
> Generalize the pattern
```

**2. 長文の理解と生成**

32Kトークンの制限により、非常に長い文章の処理が制限されます。

```
# ⚠️ 制限あり
> Summarize this 50-page document

# ✅ チャンク処理
> Summarize pages 1-10
> Summarize pages 11-20
> ...
> Combine all summaries
```

## 5.4 制限への対処法

### 5.4.1 コンテキスト制限の回避

**方法1: 関連ファイルのみ追加**

```bash
# Git履歴から関連ファイルを特定
git log --follow --oneline -- src/bug_file.py | head -10

# 関連ファイルのみ追加
aider src/bug_file.py src/related_module.py
```

**方法2: コンテキストのクリア**

```
> /clear

# コンテキストをクリアして新しいタスクを開始
```

**方法3: 要約を活用**

```
> /add large_file.py
> Summarize the main functions in this file

# 要約をメモして、詳細ファイルをdrop
> /drop large_file.py

# 要約を基に作業
```

### 5.4.2 精度の改善

**方法1: Few-Shot プロンプティング**

```
> Here are two examples of the coding style I want:
>
> Example 1:
> [コード例1を貼り付け]
>
> Example 2:
> [コード例2を貼り付け]
>
> Now, refactor this function to match this style:
> [リファクタリング対象のコードを貼り付け]
```

**方法2: ステップバイステップ指示**

```
# ❌ 曖昧
> Optimize this database query

# ✅ 具体的
> Step 1: Add indexes to frequently queried columns
> Step 2: Use query explain to identify bottlenecks
> Step 3: Rewrite subqueries as joins
```

**方法3: コードレビューの活用**

```
> /add generated_code.py

> Review this code I just generated:
> 1. Check for edge cases
> 2. Verify error handling
> 3. Suggest performance improvements
```

### 5.4.3 複数モデルの使い分け

LiteLLMの設定で複数モデルを定義し、用途に応じて切り替えます。

```yaml
# config.yaml（2025年11月最新・Qwen3 Coder構成）
model_list:
  # 高速タスク（14B・256K context）
  - model_name: gpt-3.5-turbo
    litellm_params:
      model: ollama/qwen3-coder:14b
      num_ctx: 262144

  # 通常・高品質タスク（30B Q8_0・256K context）
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      num_ctx: 262144

  # 最高品質タスク（30B Q8_0・256K context）
  - model_name: gpt-4
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      num_ctx: 262144

  # 最速タスク（7B・256K context）
  - model_name: claude-3-haiku-20240307
    litellm_params:
      model: ollama/qwen3-coder:7b
      num_ctx: 262144
```

**使用例**

```
# 高速モデルで下書き
> /model gpt-3.5-turbo
> Draft a function to parse JSON

# 高品質モデルでレビュー
> /model gpt-4
> Review and improve this function
```

## 5.5 ベストプラクティス

### 5.5.1 効果的なプロンプト

**❌ 悪い例**

```
> Fix this
```

**✅ 良い例**

```
> This function throws a KeyError when the 'name' field is missing from the input dictionary.
> Add error handling to return None instead of crashing.
> Also add type hints and a docstring.
```

### 5.5.2 段階的なアプローチ

**❌ 一度に全部**

```
> Create a complete authentication system with OAuth, JWT, password reset, email verification, and RBAC
```

**✅ ステップバイステップ**

```
> Step 1: Create a User model with SQLAlchemy
# 完了を確認
> Step 2: Add password hashing with bcrypt
# 完了を確認
> Step 3: Implement JWT token generation
# ...
```

### 5.5.3 コンテキスト管理

**定期的なクリーンアップ**

```
# 10-15回の会話ごとにクリア
> /clear

# または新しいセッションを開始
> /exit
aider
```

**関連ファイルのグループ化**

```bash
# 認証関連
aider src/auth.py src/models/user.py src/utils/jwt.py

# API関連
aider src/api.py src/routes/ src/schemas/
```

## 5.6 トラブルシューティング

### 5.6.1 応答品質が低い

**問題**: 生成されるコードの品質が期待より低い

**解決策**

1. **モデルを変更**
```
> /model gpt-4  # より大きなモデルに切り替え
```

2. **プロンプトを改善**
```
> You are an expert Python developer.
> Write production-ready code with:
> - Type hints
> - Error handling
> - Comprehensive docstrings
> - Unit tests
>
> Now, create a function to...
```

3. **Few-Shot学習**
```
> Here's an example of the code quality I expect:
> [良いコード例を貼り付け]
>
> Now, generate similar code for...
```

### 5.6.2 コンテキスト超過

**エラー**: "Context length exceeded"

**解決策**

```
# コンテキストを確認
> /tokens

# 不要なファイルを削除
> /drop unnecessary_file.py

# またはコンテキストをクリア
> /clear

# 必要最小限のファイルのみ追加
> /add essential_file.py
```

### 5.6.3 生成コードが動作しない

**問題**: 生成されたコードにバグがある

**解決策**

1. **テスト実行**
```
> /run python3 generated_code.py

# エラーメッセージをフィードバック
> The code fails with this error: [エラーメッセージを貼り付け]
> Please fix it.
```

2. **段階的テスト**
```
> Generate the code without running it first
# コードを確認
> Now let's test the function step by step
```

3. **コードレビュー**
```
> Review the generated code for potential bugs
```

## 5.7 まとめ

本章では、ローカルLLMの制限と回避方法を学びました。

**重要ポイント**

✅ **動作する機能**
- テキストベースのコード生成・編集
- リファクタリング
- テスト生成
- コードレビュー

⚠️ **制限がある機能**
- 関数呼び出し → プロンプトで代替
- 画像理解 → LLaVAで対応
- 長いコンテキスト → チャンク分割

❌ **動作しない機能**
- Claude特有のタグ
- Web検索

**対処法**
1. 明確で具体的なプロンプト
2. ステップバイステップのアプローチ
3. 適切なコンテキスト管理
4. 複数モデルの使い分け

**次のステップ**
次章では、コーディング特化モデルの比較と、MS-S1 Maxでの最適な選択について学びます。

**確認チェックリスト**
- [ ] ローカルLLMの制限を理解している
- [ ] 回避策を実践できる
- [ ] 効果的なプロンプトを書ける
- [ ] コンテキスト管理ができる

すべてチェックできたら、Chapter 06へ進みましょう！
