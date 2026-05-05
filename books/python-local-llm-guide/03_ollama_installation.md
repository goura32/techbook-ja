# 第3章 Ollamaの導入

## Ollamaとは

Ollamaは、ローカルでLLMを簡単に動かすためのツールです。モデルのダウンロードから実行まで、コマンド一つで完結します。

### Ollamaのインストール

すでに第2章でOllamaをインストール済みの方はこのセクションを飛ばしてください。

#### macOSでのインストール

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

#### Ubuntuでのインストール

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

#### Windowsでのインストール

1. [ollama.com](https://ollama.com) からWindowsインストーラーをダウンロード
2. インストーラーを実行
3. 通常は「次へ」の連続で完了

### Ollamaの起動

インストール後、Ollamaが自動的にバックグラウンドで起動します。

```bash
# Ollamaのステータス確認
ollama serving

# 手動で起動
ollama serve

# バックグラウンドプロセスの確認
ps aux | grep ollama
```

### 最初のモデルを動かす

```bash
# Llama 3.2モデルをダウンロードして実行
ollama run llama3.2

# プロンプトを入力する
> 自己紹介してください
私はLlama 3.2という大規模言語モデルです。

# Ctrl+Cで終了
```

## モデルの管理

### 利用可能なモデル一覧

```bash
# ダウンロード済みのモデル一覧
ollama list

# 例:
# namereference      size      modified
# llama3.2:latest    2.0GB     2日前
```

### モデルのダウンロード

```bash
# モデルをダウンロード（実際に動かすまでもない場合）
ollama pull llama3.2
ollama pull mistral

# リストからモデルを選ぶ
# 公式モデル一覧: https://ollama.com/library
```

### モデルの削除

```bash
# モデルの削除
ollama rm llama3.2

# 複数のモデルを一括削除
ollama rm llama3.2 mistral
```

## Ollama APIの基本

OllamaはHTTP APIも提供しています。これにより、Python等の外部アプリケーションからLLMを呼び出せます。

### APIエンドポイント

| エンドポイント | 説明 |
|-----------|------|
| `/api/tags` | ダウンロード済みモデル一覧 |
| `/api/show` | モデルの詳細表示 |
| `/api/generate` | テキスト生成 |
| `/api/chat` | チャット形式の対話 |
| `/api/embeddings` | 埋め込み生成 |

### cURLでのテスト

```bash
# テキストを生成
curl http://localhost:11434/api/generate \
  -d '{
    "model": "llama3.2",
    "prompt": "猫について3文で説明してください",
    "stream": false
  }'
```

### Pythonでの使用

```python
# Python公式パッケージを使用
import ollama

# テキスト生成
response = ollama.generate(
    model="llama3.2",
    prompt="犬について3文で説明してください"
)

print(response['response'])
# 出力例: 犬は、人間にとって最も古くから飼育されている動物の一つです。...
```

## プロンプトの最適化

### streamオプション

ストリーミングモード（リアルタイムで結果を出していく）とノンストリーミングモード（全結果を一度に返す）が選べます。

```python
# ストリーミングモード
response = ollama.generate(
    model="llama3.2",
    prompt="物語を書いてください",
    stream=True
)

for chunk in response:
    print(chunk['response'], end='', flush=True)
```

### コンテキストウィンドウ

```bash
# デフォルトのコンテキストは4096トークン
# 8192に変更するには:
ollama run llama3.2:8b -e
```

### モデルパラメータ

```bash
# tempreature（0.0～2.0）
# 低い = 保守的、高い = ランダム
ollama run llama3.2 --temperature 0.7
```

## まとめ

本章で学んだこと：

- Ollamaのインストール方法（macOS/Ubuntu/Windows）
- モデルのダウンロード・実行・削除
- Ollama HTTP APIの基本
- Pythonからの呼び出し方法

次章では、モデルの選択とダウンロードを深掘りします。

### GPUのメモリとモデルサイズの関係

GPUのVRAMサイズは、選択するモデルの最大パラメータ数を決定します。

| GPU | VRAM | 選択可能なモデル目安 |
|------|----|----|
| GTX 1080 Ti | 11GB | llama3.1:8b (Q3_K_M) |
| RTX 3060 | 12GB | llama3.1:8b (Q4_0) |
| RTX 4060 | 8GB | qwen2.5:7b (Q4_K_M) |
| RTX 4090 | 24GB | llama3.1:70b (Q4_0) |
| A100 | 80GB | llama3.1:405b (Q2_K) |

### モデルの管理コマンド

```bash
# ダウンロード済みモデルの一覧表示
ollama list

# 特定のモデルの詳細表示
ollama show llama3.1:8b

# モデルの情報を表示
ollama list --json
```

### カスタムモデルの作成

ollamaでは、カスタムモデルのModelfileからモデルを作成できます。

**Modelfileの作成：**

```docker
FROM llama3.1:8b

SYSTEM "あなたは日本語のITサポートアシスタントです。"

PARAMETER temperature 0.2
PARAMETER num_ctx 4096
```

**モデルのビルドとpush：**

```bash
# ローカルモデルとしてビルド
ollama create my-support-assistant -f Modelfile

# Ollama Library(Portal)にpush
ollama push username/my-support-assistant
```


### Ollamaの詳細設定

Ollamaはインストール後も柔軟に設定を変更できます。

#### 環境変数による設定

```bash
# Ollamaの動作ディレクトリを変更
export OLLAMA_MODELS=/mnt/data/models

# GPUオフロードのレベル（0: CPUのみ, -1: 全レイヤGPU, 0より大きい: 特定のレイヤ）
export OLLAMA_NUM_GPU=999

# フォアグラウンドで実行（デバッグ用途）
ollama serve
```

#### モデルの管理

```bash
# ダウンロードしたモデルの一覧
ollama list

# モデルの詳細情報
ollama show --verbose llama3.1

# モデルの削除
ollama rm llama3.1:8b

# カスタムモデルの作成（Modelfileを使用）
# Modelfileを作成
cat > Modelfile << 'EOF'
FROM llama3.1
PARAMETER temperature 0.7
PARAMETER num_predict 512
SYSTEM "あなたは親切なアシスタントです。"
EOF

ollama create my-llama -f Modelfile
```

### OllamaのAPI仕様

OllamaのREST APIを利用すれば、任意の言語からOLLamaにアクセスできます。

#### APIの主なエンドポイント

| エンドポイント | 説明 |
|--|--|
| GET /api/tags | ダウンロードしたモデルの一覧 |
| POST /api/generate | シークエンス生成リクエスト |
| POST /api/chat | チャットリクエスト |
| POST /api/embed | 埋め向量の生成 |
| POST /api/create | 新しいモデルの作成 |

#### curlでのAPIテスト

```bash
# モデルリストの取得
curl http://localhost:11434/api/tags | python3 -m json.tool

# テキスト生成
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.1",
  "prompt": "Pythonの利点を3つ挙げてください。",
  "stream": false
}'

# チャットリクエスト
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.1",
  "messages": [
    {"role": "user", "content": "こんにちは！自己紹介をお願いします。"}
  ]
}'
```

### Ollamaのトラブルシューティング

| 現象 | 原因 | 解決策 |
|--|--|--|
| connection refused | Ollamaが起動していない | `ollama serve`で手動起動 |
| out of memory | VRAM不足 | `OLLAMA_NUM_GPU`を減らす |
| model not found | モデル未ダウンロード | `ollama pull <model>` |
| slow performance | CPU推論に落ちている | `nvidia-smi`でGPU状態を確認 |
