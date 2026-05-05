# 第9章 GPU加速と最適化

## GPU加速とは

本章では、LLMの推論速度を大幅に向上させるGPU活用と、メモリ使用量を減らす最適化技法を学びます。

### なぜGPU加速が必要なのか

LLMの推論は数億～数千億回の行列計算を伴います。CPUでは1つのプロンプトに数秒～数分かかりますが、GPUを使えばそれを数十分の一にできます。

| 環境 | Llama 3.2 8B (2048トークン生成) | メモリ使用量 | 消費電力 |
|------|-----|------|--|------|
| M1 MacBook (CPU) | ~60秒/回 | 6GB | 15W |
| RTX 3060 | ~8秒/回 | 5GB | 120W |
| RTX 4090 | ~3秒/回 | 5GB | 300W |
| 2x A100 | ~1秒/回 | 20GB | 600W |

## GPUの確認方法

### NVIDIA GPUの確認

```bash
# CUDAのバージョン確認
nvcc --version

# GPUの確認
nvidia-smi

# 例:
# +-----------------------------------------------------------------------------+
# | NVIDIA-SMI 535.129.03   Driver Version: 535.129.03   CUDA Version: 12.2   |
# |-------------------------------+----------------------+----------------------+
# | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
# | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
# |   0  NVIDIA GeForce RTX 4090    On   | 00000000:01:00.0  On |                  N/A |
# | N/A   45C    P8    18W / 450W |   1280MiB / 24564MiB |      2%      Default |
# +-------------------------------+----------------------+----------------------+
```

### Apple Siliconの確認

```bash
# M1/M2/M3チップの確認
system_profiler SPHardwareDataType | grep "Chip"

# Metalが利用可能か確認
system_profiler SPDisplaysDataType | grep "Metal"

# Pythonから確認
import torch
print(torch.cuda.is_available())  # False (Apple Silicon)
print(torch.backends.mps.is_available())  # True (Apple Silicon)
```

### ROCm (AMD GPU)の確認

```bash
# ROCmバージョン確認
rocm-smi

# Pythonから確認
import torch
print(torch.hip.is_available())  # AMD GPU
```

## OllamaでのGPU利用設定

### Linux/NVIDIAの場合

```bash
# NVIDIAドライバーとCUDA Toolkitのインストール
sudo apt install nvidia-driver-535
sudo apt install cuda-toolkit-12-2

# GPUメモリをOllamaに割り当てる
ollama serve # 自動的にGPUを検出して利用
```

### GPUメモリのカスタマイズ

```python
import ollama

# GPUに明示的に割り当てる
ollama.generate(
    model="llama3.1",
    prompt="Pythonについて教えてください",
    options={
        "num_gpu": -1,      # -1で全レイヤーをGPUに
        "num_ctx": 8192,    # コンテキストウィンドウ
        "num_thread": 8,    # CPUスレッド数（GPU並列とは別）
    }
)
```

### Apple Silicon (Metal) の利用

```python
# macOSでMetal経由でGPU利用
import ollama

response = ollama.generate(
    model="llama3.1",
    prompt="Pythonについて教えてください"
)
# Metalが自動的に検出され、GPUで動作する
```

## 量化（Quantization）の詳細

### 量子化の仕組み

量子化は、32ビット浮動小数点数を16ビット・8ビットなどに縮小する技術です。精度はかなり保たれたまま、メモリ使用量を減らせます。

### 主要な量子化レベル

```
量子化レベル     精度       モデルサイズ(7B)     品質低下
Q0_0            4bit        ~4GB              ほぼなし
Q3_K_M          3.3bit      ~3.5GB            軽微
Q4_K_M          4.5bit      ~4.2GB            なし
Q5_K_S          5.0bit      ~4.8GB            なし
Q5_K_M          5.2bit      ~4.8GB            なし
Q6_K            6.0bit      ~5.5GB            なし
Q8_0            8bit        ~7.0GB            なし
F16             16bit       ~14GB             なし
F32             32bit       ~28GB             なし
```

### 量子化の判断基準

```
メモリが8GB以下 → Q4_K_M が推奨
メモリが16GB  → Q6_K が適切
メモリが32GB  → Q8_0 または F16
メモリが64GB以上 → F32 (フル精度)
```

