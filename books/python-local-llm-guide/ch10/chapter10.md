# 第10章 実践プロジェクト - チャットボットの構築

## プロジェクト概要

本章では、これまでに学んだすべての技術を総動員して、**ローカル完結型のチャットボットアプリケーション**を構築します。完成するアプリケーションのイメージは以下の通りです。

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Frontend   │───▶│  Backend   │───▶│  LLM Model  │
│ (Streamlit)  │◀──▶│ (FastAPI) │◀──▶│  (Ollama)   │
└─────────────┘    └─────────────┘    └─────────────┘
                          │
                    ┌─────────────┐
                    │  ChromaDB  │
                    │ (ローカルDB) │
                    └─────────────┘
```

### 機能要件
1. チャットUIでLLMと対話
2. ドキュメントのアップロードと検索（RAG）
3. 履歴の保存と再生
4. ローカル完結（クラウドなし）

## 環境セットアップ

### 必要なパッケージ

```bash
pip install fastapi uvicorn streamlit ollama langchain chromadb pypdf python-multipart
```

### ディレクトリ構成

```
local-chatbot/
├── main.py          # FastAPIバックエンド
├── app.py           # Streamフロントエンド
├── rag_pipeline.py  # RAGパイプライン
├── config.py        # 設定ファイル
├── requirements.txt  # 依存パッケージ
├── documents/       # ドキュメント格納
│   ├── docs1.txt
│   └── docs2.txt
└── db/              # ChromaDBストレージ
    └── ...
```

## 本体コード

### 設定ファイル（config.py）

```python
"""チャットボットの設定"""

# Ollamaの設定
OLLAMA_URL = "http://localhost:11434"
MODEL_NAME = "llama3.2"
EMBEDDING_MODEL = "nomic-embed-text"

# RAGの設定
CHUNK_SIZE = 500
CHUNK_OVERLAP = 50
MAX_TOKENS = 1024
TEMPERATURE = 0.7

# ChromaDBの設定
PERSIST_DIR = "./db/chroma_db"
COLLECTION_NAME = "local_chatbot"

# FastAPIの設定
HOST = "0.0.0.0"
PORT = 8000

# Streamlitの設定
ST_HOST = "0.0.0.0"
ST_PORT = 8501
```

### RAGパイプライン（rag_pipeline.py）

```python
"""RAGパイプラインの実装"""

import os
from pathlib import Path
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import (
    TextLoader,
    PyPDFLoader,
    DirectoryLoader,
)
from langchain_community.vectorstores import Chroma
from langchain_community.embeddings import OllamaEmbeddings
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_community.llms import Ollama
from config import (
    CHUNK_SIZE, CHUNK_OVERLAP, PERSIST_DIR,
    COLLECTION_NAME, EMBEDDING_MODEL, MODEL_NAME
)


class RAGPipeline:
    """RAGパイプラインのクラス"""

    def __init__(self):
        self.embeddings = OllamaEmbeddings(model=EMBEDDING_MODEL)
        self.llm = Ollama(model=MODEL_NAME, temperature=0)

    def load_documents(self, directory: Path) -> list:
        """ディレクトリからドキュメントをロード"""
        loaders = []
        
        for file_path in directory.glob("*"):
            if file_path.suffix == '.txt':
                loaders.append(TextLoader(str(file_path)))
            elif file_path.suffix == '.pdf':
                loaders.append(PyPDFLoader(str(file_path)))
        
        if not loaders:
            raise ValueError(f"{directory}にサポートされたファイルがありません")
        
        # すべてのドキュメントをロード
        docs = []
        for loader in loaders:
            docs.extend(loader.load())
        
        return docs

    def split_documents(self, documents: list) -> list:
        """ドキュメントをチャンクに分割"""
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=CHUNK_SIZE,
            chunk_overlap=CHUNK_OVERLAP,
            length_function=len,
        )
        return splitter.split_documents(documents)

    def create_vector_store(self, documents: list, persist_dir: str = PERSIST_DIR) -> Chroma:
        """ベクトルストアの作成/読み込み"""
        path = Path(persist_dir)
        path.mkdir(parents=True, exist_ok=True)
        
        # 既存のストアを読み込み、なければ作成
        vectorstore = Chroma(
            persist_directory=str(path),
            embedding_function=self.embeddings,
            collection_name=COLLECTION_NAME,
        )
        
        # 既存にドキュメントがない場合のみ追加
        if vectorstore._collection.count() == 0:
            vectorstore.add_documents(documents)
            print(f"ベクトルストアを作成しました ({vectorstore._collection.count()} ドキュメント)")
        
        return vectorstore

    def build_chain(self, vectorstore: Chroma):
        """検索→生成パイプラインの構築"""
        # テキスト抽出チェーン
        question_answer_chain = create_stuff_documents_chain(
            llm=self.llm,
            system_prompt="""あなたは技術サポートアシスタントです。
            与えられたドキュメントの情報基于いて回答してください。
            情報が足りない場合は、「その情報はドキュメントにありません」と答えてください。""",
        )

        # RAGチェーン
        retrieval_chain = create_retrieval_chain(
            retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),  # 上位3件
            combine_documents_chain=question_answer_chain,
        )

        return retrieval_chain


