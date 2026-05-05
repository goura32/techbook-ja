# 第7章 RAGの構築

## RAGとは何か

前章まででLangChainとOllamaを使ってLLMを呼び出す方法を学びました。しかし、まだ1つ大きな課題が残っています。LLMが知っている情報だけでしか回答できないという制限です。

LLMは学習データに含まれていた知識しか持ち合わせません。最新のニュース、自社独自のデータ、特定の文脈に応じたカスタマイズされた情報には対応できません。こうした課題を解決するのが**RAG（Retrieval-Augmented Generation）**です。

RAGは外部の文書知识库をLLMに接続する技術で、以下の手順で動作します。

1. ユーザーが質問する
2. 外部文書から関連情報を検索
3. 検索した文脈をプロンプトに追加
4. LLMが文脈に基づいて回答を生成

現在、RAGはLLMアプリケーション構築の**標準的なパターン**になっています。クラウドに頼らないローカル完結型RAGの需要も高まっており、本書ではOllamaとChromaDBを使った環境で実装します。

## RAGのアーキテクチャ

RAGは以下の5段階パイプラインで構成されます。図の矢印はデータの流れを示します。

```
[ドキュメント] --> [チャンキング] --> [埋め込み] --> [ベクトルストア] --> [検索] --> [生成]
```

### チャンキング

ドキュメントをそのまま処理すると膨大なデータになるため、適切なサイズの小さな塊（チャンク）に分割します。文書によっては段落単位、パラグラフ単位、あるいはトークン単位で分割します。

### 埋め込み（Embedding）

各チャンクを**意味のベクトル**に変換します。例えば「Pythonのリストとは」チャンクと「Pythonの配列について」チャンクは、意味が似ているためベクトル空間上で近い位置にマッピングされます。

### ベクトルストア

埋め込まれたベクトルを保存・管理するデータベースです。後から「この質問に最も似ている文書チャンクはどれか」を高速に検索できます。代表的なローカルベクトルストアとして**ChromaDB**が挙げられます。

### 検索（Retrieve）

ユーザーの質問を同じ埋め込みモデルでベクトルに変換し、ベクトルストアの中から最も意味的に近いチャンクを検索します。

### 生成（Generate）

検索したチャンクをコンテキストとしてプロンプトに追加し、LLMに回答を生成させます。これによりLLMは外部知識を参考にした正確な回答を返すことができます。

## ChromaDBのインストールと基本操作

ChromaDBはローカルで動作する軽量なベクトルデータベースです。ネットワーク不要でデータを保持できるため、プライバシー保護にも適しています。

```bash
pip install chromadb
```

### クライアントの初期化

```python
import chromadb

# ローカルにChromaDBクライアントを作成
client = chromadb.PersistentClient(path="./chroma_db")

# コレクション（データベースのテーブル相当）を作成
collection = client.get_or_create_collection(name="documents")

# ドキュメントと埋め込みを登録
collection.add(
    documents=["Pythonはスクリプト言語です。", "Rustはメモリ安全な言語です。"],
    metadatas=[
        {"source": "lang_tutorial.txt", "chapter": 1},
        {"source": "lang_tutorial.txt", "chapter": 2},
    ],
    ids=["doc_001", "doc_002"]
)

# 類似検索
results = collection.query(
    query_texts=["プログラミング言語について"],
    n_results=2
)
print(results["documents"][0])
```

コレクションに新しいドキュメントを追加するときは、`add()`メソッドを使います。既に存在するIDを指定するとエラーが発生するので注意してください。既存のドキュメントを更新する場合は、まず`delete()`して再登録するか、後述するLangChain経由でコレクションを操作するのが一般的です。

## LangChainのDocument Loaders

LangChainには、様々な形式のファイルをロードする機能が用意されています。本章では主要なローダーを解説します。

### テキストファイルの読み込み

最も基本的な`TextLoader`です。

