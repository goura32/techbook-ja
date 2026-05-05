# 第4章 モデルの選択とダウンロード

## モデルファミリーの分類

LLMには様々なモデルがあります。本章では主要なモデルを比較し、どう選ぶかを学びます。

### オープンソース系 vs クロップド系

| 比較項目 | オープンソース系 | クロップド系 |
|------|-------|------|
| モデル例 | Llama 3.2, Mistral, Gemma | GPT-4, Claude, Gemini |
| リソース | ローカルで動作 | クラウドAPI利用 |
| 商用利用 | モデルのライセンスによる | 利用規約による |
| カスタマイズ | 自由 | 制限 |

## 主要モデルの比較

### Llama (Meta)

- パラメータ: 1B, 3B, 8B, 70B, 405B
- 商用利用: 可能（ライセンス要確認）
- 多言語: 日本語可
- 特徴: Metaが公開した最先端のオープンモデル

### Mistral Nemo (Mistral AI)

- パラメータ: 12B
- 商用利用: Apache 2.0ライセンス（自由）
- 多言語: 日本語可
- 特徴: English-heavyだが日本語も対応

### Gemma 2 (Google)

- パラメータ: 2B, 9B, 27B
- 商用利用: Gemmaコミュニティライセンス
- 多言語: 日本語可（英語得意）
- 特徴: Googleの軽量モデル、メモリ効率がよい

## モデルの選定基準

### タスク別の推奨モデル

| タスク | 推奨モデル |
||-----|||
| 文章要約 | llama3.1:8b |
| コード生成 | llama3.1:8b or Mistral Nemo |
| チャット | llama3.1:8b |
| 翻訳 | Mistral Nemo |
| 画像理解 | llama3.1-vision |
| 高度な推論 | Llama 3.1 70B（GPU必須）|

### モデル名の読み方

```
llama3.1:8b      → Llama 3.2、8Bパラメータ
mistral-nemo:latest → Mistral Nemo（最新版）
gemma2:9b-it     → Gemma 2、9B、Instruct-Tuned版
qwen2.5:14b      → Qwen 2.5、14Bパラメータ
```

## ダウンロードの実際

### ollama pullコマンド

```bash
# 基本構文
ollama pull <model_name>

# 例: llama3.1:8bをダウンロード
ollama pull llama3.1:8b

# 例: Mistral Nemoをダウンロード
ollama pull mistral-nemo

# 例: Gemma 2をダウンロード
ollama pull gemma2:9b

# ダウンロードが完了したら確認
ollama list
```

### ダウンロードの確認

```bash
# モデル一覧
ollama list

# 例:
# NAME                          SIZE    MODIFIED
# llama3.1:8b                   4.7 GB  2日前
# llama3.1:3b                 2.0 GB  2日前
# mistral-nemo:latest           12 GB   1日前
```

### モデルの詳細

```bash
# モデルの詳細表示
ollama show llama3.1:8b
```

### マルチモデルの使用

### 複数のモデルを一度にダウンロード

```bash
# 一度に複数のモデルをダウンロード
ollama pull llama3.1:8b
ollama pull mistral-nemo:latest
ollama pull gemma2:9b
```

### モデルの切り替え

```python
import ollama

# llama3.1で生成
response1 = ollama.generate(
    model="llama3.1",
    prompt="東京の天気予報を教えて"
)

# mistral-nemoで生成
response2 = ollama.generate(
    model="mistral-nemo",
    prompt="東京の天気予報を教えて"
)

# 同じプロンプトで異なる出力
print(response1['response'][:100])
print(response2['response'][:100])
```

## 量化（Quantization）

### 量化とは

量化はモデルのパラメータを低精度に変換して、メモリ使用量を減らす技法です。

### 量化レベルの比較

| 量化レベル | ファイルサイズ | 品質 | メモリ必要量 |
|------|-------|----|------|
| Q4_0 (4bit) | ~50% | 少し低下 | 約50% |
| Q4_K_M (4.5bit) | ~55% | ほぼ同等 | 約55% |
| Q5_K_M (5bit) | ~60% | ほぼ同等 | 約60% |
| Q6_K (6bit) | ~70% | ほぼ同等 | 約70% |
| Q8_0 (8bit) | ~90% | ほぼ同等 | 約90% |

### 量化モデルのダウンロード

```bash
# Q4_0量化モデル
ollama pull llama3.1:8b-q4_0

# Q5_K_M量化モデル
ollama pull llama3.1:8b-q5_K_M

# Q8_0（高精度）
ollama pull llama3.1:8b-q8_0
```

## まとめ

本章で学んだこと：

- 主要なLLMモデルについて
- モデルの選定基準
- ollama pullコマンドでのダウンロード
- 量化によるメモリ節約

次章では、Pythonから模型を呼び出す方法を学びます。
