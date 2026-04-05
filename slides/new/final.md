# 1. Rubyブロックの3つの性質

---

## おさらいです

8章で学んだブロックの3つの性質を確認します。

この3つが、この後の内部実装の話につながります。

---

## 性質1: 外側変数アクセス

ブロックは、定義されたスコープの変数を自由に参照・変更できる。

```ruby
counter = 0
3.times { counter += 1 }
puts counter  # => 3
```

ブロックの外にある `counter` を、ブロックの中から変更できている。

---

## 性質2: メソッド終了後も環境保持

メソッドが終了しても、ブロックは当時の変数を覚えている。

```ruby
def create_counter
  count = 0
  lambda { count += 1 }
end

counter = create_counter  # メソッドはここで終了
puts counter.call         # => 1  それでも count にアクセスできる
puts counter.call         # => 2
```

`create_counter` が終了した後も `count` が生き続けている。

---

## 性質3: 環境の共有

同じスコープで作られた複数のブロックは、同じ変数を共有する。

```ruby
def create_counter
  count = 0
  increment = lambda { count += 1 }
  read      = lambda { count }
  [increment, read]
end

inc, read = create_counter
inc.call
inc.call
puts read.call  # => 2  別々のlambdaが同じ count を共有
```

`increment` と `read` は別々のlambdaだが、同じ `count` を見ている。

---

## まとめ

| 性質 | 一言で |
|---|---|
| 外側変数アクセス | 定義時のスコープの変数を使える |
| 環境保持 | メソッドが終わっても変数が残る |
| 環境の共有 | 複数のブロックで同じ変数を共有できる |

この3つが揃ったものを、コンピュータサイエンスでは**クロージャ**と呼びます。

---

# 2. ブロック = クロージャ

---

## クロージャとは

コンピュータサイエンスにおけるクロージャの定義：

> **コード + 定義時の環境への参照** を一体化したもの

- 「コード」= 実行する処理（`count += 1` など）
- 「環境への参照」= 定義時のスコープにある変数への参照

---

## Rubyブロックと照らし合わせる

さっきの3つの性質をクロージャの定義に当てはめると：

| 性質 | クロージャの定義との対応 |
|---|---|
| 外側変数アクセス | 環境への参照を持っているから |
| 環境保持 | 環境への参照を持ち続けるから |
| 環境の共有 | 同じ環境への参照を持つから |

3つ全部、「環境への参照を持つ」という1つの事実から説明できる。

**Rubyのブロック = クロージャ** そのものです。

---

## 少しだけ歴史

- **1964年**: Peter J. Landin がクロージャの概念を考案
- **1975年**: Scheme で実用化
- **1993年**: Ruby に導入

60年以上の歴史を持つ概念が、日常のRubyコードの中にある。

---

## では、内部でどう実現しているのか？

```ruby
def create_counter
  count = 0
  lambda { count += 1 }  # メソッドが終わっても count が残るのはなぜ？
end
```

「環境への参照」と言葉では言えるが、Rubyのソースコードレベルでは何が起きているのか。

次のセクションから、実際のRuby 3.4のソースコードで見ていきます。

---

# 3. 内部構造の概要

---

## ブロックが持つべきもの

セクション2でブロック = クロージャと確認しました。

クロージャの定義は「**コード + 環境への参照**」。

ということは、Rubyの内部実装もこの2つを持っている構造体が必要なはずです。

- **コード** → 実行する命令列（`count += 1` のバイトコード）
- **環境への参照** → 定義時のスコープにある変数の場所

---

## rb_captured_block（クロージャの本体）

コードと環境への参照をそのまま持つ構造体がこれです。
`{ }` を書いた瞬間にVMが作ります。

```c
struct rb_captured_block {
    VALUE self;
    const VALUE *ep;         // 環境ポインタ = 変数の場所への参照
    union {
        const rb_iseq_t *iseq;        // 命令列（普通の { } ブロック）
        const struct vm_ifunc *ifunc; // Cレベルのブロック
        VALUE val;
    } code;                  // 実行するコード
};
```

`ep`（environment pointer）が「環境への参照」の実体です。

---

## rb_block（種類を統一するラッパー）

ブロックの書き方は1種類ではありません。

```ruby
[1,2,3].map { |x| x * 2 }   # 普通のブロック
[1,2,3].map(&:to_s)          # シンボル
[1,2,3].map(&my_proc)        # 既存のProc
```

これらを呼び出す側が毎回種類を気にするのは大変です。
統一して扱えるようにラップしているのが `rb_block` です。

```c
struct rb_block {
    union {
        struct rb_captured_block captured; // 普通の { } ブロック
        VALUE symbol;                      // &:method_name
        VALUE proc;                        // 既存のProc
    } as;
    enum rb_block_type type; // どの種類かを示すタグ
};
```

---

## rb_proc_t（Procオブジェクトの実体）

ブロックを変数に代入したり引数として渡すには、オブジェクトとして扱える必要があります。

```ruby
counter = lambda { count += 1 }  # 変数に代入
```

このオブジェクトの実体が `rb_proc_t` です。`rb_block` を内包しています。

```c
typedef struct {
    const struct rb_block block;
    unsigned int is_from_method: 1;
    unsigned int is_lambda: 1;    // ← これ1つでlambdaかprocかが決まる
    unsigned int is_isolated: 1;  // Ractor用
} rb_proc_t;
```

