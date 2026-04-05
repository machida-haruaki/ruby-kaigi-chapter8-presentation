# Claude セッション継続用コンテキスト

## プロジェクト目的

「Rubyのしくみ」（Ruby Under a Microscope）8章の輪読会発表資料の作成。
**発表: 2026-04-06（今日）、45分**

---

## ファイル構成

```
slides/new/
  outline.md     # 全体構成（FIX済み）
  section1.md    # ブロックの3つの性質（完成）
  section2.md    # ブロック = クロージャ（完成）
  section3.md    # 内部構造の概要（完成）
  section4.md    # 環境はなぜ生き続けるのか（ケース①完成・ケース②未着手）
  section5.md    # 実験A・B（未着手）
  final.md       # 最後に統合（未作成）
```

---

## 確定した全体構成（outline.md）

| # | タイトル | 時間 |
|---|---|---|
| 1 | ブロックの3つの性質（おさらい） | 4分 |
| 2 | ブロック = クロージャ | 3分 |
| 3 | 内部構造の概要 | 8分 |
| 4 | 環境はなぜ生き続けるのか | 18分 |
| 5 | 実験（A + B） | 5分 |
| Q&A | | 7分 |

---

## セクション4の構成

### ケース①: シンプルなブロック（完成）
`def greet; name = "Alice"; [1,2,3].each { |i| puts name }; end`

- rb_captured_block は cfp に埋め込まれている（独立した構造体ではない）
- cfp[3]/block[0], cfp[4]/block[1], cfp[5]/block[2] のコメントが根拠
- yield = invokeblock VM命令 → vm_invokeblock_i → vm_invoke_block → vm_invoke_iseq_block → vm_push_frame
- ブロックの ep[0] に VM_GUARDED_PREV_EP(captured->ep) がセット = greet の ep[0] アドレス
- メソッド終了でスタックごと消える。ヒープコピーなし

### ケース②: lambda（未着手）
`def create_counter; count = 0; lambda { count += 1 }; end`

以下の流れを書く必要がある：
1. greet 呼び出し → cfp + ローカル変数領域（count）がスタックに
2. lambda {} を書く → rb_captured_block が cfp に埋め込まれる（ep はスタックを指す）
3. rb_vm_make_proc_lambda が呼ばれる
4. VM_ENV_ESCAPED_P チェック → まだヒープにない
5. vm_make_env_each が走る：
   - ALLOC_N でヒープにメモリ確保
   - IMEMO_NEW で rb_env_t 作成
   - MEMCPY でスタックの変数をヒープにコピー
   - cfp->ep = env_ep でepをヒープ側に付け替え
   - VM_ENV_FLAG_ESCAPED | VM_ENV_FLAG_WB_REQUIRED をセット
6. vm_proc_create_from_captured で rb_proc_t 作成（is_lambda = true）
7. create_counter が return → スタックフレームは消えるが rb_env_t はヒープに残る
8. counter.call → vm_push_frame、ep[0] = captured->ep（ヒープのrb_env_t）でアクセス

### ケース③: ネストしたブロック（未着手）
- epチェーンの仕組み
- vm_make_env_each が再帰的に外側フレームまで遡ってヒープコピー
- VM_FORCE_WRITE_SPECIAL_CONST で各epのspecvalを更新

---

## Ruby 3.4 実際のソースコード（調査済み・正確）

### 呼び出しチェーン: lambda {} 作成時

```
lambda { count += 1 } を実行
  → f_lambda()                         [proc.c]
  → rb_block_lambda()                  [proc.c]
  → proc_new(rb_cProc, TRUE)           [proc.c]  is_lambda=TRUE
  → rb_vm_make_proc_lambda()           [vm.c]    ← 核心
      if (!VM_ENV_ESCAPED_P(captured->ep)):
        → vm_make_env_object()         [vm.c]
          → vm_make_env_each()         [vm.c]    ← スタック→ヒープコピー
      → vm_proc_create_from_captured() [vm.c]    is_lambda = TRUE をセット
```

### 呼び出しチェーン: yield / .call 時

