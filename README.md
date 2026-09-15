# symbol-index

シンボル索引: `.cljk` / `.cljc` / `.cljs` / `.clj` / `.kotoba` を走査し、top-level
`(def...)` / `(ns...)` form に現れるシンボルから「シンボル → file:line」の索引を
作る。agent がコードを探すとき、full-file read の代わりに snippet を引くための道具。

## agent はこう使う (思考を減らす順)

コードを読む前に、この 3 つで済むか試す。全部 1 s 以内・数百 tok。

```
find <term>          # どこに在るか。1 hit なら snippet まで出る (往復 1 回)
outline <file|ns>    # file を Read する前の地図: 定義の行番号一覧 (実測 18 倍軽い)
show <ns/sym>        # 定義の前後 3 行。ns で曖昧性を解く
```

exit code で判断する: **0 = hit / 1 = 測って 0 件 / 2 = 拒否** (理由は `REFUSE\t…` 行)。
2 が出たら索引の問題 (top 違い・空・旧形式) —— `status` で素性を見て `build`。
`find` が 40 件超なら term を長くする。`~` 付きは reader-cond / indent 内の nested 定義。

## 実測 (append-only、正本は `measurements.edn`)

### iteration 1 (2026-09-15)

- full-file read: 平均 **~6,580 tok/query** (p50 2,212 tok)
- symbol + 前後 3 行 snippet: 平均 **~122 tok/query** → **54 倍削減**
- 最悪ケース: `catalog.cljk` 1.28M tok の full read は context を破綻させる

### iteration 2 (2026-09-15)

superproject root (orgs/ 含む、51,594 file / 612,791 symbol) で:

| 仮説 | before | after |
|---|---|---|
| find / show の wall time | 5.85 s / 5.18 s (108 MB JSON + js->clj) | **0.76 s / 0.64 s** (TSV + native RegExp、kbb 起動 0.49 s 込み) |
| 正本 path が索引から消える | `apply-pin-advance` が `worktrees/…` 側 1 件のみ | `orgs/kotoba-lang/kagami/src/kagami/db.cljk:81` |
| metadata を名前に取る | `^:private` 等 38,894 件 (5%) | 0 件 |
| linked worktree / 生成物 dir の重複 | worktrees/ 305,436 + runs/ 32,025 symbol | skip 34 worktree + 22 gitignored dir |
| file の地図 | Read 7,611 chars | `outline kagami.db` 423 chars (**18 倍**) |
| 空索引 / 空 tree | exit 0 で「0 hits」 | REFUSE exit 2 |
| test | 無し | 32 check、壊したコピー 3 種で赤を確認 |

build は 105 s (sys 37 s = stat × 5 万)。→ iteration 3 で対処。

### iteration 3 (2026-09-15)

| 仮説 | before | after |
|---|---|---|
| 読めない dir 1 つで build が落ちる (ENAMETOOLONG / EACCES) | crash | `UNREADABLE-DIRS\tn` に数えて続行 |
| walk が entry ごとに stat + realpath | 67.2 s (walk のみ) | **46.1 s** (dirent、stat は symlink だけ) |
| scan が SCI の per-line loop | 1,712 ms / 5,000 file | **387 ms** (native `gm` RegExp) |
| 複数行 `^{:doc …}` の def | 取れない | 5,018 件を新たに索引 (ns 形式も) |
| build 全体 (root corpus、A B A B 交互) | 134.1 s / 149.2 s (load 50 / 25) | **107.7 s / 121.1 s** (load 24 / 28、各周 −20%) |

### iteration 4 (2026-09-15)

観測: 136,302 dir のうち **85% (116,269) は source を 1 つも持たない subtree**
(takeout / news content / onedrive archive / .venv)。名前では分けられない。

| 仮説 | before | after |
|---|---|---|
| 連続 fileless dir 数で data subtree を prune (budget 1000、校正: source を持つ group の最長 run 737) | 全 dir を歩く | 11 group を prune、**失った file 0** (row 集合が同一) |
| build 全体 (no-prune / prune 交互、load 33–99) | 200.5 s / 179.4 s | **161.9 s / 132.4 s** (−19% / −26%) |
| walk が実は DFS だった (`(rest dirs)` への conj) | 校正と実装が不一致 | BFS に固定 |
| virtualenv | 2,776 dir を歩く | `pyvenv.cfg` で skip (8 個) |

prune した group は `status` の `pruned-group` 行に必ず出る。そこにある code を探すなら
`build --no-prune`。

## 使い方 (kbb / nbb / babashka どれでも)

```bash
# 索引生成 (1 回。superproject 全体で ~105 s、単 repo なら 1 s)
bin/symbol-index build [--include-worktrees] [--no-prune]

# シンボル部分一致検索 (大小無視) → file:line 一覧、exact > prefix > substring
bin/symbol-index find reconcile

# 定義の snippet 表示 (前後 3 行)。ns/ で曖昧性を解く
bin/symbol-index show kagami.db/apply-pin-advance

# file (path 末尾一致) か ns の定義一覧
bin/symbol-index outline kagami/db.cljk
bin/symbol-index outline kagami.db

# 索引の素性 (top / built-at / scanned / symbols / age)
bin/symbol-index status
```

