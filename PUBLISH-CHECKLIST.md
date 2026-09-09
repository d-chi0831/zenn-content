# 公開前チェックリスト

対象: `articles/ollama-vram-16gb-nodejs-pitfalls.md`
作成日: 2026-09-05 / 現在 `published: false`

**公開操作はDaichiが行う。Claudeはドラフト作成までで停止している。**

---

## A. 数値の再確認（必須）

記事中の数値は出所が2種類ある。**②は再測定していないので、公開前に確認すること。**

### ① 2026-09-05 にこのPCで実測して確認済み

| 記載値 | 確認方法 |
|---|---|
| RTX 5060 Ti / VRAM 16311 MiB / driver 595.79 | `nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv` |
| Ollama 0.33.2 | `curl -s http://localhost:11434/api/version` |
| gemma3:27b = 16.2 GiB / Q4_K_M / 27.4B | `/api/tags` の `size` |
| gpt-oss:20b = 12.85 GiB / MXFP4 / 20.9B | 同上 |

### ② コミットメッセージ由来。**未再測定**

`383fccb`（2026-09-04）の記述をそのまま使っている。

| 記載値 | 状態 |
|---|---|
| gemma3:27b = 6.4 tok/s | ⚠️ 再測定していない |
| gpt-oss:20b = 88.5 tok/s | ⚠️ 再測定していない |
| トークン速度13.8倍 / 実時間4.4倍 | ⚠️ 上記から算出 |
| 全5回 176秒 | ⚠️ 再測定していない |

> Zennガイドライン「内容の正確性を確認せずに投稿することは控えてください」に該当する。
> **測定当日の記録を信頼して出すか、1回だけ再測定するかを決めてから公開すること。**

再測定するなら:

```bash
curl -s http://localhost:11434/api/generate -d '{"model":"gpt-oss:20b","prompt":"日本語で300字の要約テストを書いてください","stream":false,"options":{"num_ctx":8192,"num_thread":12}}' | python -c "import json,sys; d=json.load(sys.stdin); print(d['eval_count']/(d['eval_duration']/1e9), 'tok/s')"
```

モデル名を `gemma3:27b` に変えて同じプロンプトでもう一度。`keep_alive:0` で前のモデルを降ろしてから測ること。

## B. Zenn規約との突き合わせ

| 項目 | 根拠 | 状態 |
|---|---|---|
| 自動生成文章の投稿（利用規約 第4条11号） | 全文を手書き。gemma3等の生成物を含まない | ✅ |
| 同一内容の繰り返し投稿（同 11号） | note既存130本と本文重複なし。VRAM関連のnote記事は購入判断・体験談で、本記事は実装の落とし穴 | ✅ |
| 広告・採用が主目的（同 15号） | 楽天アフィリエイト／ROOM／KDPリンクを一切入れていない | ✅ |
| 宣伝は記事末尾の固定メッセージ程度（ガイドライン） | 末尾にnoteへのリンク1行のみ | ✅ |
| 誇張したタイトル（ガイドライン） | 「13.8倍」は測定条件を本文で明示。要②の確認 | ⚠️ ②次第 |

## C. 公開手順（GitHub連携の場合）

1. Zennアカウント作成（**Daichiが行う**。GitHubアカウントでログイン可）
2. https://zenn.dev/dashboard/deploys から GitHub リポジトリを連携
   - このディレクトリを新規リポジトリとして push しておく
   - Public / Private どちらでもよい
3. `published: false` のままなので、**push しても公開されない**。Zennのダッシュボードに下書きとして現れる
4. Zenn上でプレビューを確認
5. 問題なければ `published: true` に変えて push → 公開

ローカルプレビュー（Zennに接続せず確認できる。0円）:

```bash
cd C:\Users\daichi\zenn-content && npm install && npx zenn preview
```

http://localhost:8000 で見られる。

## D. 公開後にやること

- [ ] [[revenue-tracking]] の10月記入時に、Zenn記事のPV/いいねを「露出」段階の指標として追加
- [ ] note側に対応記事を書く場合、**同じ本文を使わない**（Zennは実装、noteは判断の経緯）
- [ ] 反応が3件取れたら [[income-task-07-zenn-book]] の DV-009 成功基準（公開前に読みたい反応3件）の材料になるか判定する

## E. やらないこと

- 有料Bookの公開作業（DV-009ゲート未達）
- note既存記事の編集・再公開
- 記事へのアフィリエイトリンク追加