```python
from langchain_community.document_loaders import TextLoader

loader = TextLoader("documents/sample.txt", encoding="utf-8")
docs = loader.load()
print(f"ドキュメント数: {len(docs)}")
print(f"最初のチャンクのテキスト: {docs[0].page_content[:100]}")
```

### PDFファイルの読み込み

```bash
pip install pypdf
```

```python
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("documents/manual.pdf")
docs = loader.load()
print(f"ページ数: {len(docs)}")
```

### CSVファイルの読み込み

```python
from langchain_community.document_loaders import CSVLoader

loader = CSVLoader("documents/data.csv")
docs = loader.load()
```

### ディレクトリ全体の読み込み

```python
from langchain_community.document_loaders import DirectoryLoader

loader = DirectoryLoader(
    "documents/",
    glob="**/*.txt",
    loader_cls=TextLoader,
    loader_kwargs={"encoding": "utf-8"}
)
docs = loader.load()
```

## Text Splitterの選択

ロードしたドキュメントを適切なサイズのチャンクに分割します。LangChainには複数のSplitterが用意されています。

### RecursiveCharacterTextSplitter（推奨）

最も一般的に使われるスプリッターです。区切り文字を再帰的に試しながらチャンクを分割します。デフォルトでは`[\n\n, \n, ]`の順に区切り文字を試します。

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,    # 各チャンクの文字数
    chunk_overlap=50,  # チャンク間の重複文字数
)
split_docs = splitter.split_documents(docs)
print(f"分割後のチャンク数: {len(split_docs)}")
print(f"最初のチャンク: {split_docs[0].page_content[:80]}")
```

### CharacterTextSplitter

単純に指定した区切り文字で分割します。計算は速いですが、チャンクが意味の区切りと一致しない場合があります。

```python
from langchain_text_splitters import CharacterTextSplitter

splitter = CharacterTextSplitter(
    separator="\n\n",
    chunk_size=500,
    chunk_overlap=0,
)
```

### TokenTextSplitter

トークン単位で分割します。LLMへのコンテキスト送信コストを抑えたい場合に有効です。

```python
from langchain_text_splitters import TokenTextSplitter

splitter = TokenTextSplitter(
    chunk_size=256,  # トークン数
    chunk_overlap=16,
)
```

## 埋め込みモデルの選択

埋め込みモデルは、テキストを数値ベクトルに変換する重要なコンポーネントです。ローカルで完結させる場合、2つの選択肢があります。

### Ollamaの埋め込みモデルを使う

本書で使っているOllamaは埋め込みモデルも提供しています。Ollamaがインストールされていれば追加ツールは不要です。

```bash
ollama pull nomic-embed-text
```

```python
from langchain_ollama import OllamaEmbeddings

embeddings = OllamaEmbeddings(
    model='nomic-embed-text',
    base_url='http://localhost:11434'
)
vectors = embeddings.embed_documents([
    "Pythonはスクリプト言語です。",
    "Rustはメモリ安全な言語仕様を持っています。",
])
print(f"ベクトル次元数: {len(vectors[0])}")
```

### sentence-transformersを使う

```bash
pip install sentence-transformers
```

```python
from langchain_community.embeddings import HuggingFaceEmbeddings

embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2"
)
```

本章以降では、Ollamaの埋め込みモデルを使って進めます。モデルによってベクトルの次元数や精度が異なるため、同じデータセットであってもモデルを変えると検索結果が変わる可能性があります。複数の埋め込みモデルを試して最適な組み合わせを見つけることをお勧めします。

## RAGパイプラインの構築（全コード）

本章ではこれまで学んできた要素を組み合わせて、完全なRAGアプリケーションを構築します。

### 環境設定

```bash
pip install langchain langchain-ollama chromadb
```

### 実装コード

```python
from langchain_ollama import ChatOllama, OllamaEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import TextLoader, DirectoryLoader
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# --- ステップ1: 埋め込みモデルの設定 ---
embeddings = OllamaEmbeddings(model='nomic-embed-text')

