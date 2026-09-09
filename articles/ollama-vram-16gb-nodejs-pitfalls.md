---
title: "VRAMに載らないと13.8倍遅い ─ Ollamaを Docker + Node.js から叩いて踏んだ3つの落とし穴"
emoji: "🐢"
type: "tech"
topics: ["ollama", "llm", "docker", "typescript", "gpu"]
published: false
---

ローカルLLMが遅いとき、まずモデルを疑いがちです。しかし RTX 5060 Ti（VRAM 16GB）で自宅の記事生成システムを1年ほど動かしてみた結果、**速度を決めていたのはモデルの賢さではなく「VRAMに収まるかどうか」という 1bit の条件**でした。

さらに、Ollama を Docker コンテナの Node.js から HTTP で叩く構成では、モデル選定とは別に3つ踏みました。どれも**エラーにならず、静かに壊れる**種類のものです。

この記事は自分の運用リポジトリで実際に踏んで直したものだけを書きます。

## 結論

- 量子化後のモデルファイルサイズが VRAM を1GBでも超えると、一部レイヤーが CPU にオフロードされ、スループットが 1桁落ちる
- `num_ctx` を省略すると、長いシステムプロンプトが**先頭から黙って切り捨てられる**
- `num_thread` を省略すると、CPU オフロード時に全コアが飽和して**PC本体が操作不能**になる
- `stream: false` は Docker のブリッジネットワーク越しだと、長い生成の途中で TCP を切られる

## 測定環境

| 項目 | 値 |
|---|---|
| GPU | NVIDIA GeForce RTX 5060 Ti / VRAM 16311 MiB（約15.9 GiB） |
| ドライバ | 595.79 |
| CPU | Intel Core i5-14600KF（20スレッド） |
| RAM | DDR4 64GB |
| OS | Windows 11 |
| Ollama | 0.33.2（Windows ホスト側で稼働） |
| 呼び出し元 | Docker コンテナ内の Node.js から `http://host.docker.internal:11434` |

Ollama はホストで動かし、アプリだけをコンテナに置いています。GPU をコンテナに渡す必要がなくなるので、Windows ではこの分離のほうが楽でした。

## 本題：16.2 GiB のモデルは 15.9 GiB の VRAM に載らない

手元の2モデルの実サイズは `/api/tags` で確認できます。

```bash
curl -s http://localhost:11434/api/tags | jq -r '.models[] | "\(.name)\t\(.size/1024/1024/1024|floor)GiB\t\(.details.quantization_level)"'
```

```
gemma3:27b     16.2 GiB   Q4_K_M   (27.4B)
gpt-oss:20b    12.85 GiB  MXFP4    (20.9B)
```

VRAM は 15.9 GiB。つまり **gemma3:27b は 0.3 GiB ほど足りません**。パラメータ数は 27.4B と 20.9B で 1.3倍の差しかないのに、この 0.3 GiB が結果を分けます。

同一プロンプト・同一入力（RSSニュースの日本語要約）で測った実測値がこちらです。

| モデル | サイズ | 配置 | スループット |
|---|---|---|---|
| gemma3:27b | 16.2 GiB | 一部レイヤーが CPU へ溢れる | **6.4 tok/s** |
| gpt-oss:20b | 12.85 GiB | GPU 内で完結 | **88.5 tok/s** |

トークン速度で **13.8倍**、同じ処理の実時間で **4.4倍**の差です（実時間の差が小さいのは、プロンプト評価やモデルロードなど生成以外の時間が含まれるため）。4カテゴリ＋統合の全5回で 176秒 で完走しました。

:::message
測定日 2026-09-04。この数値は**この構成・この用途（日本語のニュース要約、数千トークン規模）での1回の比較**です。文脈長・バッチ・量子化方式が変われば比は変わります。「27Bは常に遅い」という話ではありません。
:::

実際に載っているかどうかは `/api/ps` の `size_vram` と `size` を比べると分かります。

```bash
curl -s http://localhost:11434/api/ps | jq '.models[] | {name, size, size_vram}'
```

`size_vram` が `size` より小さければ、その差分が CPU 側に置かれています。ここが一致していない限り、プロンプトを工夫しても速くなりません。

### 運用上どうしたか

全部を軽いモデルに寄せるのではなく、**用途ごとに分けました**。

| 用途 | モデル | 理由 |
|---|---|---|
| 記事本文の生成 | gemma3:27b | 品質優先。VRAM から溢れるのは承知の上で、遅くても許容できるバッチ処理 |
| ニュース要約 | gpt-oss:20b | 5回連続で走らせるので速度が効く。要約品質はこの用途では十分 |
| ツール呼び出し | qwen3:32b | gemma3 系は tool calling 非対応 |

「速いモデルに統一」ではなく「**遅くて困る経路だけ載るモデルにする**」のが、16GB という中途半端な容量では現実的でした。

## 落とし穴1：`num_ctx` 未指定でプロンプトが黙って切られる

Ollama の `/api/generate` は `options.num_ctx` を省略できます。省略すると環境依存のデフォルト（2048〜4096程度）が使われます。

問題は、**入力がそれを超えても例外が飛ばないこと**です。先頭から切り捨てられた上で、残りだけを見た応答が普通に返ってきます。