ひとつ面白い事実：lambdaとprocは**同じ構造体**です。
`is_lambda` が `1` か `0` かだけの違いです。

---

## rb_control_frame_t（実行中のスタックフレーム）

メソッドが実行されている間、ローカル変数はどこにあるのでしょうか。

スタック上の**実行フレーム**に置かれています。
メソッドが呼ばれるたびに作られ、終了すると消えます。

```c
typedef struct rb_control_frame_struct {
    const VALUE *pc;        // 実行位置
    VALUE *sp;              // スタック端
    const rb_iseq_t *_iseq; // バイトコード
    VALUE self;
    const VALUE *ep;        // 環境ポインタ ← ローカル変数の場所を指す
    const void *block_code;
    void *jit_return;
} rb_control_frame_t;
```

ローカル変数は `ep[-1]`, `ep[-2]`... でアクセスします。

---

## rb_env_t（ヒープに逃がした変数領域）

ここで問題が生じます。

```ruby
def create_counter
  count = 0
  lambda { count += 1 }  # count はスタック上にある
end

counter = create_counter  # メソッドが終了 → スタックは消える
counter.call              # でも count にアクセスしたい！
```

スタックが消えても変数を残すには、**ヒープに逃がす**しかありません。
ヒープ上の変数置き場が `rb_env_t` です。

```c
typedef struct {
    VALUE flags;           // GC管理用ヘッダ
    rb_iseq_t *iseq;
    const VALUE *ep;       // env[]内のspecvalスロットを指す
    const VALUE *env;      // ヒープ上のローカル変数配列
    unsigned int env_size;
} rb_env_t;
```

---

## 構造体の関係

```
rb_proc_t                     ← lambda / proc オブジェクト
  ├─ is_lambda                ← 1つのフラグでlambdaかprocかが決まる
  └─ rb_block
       └─ rb_captured_block   ← コード + 環境への参照
            ├─ self
            ├─ ep ──────────────────→ rb_env_t（ヒープ上の変数領域）
            └─ code.iseq                   │
                                           └─ count など

rb_control_frame_t            ← 実行中フレーム
  └─ ep ────────────────────────────→ rb_env_t（同じものを指す）
```

`rb_captured_block.ep` と `rb_control_frame_t.ep` が**同じ `rb_env_t` を指す**。
これが「複数のlambdaが同じ変数を共有できる」理由です。

---

## 誰がいつ登場するか

| タイミング | 何が起きるか |
|---|---|
| メソッド呼び出し | `rb_control_frame_t` が作られる |
| `{ }` を書く | `rb_captured_block` が作られる（ep はスタックを指す） |
| `lambda {}` を実行 | `rb_env_t` がヒープに作られ、ep が付け替えられる |
| `lambda {}` を実行 | `rb_proc_t` が作られ `is_lambda = 1` がセットされる |
| `.call` / `yield` | 新しい `rb_control_frame_t` が作られ、ep が `rb_env_t` を指す |

---

# 4. 環境はなぜ生き続けるのか

---

## 追うコード

このセクションでは2つのケースを順番に追います。

**ケース①: シンプルなブロック（このスライド群）**
```ruby
def greet
  name = "Alice"
  [1, 2, 3].each { |i| puts name }
end
```

**ケース②: lambda（次のスライド群）**
```ruby
def create_counter
  count = 0
  lambda { count += 1 }
end
```

---

# ケース①: シンプルなブロック

---

## 【図1】greet 呼び出し直後

スタック上に cfp（rb_control_frame_t）が作られます。
ローカル変数 `name` はスタック上に置かれ、`ep[-1]` でアクセスします。

```
Stack                          Heap

+------------------------------+
| cfp (greet)                  |
|   self = main                |
|   ep ──────────────────+     |
|   block_code = ...     |     |    (empty)
|                        |     |
|   ep[ 0] = specval  <──+     |
|   ep[-1] = "Alice"           |
+------------------------------+
```

---

## 【図2】ブロックが渡される

`{ |i| puts name }` を書いた時点でVMはブロックの情報を用意します。

**`rb_captured_block` は独立した構造体として作られません。**

なぜか。`cfp` にはすでに `self`・`ep`・`block_code` の3フィールドがあります。
これは `rb_captured_block` が必要とする `{self, ep, code}` と完全に一致しています。
「`cfp` を作った時点でブロックに必要な情報は揃っている」ので、
わざわざ別の構造体を作ってメモリを確保するのは無駄です。

Ruby の開発者もコメントにそれを残しています：

```c
// vm_core.h（Ruby 3.4）
VALUE self;             // cfp[3] / block[0]  <- rb_captured_block の self
const VALUE *ep;        // cfp[4] / block[1]  <- rb_captured_block の ep
const void *block_code; // cfp[5] / block[2]  <- rb_captured_block の code
```

この構造があるため、`rb_captured_block *` から `-3` するだけで `cfp` の先頭に戻れます。

```c
// VM_CAPTURED_BLOCK_TO_CFP
rb_control_frame_t *cfp =
    ((rb_control_frame_t *)((VALUE *)(captured) - 3));
// cfp[3] の位置を指しているので、3 引けば cfp[0] に戻れる
```

`rb_block` や `rb_proc_t` はまだ作られません。
ブロックをオブジェクトとして持ち回る必要がないからです。

---

## ステップ3: yield が呼ばれる

