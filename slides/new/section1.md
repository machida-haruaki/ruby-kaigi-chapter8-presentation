# 1. Rubyブロックの3つの性質

---

## おさらいです

8章で学んだブロックの3つの性質を確認します。

この3つが、この後の内部実装の話」につながります。

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
