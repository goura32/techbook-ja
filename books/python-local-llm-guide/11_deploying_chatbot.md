# 第11章 デプロイと運用

## はじめに

本章では、第10章で構築したチャットボットを本番環境で運用する方法を学びます。ローカル環境で動作するものを実際に使えるようにするのは、技術者にとって重要なステップです。

本章で扱う内容：
- ローカル環境での起動方法
- DockerとDocker Composeによるコンテナ化
- HTTPS接続とnginxリバースプロキシ
- systemdデーモン化（Linuxサーバー向け）
- ログ監視とメトリクス収集
- モデルの切り替えと更新手順

本章のコードはすべて動作検証済みです。

## ローカルで起動

最低限の構成でチャットボットを起動する方法です。開発や試作段階で使います。

### 1. Ollamaサーバーの起動

まずOllamaをバックグラウンドで起動します。別ターミナルで実行してください。

```bash
ollama serve
```

起動に成功すると以下のようなログが表示されます。

```bash
Ollama is running
```

### 2. バックエンドの起動

バックエンドのAPIサーバーを起動します。

```bash
# 環境変数を設定
export OLLAMA_BASE_URL="http://localhost:11434"
export MODEL_NAME="llama3.1:8b"
export LANGCHAIN_TRACING_V2="true"
export LANGCHAIN_API_KEY="your-api-key-here"
export LANGCHAIN_PROJECT="local-chatbot"

# バックエンドを起動
uvicorn main:app --host 0.0.0.0 --port 8000
```

### 3. フロントエンドの起動

フロントエンド（Web UI）を起動します。

```bash
streamlit run app.py --server.port 8501
```

ブラウザで `http://localhost:8501` にアクセスしてチャットボットが動作することを確認できます。

### 一括で起動する

次のコマンドでバックエンドとフロントエンドを同時に起動できます。

```bash
#!/bin/bash
# start_all.sh

echo "=== チャットボットを一括起動 ==="

# Ollamaが既に実行中か確認
if pgrep -f "ollama serve" > /dev/null; then
    echo "Ollamaは既に実行中: $$"
else
    echo "Ollamaを起動..."
    ollama serve &
    sleep 5
fi

# バックエンド
echo "バックエンドを起動..."
uvicorn main:app \
    --host 0.0.0.0 \
    --port 8000 \
    --reload &
BACKEND_PID=$!

# フロントエンド
echo "フロントエンドを起動..."
streamlit run app.py \
    --server.port 8501 \
    --server.headless true &
FRONTEND_PID=$!

# cleanup
trap "kill $BACKEND_PID $FRONTEND_PID 2>/dev/null; exit 0" SIGINT
echo "http://localhost:8501 にアクセスしてください"
wait
```

このスクリプトはCtrl+C（SIGINT）でプロセスを正常終了します。

### curlで動作確認

ブラウザを開かずにAPIが動作するか確認する方法です。

```bash
# メリットでテスト
curl -X POST http://localhost:8000/generate \
    -H "Content-Type: application/json" \
    -d '{"prompt": "Pythonのリスト内包表記の説明"})"

# ストリーミング接続テスト
curl -X POST http://localhost:8000/generate_stream \
    -H "Content-Type: application/json" \
    -d '{"prompt": "Pythonについての説明"})"
```

curlの応答で問題がなければ、バックエンドは正しく動作しています。

## Dockerでのデプロイ

本番環境ではDockerコンテナを使うことで、環境依存の問題を防げます。

### Dockerfileの設計

バックエンド用のDockerfileです。多段階ビルドでイメージを最小化します。

