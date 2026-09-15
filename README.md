# symbol-index

シンボル索引: `.cljk` / `.cljc` / `.cljs` / `.clj` / `.kotoba` を走査し、top-level
`(def...)` / `(ns...)` form に現れるシンボルから「シンボル → file:line」の索引を
作る。agent がコードを探すとき、full-file read の代わりに snippet を引くための道具。

## 実測 (2026-09-15, Co-Scientist iteration)

- full-file read: 平均 **~6,580 tok/query** (p50 2,212 tok)
- symbol + 前後 3 行 snippet: 平均 **~122 tok/query** → **54 倍削減**
- 最悪ケース: `catalog.cljk` 1.28M tok の full read は context を破綻させる

## 使い方 (kbb / nbb / babashka どれでも)

```bash
# 索引生成 (1 回。対象 tree の規模で数十秒)
bin/symbol-index build

# シンボル部分一致検索 → file:line 一覧
bin/symbol-index find reconcile

# 定義の snippet 表示 (前後 3 行)
bin/symbol-index show kagami.db/apply-pin-advance
```

`build` は `.kotoba-cache/symbol-index.json` (gitignore 推奨) にキャッシュを
落とす。`find` / `show` はキャッシュが無ければ自動で build する。

## 挙動

- 走査対象: `.cljk` `.cljc` `.cljs` `.clj` `.kotoba` (4 MB 未満の file)
- symlink な checkout (west の子 repo 等) も statSync で指先を辿る。
  visited set (realpath) で symlink loop を断つ。
- Node Stats の `isFile`/`isDirectory` は method — `(.-isFile stat)` は cljs では
  関数が返るだけで真偽にならない。この罠の実装例は src の walk を参照。
- 拒否は fail-closed: 無引数 / 未知 subcommand / `find` 欠 term / `show` 不在
  symbol は理由を印字して **exit 2**。hit は exit 0。

## 前提

- 実行 host: `kbb --backend sci` (本命) / `nbb` / babashka。Node の `fs`/`path`
  を require する。JVM 不要。
- superproject (west / submodule) 配下でも、root の `git grep` が子 repo を
  引けない (gitlink) ことの代替として機能する。

## Co-Scientist 手法 (この数値の取り方)

この repo の数値は 2026-09-15 の Co-Scientist iteration (観測 → 仮説 → 実測 →
着地 → 反証) で出した。再現手順:

1. **観測**: agent log (767 API call) を 1 call ごとに token / cache hit /
   upstream latency で集計。file read を伴う query の token 分布を取る。
2. **仮説**: 「code 理解の cost は file 全文 read に支配されている」
3. **実測 (A/B)**:
   - A: 実運用で起きた full-file read の token 数を 1 query ごとに平均
   - B: 同じ symbol を `find` + 前後 3 行 snippet で引いたときの token 数
   - 同一 corpus・同一 query 意図で比較 (p50 も併記)
4. **着地**: 差が測定値として確定してから tool を着地する。推定値は書かない。
5. **反証**: 「入力が無いとき 0 を返す」「実行できないとき pass を返す」
   自己検査を fail-closed で挟む (無引数 / 未知 cmd / 不在 symbol → exit 2)。

実測は iteration ごとに上書きではなく**追記**する。数値の横に測定日を置く。

## 関連

- kotoba-lang/amu — `.kotoba` compiler
- orgs/kotoba-lang/org-babashka-nbb (kbb engine)
