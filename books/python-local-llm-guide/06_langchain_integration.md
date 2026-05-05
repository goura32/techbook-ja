# 第6章 LangChainとの統合

## LangChainとは

前章までで、OllamaをPythonの`ollama`パッケージを使って直接呼び出す方法を学びました。本章では、より高レベルなフレームワークである**LangChain**を使ってOllamaを操作する方法を解説します。

LangChainはLLMを活用したアプリケーションを構築するためのオープンソースフレームワークです。プロンプトの生成、モデルの呼び出し、結果の後処理などの部品をモジュール化されており、RAG（Retrieval-Augmented Generation）などの複雑な処理も短く記述できます。

LangChainで最も重要なのは**LCEL（LangChain Expression Language）**という独自の記法です。LCELはチェーン（処理の流れ）をDSL的に宣言できる仕組みで、`|`演算子を使って処理を連結するのが基本です。

```
プロンププト --> モデル --> 出力
prompt | model | output_parser
```

## LangChain + Ollamaのセットアップ

### インストール

LangChainとOllama統合パッケージをインストールします。

```bash
pip install langchain langchain-ollama
```

### オプション: LangSmithの追加

 LangChainのトレーリングにLangSmithを利用する場合は、以下のパッケージもインストールします。

```bash
pip install langsmith
```

### 動作確認

インストールが完了したら、まずは簡単なチェーンを組んで動作を確認します。

```python
from langchain_ollama import ChatOllama
from langchain_core.prompts import ChatPromptTemplate

# Ollamaモデルの読み込み
llm = ChatOllama(model='llama3.1', temperature=0.7)

# シンプルなプロンプト
prompt = ChatPromptTemplate.from_messages([
    ("system", "あなたは日本語のAIアシスタントです。"),
    ("user", "{text}"),
])

# Chainの構築: prompt -> model
chain = prompt | llm
response = chain.invoke({"text": "Pythonとは何ですか？"})

print(response.content)
```

このように`|`演算子で`ChatPromptTemplate`と`ChatOllama`を連結すると、テンプレート適用からLLM呼び出しまでの処理が1行で完結します。

## LCEL（LangChain Expression Language）基本

LCELは複数のコンポーネントを連結する仕組みです。LCELで使える要素は大きく分けて3つあります。

### 主要コンポーネント

| コンポーネント | 役割 |
|---------------|------|
| `ChatPromptTemplate` | プロンプトテンプレートの構築 |
| `ChatOllama` | Ollamaモデルとの接続 |
| `StringOutputParser` | 生の文字列として出力を返す |

### コネクトの仕組み

LCELは`|`演算子でコンポーネントを左から右へ連結します。

```python
from langchain_ollama import ChatOllama
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# 各コンポーネント
prompt = ChatPromptTemplate.from_messages([
    ("system", "あなたは簡潔な回答をするAIです。"),
    ("user", "{question}"),
])
llm = ChatOllama(model='llama3.1', temperature=0.5)
parser = StrOutputParser()

# 連結: prompt -> llm -> parser
chain = prompt | llm | parser

# 実行
result = chain.invoke({"question": "Pythonの利点は？"})
print(result)
# Pythonの利点:
# 1. 構文がシンプルで学習しやすい
# 2. 豊富な標準ライブラリ
# 3. 大規模なエコシステムとコミュニティ
```

`StrOutputParser`を追加するのは、`llm`の出力が`AIMessage`オブジェクトだからです。`StrOutputParser`を挟むと、`content`属性の内容が直接文字列として得られます。

### 関数の挿入

カスタム処理を`lambda`関数として差し込むこともできます。

```python
chain = prompt | llm | parser | (lambda text: text.strip())

result = chain.invoke({"question": "  空白を含むテキスト  "})
print(repr(result))  # '空白が含ま除去される'
```

## チャット用のChain構築

### プログラマチックな対話