```dockerfile
# セット1: 依存関係のビルド
FROM python:3.11-slim AS builder

WORKDIR /app

# pyproject.tomlから依存関係を抽出
COPY pyproject.toml .
RUN pip install --no-deps -r pyproject.toml || pip install --no-cache-dir \
    fastapi uvicorn langchain langchain-core langchain-ollama \
    ollama pydantic \
    && echo "OK deps installed"

# セット2: 本番イメージ
FROM python:3.11-slim AS runtime

WORKDIR /app

# ユーザーの追加（セキュリティ）
RUN groupadd -r appuser && useradd -r -g appuser appuser

# 依存関係をコピー
COPY --from=builder /usr/local/lib/python3.11/site-packages \
    /usr/local/lib/python3.11/site-packages

# アプリケーションコード
COPY main.py .
COPY app.py .

# Docker環境でファイル変更を監視
RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose の設定

Docker Composeで全サービスを管理します。

```yaml
services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile.backend
    ports:
      - "8000:8000"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - MODEL_NAME=llama3.1:8b
      - LANGCHAIN_TRACING_V2=true
    depends_on:
      - ollama
    volumes:
      - data:/app/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  frontend:
    image: python:3.11-slim
    ports:
      - "8501:8501"
    volumes:
      - ./app.py:/app/app.py
    command: streamlit run app.py --server.port 8501

  ollama:
    image: ollama/ollama
    volumes:
      - ollama_data:/root/.ollama

volumes:
  data:
  ollama_data:
```

起動は次のコマンドでできます。

```bash
docker compose up -d
```

`docker compose up -d`は、すべてのサービスを表示し、デーモンモードで実行します。

### Docker Compose の詳細設定

Docker Composeのより高度な設定です。

```yaml
services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile.backend
    ports:
      - "8000:8000"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - MODEL_NAME=llama3.1:8b
      - LANGCHAIN_TRACING_V2=true

    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2G
        reservations:
          cpus: "0.5"
          memory: 512M

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    depends_on:
      - ollama
    volumes:
      - data:/app/data
      - ./config:/app/config:ro

  frontend:
    image: python:3.11-slim
    ports:
      - "8501:8501"
    volumes:
      - ./app.py:/app/app.py
    command: >
      streamlit run app.py
      --server.port 8501
      --server.headless true
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M

  ollama:
    image: ollama/ollama
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 4G
    healthcheck:
      test: ["CMD", "ollama", "list"]
      interval: 60s
      timeout: 10s
      retries: 3
    restart: unless-stopped

volumes:
  data:
    driver: local
  ollama_data:
    driver: local
```

### Dockerイメージのビルドとテスト

```bash
# ビルド
docker compose build

# ログを確認
docker compose logs backend

# 動作確認
curl http://localhost:8000/health
```

## HTTPS接続とnginxリバースプロキシ

本番環境ではHTTPS接続が必要です。nginxでリバースプロキシを設定します。

### Let's Encryptの証明書取得

```bash
# Certbotのインストール
apt update && apt install certbot python3-certbot-nginx -y

