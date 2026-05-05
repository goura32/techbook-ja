# 第2章 環境構築

## Python開発環境の準備

本章では、ローカルLLMを動かすためのPython開発環境を整えます。

### Pythonのインストール

#### macOSの場合

```bash
# Homebrewをインストール（まだの場合は）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Python 3.11をインストール
brew install python@3.11

# バージョン確認
python3.11 --version
```

#### Ubuntu/Debianの場合

```bash
# Python 3.11のインストール
sudo apt update
sudo apt install python3.11 python3.11-venv python3.11-dev

# バージョン確認
python3.11 --version
```

#### Windowsの場合

1. python.orgからPython 3.11をダウンロード
2. インストーラ実行時に「Add Python to PATH」にチェック
3. コマンドプロンプトを開いて確認
```cmd
python --version
```

### 仮想環境の作成

Pythonプロジェクトでは、**仮想環境**を使うのがベストプラクティスです。各プロジェクトごとに独立したパッケージ環境を作れます。

```bash
# プロジェクトフォルダの作成
mkdir local-llm-demo
cd local-llm-demo

# 仮想環境を作成
python3.11 -m venv llm-env

# 仮想環境をアクティベート
# macOS/Linuxの場合
source llm-env/bin/activate

# Windowsの場合
llm-env\Scripts\activate

# 仮想環境がアクティブになっていることを確認
# プロンプトの前に(llm-env)と表示される
which python
(Windowsの場合はwhere python)
```

### 必要なパッケージのインストール

```bash
# pipを更新
pip install --upgrade pip

# 主要パッケージの一括インストール
pip install requests
pip install ollama
pip install langchain-community
pip install langchain
pip install sentence-transformers
pip install chromadb
pip install unstructured
pip install rich
```

### コマンドラインツール

```bash
# Ollamaのインストール（macOS）
curl -fsSL https://ollama.com/install.sh | sh

# Ollamaのインストール（Ubuntu）
curl -fsSL https://ollama.com/install.sh | sh

# Ollamaのバージョン確認
ollama --version
```

## 動作確認

環境構築が完了したら、実際にテストコードを実行して動作確認します。

### テストスクリプト

以下のファイルを新規作成してください：

```python
# test_environment.py
import sys
import subprocess
import requests

def check_python_version():
    """Pythonバージョンの確認"""
    version = sys.version_info
    print(f"Pythonバージョン: {version.major}.{version.minor}")
    assert version.major == 3, "Python 3が必要です"
    assert version.minor >= 10, "Python 3.10以上が必要です"
    print("✓ Pythonバージョン OK")

def check_packages():
    """必要なパッケージがインストールされているか確認"""
    packages = {
        'requests': 'requests',
        'ollama': 'ollama',
        'langchain': 'langchain',
        'langchain_community': 'langchain_community',
    }
    for name, import_name in packages.items():
        try:
            __import__(import_name)
            print(f"✓ {name} インストール済み")
        except ImportError:
            print(f"✗ {name} がインストールされていません")
            print(f"  実行: pip install {name}")

def check_ollama():
    """Ollamaが起動しているか確認"""
    try:
        response = requests.get("http://localhost:11434/api/tags")
        if response.status_code == 200:
            models = response.json().get("models", [])
            if models:
                print(f"✓ Ollama接続OK（{len(models)}個のモデルあり）")
            else:
                print("✓ Ollama接続OK（モデルなし）")
        else:
            print("✗ Ollamaに接続できません。ollama serveを確認してください")
    except requests.exceptions.ConnectionError:
        print("✗ Ollamaが起動していません。以下のコマンドで起動: ollama serve")

if __name__ == "__main__":
    check_python_version()
    print("---")
    check_packages()
    print("---")
    check_ollama()
```

### 実行と確認

```bash
# 仮想環境をアクティベート（まだの場合は）
source llm-env/bin/activate

# テストスクリプトを実行
python test_environment.py
```

期待される出力：
```
Pythonバージョン: 3.11
✓ Pythonバージョン OK
---
✓ requests インストール済み
✓ ollama インストール済み
✓ langchain インストール済み
✓ langchain_community インストール済み
---
✓ Ollama接続OK（0個のモデルあり）
```

##トラブルシューティング

### よくあるエラー

**Q: pip install がエラーになる**
```bash
# パッケージのバージョンを固定してインストール
pip install --no-cache-dir requests ollama
```

**Q: Pythonのバージョンが古い**
```bash
# バージョンを確認
python --version

# 新しいバージョンをインストール（Ubuntuの場合）
sudo apt install software-properties-common
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.11
```

**Q: Ollamaが起動しない**
```bash
# Ollamaのステータス確認
ollama --version

# サービス再起動
sudo systemctl restart ollama

# または直接起動
ollama serve
```

**Q: GPUが使えない**
- CUDA 12に対応したNVIDIA GPUが必要です
- AMDのGPUには ROCm を使用
- Apple siliconはMetal経由で最適化

## まとめ

本章で学んだこと：

- Python 3.10以上のインストール
- 仮想環境の作成とアクティベーション
- 主要パッケージのインストール
- Ollamaのインストールと動作確認

次章では、インストールしたOllamaを使ってLLMを動かす方法を学びます。


### Virtualenvの作成とアクティベーション

物理的なパッケージの衝突を避けるため、仮想環境の利用を強く推奨します。仮想環境はPythonの環境を隔離し、プロジェクトごとに独立した依存関係を維持できます。

```bash
# 仮想環境の作成
cd ~/projects/my-llm-app
python3.11 -m venv .venv

# macOS/Linux で仮想環境を有効化
source .venv/bin/activate

# Windows の場合
.venv\Scripts\activate

# パッケージのインストール
pip install ollama langchain pydantic

# 仮想環境から退出
deactivate
```

### pipパッケージの管理

```bash
# ライフサイクルの管理
pip install --upgrade pip setuptools wheel

# 依存関係の固定
pip freeze > requirements.txt

# 必要パッケージの確認
pip list --outdated

# 特定のバージョンのインストール
pip install langchain==0.2.0
```

### GPU環境のセットアップ

GPUで高速化するには、CUDAツールチェーンが必要です。

#### Ubuntu 22.04 でのCUDAインストール

```bash
# NVIDIAドライバの確認
nvidia-smi

# CUDAツールキットのインストール（例: CUDA 12.2）
wget https://developer.download.nvidia.com/compute/cuda/12.2.0/local_installers/cuda_12.2.0_535.54.03_linux.run
sudo sh cuda_12.2.0_535.54.03_linux.run
```

#### NVIDIAドライバのバージョン管理

```bash
# インストール可能なドライバの一覧
apt-cache policy nvidia-driver-550

# ドライバのアンインストール
sudo apt remove --purge nvidia-driver-*
sudo apt autoremove
```

### Dockerでの環境構築

Dockerを使えば、複雑な依存関係もコンテナの中で管理できます。

```dockerfile
FROM python:3.11-slim

WORKDIR /app
RUN apt-get update && apt-get install -y     build-essential     && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
CMD ["python", "main.py"]
```

### 環境の検証

```bash
# 全パッケージのバージョン確認
pip freeze | grep -E "ollama|langchain|torch|cuda"

# GPUの可用性確認
python3 -c "import torch; print(f'CUDA: {torch.cuda.is_available()}')"

# Ollamaのコネクション確認
curl -s http://localhost:11434/api/tags | python3 -m json.tool
```
