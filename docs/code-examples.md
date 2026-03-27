# 第8章 コード例集

## 基本的なブロック例

### 1. 簡単なブロック
```ruby
10.times do |i|
  puts "Hello #{i}"
end
```

### 2. 変数スコープの例
```ruby
str = "The quick brown fox"
10.times do
  str2 = "jumps over the lazy dog."
  puts "#{str} #{str2}"
end
```

## lambda/Procの例

### 3. lambda基本例
```ruby
def message_function
  str = "The quick brown fox"
  lambda do |animal|
    puts "#{str} jumps over the lazy #{animal}."
  end
end

function_value = message_function
function_value.call('dog')
```

### 4. 後から変数を変更する例
```ruby
def message_function
  str = "The quick brown fox"
  func = lambda do |animal|
    puts "#{str} jumps over the lazy #{animal}."
  end
  str = "The sly brown fox"
  func
end

function_value = message_function
function_value.call('dog')
# 出力: "The sly brown fox jumps over the lazy dog."
```

### 5. 同じスコープでlambdaを複数回呼び出し
```ruby
i = 0
increment_function = lambda do
  puts "incrementing from #{i} to #{i+1}"
  i += 1
end

decrement_function = lambda do
  i -= 1
  puts "decrementing from #{i+1} to #{i}"
end

increment_function.call  # incrementing from 0 to 1
decrement_function.call  # decrementing from 1 to 0
increment_function.call  # incrementing from 0 to 1
increment_function.call  # incrementing from 1 to 2
decrement_function.call  # decrementing from 2 to 1
```

## ベンチマーク用コード

### 6. while ループのベンチマーク
```ruby
require 'benchmark'
ITERATIONS = 1000000

Benchmark.bm do |bench|
  bench.report("iterating from 1 to 10, one million times") do
    ITERATIONS.times do
      sum = 0
      i = 1
      while i <= 10
        sum += i
        i += 1
      end
    end
  end
end
```

### 7. each ブロックのベンチマーク
```ruby
require 'benchmark'
ITERATIONS = 1000000

Benchmark.bm do |bench|
  bench.report("iterating from 1 to 10, one million times") do
    ITERATIONS.times do
      sum = 0
      (1..10).each do |i|
        sum += i
      end
    end
  end
end
```

## C構造体（参考）

### 8. rb_block_t構造体
```c
typedef struct rb_block_struct {
    VALUE self;           // ブロックが最初に参照された際のselfポインタの値
    VALUE klass;          // 現在のオブジェクトのクラス
    VALUE *ep;            // 環境ポインタ（クロージャの環境の重要な部分）
    rb_iseq_t *iseq;     // ブロック中のRubyコードを表現するYARV命令列へのポインタ
    VALUE proc;           // ブロックからProcオブジェクトを作成した際に使用
} rb_block_t;
```

### 9. rb_control_frame_t構造体（抜粋）
```c
typedef struct rb_control_frame_struct {
    VALUE *pc;            // プログラムカウンタ
    VALUE *sp;            // スタックポインタ
    rb_iseq_t *iseq;     // 命令列
    VALUE flag;           // フラグ
    VALUE self;           // selfオブジェクト
    VALUE klass;          // クラス
    VALUE *ep;            // 環境ポインタ
    rb_iseq_t *block_iseq; // ブロック命令列
    VALUE proc;           // Procオブジェクト
    // その他のメンバ...
} rb_control_frame_t;
```

## 実験・検証用コード

### 10. メモリ使用量確認用
```ruby
# GCの動作を観察
def test_closure_memory
  str = "The quick brown fox" * 1000  # 大きな文字列
  
  lambda_func = lambda do
    puts str.length
  end
  
  str = nil  # 元の参照を削除
  GC.start   # ガベージコレクション実行
  
  lambda_func.call  # まだ文字列にアクセスできる
end
```

### 11. 複数のlambdaでの環境共有確認
```ruby
def create_shared_counter
  count = 0
  
  increment = lambda { count += 1 }
  decrement = lambda { count -= 1 }
  get_value = lambda { count }
  
  [increment, decrement, get_value]
end

inc, dec, get = create_shared_counter
puts get.call  # 0
inc.call
inc.call
puts get.call  # 2
dec.call
puts get.call  # 1
```