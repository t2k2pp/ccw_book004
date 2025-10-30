# 第8章：実践的な使い方

## 8.1 チャットインターフェースの活用

### 8.1.1 基本的な対話

LM Studioのチャットインターフェースは、ChatGPTと同様の使い勝手を提供します。

#### 効果的なプロンプトの書き方

**原則1: 明確かつ具体的に**

```
❌ 悪い例:
"プログラミングについて教えて"

✅ 良い例:
"Pythonで辞書型データをJSON形式のファイルに保存する方法を、
サンプルコード付きで教えてください。"
```

**原則2: コンテキストを提供**

```
❌ 悪い例:
"これをどう改善できますか？"

✅ 良い例:
"以下のPython関数の実行速度を改善したいです。
大量のデータ（10万行以上）を処理する際に遅くなります。
最適化の提案をお願いします。

```python
def process_data(data_list):
    result = []
    for item in data_list:
        if item > 100:
            result.append(item * 2)
    return result
```
"
```

**原則3: 段階的に質問**

```
会話の流れ:

1. "Pythonでウェブスクレイピングを学びたいです。
   初心者向けのステップを教えてください。"

2. "Beautiful Soupについて詳しく教えてください。"

3. "Beautiful Soupで特定のCSSクラスを持つ要素を
   抽出するサンプルコードを示してください。"
```

### 8.1.2 システムプロンプトの活用

**システムプロンプト**は、AIの振る舞いや役割を定義します。

#### アクセス方法

```
Chat画面 → 上部の「⚙️」アイコン
→ System Prompt セクション
```

#### 効果的なシステムプロンプト例

**例1: プログラミングアシスタント**

```
あなたは経験豊富なソフトウェアエンジニアです。
以下のルールに従って応答してください：

1. コードは必ず動作する完全な例を提供する
2. コメントで各ステップを説明する
3. エッジケースやエラー処理についても言及する
4. 複数の解決方法がある場合は、それぞれの長所短所を説明する
5. コードの効率性とベストプラクティスを重視する

使用言語: 日本語
コードのコメント: 日本語
```

**例2: 執筆アシスタント**

```
あなたはプロフェッショナルなライターです。
以下の特性を持って応答してください：

1. 明確で読みやすい文章を書く
2. 適切な段落分けと見出しを使う
3. 専門用語は必要に応じて説明する
4. 具体例を交えて説明する
5. 敬体（です・ます調）を使用する

対象読者: 一般的なビジネスパーソン
文章の長さ: 質問に応じて調整（デフォルト: 中程度）
```

**例3: 日本語学習アシスタント**

```
You are a Japanese language tutor helping English speakers learn Japanese.

Rules:
1. Explain grammar points in English
2. Provide example sentences in Japanese with romaji and English translations
3. Point out common mistakes learners make
4. Give cultural context when relevant
5. Be encouraging and patient

Format:
Japanese sentence
Romaji
English translation
Grammar explanation
```

### 8.1.3 マルチターン対話の管理

**会話履歴の活用**

LM Studioは会話履歴を自動的に保持します。

```
効果的な使い方:

1. 初回質問で十分な背景情報を提供
2. 以降の質問では「それ」「前述の」などの参照を使用
3. トピックを変える際は明示する

例:
ユーザー: "Pythonのリスト内包表記について教えてください。"
AI: [詳しい説明]
ユーザー: "それを使って、1から100までの偶数のリストを作るには？"
AI: [リスト内包表記を使った解答]
ユーザー: "では、ディクショナリ内包表記についても教えてください。"
```

**会話のリセットタイミング**

```
リセット推奨ケース:
  ✓ トピックが完全に変わる
  ✓ 長時間使用後（メモリ解放）
  ✓ モデルの応答品質が下がった
  ✓ コンテキスト長の上限に近づいた

方法:
  Chat画面右上 → 「New Chat」ボタン
```

## 8.2 実践的なユースケース