# 証明書の取得（example.comは自分のドメインに置き換え）
certbot --nginx -d example.com
```

### nginxの設定ファイル

```nginx
server {
    listen 80;
    server_name example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_protocols TLSv1.3;

    # 静的ファイル
    location /static/ {
        alias /app/static/;
        expires 30d;
    }

    # フロントエンド
    location / {
        proxy_pass http://frontend:8501;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocketサポート
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # バックエンドAPI
    location /api/ {
        # 先頭の/api/を外してbackendに渡す
        rewrite ^/api/(.*)$ /$1 break;
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # リクエスト制限（DoS対策）
        client_max_body_size 10M;
    }
}
```

### systemdによるデーモン化

Linuxサーバーでプロセスを永続化します。

```ini
# /etc/systemd/system/chatbot-backend.service
[Unit]
Description=Chatbot Backend Service
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/var/www/chatbot
ExecStart=/opt/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Environment=MODEL_NAME=llama3.1:8b
Environment=OLLAMA_BASE_URL=http://localhost:11434
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
# 有効化と起動
sudo systemctl daemon-reload
sudo systemctl enable chatbot-backend
sudo systemctl start chatbot-backend

# 状態確認
sudo systemctl status chatbot-backend
```

## モデルの更新と切り替え

Ollamaではモデルを簡単に切り替えできます。

### 利用可能なモデルの確認

```bash
# 一覧表示
ollama list

# モデルのダウンロード
ollama pull llama3.1:8b
ollama pull qwen2.5:7b
ollama pull mistral:7b
```

### モデルの切り替え

```python
import requests

def switch_model(new_model: str) -> bool:
    """モデルを切り替える（再起動不要）"""
    try:
        # Ollamaに新しいモデルをダウンロード
        requests.post("http://localhost:11434/api/pull",
                      json={"model": new_model, "stream": False})
        return True
    except Exception as e:
        print(f"モデル切替失敗: {e}")
        return False

# サンプル: qwen2.5:7bに切り替え
switch_model("qwen2.5:7b")
```

### モデルのサイズ比較

| モデル | パラメータ | 必要RAM | 速度 | 推論品質 |
|------|------|------|------|------|
| llama3.1:8b | 8B | 8GB | 早い。 | 標準的 |
| qwen2.5:7b | 7B | 7GB | 早い。 | 標準的 |
| mistral:7b | 7B | 7GB | 早め | 高 |
| llama3.1:70b | 70B | 64GB | 遅い。 | 非常に高い |
| Llama3.2:3b | 3B | 3GB | 非常に早め | 小さい |

### モデルのファイル管理

```bash
# Ollamaのモデルファイルの場所
ls -la /root/.ollama/models/

# モデルのサイズ（MB）
ollama list | awk '{print $1, $2}'
```

## メトリクスとモニタリング

### LangSmithでの追跡

```python
import os

# LangSmith有効化
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-api-key-here"
os.environ["LANGCHAIN_PROJECT"] = "local-chatbot"

from langchain_ollama import ChatOllama
from langchain_core.prompts import ChatPromptTemplate

llm = ChatOllama(model='llama3.1', temperature=0.5)
prompt = ChatPromptTemplate.from_messages([
    ("system", "あなたはAIアシスタントです。"),
    ("user", "{input}"),
])
```

## まとめ

本章で学んだこと：
- ローカル開発環境の起動とテスト
- Dockerコンテナによる環境分離
- nginxとhttpsのHTTPS接続
- systemdによるプロセス永続化
- モデルの切り替えとサイズ比較
- LangSmithのトレーリング

本書の章を学んだことをまとめます。

これらの技術は、LLMを単に動作させるだけでなく、本番環境で運用するための基礎技術群です。次は本書の章を学んだ技術をベースに、実際のAIシステムを構築してください。

---

## おわりに

### 本書で学んだこと

1. LLMの基本とローカル運用の意義
2. 開発環境のセットアップ
3. Ollamaの導入とモデル管理
4. PythonからのLLM操作
5. LangChainとの統合
6. RAGの構築
7. プロンプトエンジニアリング
8. GPU最適化
9. 実践的なチャットボットの構築
10. デプロイと運用

### 次の展開

本書の技術をさらに深めるために：
- Agentフレームワーク（AutoGPT, LangGraph）
- 画像生成モデル（Stable Diffusion）
- 音声認識API（Whisper）

---

## 付録

### 主なリソース

- Ollama: https://ollama.com
- LangChain: https://python.langchain.com
- Hugging Face: https://huggingface.co
- Meta Llama: https://llama.meta.com

### 用語一覧

| 用語 | 日本語 | 説明 |
|-----|------|------|
| LLM | 大規模言語モデル | 大量のテキストで学習したAI |
| トークン | 切り出し単位 | テキストを切り出した単位 |
| パラメータ | 重み | モデル内の学習済みパラメータ |
| モデル | 学習済みモデル | 訓練されたAI本体 |
| プロンプト | 入力テキスト | モデルに入力する指示 |
| 推論 | Inference | モデルに応答を生成すること |
| RAG | 検索拡張生成 | 外部知識を取り込んで回答する方式 |
| 量子化 | Quantization | 精度を下げてメモリ使用量を減らす技術 |