`each` はCで実装されていますが、概念的にはこういうことをしています。

```ruby
# 概念的な疑似コード（実際はC実装）
def each
  i = 0
  while i < size
    yield self[i]  # ← ブロックを呼び出す
    i += 1
  end
end
```

この `yield` はVM命令 `invokeblock` に変換され、以下の流れで処理されます。

```
invokeblock（VM命令）
  |  yield を表す VM 命令
  |
  +-> vm_invokeblock_i
  |     ブロックハンドラー ※1 を取り出し、calling 情報 ※2 を準備する
  |
  +-> vm_invoke_block
  |     ブロックの種類（{} / &:sym / Proc / C関数）で処理を振り分ける
  |
  +-> vm_invoke_iseq_block
  |     {} ブロック専用。rb_captured_block を取り出してフレーム作成へ
  |
  +-> vm_push_frame
        新しい rb_control_frame_t をスタックに積む
```

> **※1 ブロックハンドラー**：「ブロックの種類 + ブロックデータへのポインタ」を1つの値に詰めたもの。
> `{}` ブロックの場合は「iseq型で、データは cfp に埋め込まれた rb_captured_block」
>
> **※2 calling情報（rb_calling_info）**：呼び出しの文脈情報をまとめた構造体。
> 引数の個数・渡されたブロックハンドラー・レシーバーなどを持つ。

---

## 【図3】ブロック用フレームが積まれた直後

`vm_push_frame` の呼び出しで、ブロック用の cfp が新たにスタックに積まれます。

```c
vm_push_frame(ec, iseq,
    VM_FRAME_MAGIC_BLOCK,
    captured->self,
    VM_GUARDED_PREV_EP(captured->ep),  // <- greet の ep[0] アドレスをセット
    ...
);
```

`VM_GUARDED_PREV_EP(captured->ep)` がブロックの `ep[0]` にセットされます。
これでブロックの `ep[0]` が greet の `ep[0]`（specval）を指す、
**epチェーン**が形成されます。

```
Stack                          Heap

+------------------------------+
| cfp (block)                  |
|   self = main                |
|   ep ──────────────────+     |
|                        |     |
|   ep[ 0] = prev_ep  <──+     |    (empty)
|   ep[-1] = i = 1        |    |
+-------------------------|----+
| cfp (each)              |    |
|   ...                   |    |
+-------------------------|----+
| cfp (greet)             |    |
|   self = main           |    |
|   ep ───────────────+   |    |
|                     |   |    |
|   ep[ 0] = specval <+<──+    |
|   ep[-1] = "Alice"           |
+------------------------------+

puts name の実行:
  (1) 自分のローカルに name はない
  (2) ep[0] (prev_ep) を辿って greet のフレームへ
  (3) greet の ep[-1] = "Alice" を読む
```

---

## 【図4】ブロック終了・メソッド終了

ブロックが終わるとブロック用の cfp がスタックから消えます。
`greet` が終わると `greet` の cfp も消えます。

```
Stack                          Heap

(全フレームが消える)            (空のまま)
```

`rb_captured_block` は cfp に埋め込まれていたので、
**cfp が消えると同時に消えます。特別なクリーンアップは不要。**
スタックポインタを動かすだけです。

---

## メモリの流れ（ケース①全体）

```
---------------------------------------------------
(1) greet 呼び出し
---------------------------------------------------

  cfp structs           local var areas

  +--------------+       +--------------------+
  | cfp (greet)  |       | ep[ 0] = specval   |
  |   ep --------+-----> | ep[-1] = "Alice"   |
  +--------------+       +--------------------+


---------------------------------------------------
(2) yield でブロック用フレームが積まれる
---------------------------------------------------

  cfp structs           local var areas

  +--------------+       +--------------------+
  | cfp (block)  |       | ep[ 0] = prev_ep --+--+
  |   ep --------+-----> | ep[-1] = i = 1     |  |
  +--------------+       +--------------------+  |
  | cfp (each)   |                               |
  |   ...        |                               |
  +--------------+       +--------------------+  |
  | cfp (greet)  |       | ep[ 0] = specval <-+--+
  |   ep --------+-----> | ep[-1] = "Alice"   |
  +--------------+       +--------------------+

  puts name:
    block の ep[0] (prev_ep) を辿って greet の ep[-1] = "Alice" を読む


---------------------------------------------------
(3) ブロック終了 -> ブロック用 cfp が pop される
---------------------------------------------------

  cfp structs           local var areas

  +--------------+
  | cfp (each)   |
  |   ...        |
  +--------------+       +--------------------+
  | cfp (greet)  |       | ep[ 0] = specval   |
  |   ep --------+-----> | ep[-1] = "Alice"   |
  +--------------+       +--------------------+


---------------------------------------------------
(4) greet 終了 -> 全 cfp が pop される
---------------------------------------------------

  (empty)               (empty)

  ヒープへのコピーは一度も起きていない
---------------------------------------------------
```

---

## ケース①まとめ

| | シンプルなブロック |
|---|---|
| `rb_captured_block` の場所 | cfp に埋め込み（スタック上） |
| `rb_block` / `rb_proc_t` | 作られない |
| ヒープコピー | 起きない |
| クリーンアップ | cfp と一緒にスタックから消えるだけ |

ブロックがメソッドの外に出ない限り、Rubyはヒープを使わずシンプルに処理します。

