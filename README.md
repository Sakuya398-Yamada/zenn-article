# zenn-article

Zenn記事のデプロイ用リポジトリ兼、Zenn記事作成用のClaude Codeスキルです。旧名は`zenn-article-skill`です。

## 機能

- Zennのマークダウン記法・frontmatter・ファイル構造に準拠した記事を生成
- `/zenn-article [テーマ]` で呼び出し、または自然言語で自動トリガー
- タイトルに `[生成AI利用]` を自動付与
- 参考文献・リポジトリリンク・AI生成表記を記事末尾に自動挿入

## インストール

ユーザースコープ（全プロジェクト共通）で使う場合:

```bash
git clone https://github.com/Sakuya398-Yamada/zenn-article.git ~/.claude/skills/zenn-article
```

プロジェクトスコープ（特定プロジェクトのみ）で使う場合:

```bash
git clone https://github.com/Sakuya398-Yamada/zenn-article.git .claude/skills/zenn-article
```

## 使い方

Claude Codeで以下のように呼び出せます:

```
/zenn-article React Hooksの基本
```

または自然言語でも自動的にトリガーされます:

```
Zennの記事を書きたい。テーマはDocker入門。
```

## 生成される記事の構成

1. 導入部
2. 本文（`##` から開始）
3. まとめ
4. リポジトリリンク
5. 参考文献
6. AI生成表記

## ファイル構成

```
zenn-article/
├── SKILL.md                     # メインスキル定義
├── README.md
├── articles/                    # 作者のZennの記事
└── references/
    └── zenn-markdown.md         # Zenn Markdown記法リファレンス
```

## Zennとの連携

このリポジトリは、作者のZennのGitHub連携の対象でもあります。`master`ブランチの`articles/`に入れた記事がZennに同期されます。

公開リポジトリなので、`published: false`の下書きも誰でも読めます。公開すると決めた記事だけをpushします。