```
counter.call / yield
  → invokeblock（VM命令）
  → vm_invokeblock_i
  → vm_invoke_block    ← 種類（iseq/ifunc/symbol/proc）で振り分け
  → vm_invoke_iseq_block
  → vm_push_frame      ← 新しい cfp 作成
      VM_GUARDED_PREV_EP(captured->ep) を ep[0] にセット
```

### struct rb_captured_block（vm_core.h）
```c
struct rb_captured_block {
    VALUE self;
    const VALUE *ep;
    union {
        const rb_iseq_t *iseq;
        const struct vm_ifunc *ifunc;
        VALUE val;
    } code;
};
```

### rb_control_frame_t（vm_core.h）
```c
typedef struct rb_control_frame_struct {
    const VALUE *pc;        // cfp[0]
    VALUE *sp;              // cfp[1]
    const rb_iseq_t *_iseq; // cfp[2]
    VALUE self;             // cfp[3] / block[0]
    const VALUE *ep;        // cfp[4] / block[1]
    const void *block_code; // cfp[5] / block[2]
    void *jit_return;       // cfp[6]
} rb_control_frame_t;
```

### rb_proc_t（vm_core.h）
```c
typedef struct {
    const struct rb_block block;
    unsigned int is_from_method: 1;
    unsigned int is_lambda: 1;       // これ1つでlambdaかprocかが決まる
    unsigned int is_isolated: 1;
} rb_proc_t;
```

### rb_env_t（vm_core.h）
```c
typedef struct {
    VALUE flags;         // imemoヘッダ（GC管理用）
    rb_iseq_t *iseq;
    const VALUE *ep;     // env[]内のspecvalスロットを指す
    const VALUE *env;    // ヒープ上のローカル変数配列
    unsigned int env_size;
} rb_env_t;
```

### vm_make_env_each の処理（vm.c ~1088行）
```c
static VALUE
vm_make_env_each(const rb_execution_context_t * const ec, rb_control_frame_t *const cfp)
{
    if (VM_ENV_ESCAPED_P(ep)) return VM_ENV_ENVVAL(ep);  // すでにヒープ済みなら即return

    // ネストしたブロックの場合は外側を再帰的に処理
    if (!VM_ENV_LOCAL_P(ep)) {
        // 外側フレームを再帰的に vm_make_env_each する
        vm_make_env_each(ec, prev_cfp);
        // 内側のep[0]を外側の新しいヒープepに更新
        VM_FORCE_WRITE_SPECIAL_CONST(&ep[VM_ENV_DATA_INDEX_SPECVAL],
                                      VM_GUARDED_PREV_EP(prev_cfp->ep));
    }

    env_body = ALLOC_N(VALUE, env_size);            // ヒープにメモリ確保
    rb_env_t *env = IMEMO_NEW(rb_env_t, imemo_env, 0); // rb_env_t 作成
    MEMCPY(env_body, ep - (local_size - 1), VALUE, local_size); // スタック→ヒープコピー
    env_ep = &env_body[local_size - 1];             // epはenv[]内のspecvalを指す
    env->ep = env_ep;
    env->env = env_body;
    cfp->ep = env_ep;                               // cfpのepをヒープ側に付け替え
    VM_ENV_FLAGS_SET(env_ep, VM_ENV_FLAG_ESCAPED | VM_ENV_FLAG_WB_REQUIRED);
    return (VALUE)env;
}
```

---

## 重要な設計決定・方針

- **図のスタイル**: 英語ラベルのみ（日本語はボックス外）、`+`, `-`, `|` で罫線
- **cfp とローカル変数領域は別ボックス**で描く（物理的に隣り合っていないため）
- **左: cfp structs / 右: local var areas** のレイアウトで統一
- セクション5（lambda vs proc）は削除し、is_lambda フラグはセクション3で一言触れるだけ
- 各セクションは独立ファイル。統合は全完成後に1回だけ

---

## 開発環境

- Ruby 3.4.9（rbenv、グローバル設定済み）
- `~/src/ruby` に v3.4.9 ソースをclone済み
- ctags（27,432エントリ）と compile_commands.json（clangd用）生成済み
- neovim で clangd LSP 有効（`~/.config/nvim/lua/plugins/lsp.lua` に clangd 追加済み）
  - `~/.config/nvim` は `~/dotfiles/nvim/.config/nvim` へのシムリンク