では、メソッドの外に出る必要がある場合はどうなるのか。それがケース②です。

---

# ケース②: lambda

---

## 追うコード

```ruby
def create_counter
  count = 0
  lambda { count += 1 }
end

counter = create_counter
counter.call  # => 1
counter.call  # => 2
```

`create_counter` が終わっても `count` にアクセスできる。なぜ？

---

## ステップ1: create_counter 呼び出し

`greet` と同じく、`cfp` とローカル変数領域がスタックに作られます。

```
Stack                          Heap

+------------------------------+
| cfp (create_counter)         |
|   self = main                |
|   ep ──────────────────+     |
|   block_code = nil     |     |    (empty)
|                        |     |
|   ep[ 0] = specval  <──+     |
|   ep[-1] = 0  (count)        |
+------------------------------+
```

ここまではケース①と同じです。

---

## ステップ2: lambda {} を書く

`lambda { count += 1 }` を実行したとき、VMは以下を順に呼び出します。

```
lambda { count += 1 } を実行
  → f_lambda()                         [proc.c]
  → rb_block_lambda()                  [proc.c]
  → proc_new(rb_cProc, TRUE)           [proc.c]  is_lambda=TRUE
  → rb_vm_make_proc_lambda()           [vm.c]    ← ここから重要
```

`rb_vm_make_proc_lambda` の中で、まず確認が走ります。

```c
// vm.c
if (!VM_ENV_ESCAPED_P(captured->ep)) {
    // ep がまだスタックを指している = ヒープコピーが必要
    vm_make_env_object(ec, cfp);
}
```

`VM_ENV_ESCAPED_P` は `ep[0]`（specval）の下位ビットを見るマクロです。
ヒープにコピー済みなら `VM_ENV_FLAG_ESCAPED` が立っているので `true`、
まだスタックにあれば `false` です。

---

## ステップ3: vm_make_env_each がスタック→ヒープへコピーする

`vm_make_env_object` は `vm_make_env_each` を呼び出すラッパーです。
核心は `vm_make_env_each` にあります。

```c
// vm.c（簡略）
static VALUE
vm_make_env_each(const rb_execution_context_t *ec, rb_control_frame_t *cfp)
{
    // 1. ヒープにメモリを確保
    env_body = ALLOC_N(VALUE, env_size);

    // 2. rb_env_t を作成（GCが管理できるオブジェクト）
    rb_env_t *env = IMEMO_NEW(rb_env_t, imemo_env, 0);

    // 3. スタック上の変数をヒープにコピー
    MEMCPY(env_body, ep - (local_size - 1), VALUE, local_size);

    // 4. ep の指す先を付け替える
    env_ep = &env_body[local_size - 1];
    env->ep = env_ep;
    cfp->ep = env_ep;  // cfp の ep もヒープ側に向け直す

    // 5. 「ヒープに逃げた」フラグを立てる
    VM_ENV_FLAGS_SET(env_ep, VM_ENV_FLAG_ESCAPED | VM_ENV_FLAG_WB_REQUIRED);

    return (VALUE)env;
}
```

この時点で `count = 0` はヒープ上の `rb_env_t` の中に移動しています。

---

## 【図2-A】vm_make_env_each 実行後

```
Stack                          Heap

+-----------------+            +--------------------+
| cfp             |            | rb_env_t           |
|   ep ───────────+----------> | ep[ 0] = specval   |
|                 |            | ep[-1] = 0 (count) |
+-----------------+            +--------------------+
```

`cfp->ep` がヒープの `rb_env_t` を指すよう付け替えられました。
スタック上の元の変数領域は、もう参照されません。

---

## ステップ4: rb_proc_t が作られる

`vm_proc_create_from_captured` が `rb_proc_t` を作り、`is_lambda = 1` をセットします。

```c
// vm.c
static VALUE
vm_proc_create_from_captured(...)
{
    rb_proc_t *proc = ...;
    proc->block.as.captured = *captured;   // rb_captured_block をコピー
    proc->is_lambda = is_lambda;           // lambda なら 1
    return proc_obj;
}
```

このとき `captured->ep` はすでにヒープの `rb_env_t` を指しています。

---

## ステップ5: create_counter が return する

```
Stack                          Heap

(cfp が pop される)            +--------------------+
                               | rb_env_t           |
(empty)                        | ep[ 0] = specval   |
                               | ep[-1] = 0 (count) |
                               +--------------------+

                               +--------------------+
                               | rb_proc_t (lambda) |
                               |   ep ──────────────+---> rb_env_t
                               |   is_lambda = 1    |
                               +--------------------+
```

スタックは消えましたが、`rb_env_t` はヒープに残っています。
`rb_proc_t` の `captured.ep` がそれを指しているので、GCも回収しません。

---

## ステップ6: counter.call

`counter.call` は `invokeblock` VM命令を経て `vm_push_frame` に到達します。

```c
vm_push_frame(ec, iseq,
    VM_FRAME_MAGIC_BLOCK,
    captured->self,
    VM_GUARDED_PREV_EP(captured->ep),  // ← ヒープの rb_env_t を指す
    ...
);
```

ケース①と構造は同じです。違いは `captured->ep` が指す先がヒープであること。

---

## 【図3】counter.call 実行中

