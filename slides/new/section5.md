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