### 8.2.1 文章作成・編集

#### ブログ記事の作成

**ステップ1: アウトライン生成**

```
プロンプト:
"「自宅でできる効果的な運動習慣」というテーマで
ブログ記事を書きたいです。
記事のアウトライン（見出し構成）を提案してください。
対象読者は運動初心者、記事の長さは2000文字程度を想定しています。"
```

**ステップ2: 各セクションの執筆**

```
プロンプト:
"先ほどのアウトラインの「2. 自宅運動のメリット」セクションを
300-400文字で執筆してください。具体例を2-3個含めてください。"
```

**ステップ3: 推敲**

```
プロンプト:
"以下の文章を、よりキャッチーで読みやすく改善してください：

[元の文章を貼り付け]
"
```

#### メール作成

**ビジネスメール**

```
プロンプト:
"以下の内容でビジネスメールを作成してください。

宛先: 取引先の田中様
目的: 来週の打ち合わせ日程の変更依頼
理由: 急な出張が入ったため
提案日時: 2つの候補日時を提示
トーン: 丁寧かつプロフェッショナル"
```

### 8.2.2 コード生成とレビュー

#### コード生成

**Pythonスクリプト生成**

```
プロンプト:
"以下の要件を満たすPythonスクリプトを作成してください：

要件:
1. CSVファイルを読み込む
2. 特定の列（'age'）の値が30以上の行を抽出
3. 結果を新しいCSVファイルに保存
4. エラーハンドリングを含める
5. コメントで各ステップを説明

入力ファイル: data.csv
出力ファイル: filtered_data.csv"
```

#### コードレビュー

```
プロンプト:
"以下のコードをレビューして、改善点を指摘してください。
特に、パフォーマンス、可読性、エラー処理の観点からお願いします。

```python
[コードを貼り付け]
```
"
```

#### デバッグ支援

```
プロンプト:
"以下のPythonコードでエラーが発生します。
原因と修正方法を教えてください。

エラーメッセージ:
KeyError: 'name'

コード:
```python
[エラーが発生するコードを貼り付け]
```
"
```

### 8.2.3 データ分析と要約

#### 長文の要約

```
プロンプト:
"以下の文章を300文字程度で要約してください。
主要なポイントを箇条書きで整理してください。

[長文を貼り付け]
"
```

**💡 TIP**: MS-S1 Maxの大容量メモリを活かして、32Kコンテキスト設定で書籍の章全体を要約できます。

#### データの比較分析

```
プロンプト:
"以下の2つの製品仕様を比較し、それぞれの長所・短所を
表形式でまとめてください。

製品A:
[仕様を貼り付け]

製品B:
[仕様を貼り付け]

比較観点: 価格、性能、拡張性、サポート"
```

### 8.2.4 学習支援

#### 概念の説明

```
プロンプト:
"機械学習の「過学習（overfitting）」について、
プログラミング初心者でも理解できるように、
身近な例えを使って説明してください。"
```

#### 練習問題の生成

```
プロンプト:
"Pythonの基本文法（if文、for文、関数）を学習中です。
これらを組み合わせた練習問題を3つ作成してください。
難易度は初級から中級です。模範解答も含めてください。"
```

## 8.3 APIサーバーモードの活用

### 8.3.1 ローカルサーバーの起動

LM Studioは、OpenAI API互換のローカルサーバーとして動作できます。

#### 起動手順

```
1. LM Studioを開く
2. 左側のタブから「🌐 Local Server」を選択
3. 使用するモデルを選択
4. 「Start Server」をクリック

サーバー情報:
  URL: http://localhost:1234
  エンドポイント: /v1/chat/completions
  認証: 不要（ローカル環境）
```

#### サーバー設定

```
Settings:
  Port: 1234（デフォルト、変更可能）
  CORS: 有効（ブラウザアプリからアクセス可能）
  Log Level: Info
  Auto-start: オフ（必要に応じて有効化）
```