# 使用例
if __name__ == "__main__":
    pipeline = RAGPipeline()
    docs = pipeline.load_documents(Path("./documents"))
    chunks = pipeline.split_documents(docs)
    vectorstore = pipeline.create_vector_store(chunks)
    chain = pipeline.build_chain(vectorstore)
    
    result = chain.invoke({"input": "Pythonについて教えてください"})
    print(result["answer"])
```

### バックエンド（main.py）

```python
"""FastAPIバックエンド"""

from fastapi import FastAPI, UploadFile, File, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from pathlib import Path
from rag_pipeline import RAGPipeline
from config import HOST, PORT

app = FastAPI(title="Local LLM Chat API")

# CORS設定
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# グローバル変数
pipeline = RAGPipeline()
chain = None


class ChatRequest(BaseModel):
    """チャットリクエスト"""
    message: str
    history: list = []  # 会話履歴


class ChatResponse(BaseModel):
    """チャットレスポンス"""
    response: str
    sources: list = []
    error: str = ""


@app.on_event("startup")
async def startup_event():
    """アプリ起動時にRAGパイプラインを初期化"""
    global chain
    docs = pipeline.load_documents(Path("./documents"))
    chunks = pipeline.split_documents(docs)
    vectorstore = pipeline.create_vector_store(chunks)
    chain = pipeline.build_chain(vectorstore)
    print(f"RAGパイプライン初期化完了（{len(docs)} ドキュメント）")


@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """チャットエンドポイント"""
    try:
        if chain is None:
            return ChatResponse(response="", error="RAGパイプラインが初期化されていません")
        
        result = chain.invoke({"input": request.message})
        return ChatResponse(response=result["answer"],)
    except Exception as e:
        return ChatResponse(response="", error=str(e))


@app.post("/upload")
async def upload_document(file: UploadFile = File(...)):
    """ドキュメントのアップロード"""
    try:
        docs_dir = Path("./documents")
        docs_dir.mkdir(parents=True, exist_ok=True)
        
        file_path = docs_dir / file.filename
        with open(file_path, "wb") as f:
            content = await file.read()
            f.write(content)
        
        return {"message": f"アップロード成功: {file.filename}", "path": str(file_path)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host=HOST, port=PORT)
```

### フロントエンド（app.py）

```python
"""Streamlitフロントエンド"""

import streamlit as st
import requests
from config import ST_PORT

# ページ設定
st.set_page_config(page_title="Local LLM Chat", layout="wide")
st.title("ローカルLLMチャット")
st.caption("Ollama + Llama 3.2 駆動のローカルチャットボット")

# チャット履歴の初期化
if "messages" not in st.session_state:
    st.session_state["messages"] = []


def send_message(message):
    """バックエンドにメッセージを送信"""
    history = st.session_state["messages"]
    response = requests.post(
        "http://localhost:8000/chat",
        json={"message": message, "history": history}
    )
    return response.json()