`RunnableWithMessageHistory`を使えば、会話履歴を管理するChatボットが構築できます。

```python
from langchain_ollama import ChatOllama
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.messages import HumanMessage, AIMessage

llm = ChatOllama(model='llama3.1', temperature=0.5)

prompt = ChatPromptTemplate.from_messages([
    ("system", "あなたは質問に答えるAIアシスタントです。"),
    MessagesPlaceholder(variable_name="history"),
    ("user", "{question}"),
])

# 簡易的な履歴ストア（メモリ内）
store = {}

def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = []
    return store[session_id]

chain_with_history = RunnableWithMessageHistory(
    prompt | llm,
    get_session_history,
    input_messages_key="question",
    history_messages_key="history",
)

session_id = "session_1"

# 第1回質問
response1 = chain_with_history.invoke(
    {"question": "Pythonのリストとタupleの違いは？"},
    config={"configurable": {"session_id": session_id}},
)
print(response1.content)

# 第2回質問（前回の文脈が残る）
response2 = chain_with_history.invoke(
    {"question": "それは何で使いますか？"},
    config={"configurable": {"session_id": session_id}},
)
print(response2.content)
```

`MessagesPlaceholder`がメッセージ履歴の挿入点を表し、`RunnableWithMessageHistory`がセッションごとに履歴を管理します。

### ストリーミング出力

LCELではストリーミングも容易です。

```python
chain = prompt | llm

for chunk in chain.stream({"question": "Pythonの基本的な型を5つ挙げてください"}):
    print(chunk.content, end="", flush=True)
```

`invoke`が全体の結果を返すのに対し、`stream`は各チャンクを逐次返します。

## カスタムChainの構築

### 複数ステップのチェーン

実務では、複数の処理を組み合わせる必要があります。

```python
from langchain_ollama import ChatOllama
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatOllama(model='llama3.1', temperature=0)

# ステップ1: テキストを要約
summarize_prompt = ChatPromptTemplate.from_messages([
    ("system", "入力を50字以内で要約してください。"),
    ("user", "{text}"),
])

# ステップ2: 要約結果を日本語で解説
explain_prompt = ChatPromptTemplate.from_messages([
    ("system", "要約をもう少し詳しく解説してください。"),
    ("user", "{summary}"),
])

# 2ステップチェーンの構築
summarize_chain = summarize_prompt | llm | StrOutputParser()
explain_chain = explain_prompt | llm | StrOutputParser()

# 全体のチェーン: text -> summarize -> explain
full_chain = summarize_chain | (lambda s: {"summary": s}) | explain_chain

# 実行
result = full_chain.invoke({
    "text": "Pythonは Guido van Rossum によって1991年にリリースされたプログラミング言語です。 interpretedで高水準で汎用的で、スクリプト言語として広く使われています。"
})

print(result)
# 要約→解説の結果が返される
```

### カスタムランナブル

複雑な処理が必要な場合は`@chain`デコレータを使います。

```python
from operator import itemgetter
from langchain_core.runnables import chain
from langchain_ollama import ChatOllama
from langchain_core.prompts import ChatPromptTemplate

llm = ChatOllama(model='llama3.1')

# JSONレスポンス用のプロンプト
prompt = ChatPromptTemplate.from_messages([
    ("system", "{instruction}"),
    ("user", "{text}"),
])

@chain
def analyze_text(input_dict: dict) -> dict:
    """カスタム分析関数"""
    text = input_dict["text"]
    instruction = input_dict["instruction"]

    chain = prompt | llm
    msg = chain.invoke({"instruction": instruction, "text": text}).content

    # 簡易的な分析
    return {
        "original": text,
        "instruction": instruction,
        "response": msg,
        "length": len(msg),
    }

result = analyze_text.invoke({
    "text": "Pythonは学習コストが低く、Web開発からデータ科学まで幅広く使われています。",
    "instruction": "この文の要約と分析をしてください。",
})

print(result)
```