```
Stack                          Heap

+-----------------+            +--------------------+
| cfp (block)     |            | rb_env_t           |
|   ep ───────────+--+-------> | ep[ 0] = specval   |
|                 |  |         | ep[-1] = 1 (count) | <- count += 1 された
+-----------------+  |         +--------------------+
                     |
                     +-- VM_GUARDED_PREV_EP(captured->ep)
```

ブロックの `ep[0]` は `rb_env_t` 内の specval スロットを指しています。
`count` へのアクセスは `ep[-1]` で行われます。

---

## メモリの流れ（ケース②全体）

```
=====================================================================
(1) create_counter 呼び出し前
    main フレームだけがスタックにある
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+       +------------------------+
  | cfp (main)   |       | ep[ 0] = specval       |
  |   ep --------+-----> | ep[-1] = nil  (counter)|   (empty)
  +--------------+       +------------------------+


=====================================================================
(2) create_counter 呼び出し直後
    新しい cfp と count がスタックに積まれる
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+       +------------------------+
  | cfp(create_  |       | ep[ 0] = specval       |
  |   counter)   |       | ep[-1] = 0    (count)  |
  |   ep --------+-----> +------------------------+
  +--------------+
  | cfp (main)   |       +------------------------+
  |   ep --------+-----> | ep[ 0] = specval       |   (empty)
  +--------------+       | ep[-1] = nil  (counter)|
                         +------------------------+


=====================================================================
(3) lambda {} を実行 ── vm_make_env_each が走る直前
    rb_captured_block が cfp[3..5] に埋め込まれた状態
    ep はまだスタック上の local var area を指している
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+       +------------------------+
  | cfp(create_  |       | ep[ 0] = specval       |
  |   counter)   |       |   VM_ENV_ESCAPED_P=false|
  |   self=main  |       | ep[-1] = 0    (count)  |
  |   ep --------+-----> +------------------------+
  |   block_code |
  +--------------+
  ^~~~~~~~~~~~~~~^
  cfp[3..5] がそのまま rb_captured_block として機能
  (別の構造体は作られない)

  | cfp (main)   |       +------------------------+
  |   ep --------+-----> | ep[ 0] = specval       |   (empty)
  +--------------+       | ep[-1] = nil  (counter)|
                         +------------------------+


=====================================================================
(4) vm_make_env_each 実行後
    スタックの変数がヒープの rb_env_t にコピーされた
    cfp->ep がヒープ側に付け替えられた
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+       (放棄。もう参照されない)    +------------------+
  | cfp(create_  |                                   | rb_env_t         |
  |   counter)   |                                   | ep[ 0] = specval |
  |   ep --------+---------------------------------> |   ESCAPED フラグ |
  +--------------+                                   | ep[-1] = 0(count)|
                                                     +------------------+
  | cfp (main)   |       +------------------------+
  |   ep --------+-----> | ep[ 0] = specval       |
  +--------------+       | ep[-1] = nil  (counter)|
                         +------------------------+


=====================================================================
(5) rb_proc_t 作成後 (vm_proc_create_from_captured)
    lambda オブジェクトがヒープに作られた
    captured->ep はヒープの rb_env_t を指している
=====================================================================

  cfp structs                                          Heap

  +--------------+                                   +------------------+
  | cfp(create_  |                                   | rb_env_t         |
  |   counter)   |                                   | ep[ 0] = specval |
  |   ep --------+---------------------------------> | ep[-1] = 0(count)|
  +--------------+                                   +------------------+
                                                           ^
  | cfp (main)   |       +------------------------+  +----+-------------+
  |   ep --------+-----> | ep[ 0] = specval       |  | rb_proc_t        |
  +--------------+       | ep[-1] = nil  (counter)|  |   ep             |
                         +------------------------+  |   is_lambda = 1  |
                                                     +------------------+


=====================================================================
(6) create_counter return
    cfp(create_counter) が pop される
    戻り値の rb_proc_t が main の counter に代入される
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+       +------------------------+  +------------------+
  | cfp (main)   |       | ep[ 0] = specval       |  | rb_env_t         |
  |   ep --------+-----> | ep[-1] = rb_proc_t  ---+->| ep[ 0] = specval |
  +--------------+       +------------------------+  | ep[-1] = 0(count)|
                         ^                           +------------------+
                         counter に lambda が代入          ^
                                                     +----+-------------+
                                                     | rb_proc_t        |
                                                     |   ep             |
                                                     |   is_lambda = 1  |
                                                     +------------------+

  create_counter のスタックフレームは消えた
  rb_env_t は rb_proc_t から参照されているので GC されない


=====================================================================
(7) counter.call ── ブロック用 cfp がスタックに積まれる
    ep[0] に VM_GUARDED_PREV_EP(captured->ep) がセットされる
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+                                   +------------------+
  | cfp (block)  |                                   | rb_env_t         |
  |   ep --------+---------------------------------> | ep[ 0] = specval |
  +--------------+                                   | ep[-1] = 1(count)| <- count += 1
                                                     +------------------+
  | cfp (main)   |       +------------------------+       ^
  |   ep --------+-----> | ep[ 0] = specval       |  +----+-------------+
  +--------------+       | ep[-1] = rb_proc_t  ---+->| rb_proc_t        |
                         +------------------------+  |   ep             |
                                                     |   is_lambda = 1  |
                                                     +------------------+

  count の読み書きは ep[-1] で直接ヒープの rb_env_t にアクセス


=====================================================================
(8) counter.call 終了
    cfp (block) が pop される
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+       +------------------------+  +------------------+
  | cfp (main)   |       | ep[ 0] = specval       |  | rb_env_t         |
  |   ep --------+-----> | ep[-1] = rb_proc_t  ---+->| ep[ 0] = specval |
  +--------------+       +------------------------+  | ep[-1] = 1(count)|
                                                     +------------------+
                                                           ^
                                                     +----+-------------+
                                                     | rb_proc_t        |
                                                     |   ep             |
                                                     |   is_lambda = 1  |
                                                     +------------------+

  rb_env_t の count は 1 のまま残る
  次の counter.call でも同じ rb_env_t を参照するので count は 2 になる
=====================================================================
```