# チャットUI
container = st.container()
with container:
    for msg in st.session_state["messages"]:
        with st.chat_message(msg["role"]):
            st.write(msg["content"])

    if prompt := st.chat_input("メッセージを入力..."):
        st.session_state["messages"].append({"role": "user", "content": prompt})
        with st.chat_message("user"):
            st.write(prompt)

        # バックエンドに応答
        api_response = send_message(prompt)
        response_text = api_response.get("response", "エラーが発生しました")
        
        st.session_state["messages"].append({"role": "assistant", "content": response_text})
        with st.chat_message("assistant"):
            st.write(response_text)

# ドキュメントアップロード
st.sidebar.header("ドキュメント管理")
uploaded_file = st.sidebar.file_uploader("ドキュメントをアップロード", type=['txt', 'pdf'])
if uploaded_file:
    with open(f"documents/{uploaded_file.name}", "wb") as f:
        f.write(uploaded_file.getbuffer())
    st.sidebar.success(f"アップロード完了: {uploaded_file.name}")
```

## デプロイ

### ローカルで起動

```bash
# 1. Ollamaを起動（別ターミナル）
ollama serve

# 2. バックエンドを起動
uvicorn main:app --host 0.0.0.0 --port 8000

# 3. 別ターミナルでフロントエンドを起動
streamlit run app.py --server.port 8501
```

### Dockerでのデプロイ

```dockerfile
# Dockerfile（バックエンド用）
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  backend:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./documents:/app/documents
      - ./db:/app/db

  frontend:
    image: python:3.11-slim
    ports:
      - "8501:8501"
    volumes:
      - ./app.py:/app/app.py
    command: streamlit run app.py --server.port 8501

  ollama:
    image: ollama/ollama
    volumes:llama:
      - ollama_data:/root/.ollama
```

```bash
# Docker Composeで起動
docker-compose up -d
```

## 改善の方向性

### 今後の拡張パス

1. **Agent機能の追加**
   - ツール呼び出し（API, Web検索）
   - 並列処理のエージェント

2. **多モーダル対応**
   - 画像理解（多モーダルモデルへの切り替え
   - 音声入力

3. **モニタリング**
   - LangSmithとの統合
   - ロギングと分析

4. **スケーリング**
   - 複数GPUでの分散推論
   - vLLMへの移行

## まとめ

本書で学んだこと：

1. LLMの基本とローカル運用の意義
2. 開発環境のセットアップ
3. Ollamaの導入とモデル管理
4. PythonからのLLM操作
5. LangChainとの統合
6. RAGの構築
7. プロンプトエンジニアリング
8. GPU最適化
9. 実践的なチャットボットの構築

これらは起点にすぎません。LLMの世界は日進月歩です。新しいモデル、新しいライブラリ、新しい技法に触れながら、あなた自身のAIアプリを作り続けてください。

## おわりに

本書を通じて「AIを他人のシステムに頼らず、自分で制御できる」力を身につけてもらえたら幸いです。AIはまだ進化中です。あなたの創造性で、可能性を最大化してください。

---

## 付録

### 主なリソース

- Ollama: https://ollama.com
- LangChain: https://python.langchain.com
- Hugging Face: https://huggingface.co
- Meta Llama: https://llama.meta.com

### 用語一覧

| 用語 | 日本語 | 説明 |
|--------|-----|------|
| LLM | 大規模言語モデル | 大量のテキストで学習したAI |
| トークン | 単語（切り出し） | テキストを切り出した単位 |
| パラメータ | 重み | モデル内の学習済みパラメータ |
| モデル | 学習済みモデル | 訓練されたAI本体 |
| プロンプト | 入力テキスト | モデルに入力する指示 |
| 推論 | Inference | モデルに応答を生成すること |
| RAG | 検索拡張生成 | 外部知識を取り込んで回答する方式 |
| 量子化 | Quantization | 精度を下げてメモリ使用量を減らす技術 |