`@chain`デコレータ付き関数はLCELチェーンとして自動登録されます。`invoke`で通常のChainと同じように扱えます。

## エラーハンドリングとロギング

### エラーハンドリング

LCELチェーンでも例外が発生します。代表的なものを捕获して処理します。

```python
import time
from langchain_ollama import ChatOllama
from langchain_core.prompts import ChatPromptTemplate
import logging

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)

async_llm = ChatOllama(model='llama3.1', temperature=0.5)
prompt = ChatPromptTemplate.from_messages([
    ("system", "あなたはAIアシスタントです。"),
    ("user", "{question}"),
])

def safe_chain(question: str, retries: int = 3) -> str:
    """エラーハンドリング付きのChain呼び出し"""
    for attempt in range(retries):
        try:
            chain = prompt | async_llm
            response = chain.invoke({"question": question})
            return response.content
        except Exception as e:
            logger.warning(f"リトライ {attempt+1}/{retries}: {e}")
            if attempt == retries - 1:
                raise RuntimeError(f"Chain呼び出し失敗: {e}")
            time.sleep(2 ** attempt)  # 指数バックオフ

result = safe_chain("Pythonについて教えてください")
print(result)
```

### テンプレートレンダリング時のエラー

プロンプトテンプレート自体に問題がある場合は、`invoke`前に検証しておくとよいです。

```python
from langchain_core.prompts import PromptTemplate, ChatPromptTemplate

# プロンプトの検証
try:
    prompt = ChatPromptTemplate.from_messages([
        ("system", "{greeting}"),
        ("user", "{query}"),
    ])
    # 必須フィールドが補完できるかテスト
    test = prompt.format(greeting="こんにちは", query="テスト")
    print("テンプレート有効:", test)
except KeyError as e:
    print(f"必須変数不足: {e}")
```

## LangSmith連携（オプション）

LangChainには**LangSmith**というトレーリングプラットフォームがあります。ローカルでのチェーン動作を可視化・デバッグできます。

### LangSmithの設定

```bash
pip install langsmith
```

`.env`ファイルまたは環境変数でAPIキーを設定します。

```bash
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY=your-api-key-here
export LANGCHAIN_PROJECT=local-llm
```

### コード例

```python
import os

# LangSmith有効化
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-api-key"
os.environ["LANGCHAIN_PROJECT"] = "local-llm"

from langchain_ollama import ChatOllama
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatOllama(model='llama3.1', temperature=0.5)
prompt = ChatPromptTemplate.from_messages([
    ("system", "あなたは簡潔に答えるAIです。"),
    ("user", "{question}"),
])
chain = prompt | llm | StrOutputParser()

# この呼び出しはLangSmithに自動ログ送信される
result = chain.invoke({"question": "Pythonの利点は？"})
print(result)
```

LangSmithダッシュボード（`https://smith.langchain.com`）で、各チェーン呼び出しの入力・出力・遅延・トークン量が詳細に記録されます。デバッグだけでなく、モデルやパラメータの変更による出力品質の比較にも利用できます。

**注意**: LangSmithの利用には有料プランが必要です。ローカル開発中は無料枠で十分利用可能です。

## まとめ

本章で学んだこと：

- **LangChain**はLLMアプリ構築のオープンソースフレームワーク
- **LCEL**は`|`演算子でコンポーネントを連結する記法
- **ChatOllama**でローカルのOllamaモデルを容易に利用可能
- **RunnableWithMessageHistory**で会話履歴を管理するチャットボットが構築できる
- **@chain**デコレータでカスタム処理をLCELチェーンに統合できる
- **LangSmith**トレーリングでチェーンの動作を追跡できる

LangChainを使うと、プロンプトテンプレートの管理、複数ステップの処理、履歴の追跡が統一的なAPIで実現できます。次章では、RAGの構築方法を学び、テキスト検索とLLMの連携に挑戦します。
