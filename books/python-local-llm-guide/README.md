# Pythonで学ぶローカルLLM構築入門

> クラウドに頼らない。あなたのPCでAIを動かす。

この本は、PythonとOllamaを使って**ローカルLLMの構築〜運用**までを初心者向けに解説する実践ガイドです。

## 📖 目次

| 章 | タイトル | 内容 |
|---|---|---|
| [第0章](books/python-local-llm-guide/ch00/chapter00.md) | はじめに | 本書の目的・対象読者・必要環境 |
| [第1章](books/python-local-llm-guide/ch01/chapter01.md) | LLMとは何か | ローカルLLMの意義・モデルの分類 |
| [第2章](books/python-local-llm-guide/ch02/chapter02.md) | 環境構築 | Python・仮想環境・必要なパッケージ |
| [第3章](books/python-local-llm-guide/ch03/chapter03.md) | Ollamaの導入 | インストール・モデル管理・API基 |
| [第4章](books/python-local-llm-guide/ch04/chapter04.md) | モデルの選択 | Llama・Mistral・Gemmaの比較と量化 |
| [第5章](books/python-local-llm-guide/ch05/chapter05.md) | Pythonからの呼び出し | ollamaパッケージ・ストリーミング・埋め込み |
| [第6章](books/python-local-llm-guide/ch06/chapter06.md) | LangChainとの統合 | LCEL・チェーン・チャットHistory |
| [第7章](books/python-local-llm-guide/ch07/chapter07.md) | RAGの構築 | ChromaDB・Retrieval Pipeline |
| [第8章](books/python-local-llm-guide/ch08/chapter08.md) | プロンプトエンジニアリング | Few-shot・CoT・ReAct |
| [第9章](books/python-local-llm-guide/ch09/chapter09.md) | GPU加速と最適化 | GPU活用・量化・vLLM |
| [第10章](books/python-local-llm-guide/ch10/chapter10.md) | 実践プロジェクト | チャットボットの構築・デプロイ |

## 🚀 始め方

```bash
# 1. このリポジトリをクローン
git clone https://github.com/USER/python-local-llm-guide.git
cd python-local-llm-guide

# 2. 仮想環境を作成してアクティベート
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 3. 必要なパッケージをインストール
pip install ollama langchain langchain-community chromadb sentence-transformers pypdf streamlit fastapi uvicorn

# 4. Ollamaをインストール
curl -fsSL https://ollama.com/install.sh | sh

# 5. モデルをダウンロード
ollama pull llama3.2
ollama pull nomic-embed-text

# 6. 各章のコード例を実行
python <chapter_code_example>
```

## 📚 構成

本書は3フェーズで構成されます。

### フェーズ1: 基礎 (第0章〜第3章)
- LLMの基本理解
- 開発環境の準備
- Ollamaのインストールとモデル管理

### フェーズ2: Python連携 (第4章〜第6章)
- モデルの選択とダウンロード
- PythonからのLLM操作
- LangChainとの統合

### フェーズ3: 実践 (第7章〜第10章)
- RAGパイプラインの構築
- プロンプトエンジニアリング
- GPU加速と最適化
- チャットボットの構築・デプロイ

## 🛠 技術スタック

- **言語**: Python 3.10+
- **LLMランタイム**: Ollama
- **モデル**: Llama 3.2, Mistral Nemo, Gemma 2
- **フレームワーク**: LangChain v1+
- **ベクトルストア**: ChromaDB
- **Embedding**: Ollama Embeddings / SentenceTransformers
- **フロントエンド**: Streamlit
- **バックエンド**: FastAPI

## 📄 LICENSE

MIT License
