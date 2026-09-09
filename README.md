# zenn-content

Zenn記事のソース。**AI・PC領域（note `lush_kudu2534`）の露出装置**として使う。

方針の正本: Obsidian `Projects/zenn-free-article-exposure.md`

## いまの状態

| 記事 | 状態 |
|---|---|
| `articles/ollama-vram-16gb-nodejs-pitfalls.md` | `published: false`（未公開） |

**公開はDaichiが行う。** 公開前に `PUBLISH-CHECKLIST.md` を通すこと。

## ローカルプレビュー（Zennに接続しない・0円）

```bash
npm install
npx zenn preview
```

http://localhost:8000

## 書くときのルール

1. **本文をAIに生成させない。** Zenn利用規約 第4条11号「機械により自動生成された文章の投稿」は禁止行為。noteの自動生成記事をここへ持ってこない
2. **アフィリエイト・ROOM・KDPリンクを入れない。** 同15号「広告または採用を主な目的としたコンテンツ」。宣伝は末尾のnoteリンク1行まで
3. **素材はリポジトリの一次情報から取る。** コード・設定・コミットの実測値。note記事からではない
4. **数値には測定日と条件を書く。** 再測定していない値はチェックリストに明記する
5. **noteと本文を重ねない。** Zennは外部canonicalに非対応なので、重複回避は内容を分けるしかない。Zenn＝実装、note＝判断の経緯
6. **月1〜2本まで。** ガイドラインが「乱造」を避けるよう求めている。note 56本で成果0だった事実もある

## frontmatter

```yaml
---
title: ""        # 記事タイトル
emoji: "😸"      # 1文字
type: "tech"     # tech | idea
topics: []       # 最大5個
published: false # 公開設定。falseなら push しても公開されない
---
```

ファイル名（slug）は英小文字・数字・ハイフン・アンダースコアで12〜50文字。

## GitHub連携

1. このディレクトリをリポジトリとして push（Public/Privateどちらでも可）
2. https://zenn.dev/dashboard/deploys で連携（最大2リポジトリ）
3. 登録ブランチへの push で自動デプロイ。`published: false` の記事は下書き扱い
4. 記事の削除はZennダッシュボードからのみ

**note-poster リポジトリとは連携しないこと**（Cookie等を含むローカル専用リポジトリのため）。