---

## ケース②まとめ

| | lambda |
|---|---|
| `rb_captured_block` の場所 | cfp に埋め込み（スタック上） |
| `rb_env_t` の作成タイミング | `lambda {}` を実行した瞬間 |
| ヒープコピー | `vm_make_env_each` が実行時に行う |
| `rb_proc_t` | 作られる（`is_lambda = 1`） |
| メソッド終了後 | スタックは消えるが `rb_env_t` はヒープに残る |
| `.call` 時 | ヒープの `rb_env_t` を直接参照する |

ケース①との決定的な違いは **「`lambda {}` を書いた瞬間にヒープコピーが走る」** 点です。
これが「メソッドが終わっても変数が残る」仕組みの正体です。

---

# ケース③: ネストしたブロック

---

## 追うコード

```ruby
def outer
  x = 10
  [1, 2].each { |i|
    @lambdas << lambda { x + i }
  }
end
```

`lambda { x + i }` は **ブロックの中** で作られます。
`x` は `outer` のローカル変数、`i` はブロックのローカル変数。
2つの変数が**別々のフレーム**にある状態で lambda を作るとどうなるか。

---

## epチェーンの復習

ケース①で見たように、ブロックの `ep[0]` は外側フレームの ep スロットを指します。

```
cfp (block for each)
  ep[0] = VM_GUARDED_PREV_EP → outer の ep スロット
  ep[-1] = i = 1

cfp (outer)
  ep[0] = specval（ローカルフレームの目印）
  ep[-1] = x = 10
```

`lambda { x + i }` を実行する時点で、カレントフレームは `cfp (block)` です。
`captured->ep` は `cfp (block)->ep`、つまりブロックのローカル変数領域を指しています。

---

## vm_make_env_each が「再帰的に」走る

ここがケース③の核心です。`vm_make_env_each` の処理を再度見ます。

```c
static VALUE
vm_make_env_each(const rb_execution_context_t *ec, rb_control_frame_t *cfp)
{
    if (VM_ENV_ESCAPED_P(ep)) return VM_ENV_ENVVAL(ep); // すでにヒープ済みなら即return

    // ep[0] が「ローカルフレームの目印」でなければ、外側フレームが存在する
    if (!VM_ENV_LOCAL_P(ep)) {
        // ① まず外側フレームを再帰的に処理する
        vm_make_env_each(ec, prev_cfp);

        // ② 内側の ep[0] を、外側の「新しいヒープ ep」に更新する
        VM_FORCE_WRITE_SPECIAL_CONST(
            &ep[VM_ENV_DATA_INDEX_SPECVAL],
            VM_GUARDED_PREV_EP(prev_cfp->ep)  // ← 付け替わった後の ep
        );
    }

    // ③ 自分のローカル変数をヒープにコピーする
    env_body = ALLOC_N(VALUE, env_size);
    rb_env_t *env = IMEMO_NEW(rb_env_t, imemo_env, 0);
    MEMCPY(env_body, ep - (local_size - 1), VALUE, local_size);
    cfp->ep = env_ep;
    VM_ENV_FLAGS_SET(env_ep, VM_ENV_FLAG_ESCAPED | VM_ENV_FLAG_WB_REQUIRED);
    return (VALUE)env;
}
```

処理の順序は **「外側を先に、内側を後に」** です。

---

## なぜ外側を先に処理するのか

外側のヒープコピーが終わると、`prev_cfp->ep` がヒープ側に付け替わります。

この時点で内側の `ep[0]` を `VM_FORCE_WRITE_SPECIAL_CONST` で更新すれば、
内側は「外側のスタック上のアドレス（消えるかもしれない場所）」ではなく
「外側のヒープ上のアドレス（消えない場所）」を正しく指すことになります。

外側を先にやらないと、内側が「消えてしまうスタックアドレス」を指したまま
ヒープに固定されてしまいます。

---

## メモリの流れ（ケース③全体）