### Ollamaでの量子化モデルの選択

```bash
# 量子化モデルを一覧
ollama list | grep llama3.1

# 例:
# llama3.1:8b         16GB (F16)
# llama3.1:8b-q4_0      4GB (Q4_0)
# llama3.1:8b-q6_K      5.5GB (Q6_K)
# llama3.1:8b-q8_0      7GB (Q8_0)
```

### ベンチマーク

```python
import ollama
import time

def benchmark_model(model_name, prompt, iterations=5):
    """モデルの推論速度をベンチマーク"""
    durations = []
    
    for i in range(iterations):
        start = time.time()
        response = ollama.generate(
            model=model_name,
            prompt=prompt,
            options={"stream": False}
        )
        end = time.time()
        durations.append(end - start)
    
    avg_duration = sum(durations) / len(durations)
    avg_tokens = len(response['response']) / avg_duration  # tok/sec
    
    print(f"モデル: {model_name}")
    print(f"平均処理時間: {avg_duration:.2f}秒")
    print(f"トークン/sec: {avg_tokens:.1f}")
    print(f"メモリ使用量: {response.get('total_size', 'N/A')} bytes")
    return avg_duration

# 比較ベンチマーク
benchmark_model("llama3.1:8b-q4_0", "Pythonの利点について100字で説明してください")
benchmark_model("llama3.1:8b-q6_K", "Pythonの利点について100字で説明してください")
benchmark_model("llama3.1:8b", "Pythonの利点について100字で説明してください")
```

## vLLMの基礎

### vLLMとは

vLLMは、高速なLLM推論のためのオープンソースフレームワークです。PagedAttentionという技術でGPUメモリを効率的に管理し、スループットを大幅に向上させます。

### vLLMのインストール

```bash
pip install vllm
```

### vLLMでの推論

```python
from vllm import LLM, SamplingParams

# モデルの初期化（初回のみ数分待つ）
llm = LLM(
    model="meta-llama/Llama-3.2-3B",
    gpu_memory_utilization=0.8,  # GPUメモリの80%を使用
    tensor_parallel_size=1,       # GPUの数
)

# サンプリングパラメータ
sampling_params = SamplingParams(
    temperature=0.7,
    top_p=0.9,
    max_tokens=256,
)

# 推実行
inputs = ["Pythonの利点について"]
outputs = llm.generate(inputs, sampling_params)

for output in outputs:
    print(output.outputs[0].text)
```

## パフォーマンス最適化

### 設定による最適化

```python
import ollama

# 最適設定
optimized_response = ollama.generate(
    model="llama3.1",
    prompt="Pythonについて教えてください",
    options={
        "num_gpu": -1,         # 全レイヤーをGPU
        "num_ctx": 8192,       # コンテキストウィンドウ
        "num_batch": 512,      # バッチサイズ
        "num_thread": 8,       # CPUスレッド数
        "num_keep": 0,         # キーのキャッシュ
        "flash_attention": True,  # クラッシュアテンション
        "num_negative": 0,     # ネガティブプロンプト数
        "repeat_penalty": 1.1, # 繰り返しペナルティ
        "temperature": 0.7,    # 温度パラメータ
        "top_k": 40,           # トップ-Kサンプリング
        "top_p": 0.9,          # トップ-Pサンプリング
    }
)
```

### 最適化の優先順位

```
優先度  最適化項目                    効果      難易度
1       GPU活用 (Ollama自動検出)       高        低
2       量化 (Q4_K_M)                 中～高     低
3       num_gpu設定                   高        低
4       num_batch, num_thread         中        中
5       vLLM使用                      高        高
6       TensorRT-LLM併用              高        高
7       Flash Attention               中        中
8       サンプリングパラメータ調整      低        低
```

## まとめ

本章で学んだこと：

- GPU活用で推論速度を数倍～数十倍に向上できる
- 量子化は精度をほぼ保ったままメモリ使用量を減らせる
- vLLMはより高度なGPU最適化を提供する
- 最適な設定はハードウェアと要件次第

本書ではここまでの技術を活用して、最終的にチャットボットを構築します。
