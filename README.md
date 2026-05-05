# techbook-ja

日本語技術書の収集・執筆・公開を行うプロジェクト。

## リポジトリ構成

```
.
├── books/              # 技術書プロジェクト一覧
│   ├── python-local-llm-guide/    # Pythonで学ぶローカルLLM構築入門（11章）
│   ├── rust-system-programming/   # Rustシステムプログラミング入門（9章）
│   └── kubernetes-linux-system-design/ # Kubernetes × Linuxシステム設計（10章）
├── keywords/              # キーワード管理（seed_keywords.yaml, all_keywords.yaml）
├── progress/              # 進捗ダッシュボード（dashboard.json）
├── skills/               # 再利用スキル
└── references/            # リファレンス資料
```

## 登録済みの技術書

| 書籍 | 章数 | ステータス |
|------|------|------------|
| Pythonで学ぶローカルLLM構築入門 | 11章 | completed |
| Rustシステムプログラミング入門 | 9章 | completed |
| Kubernetes × Linux システム設計 | 10章 | completed |

各書籍の詳細は `books/<book_id>/README.md` を参照。

## 進捗確認

```bash
cat progress/dashboard.json
```

## LICENSE

MIT License
