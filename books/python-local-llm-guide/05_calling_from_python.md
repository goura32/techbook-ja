# 第5章 Pythonからの呼び出し

## ollama Pythonパッケージ

本章では、PythonコードからOllamaを操作する方法を詳しく学びます。

### Pythonパッケージのインストール

```bash
pip install ollama
```

### 基本的な利用

```python
import ollama

# 基本構文
response = ollama.generate(
    model='llama3.1',
    prompt='Pythonの利点を3つ挙げてください'
)

print(response['response'])
```

### 応答の構造

```python
{
    'model': 'llama3.1',
    'created_at': '2000-01-01T00:00:00.000000Z',
    'response': 'Pythonの利点は以下の通りです...',
    'done': True,
    'total_duration': 1234567890,
    'load_duration': 1234567890,
    'prompt_eval_count': 100,
    'eval_count': 200
}
```

## プロンプトの設計

### プロンプトの基本的なルール

```python
import ollama

# 悪い例: 曖昧すぎる
response1 = ollama.generate(
    model='llama3.1',
    prompt='Pythonについて書いて'
)

# 良い例: 具体的で制約がある
response2 = ollama.generate(
    model='llama3.1',
    prompt="""Pythonの利点について教えてください。
以下の条件を満たしてください：
1. Pythonの利点を3つ挙げること
2. 各利点の実例を1つずつ示すこと
3. 各項目は30字以内で簡潔に書くこと""""
)
```

### システムメッセージの利用

```python
# システムメッセージで役割を与え、モデルの出力を制御
response = ollama.chat(
    model='llama3.1',
    messages=[
        {
            'role': 'system',
            'content': 'あなたは日本語の技術書執筆アシスタントです。簡潔に答えてください。'
        },
        {
            'role': 'user',
            'content': 'Pythonのリスト内包表記の利点を教えてください'
        }
    ]
)

print(response['message']['content'])
```

## ストリーミング処理

### リアルタイム出力

```python
import ollama

# stream=Trueでリアルタイム出力
response = ollama.generate(
    model='llama3.1',
    prompt='Pythonの機能について教えてください',
    stream=True
)

for chunk in response:
    # chunkには'token'フィールドが含まれる
    print(chunk['token'], end='', flush=True)
```

### ストリーミングで文章を構築

```python
import ollama

def complete_sentence(text):
    """ストリーミングで完了した文章を返す"""
    response = ollama.generate(
        model='llama3.1',
        prompt='Pythonの基本的な機能を解説してください',
        stream=True
    )
    
    full_text = ''
    for chunk in response:
        token = chunk['token']
        full_text += token
        print(token, end='', flush=True)
    
    print()  # 改行
    return full_text
```

## 埋め込み(embedding)の利用

### 埋め込みとは

埋め込み(vector)は、テキストを数値のベクトルに変換するものです。これにより、文章の類似度を比較できます。

```python
import ollama

# 埋め込みを生成
vectors = ollama.embed(
    model='llama3.1',
    input='Pythonのリスト内包表記について'
)

print(vectors['embeddings'])
# [[0.030691541, -0.024131535, 0.012345678, ...]]
```

### 文章の類似度計算

```python
import ollama
import numpy as np

def get_similarity(text1, text2):
    """2つの文章の類似度を計算"""
    emb1 = ollama.embed(model='llama3.1', input=text1)['embeddings'][0]
    emb2 = ollama.embed(model='llama3.1', input=text2)['embeddings'][0]
    
    cosine_sim = np.dot(emb1, emb2) / (np.linalg.norm(emb1) * np.linalg.norm(emb2))
    return cosine_sim

# テスト
sim = get_similarity(
    'Pythonのリスト内包表記',
    'Pythonでリストを効率的に生成する方法'
)
print(f"類似度: {sim:.3f}")
```

## エラー処理

### よくあるエラー