# --- ステップ2: ドキュメントの読み込みと分割 ---
# documents/ディレクトリ内の全txtファイルをロード
loader = DirectoryLoader(
    "documents/",
    glob="**/*.txt",
    loader_cls=TextLoader,
    loader_kwargs={"encoding": "utf-8"}
)
docs = loader.load()

# 500文字ごとにチャンク分割（50文字の重複あり）
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
)
split_docs = splitter.split_documents(docs)
print(f"合計{len(split_docs)}個のチャンクに分割しました。")

# --- ステップ3: ベクトルストアの構築 ---
# ChromaDBにチャンクを保存
vectorstore = Chroma.from_documents(
    documents=split_docs,
    embedding=embeddings,
    collection_name="my_rag_collection"
)
print("ベクトルストアの構築が完了しました。")

# --- ステップ4: 検索コンポーネントの作成 ---
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 3}  # 上位3件のチャンクを検索
)

# --- ステップ5: プロンプトの設定 ---
prompt = ChatPromptTemplate.from_messages([
    ("system", """あなたは信頼性の高いAIアシスタントです。
以下の[検索結果]から回答を構築してください。
[検索結果]に情報がない場合は、その旨を正直に答えてください。
回答は簡潔に、箇条書きまたは短文で構成してください。"""),
    ("human", "[搜索結果]\n{context}\n\n質問: {question}")
])

# --- ステップ6: LLMの設定 ---
llm = ChatOllama(
    model='llama3.1',
    temperature=0.3
)

# --- ステップ7: 全処理をLCELで接続 ---
chain = {
    "context": retriever | StrOutputParser(),
    "question": RunnablePassthrough()
} | prompt | llm | StrOutputParser()

# --- 使用例 ---
query = "本書で扱っている主なトピックは何ですか？"
result = chain.invoke(query)
print(result)
```

このパイプラインのデータの流れをまとめると以下の通りです。

```
ドキュメント
    |
   ローディング
    |
分割（チャンキング）
    |
埋め込み + 保存（ChromaDB）
    |
検索（retriever）
    |
コンテキスト結合 + プロンプト
    |
生成（LLM）
    |
出力
```

1つのPythonコードで完結しており、ドキュメントの追加・更新・削除だけで知識ベースをいつでも更新できます。

## 精度改善のヒント

構築したRAGパイプラインをもっと正確にするためのポイントを紹介します。

### チャンクサイズと重複の調整

`chunk_size`が小さすぎると重要な情報が分割されてしまいます。一方、大きすぎると検索の精度が下がります。対象のドキュメントの種類に応じて最適なサイズを探してください。

```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,    # 文書の内容に合わせて調整
    chunk_overlap=50,  # 文脈の切れ目にならないように重複を持たせる
)
```

### 検索数の調整

`k`（検索件数）を適度に増やすことで、文脈に豊富な情報を提供できます。ただし多すぎるとLLMの注意が分散する可能性があります。

```python
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 5}
)
```

### 埋め込みモデルの変更

`nomic-embed-text`だけでなく、`all-MiniLM-L6-v2`など異なる埋め込みモデルを試してください。データセットに合うモデルが異なることがあります。

### コレクションの更新

新しいドキュメントを追加した場合、コレクションも更新が必要です。コレクションの削除と再構築は本番環境では適さないため、追加・更新APIを使います。

```python
# コレクションの更新例
vectorstore.add_documents(new_docs)

# 特定のドキュメントを削除する場合はIDで指定
vectorstore.collection.delete(ids=["doc_001"])
```

## まとめ

本章ではRAGの基本概念から実装までを学びました。RAGは外部知識をLLMに接続する強力な技術で、ドキュメントを追加・更新するだけで知識ベースを柔軟に拡張できます。本章で作成したパイプラインをベースに、自身の用途に合わせてドキュメントの形式や分割方法、埋め込みモデルを変えて試してみてください。

次回第8章では、RAGの実践的な応用として、複数のデータソースを統合する高度なRAGについて解説します。