### 8.3.2 他のアプリケーションとの連携

#### VS Code + Continue拡張機能

**Continueのインストール**

```
1. VS Codeを開く
2. 拡張機能マーケットプレイスで「Continue」を検索
3. インストール
```

**LM Studioとの連携設定**

```json
// ~/.continue/config.json

{
  "models": [
    {
      "title": "LM Studio",
      "provider": "openai",
      "model": "local-model",
      "apiBase": "http://localhost:1234/v1",
      "apiKey": "not-needed"
    }
  ],
  "tabAutocompleteModel": {
    "title": "LM Studio",
    "provider": "openai",
    "model": "local-model",
    "apiBase": "http://localhost:1234/v1"
  }
}
```

**使用方法**

```
1. LM StudioでLocal Serverを起動（DeepSeek-Coder推奨）
2. VS Codeでコードを開く
3. Ctrl+L（Windows/Linux）でContinueを開く
4. コードに関する質問、リファクタリング依頼等を入力
```

#### Pythonスクリプトからの利用

```python
import openai

# LM Studio APIの設定
openai.api_base = "http://localhost:1234/v1"
openai.api_key = "not-needed"

# チャット補完の実行
response = openai.ChatCompletion.create(
    model="local-model",
    messages=[
        {"role": "system", "content": "あなたは親切なアシスタントです。"},
        {"role": "user", "content": "Pythonのリスト内包表記について説明してください。"}
    ],
    temperature=0.7,
    max_tokens=1000
)

print(response.choices[0].message.content)
```

#### curlコマンドでのテスト

```bash
curl http://localhost:1234/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "local-model",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Hello!"}
    ],
    "temperature": 0.7,
    "max_tokens": 100
  }'
```

### 8.3.3 Webアプリケーションの構築

#### Streamlitを使ったチャットアプリ

```python
import streamlit as st
import openai

# LM Studio API設定
openai.api_base = "http://localhost:1234/v1"
openai.api_key = "not-needed"

st.title("ローカルAIチャット")

# セッション状態の初期化
if "messages" not in st.session_state:
    st.session_state.messages = []

# 過去のメッセージを表示
for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# ユーザー入力
if prompt := st.chat_input("メッセージを入力してください"):
    # ユーザーメッセージを追加
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.markdown(prompt)

    # AI応答を生成
    with st.chat_message("assistant"):
        response = openai.ChatCompletion.create(
            model="local-model",
            messages=st.session_state.messages,
            temperature=0.7,
            stream=True
        )

        full_response = ""
        message_placeholder = st.empty()

        for chunk in response:
            if chunk.choices[0].delta.get("content"):
                full_response += chunk.choices[0].delta.content
                message_placeholder.markdown(full_response + "▌")

        message_placeholder.markdown(full_response)

    # アシスタント応答を保存
    st.session_state.messages.append({"role": "assistant", "content": full_response})
```

**実行方法:**

```bash
# Streamlitのインストール
pip install streamlit openai

# アプリの実行
streamlit run chat_app.py
```

## 8.4 効率的なワークフロー

### 8.4.1 テンプレートの活用

よく使うプロンプトは、テンプレートとして保存しましょう。

#### プロンプトテンプレート例

**コードレビューテンプレート**

```
以下のコードをレビューしてください。

【確認観点】
1. バグや潜在的な問題
2. パフォーマンスの改善余地
3. 可読性の向上
4. ベストプラクティスへの準拠

【コード】
```
[言語名]
[コードを貼り付け]
```

【特に重視する点】
[例: セキュリティ、パフォーマンス等]
```

**文章添削テンプレート**

```
以下の文章を添削してください。

【目的】[例: ビジネスメール、ブログ記事]
【対象読者】[例: 一般ユーザー、専門家]
【文体】[例: 敬体、常体]

【元の文章】
[文章を貼り付け]

【添削の観点】
1. 文法や表現の誤り
2. より適切な語彙の提案
3. 文章構成の改善
4. 読みやすさの向上
```