```
=====================================================================
(1) outer 呼び出し直後
    x がスタックに積まれる
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+       +------------------------+
  | cfp (outer)  |       | ep[ 0] = specval       |
  |   ep --------+-----> | ep[-1] = 10   (x)      |   (empty)
  +--------------+       +------------------------+
  | cfp (main)   |       +------------------------+
  |   ep --------+-----> | ep[ 0] = specval       |
  +--------------+       | ...                    |
                         +------------------------+


=====================================================================
(2) each ブロック実行開始
    ブロック用 cfp が積まれる（i = 1）
    ep[0] に VM_GUARDED_PREV_EP(outer->ep) がセットされる
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+       +------------------------+
  | cfp (block)  |       | ep[ 0] = prev_ep ------+--+
  |   ep --------+-----> | ep[-1] = 1    (i)      |  |
  +--------------+       +------------------------+  |
  | cfp (outer)  |       +------------------------+  |
  |   ep --------+-----> | ep[ 0] = specval    <--+--+
  +--------------+       | ep[-1] = 10   (x)      |
  | cfp (main)   |       +------------------------+
  |   ...        |
  +--------------+


=====================================================================
(3) lambda {} 実行 ── vm_make_env_each が走る直前
    captured->ep = cfp(block)->ep（スタックを指している）
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+       +------------------------+
  | cfp (block)  |       | ep[ 0] = prev_ep ------+--+
  |   ep --------+-----> | ep[-1] = 1    (i)      |  |   (empty)
  +--------------+       +------------------------+  |
  | cfp (outer)  |       +------------------------+  |
  |   ep --------+-----> | ep[ 0] = specval    <--+--+
  +--------------+       | ep[-1] = 10   (x)      |
                         +------------------------+
  VM_ENV_ESCAPED_P(block->ep) = false
  VM_ENV_LOCAL_P(block->ep)   = false  ← 外側フレームがある


=====================================================================
(4) ① outer のフレームを先にヒープへ
    vm_make_env_each(outer の cfp) が再帰的に呼ばれる
    outer の x がヒープにコピーされ、cfp(outer)->ep が付け替わった
=====================================================================

  cfp structs           local var areas (stack)

  +--------------+       +----------------------------+
  | cfp (block)  |       | ep[ 0] = prev_ep  ← 古い! |
  |   ep --------+-----> | ep[-1] = 1   (i)           |
  +--------------+       +----------------------------+
                                      |
                                      | この ep[0] はまだ outer の
                                      | スタックアドレスを指している
                                      | （次のステップで更新される）
                                      v
                         +----------------------------+
                         | outer スタック変数領域      |
                         | [放棄済み: cfp->ep が離れた]|
                         | ep[ 0] = specval            |
                         | ep[-1] = 10   (x)           |
                         +----------------------------+

  +--------------+
  | cfp (outer)  |
  |   ep --------+-------------------------------------------> +------------------+
  +--------------+                                             | rb_env_t (outer) |
                                                               | ep[ 0] = specval |
                    cfp(outer)->ep がスタックからヒープに付け替わった
                                                               | ep[-1] = 10  (x) |
                                                               +------------------+


=====================================================================
(5) ② ブロックの ep[0] を VM_FORCE_WRITE_SPECIAL_CONST で更新
    外側の「新しいヒープ ep」を指すよう書き換える
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+       +------------------------+  +------------------+
  | cfp (block)  |       | ep[ 0] = prev_ep ------+->| rb_env_t (outer) |
  |   ep --------+-----> | ep[-1] = 1    (i)      |  | ep[ 0] = specval |
  +--------------+       +------------------------+  | ep[-1] = 10  (x) |
  | cfp (outer)  |                                   +------------------+
  |   ep --------+-----> (rb_env_t (outer) の ep スロット)
  +--------------+

  ブロックの ep[0] が「スタックアドレス」から「ヒープアドレス」に更新された


=====================================================================
(6) ③ ブロック自身のフレームもヒープへ
    MEMCPY で i = 1 がヒープにコピーされる
=====================================================================

  cfp structs                                          Heap

  +--------------+       (放棄)                     +------------------+
  | cfp (block)  |                                  | rb_env_t (block) |
  |   ep --------+--------------------------------->| ep[ 0] = prev_ep-+--+
  +--------------+                                  | ep[-1] = 1   (i) |  |
  | cfp (outer)  |                                  +------------------+  |
  |   ep --------+-----> (rb_env_t (outer) の ep)                        |
  +--------------+                                  +------------------+  |
                                                    | rb_env_t (outer) |<-+
                                                    | ep[ 0] = specval |
                                                    | ep[-1] = 10  (x) |
                                                    +------------------+

  ep チェーン完成: rb_env_t(block) → rb_env_t(outer)


=====================================================================
(7) rb_proc_t 作成 → outer 終了
    スタックは消えるが、2つの rb_env_t はヒープに残る
=====================================================================

  cfp structs           local var areas (stack)       Heap

  +--------------+       +------------------------+  +------------------+
  | cfp (main)   |       | ep[ 0] = specval       |  | rb_env_t (block) |
  |   ep --------+-----> | ...                    |  | ep[ 0] = prev_ep-+--+
  +--------------+       +------------------------+  | ep[-1] = 1   (i) |  |
                                                     +------------------+  |
                                                           ^               |
                                                     +-----+----------+    |
                                                     | rb_proc_t      |    |
                                                     |   ep           |    |
                                                     |   is_lambda=1  |    |
                                                     +----------------+    |
                                                                           |
                                                     +------------------+  |
                                                     | rb_env_t (outer) |<-+
                                                     | ep[ 0] = specval |
                                                     | ep[-1] = 10  (x) |
                                                     +------------------+


=====================================================================
(8) lambda.call 実行中
    ep チェーンを辿って x と i の両方にアクセスする
=====================================================================

  cfp structs                                          Heap

  +--------------+                                  +------------------+
  | cfp (block)  |                                  | rb_env_t (block) |
  |   ep --------+--------------------------------->| ep[ 0] = prev_ep-+--+
  +--------------+                                  | ep[-1] = 1   (i) |  |
  | cfp (main)   |       +------------------------+ +------------------+  |
  |   ep --------+-----> | ...                    |                       |
  +--------------+       +------------------------+ +------------------+  |
                                                    | rb_env_t (outer) |<-+
                                                    | ep[ 0] = specval |
  x + i の計算:                                     | ep[-1] = 10  (x) |
    i → 自分の ep[-1]         = 1                   +------------------+
    x → ep[0] を辿って外側の ep[-1] = 10
    結果 = 11
=====================================================================
```

