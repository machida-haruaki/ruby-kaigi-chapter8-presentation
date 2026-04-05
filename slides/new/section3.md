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