### 8.4.2 ショートカットとTips

#### キーボードショートカット

```
Windows/Linux:
  Ctrl+L: プロンプト入力欄にフォーカス
  Ctrl+Enter: メッセージ送信
  Ctrl+Shift+C: 会話のコピー
  Ctrl+Shift+N: 新しいチャット
  Ctrl+Shift+H: ハードウェア設定を開く

macOS:
  Cmd+L: プロンプト入力欄にフォーカス
  Cmd+Enter: メッセージ送信
  Cmd+Shift+C: 会話のコピー
  Cmd+Shift+N: 新しいチャット
  Cmd+Shift+H: ハードウェア設定を開く
```

#### 効率化のTips

**Tip 1: マークダウン形式で出力を依頼**

```
プロンプト:
"以下の情報をMarkdown形式の表で整理してください：
[データを貼り付け]"

→ 出力をそのままドキュメントに貼り付け可能
```

**Tip 2: コードブロックの指定**

```
プロンプト:
"Pythonのサンプルコードを提供してください。
必ず```python```のコードブロックで囲んでください。"

→ シンタックスハイライトされた出力
```

**Tip 3: 段階的な出力**

```
プロンプト:
"長い説明になる場合は、セクションごとに区切って出力してください。
各セクションの後に、続きを読むか確認してください。"

→ 長文を管理しやすく
```

## 8.5 トラブルシューティング

### 8.5.1 よくある問題と解決法

#### 問題1: 応答が途中で止まる

```
原因:
  - Max Tokensが小さすぎる
  - コンテキスト長の上限到達

解決法:
  1. Max Tokensを増やす（2048 → 4096）
  2. 会話をリセット（New Chat）
  3. より大きいコンテキスト長を設定
```

#### 問題2: 応答品質が低い

```
原因:
  - モデルが小さすぎる
  - プロンプトが不明確
  - 不適切なパラメータ設定

解決法:
  1. より大きいモデルを試す（7B → 14B → 32B）
  2. プロンプトを具体的に書き直す
  3. Temperatureを調整（低すぎる or 高すぎる）
```

#### 問題3: 日本語の品質が低い

```
原因:
  - 日本語に弱いモデルを使用

解決法:
  1. Qwenシリーズを使用（日本語に強い）
  2. システムプロンプトで日本語を明示
     "必ず日本語で応答してください。"
  3. Few-shot例を提供
```

### 8.5.2 パフォーマンス改善

#### 応答速度を上げる

```
方法1: より小さいモデルを使用
  70B → 32B → 14B → 7B

方法2: より軽い量子化
  Q6 → Q5 → Q4

方法3: コンテキスト長を減らす
  32K → 16K → 8K

方法4: 性能モードを変更
  Quiet → Balance → Performance
```

#### メモリ使用量を減らす

```
方法1: 会話をこまめにリセット
方法2: コンテキスト長を減らす
方法3: 不要なモデルをアンロード
方法4: 他のアプリケーションを終了
```

## 8.6 本章のまとめ

本章では、LM Studioの実践的な使い方を学習しました。

✅ **チャットインターフェース**
- 効果的なプロンプト作成
- システムプロンプトの活用
- マルチターン対話の管理

✅ **実践的なユースケース**
- 文章作成・編集
- コード生成とレビュー
- データ分析と要約
- 学習支援

✅ **APIサーバーモード**
- ローカルサーバーの起動
- VS Code、Pythonとの連携
- Webアプリケーション構築

✅ **効率的なワークフロー**
- テンプレート活用
- ショートカット
- トラブルシューティング

次章では、高度な機能とカスタマイズを学びます。

---

**前章へ**: [第7章 MS-S1 Max向け最適化設定](chapter07_optimization.md)
**次章へ**: [第9章 高度な機能とカスタマイズ](chapter09_advanced_features.md)