自分の場合、システムプロンプトに「文体ルール」「禁止表現」「商品指定」を積んでいたので、切られるのはちょうどそのルール部分でした。結果として「なぜか指示を無視した記事が生成される」という症状になり、プロンプトの書き方を疑ってしばらく時間を溶かしました。

```ts
const resp = await fetch(`${OLLAMA_URL}/api/generate`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    model, prompt, system,
    stream: true,
    keep_alive: 0,
    options: {
      num_thread: OLLAMA_NUM_THREAD,
      num_ctx:    OLLAMA_NUM_CTX,   // ← 省略しない
    },
  }),
  signal: AbortSignal.timeout(600_000),
})
```

`num_ctx` を上げると KV キャッシュのぶん VRAM を余分に食う点には注意が要ります。載るギリギリを狙っているときは、`num_ctx` を上げたせいで今度は溢れる、ということが起きます。手元では 8192 にしています。

## 落とし穴2：`num_thread` 未指定で PC が操作不能になる

これは CPU オフロードとセットで効いてきます。

GPU に載りきっているうちは CPU はほぼ使いません。ところが溢れた瞬間、Ollama は CPU 側の計算に使えるスレッドを取りにいきます。`num_thread` を指定しないと利用可能なコアを埋めにいくので、**生成中はマウスカーソルすら引っかかる**状態になりました。

自宅PCは記事生成だけをしているわけではなく、他のコンテナも動いています。20スレッドのうち 12 に制限して、残りを OS と他プロセスに残すようにしました。

```ts
// 27BモデルはVRAM 16GBに収まらず一部レイヤーがCPU実行になる。
// スレッド無制限だと全コア飽和でPCが操作不能になるため上限を設ける
const OLLAMA_NUM_THREAD = Number(process.env.OLLAMA_NUM_THREAD ?? 12)
```

重要なのは、**Ollama を叩く経路が複数あるなら全部に付ける必要がある**ことです。自分は最初、記事生成のモジュールにだけ入れて満足していました。別モジュールから呼んでいる経路が残っていて、そちらが走ったときだけ PC が固まる、という再現性の低いバグになりました。共通のラッパー関数を1つ作って、直接 `fetch` を書かないようにするのが結局早いです。

## 落とし穴3：`stream: false` は Docker 越しに切られる

`stream: false` にすると、生成が終わるまでレスポンスが1バイトも返りません。CPU オフロードで 6.4 tok/s まで落ちている状態で数千トークン生成すると、**数分間まったくパケットが流れない TCP 接続**ができます。

Docker のブリッジネットワークにはアイドル接続を落とす挙動があり、これに引っかかると生成完了直前に接続が切れます。アプリ側からは「たまに失敗する」としか見えません。

`stream: true` にすると、トークンが出るたびに JSON 行が流れるのでアイドルになりません。パースは1行ずつやります。

```ts
const reader = resp.body!.getReader()
const decoder = new TextDecoder()
let result = ''
while (true) {
  const { done, value } = await reader.read()
  if (done) break
  for (const line of decoder.decode(value, { stream: true }).split('\n')) {
    if (!line.trim()) continue
    // { response: "...", done: false } を1行ずつ受け取る
    result += JSON.parse(line).response ?? ''
  }
}
```

ストリーミングが不要でも、**接続を維持する目的で `stream: true` を選ぶ**ことがある、というのがここでの学びでした。

## おまけ：VRAM を空けてから使う

複数モデルを使い分けていると、前の実行で載ったモデルが `keep_alive` の間 VRAM に残ります。次のモデルが載らずに CPU へ溢れる原因になるので、実行前に他モデルを明示的に降ろしています。

```ts
const res = await fetch(`${OLLAMA_URL}/api/ps`)
const data = await res.json()
for (const item of data.models ?? []) {
  const loaded = item.name ?? item.model ?? ''
  if (!loaded || loaded.startsWith(selectedModel.split(':')[0])) continue
  // keep_alive: 0 で即座にアンロード
  await fetch(`${OLLAMA_URL}/api/generate`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ model: loaded, keep_alive: 0 }),
  })
}
```

アンロードは非同期なので、`/api/ps` を再度見て消えるまで待たないと、結局重なった状態でロードが始まります。

## この記事で確認していないこと

- 32GB / 24GB など**他のVRAM容量での比較はしていません**。「16GBならこうなる」という1台ぶんの記録です
- gemma3:27b と gpt-oss:20b の**出力品質を定量評価していません**。日本語要約という自分の用途で許容範囲だった、という主観的判断です
- Docker のアイドル切断について、**タイムアウト値そのものは特定していません**。`stream: true` で症状が消えたところで調査を止めています
- Linux + Docker で GPU をコンテナに直接渡す構成は試していません

## まとめ

VRAM 16GB は「27B クラスがギリギリ載らない」という、いちばん判断が要る容量でした。買う前に見るべきなのはパラメータ数ではなく、**使いたい量子化での実ファイルサイズが VRAM を下回るか**です。`/api/tags` の `size` を見れば買う前でも（同じ量子化のモデルカードから）見積もれます。

そして Ollama を HTTP から叩くなら、`num_ctx` と `num_thread` は省略しないこと。どちらも黙って壊れる側に倒れるので、動いているように見えているうちは気づけません。

---

普段は自宅PCのAI自動化や自作PCまわりのことを note に書いています → https://note.com/lush_kudu2534