```python
import ollama

# エラーケース1: モデルが存在しない
try:
    response = ollama.generate(
        model='non-existent-model',
        prompt='テスト'
    )
except ollama.ResponseError as e:
    print(f'Ollamaエラー: {e.error}')
    print('ollama pull non-existent-model で解決')

# エラーケース2: API接続エラー
try:
    response = ollama.generate(
        model='llama3.1',
        prompt='テスト'
    )
except ollama.ConnectionError:
    print('Ollamaが起動していません')

# エラーケース3: システムオーバーロード
try:
    response = ollama.generate(
        model='llama3.1',
        prompt='テスト'
    )
except ollama.ResponseError as e:
    print(f'リソース不足: {e.error}')
```

### 自動リトライ

```python
import ollama
import time

def generate_with_retry(prompt, retries=3, delay=2):
    """リトライ付き生成"""
    for attempt in range(retries):
        try:
            response = ollama.generate(
                model='llama3.1',
                prompt=prompt
            )
            return response['response']
        except ollama.ResponseError as e:
            if attempt == retries - 1:
                raise
            print(f"リトライ {attempt + 1}/{retries}")
            time.sleep(delay)

response = generate_with_retry('Pythonについて教えてください', retries=3)
print(response)
```

## まとめ

本章で学んだこと：

- ollama Pythonパッケージの基本的な使い方
- システムメッセージ・プロンププトの設計
- ストリーミングと埋め込みの活用
- エラー処理と自動リトライ

次章では、LangChainとOllamaを統合する方法を学びます。


### チャットAPIによるマルチターン会話

OllamaのチャットAPIを使えば、以前の会話の内容を文脈として保持できます。

```python
import ollama

messages = [
    {"role": "system", "content": "あなたは数学の講師です。"},
    {"role": "user", "content": "微積分の基本的な概念を教えてください。"},
]

response = ollama.chat(
    model="llama3.1",
    messages=messages,
    stream=True
)

for chunk in response:
    print(chunk["message"]["content"], end="", flush=True)

print()

# 次のメッセージを追記する
messages.append({"role": "assistant", "content": response["message"]["content"]})
messages.append({"role": "user", "content": "具体的な例も教えてください。"})

response2 = ollama.chat(model="llama3.1", messages=messages)
print(response2["message"]["content"])
```

### バッチ処理とストリーミング

大量のテキストを処理する際、ストリーミングはリアルタイム性の向上に寄与します。

```python
import ollama

# ストリーミング応答
response = ollama.generate(
    model="llama3.1",
    prompt="Pythonでの機械学習のステップを10段階で説明してください。",
    stream=True
)

for chunk in response:
    if "response" in chunk:
        print(chunk["response"], end="", flush=True)
```

### Ollamaの主要なパラメータ

Ollama APIで利用可能な主要なパラメータの一覧です。

| パラメータ | 説明 | デフォルト | 有効範囲 |
|--|--|--|--|
| model | 使用するモデル名 | -- | -- |
| prompt | プロンプトテキスト | -- | -- |
| temperature | 出力のランダム性（高いほど多様） | 0.8 | 0.0〜2.0 |
| num_predict | 最大生成トークン数 | -1（無限） | 1〜4096 |
| top_p | トップPサンプリング（高いほど多様） | 0.9 | 0.0〜1.0 |
| top_k | トップKサンプリング（高いほど多様） | 40 | 1〜100 |
| repeat_penalty | 繰り返しの罰則 | 1.1 | 1.0〜2.0 |
| num_ctx | コンテキスト窓のサイズ | 2048 | 128〜32768 |
| stream | ストリーミングの有無 | true | true/false |

### エラーハンドリングの実装例

```python
import ollama
import time

def retry_generate(prompt, retries=3, delay=2):
    for attempt in range(retries):
        try:
            response = ollama.generate(
                model="llama3.1",
                prompt=prompt,
                timeout=60  # 60秒のタイムアウト
            )
            return response
        except ollama.ResponseError as e:
            print(f"リクエストエラー（{attempt+1}/{retries}回目）: {e}")
            time.sleep(delay * (attempt + 1))  # エクスポネンシャルバックオフ
    
    raise Exception("すべてのリトライが失敗しました")
```
