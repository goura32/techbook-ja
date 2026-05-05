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