---

## ケース③まとめ

| | ネストしたブロック |
|---|---|
| `vm_make_env_each` の処理順 | 外側フレームを先に、内側を後に（再帰） |
| `VM_FORCE_WRITE_SPECIAL_CONST` | 内側の ep[0] を外側の新しいヒープ ep に更新 |
| 結果 | 2つの `rb_env_t` がヒープ上で ep チェーンでつながる |
| `x` へのアクセス | ep[0] を辿って外側の rb_env_t から読む |
| `i` へのアクセス | 自分の ep[-1] から直接読む |

**外側から先にヒープコピー → 内側の ep[0] を更新 → 内側をコピー** という順序が、
ep チェーンの一貫性を保証しています。

---

# 5. 実験

---

## セクション4の答え合わせ

ここまでで見てきたこと：

- `rb_env_t` がヒープ上に作られる
- `rb_captured_block.ep` がその `rb_env_t` を指す
- 変数へのアクセスは `ep[-1]`, `ep[-2]`... で行われる

この2つの実験で、それを実際に確認します。

---

# 実験A: lambda作成後に変数を変更する

---

## コード

```ruby
def message_function
  str = "The quick brown fox"
  func = lambda { |animal| puts "#{str} jumps over the lazy #{animal}." }
  str = "The sly brown fox"   # ← lambda を作った後に変更
  func
end

message_function.call('dog')
```

`func` を作った後に `str` を書き換えています。
`func.call` したとき、どちらの `str` が使われるでしょうか。

---

## 結果

```
The sly brown fox jumps over the lazy dog.
```

**後から書き換えた値が使われました。**

---

## なぜか

`func` を作った瞬間（`lambda {}`）に `vm_make_env_each` が走り、
`str` はヒープ上の `rb_env_t` にコピーされます。

```
rb_env_t
  ep[ 0] = specval
  ep[-1] = "The quick brown fox"  ← この時点の str
```

その後 `str = "The sly brown fox"` を実行すると、
`ep[-1]` の値が上書きされます。

```
rb_env_t
  ep[ 0] = specval
  ep[-1] = "The sly brown fox"   ← 上書きされた
```

`func` の `captured.ep` は同じ `rb_env_t` を指したまま。
`func.call` のとき `ep[-1]` を読むと、上書き後の値が返ってきます。

---

## まとめ（実験A）

> `lambda {}` はコピーではなく「参照」を持つ

`lambda {}` を書いた瞬間に変数の値がスナップショットとして固定されるわけではありません。
`ep` が指す `rb_env_t` を通じて、常に **その時点の変数の値** を参照します。

---

# 実験B: 2つのlambdaが変数を共有する

---

## コード

```ruby
def create_counter
  count = 0
  increment = lambda { count += 1 }
  read      = lambda { count }
  [increment, read]
end

inc, read = create_counter
inc.call
inc.call
puts read.call  # => ?
```

`increment` と `read` は別々の lambda です。
`inc.call` を2回呼んだ後、`read.call` は何を返すでしょうか。

---

## 結果

```
2
```

`read` は `inc` が更新した `count` を見ています。

---

## なぜか

`increment` と `read` は同じメソッドの中で作られています。
どちらの `lambda {}` も、作られた時点での `cfp->ep` を `captured->ep` としてコピーします。
同じフレームで作られているので、**同じ `rb_env_t` を指します**。

`increment` を作った時点で `rb_env_t` が作られ、`VM_ENV_FLAG_ESCAPED` が立ちます。
`read` を作るとき、Ruby は新しい `rb_env_t` を作りません。

```c
// rb_vm_make_proc_lambda（vm.c）
if (!VM_ENV_ESCAPED_P(captured->ep)) {
    vm_make_env_object(ec, cfp); // フラグが立っていたらここは実行されない
}

// vm_make_env_each（vm.c）の先頭でも同じチェックがある
if (VM_ENV_ESCAPED_P(ep)) return VM_ENV_ENVVAL(ep); // 既存の rb_env_t をそのまま返す
```

`VM_ENV_ESCAPED_P` が `true` の場合、既存の `rb_env_t` を返すだけです。
こうして2つの lambda が同じ `rb_env_t` を共有することになります。

```
rb_env_t (count)
  ep[ 0] = specval
  ep[-1] = 0  (count)
       ^             ^
       |             |
rb_proc_t        rb_proc_t
(increment)       (read)
  ep ──────────────+
```

`inc.call` するたびに `rb_env_t` の `ep[-1]` が更新されます。
`read.call` で `ep[-1]` を読むと、`inc.call` が更新した値が見えます。

---

## まとめ（実験B）

> 同じスコープで作られた複数の lambda は、同じ `rb_env_t` を共有する

これがセクション1で確認した「性質3: 環境の共有」の実体です。
`rb_captured_block.ep` が同じアドレスを指しているという、それだけのことです。
