# 第8章 プロンプトエンジニアリング

## プロンプトエンジニアリングとは

本章では、LLMに「より正しい、より有用な回答」をさせるための技法を学びます。プロンプトエンジニアリングは、入力するテキスト（プロンプト）の設計手法を体系的に学ぶ分野です。

### なぜプロンプトエンジニアリングが重要なのか

LLMは「何を尋ねられたか」によって、出力の品質が大きく変わります。同じモデルでも、プロンプトの良し悪しで回答精度が数倍変わることは珍しくありません。

```
悪いプロンプト: 「Pythonについて」
→ 出力: Pythonはプログラミング言語です。（広すぎる）

良いプロンプト: 「Pythonでデータベースを操作するための主要なライブラリを3つ挙げ、各ライブラリの使いどころを30字以内で説明してください」
→ 出力: 
1. sqlite3: 組み込みDB。小規模アプリ・テスト向け
2. SQLAlchemy: O/Rマッパー。柔軟なクエリ構築向け
3. psycopg2: PostgreSQL専用。高機能なDB操作向け
```

## プロンプト設計の基本原則

### 6つの原則

1. **具体的に**: 曖昧な指示ではなく、具体的な出力形式や制約を明記
2. **構造化して**: 段落分け、箇条書きを活用
3. **例を示す**: Few-shot（後述）で正解パターンを提示
4. **制約を設ける**: 文字数、フォーマット、禁止事項を明記
5. **役割を与えて**: 「あなたは〇〇の専門家です」と役割を設定
6. **ステップバイステップ**: 複雑なタスクは段階的に分解

### プロンプトテンプレートの基本構造

```python
from langchain.prompts import PromptTemplate

# 基本テンプレート
template = PromptTemplate(
    input_variables=["topic", "num_items", "word_limit"],
    template="""あなたは{name_role}の専門家です。

{topic}について、以下を条件にリスト形式で回答してください：
- 項目数: {num_items}個
- 各項目の文字数: {word_limit}字以内
- 専門用語を使う場合は30字以内の解説も付加すること"""
)

prompt = template.format(
    name_role="Python",
    topic="データ分析ライブラリ",
    num_items="3",
    word_limit="30"
)

# 生成結果:
# あなたはPythonの専門家です。
# データ分析ライブラリについて、以下を条件にリスト形式で回答してください：
# ...
```

## Few-shot Learning

### 基本概念

Few-shot learningは、プロンプトの中に「入力-正解出力の例」をいくつか含める技法です。モデルが例のパターンを学習して、同様の形式で回答します。

```python
from langchain.llms import Ollama
from langchain.chains import LLMChain

llm = Ollama(model="llama3.2")

few_shot_prompt = """以下の例の形式に従って、感情分析を行ってください。

例1:
入力: この映画はとても面白かった！
感情: 肯定的
置信度: 0.95

例2:
入力: 期待外れだった。もう見ない。
感情: 否定的
置信度: 0.91

例3:
入力: まぁまぁだった。
感情: 中立的
置信度: 0.75

例4:
入力: Pythonのコードが動いた！感動！
感情: """

# このプロンプトをLLMに入力すると、例のパターンに従って出力する
result = llm.generate(text=few_shot_prompt)
print(result.text.strip())
# 出力例: 肯定的\n置信度: 0.93
```

### Few-shotを自動生成する

```python
import ollama

def generate_few_shot_examples(topic, num=3):
    """LLM自身がFew-shot例を生成する"""
    prompt = f"""次のテーマに関するFew-shot例を{num}組生成してください。
    形式は「入力: ...\n正解: ...」です。

    テーマ: {topic}

    例1:
    入力: {topic}の具体例その1
    正解: 例1への対応する回答
    ...を{num}組生成してください。"""

    response = ollama.generate(model="llama3.2", prompt=prompt)
    return response["response"]

# 実用上は、Few-shot例は事前に手動で作成しておくことを推奨
```

## Chain-of-Thought 推論

### 基本概念

Chain-of-Thought (CoT) は、モデルに「思考プロセス」を出力させる技法です。段階的に考えてもらうことで、複雑な問題の精度が大幅に向上します。

```python
import ollama

# CoTなし
no_cot_prompt = "17 x 24 = ?"
result1 = ollama.generate(
    model="llama3.2",
    prompt=no_cot_prompt
)

# CoTあり
cot_prompt = """17 x 24を計算してください。
以下のステップで考えてください：
1. 17 の一の位(7) x 24 を計算
2. 17 の十の位(10) x 24 を計算
3. 1と2の結果を合計
4. 最終的な答えを書く"""

result2 = ollama.generate(
    model="llama3.2",
    prompt=cot_prompt
)

print(f"CoTなし: {result1['response']}")
print(f"CoTあり: {result2['response']}")
```

### PythonでのCoTチェーン

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain.chains import LLMChain
from langchain_community.llms import Ollama

llm = Ollama(model="llama3.2", temperature=0)

# Chain-of-Thought Chain
cot_chain = LLMChain(
    llm=llm,
    prompt=ChatPromptTemplate.from_messages([
        ("human", """あなたは論理的思考の専門家です。
質問に対して、以下のステップで回答してください：

1. まず問題を分解する
2. 各要素を独立に考える
3. 結果を統合する
4. 最終解答を書く

質問: {question}

答え: """)
    ])
)

result = cot_chain.run(question="Pythonで最大公約数を求める関数を書いてください")
print(result)
```

## ReAct（Reasoning + Acting）

### ReActとは

ReActは、推論（Reasoning）と行動（Action）を交互に行う手法です。LLMが「考えて→行動して→結果を見て→さらに考える」というルートを踏みます。

```python
import ollama
import subprocess