`build` は `.kotoba-cache/symbol-index.tsv` (v2、gitignore 推奨) に落とす。
`find` / `show` / `outline` は**索引が無ければ REFUSE する** (exit 2、自動 build
しない —— iteration 3 で Hermes Agent に導入したとき、agent の terminal が `$HOME`
に居る状態で呼ばれ、home 全体を build しに行って tool の 120s timeout に当たった
実測がある)。build は明示の `build` だけ。

top の決め方 (優先順): `SYMBOL_INDEX_ROOT` > `CLAUDE_PROJECT_DIR` > cwd から上へ
辿って `.kotoba-cache/symbol-index.tsv` を持つ最寄りの dir > cwd。project の
subdir から呼んでも root の索引を引く。root の外 (例: `$HOME`) から呼ぶときは
`SYMBOL_INDEX_ROOT=/path/to/root` を付ける。いずれも realpath に揃えてから
索引 header の top と比べる (macOS の `/var` → `/private/var` で不一致になった)。
索引の top と違う場所から引くと REFUSE する (別 tree の答えを返さない)。

`bin/symbol-index` は symlink 越し (`ln -s .../bin/symbol-index ~/.local/bin/`)
でも自分の実体を辿って `scripts/` を見つける (`BASH_SOURCE[0]` は link 側なので、
辿らないと `~/.local/scripts/` を探して ENOENT になった)。

## Hermes Agent への導入 (iteration 5)

skill は `~/.hermes/skills/software-development/symbol-index/SKILL.md` (user-local)、
実行体は `~/.local/share/kotoba-lang/symbol-index` を `~/.local/bin/symbol-index` に
symlink。A/B の実測は `bench/hermes_ab.cljk` と `measurements.edn` の `:iteration 5`:
skill を**先読み** (`hermes chat -s symbol-index` / cron job の `skills:`) すると
8/8 正解・token 2.7 分の 1・API call 2.5 分の 1・壁時計 10 分の 1。**置くだけ**では
agent が選ぶのは 2/8 —— 導入 = 先読みまで。

## 自己検査

```bash
kbb --backend sci test/symbol_index_test.cljk    # CHECKS<TAB>n / FAILED<TAB>n
```

fixture tree を tmp に作り、script を子プロセスで実行して exit code と `REFUSE`
の文言を pin する。8 問 (verification-discipline) の Q1/Q2/Q4/Q5/Q6/Q7 に対応。

## 挙動

- 走査対象: `.cljk` `.cljc` `.cljs` `.clj` `.kotoba` (4 MB 未満の file)
- symlink な checkout (west の子 repo 等) も statSync で指先を辿る。
  visited set (realpath) で symlink loop を断つ。
- Node Stats の `isFile`/`isDirectory` は method — `(.-isFile stat)` は cljs では
  関数が返るだけで真偽にならない。この罠の実装例は src の walk を参照。
- 走査しないもの: linked worktree (`.git` file が `/worktrees/` を指す dir)、
  top 直下で gitignore され `.git` を持たない dir (生成物)。名前でなく構造で判定。
  symlink は realpath で 1 回だけ数え、path も realpath 相対で報告する。
- 索引する形: 行頭の def form + 名前の前の metadata 読み飛ばし (複数行の
  `^{:doc …}` も) + 行頭 `#?(:clj (defn` + indent ≤3 (nested、`~` 印)。indent 4+ は
  取らない。行番号は `(def` のある行。
- 読めない dir (権限 / 異常な名前) は crash せず `UNREADABLE-DIRS` に数える。
- 走査は BFS。group (repo 直下 2 段 / repo 外は 4 段) 内で source を持たない dir が
  1,000 個続いたらその group の残りを歩かず、`PRUNED` と header `pruned-groups` に
  名前を残す (`status` で見える)。`pyvenv.cfg` を持つ dir は virtualenv として歩かない。
- 拒否は fail-closed: 無引数 / 未知 subcommand / 欠 term / 不在 symbol / 索引の
  top 不一致 / 空索引 / 旧 JSON 形式は `REFUSE\t<理由>` を印字して **exit 2**。
  hit は exit 0、測って 0 件は exit 1。

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
正本は `measurements.edn` (1 entry = 1 仮説、`:before` / `:after` / `:verdict`)。

iteration 2 で足した反証の型: **壊したコピーで test が落ちることを確かめてから
landed とする**。落ちない test は劇場。壊した経路の check だけが落ちることまで見る。

## 関連

- kotoba-lang/amu — `.kotoba` compiler
- orgs/kotoba-lang/org-babashka-nbb (kbb engine)