# ReActプロンプト
react_template = """以下の質問に答えてください。ReActフレームワークに従います。

Format:
思考: <このステップで何を考えているか>
行動: <実行する行動>
行動入力: <行動の入力>
結果: <行動の結果>
思考: <結果を見てどう考えるか>
...
最終解答: <最終的な回答>

質問: {question}

思考: """

# 実際のコード例
def react_agent(question, max_steps=5):
    """簡易ReActエージェント"""
    chain_of_thought = "思考: "
    
    for step in range(max_steps):
        full_prompt = react_template.format(
            question=question,
            current_context=chain_of_thought.strip()
        )
        
        # 行動の決定
        result = ollama.generate(
            model="llama3.2",
            prompt=full_prompt
        )
        
        response = result['response']
        
        # 行動から文字列を抽出（簡易版）
        if "最終解答:" in response:
            return response.split("最終解答:")[1].strip()
        
        chain_of_thought += response + "\n"
        chain_of_thought += "行動: [" + question + "]\n"
    
    return "最終解答: 十分な情報が得られませんでした"

answer = react_agent(「Pythonのバージョン情報を取得してください」)
print(answer)
```

### OpenAI関数呼び出しとの組み合わせ

```python
import ollama

def get_weather(location):
    """天気情報を取得する関数"""
    import requests
    response = requests.get(f"https://api.weatherapi.com/v1/current.json?key=YOUR_API&q={location}")
    return response.json()

# レキシコン関数の呼び出しをプロンプトに含める
def react_with_tools(question, tools):
    """LLMがツールを選択して実行"""
    tool_desc = "\n".join([f"- {name}: {desc}" for name, _, desc in tools])
    
    prompt = f"""ツールを使用して質問に答えてください。
    利用可能なツール:
    {tool_desc}

    質問: {question}

    Format:
    思考: タスクを分析
    行動: 使用するツール名
    行動入力: ツールの引数
    結果: ツールの出力
    最終解答: """
    
    response = ollama.generate(
        model="llama3.2",
        prompt=prompt
    )
    return response['response']
```

## 出力フォーマットの制御

### JSON出力

```python
import ollama
import json

# JSON出力プロンプト
json_prompt = """以下の形式でJSONを出力してください（他のテキストは一切出力しないこと）:

{{
    "category": "カテゴリ名",
    "items": [
        {{
            "name": "アイテム名",
            "description": "説明",
            "difficulty": "易/中/難"
        }}
    ]
}}

テーマ: 【Pythonのデータ可視化ライブラリ】"""

response = ollama.generate(
    model="llama3.2",
    prompt=json_prompt,
    format={"type": "object"}  # JSONモード（Ollama APIでサポート）
)

# JSONとしてパース（失敗する可能性があるので安全に）
try:
    result = json.loads(response['response'])
    print(result['items'])  # リストとしてアクセス可能
except (json.JSONDecodeError, KeyError):
    print(response['response'])
```

### グレード付き出力（Pydantic）

```python
from pydantic import BaseModel, Field
from typing import List

class SearchResult(BaseModel):
    title: str = Field(..., description="検索結果のタイトル")
    url: str = Field(..., description="URL")
    relevance_score: float = Field(..., description="関連度(0.0-1.0)")

class RAGResult(BaseModel):
    answer: str = Field(..., description="回答本文")
    sources: List[SearchResult] = Field(..., description="参照元")
    confidence: float = Field(..., description="自信度(0.0-1.0)")

# Pydantic経由で構造を強制
response = ollama.generate(
    model="llama3.2",
    prompt="Pythonに関する最新ニュースを3件検索してください",
    format=SearchResult.schema()  # Pydanticスキーマを自動適用
)
```

## 評価と改善サイクル

### プロンプトの評価方法

```python
from langchain.evaluation import load_evaluator
import ollama

def evaluate_prompt(prompt_template, test_cases, llm_name="llama3.2"):
    """プロンプトをテストケースで評価"""
    results = []
    
    for case in test_cases:
        prompt = prompt_template.format(
            question=case["question"],
            expected_type=case["expected"]
        )
        
        response = ollama.generate(
            model=llm_name,
            prompt=prompt
        )
        
        # 簡易評価（文字数、キーワード一致など）
        score = 0
        response_text = response['response']
        
        if case.get("min_length") and len(response_text) >= case["min_length"]:
            score += 1
        if case.get("require_keywords"):
            for kw in case["require_keywords"]:
                if kw in response_text:
                    score += 1
        
        results.append({
            "question": case["question"],
            "output": response_text[:100],
            "score": score,
            "max_score": case.get("max_score", 2)
        })
    
    total_score = sum(r["score"] for r in results)
    total_max = sum(r["max_score"] for r in results)
    print(f"全体のスコア: {total_score}/{total_max} ({total_score/total_max*100:.1f}%)")
    
    return results

# テストケース
test_cases = [
    {
        "question": "Pythonの利点を3つ挙げてください",
        "expected": "リスト形式",
        "min_length": 50,
        "require_keywords": ["利点", "機能"],
        "max_score": 3
    },
    {
        "question": "Pythonのリスト内包表記の説明してください",
        "expected": "例を含む説明",
        "min_length": 100,
        "require_keywords": ["内包"],
        "max_score": 3
    }
]

results = evaluate_prompt(
    prompt_template="あなたはPythonの専門家です。\n{question}",
    test_cases=test_cases
)
```

## まとめ

本章で学んだこと：

- プロンプトエンジニアリングはLLMの出力品質を決定する
- Few-shot、Chain-of-Thought、ReActは強力な技法
- 出力フォーマットを制御することで、パイプラインへの組み込みが容易になる
- 評価と改善のサイクルを回し続けることが重要

次章では、GPU加速と最適化を学びます。
