# Rubyブロックの内部実装
## クロージャはどう実現されているか

---

## はじめに

### この章では何を学ぶのか

Ruby内部でブロックがどのように実装され、動作するかを学ぶ。特に以下2つの核心概念を理解する：

| 概念 | 説明 | Ruby内部実装 |
|------|------|-------------|
| ブロック | コードの塊と環境を組み合わせた機能 | rb_block_t構造体 |
| クロージャ | 定義時の環境を保持する関数的オブジェクト | 環境ポインタ（EP）によるスコープ管理 |

### 日常的に使用されているブロック機能

```ruby
[1,2,3].each { |x| puts x }
File.open("data.txt") { |f| puts f.read }
benchmark { heavy_calculation }
```

これらは1970年代に確立されたクロージャ理論に基づく実装である。

---

## 目次

1. **ブロックの基本機能**
2. **ブロックとメソッドの違い**
3. **クロージャの定義と歴史**  
4. **内部実装メカニズム**
5. **lambda vs proc の実装差異**
6. **設計パラダイムとしての価値**

---

## 1. ブロックの基本

```ruby
10.times do |i|
  puts "Hello #{i}"
end

[1, 2, 3].each { |x| puts x * 2 }

File.open("data.txt") do |file|
  puts file.read
end
```

```ruby
my_proc = proc { |x| puts x }
my_lambda = lambda { |x| puts x }
```

普段何気なく使ってるこれら、内部で何が起きてるかを見てみます。

---

## 2. ブロックとメソッドの違い

### メソッドの場合
```ruby
prefix = "Hello"

def greet(name)
  puts "#{prefix} #{name}"  # エラー！prefixが見えない
end

greet("World")
```

### ブロックの場合
```ruby
prefix = "Hello"

def greet_with_block
  yield(prefix)  # OK！prefixが使える
end

greet_with_block { |p| puts "#{p} World" }  # "Hello World"
```

### ブロックの嬉しいポイント

#### 1. 定義された時点のスコープの変数にアクセスできる

```ruby
def create_greeter
  name = "Alice"    # このスコープの変数
  age = 25
  
  # ブロックでgreetを実行
  3.times do |i|
    puts "#{name}さん、#{age}歳です (#{i + 1}回目)"
  end
end

def run_somewhere_else
  # 全然違うスコープ
  name = "Bob"  # こっちのnameは関係ない
  create_greeter  # でもAliceが出力される
end

run_somewhere_else
# "Aliceさん、25歳です (1回目)"
# "Aliceさん、25歳です (2回目)"  
# "Aliceさん、25歳です (3回目)"
```

**何が嬉しい？**: 設定を「覚えて」くれるので、毎回渡す必要がない

#### 2. 固有の処理と共通処理を分離できる

```ruby
# 共通処理（時間測定）をメソッドに、固有処理をブロックに
def benchmark
  start_time = Time.now
  yield  # 固有の処理を実行
  end_time = Time.now
  puts "実行時間: #{end_time - start_time}秒"
end

benchmark { heavy_calculation }  # 共通処理に固有処理を渡す
benchmark { another_task }       # 同じ共通処理で別の固有処理
```

**メソッドでやろうとすると...**
```ruby
def benchmark_method(method_name)
  start_time = Time.now
  send(method_name)  # メソッド名で呼び出し
  end_time = Time.now
  puts "実行時間: #{end_time - start_time}秒"
end

benchmark_method(:heavy_calculation)
benchmark_method(:another_task)
# メソッド名（シンボル）で渡す必要がある
# 複雑な処理はメソッドとして事前定義が必要
```

**ブロックなら**
```ruby
benchmark { heavy_calculation }
benchmark { complex_calc(x, y, z) }  # 引数付きも簡単
benchmark { puts "Hello"; sleep(1) }  # 複数行もその場で
# 任意のコードをその場で書ける
```

#### 3. 変数の状態を保持し続ける

```ruby
total = 0

[1, 2, 3].each do |n|
  total += n  # 外側のtotalを更新
  puts "現在の合計: #{total}"
end

puts "最終合計: #{total}"  # 6
```

**何が嬉しい？**: ブロック内で外側の変数を更新できる

#### 補足例：独立した環境を持つ関数

```ruby
def create_counter(start = 0)
  lambda { start += 1 }
end

counter1 = create_counter(0)
counter2 = create_counter(100)

puts counter1.call  # 1
puts counter1.call  # 2
puts counter2.call  # 101  ← 独立した環境
puts counter1.call  # 3   ← counter2の影響を受けない
```

### まとめると

ブロックの3つの特徴：

1. **定義時のスコープの変数にアクセスできる**
   → 設定や環境を「記憶」して使える

2. **固有の処理と共通処理を分離できる**
   → 柔軟で再利用可能なコードが書ける

3. **外側の変数の状態を変更できる**
   → ブロック実行前後で状態を共有できる

**結果**: メソッドより柔軟で、コードの重複を避けられる

---

## 3. クロージャの定義と歴史

### クロージャの正式な定義

上記で見たブロックの特徴は、コンピュータサイエンスでは**クロージャ**と呼ばれる概念である。

Sussman & Steele (1975)による定義:
> **クロージャ = lambda式 + それが定義された環境**

この定義は、関数型プログラミングの基礎となっている。

### クロージャの歴史と発展

**1958年**: John McCarthy - LISP開発（初期の関数型言語）
**1964年**: Peter J. Landin - クロージャ概念の理論的基礎を確立
**1975年**: Guy Steele & Gerald Sussman - Schemeで初の実用的なクロージャ実装
**1995年**: Brendan Eich - JavaScriptでWebにクロージャを導入
**2000年代**: Python, C#, Rubyなどで幅広く採用

### クロージャの本質的価値

従来の関数：引数のみで動作
```javascript
function add(x, y) {
  return x + y;  // xとyのみアクセス可能
}
```

クロージャ：定義時の環境を保持
```javascript
var multiplier = 3;  // 定義時の環境

function multiply(x) {
  return x * multiplier;  // 環境のmultiplierを「記憶」
}
```

この「記憶」機能が、関数を極めて柔軟で再利用可能なコンポーネントに変えた。

![Procオブジェクト構造詳細](../images/proc_structure.png)

**図8-21: Procオブジェクトの完全なクロージャ構造**

**1975年のクロージャ定義**「関数 + 環境」の**現代的実装**：

**関数部分（左上）**:
- YARV命令列：putstring, getlocal, tostring等の実行可能コード
- オブジェクトアクセス：「&lt;callinfo!mid:puts, argc:1...」

**環境部分（右下）**:
- **rb_proc_t**: Rubyオブジェクト化されたプロシージャ
- **rb_block_t**: iseq（関数）とEP（環境）を結合  
- **ヒープ上のスタックフレームコピー**: ローカル変数strの永続化

**結合メカニズム**: EPの矢印が示すように、関数と環境が**ポインタで連結**されており、Sussman & Steeleの理論的定義をC言語レベルで忠実に実装している。

### Rubyにおけるクロージャの位置づけ

Rubyのブロックは、この歴史あるクロージャ理論の現代的実装であり、特にイテレータやコールバックパターンでその価値を発揮している。

---

## 4. 内部実装メカニズム

### Ruby内部でのメソッド実行メカニズム

```ruby
def calculate(x, y)
  result = x + y
  return result
end

calculate(3, 5)
```

#### Rubyの内部動作

**1. スタックフレーム作成**
```
┌─────────────────────┐
│ calculate メソッド   │ ← rb_control_frame_t構造体
│ ├── x = 3          │   （メソッド実行コンテキスト管理用の制御構造体）
│ ├── y = 5          │   - ローカル変数の位置情報
│ └── result = 8     │   - 実行ポイント（プログラムカウンタ）
└─────────────────────┘   - 環境ポインタ（EP）: 変数アクセス管理
```

**2. メソッド終了時**
```
スタックフレーム削除 → 全ての変数が消失
```

![スタックフレーム図](../images/stack_frame.png)

**図8-17: lambda呼び出し時のスタックフレームのヒープコピー**
- Rubyが現在のスタックフレームをヒープにコピーする仕組み
- rb_proc_tとrb_env_t構造体の関係
- is_lambdaフラグによる動作切り替え

**問題**: メソッド終了と同時に変数が消えるため、後からアクセスできない

### ブロック専用構造体：rb_block_t

Rubyはクロージャ実現のために特別な構造体を定義している：

```c
typedef struct rb_block_struct {
    VALUE *ep;            // Environment Pointer: 変数スコープへのポインタ（クロージャ実現の核心）
    rb_iseq_t *iseq;     // Instruction Sequence: コンパイル済み実行コード（Ruby→バイトコード）  
    VALUE self;           // ブロック実行時のselfオブジェクト参照
    VALUE klass;          // メソッド探索用のクラス情報
    VALUE proc;           // Proc化時の相互参照（循環参照管理）
} rb_block_t;
```

#### VALUE型について
Rubyの内部オブジェクト表現。ポインタまたは即値を格納する統一型：
- **ポインタ**: 文字列、配列、ハッシュ等のヒープオブジェクトへの参照  
- **即値**: 数値、シンボル、nil、true/false等の直接値

#### 各フィールドの役割詳細

| フィールド | 機能 | 重要度 |
|----------|------|----------|
| **ep** | クロージャの環境保持 | ★★★ |
| **iseq** | YARVバイトコード実行 | ★★★ |
| **self** | ブロック内selfコンテキスト | ★★ |
| **klass** | メソッド探索用クラス | ★★ |
| **proc** | Procオブジェクト化管理 | ★ |

![rb_block_t構造](../images/rb_block_structure.png)

**図8-4: ブロック作成時の構造体関係**

この図は重要な3つの要素を示している：
1. **YARVの内部スタック**: ローカル変数strが配置される一時的メモリ領域
2. **rb_control_frame_t**: メソッド実行管理用の制御構造体（EPを含む）
3. **rb_block_t**: ブロック専用構造体（iseqとEPを含む）

EPが両方の構造体に存在し、**同一のスタック位置を指す**ことで変数アクセスを実現している点に注目。これがクロージャの環境保持メカニズムの基礎である。

#### クロージャ実現の2本柱

**1. iseq (Instruction Sequence) - 実行コード**
- **定義**: RubyソースコードをYARVバイトコードにコンパイルした中間表現
- **形式**: プラットフォーム非依存の仮想マシン命令列
- **最適化**: 実行時の高速化とメモリ効率を両立

**2. EP (Environment Pointer) - 環境ポインタ** 
- **定義**: ブロック定義時点での変数スコープへのポインタ
- **機能**: スタックフレーム上の変数位置情報を保持
- **重要性**: クロージャの「環境保持」を実現する最核心メカニズム
- **生存期間**: メソッド終了後もヒープコピーで変数アクセスを継続

---

## 環境ポインタ（EP）の仕組み

### 1. ブロック作成時
```ruby
def create_greeter
  prefix = "Hello"        # ← スタック上
  lambda do |name|        # ← EPがprefixの場所を記録
    puts "#{prefix} #{name}"
  end
end
```

### 2. メソッド終了直前
Rubyが賢く動作：
- スタック上の変数をヒープにコピー
- EPをヒープの場所に変更

### 3. あとで実行
```ruby
greeter.call("World")  # ヒープ上のprefixにアクセス
```

メソッド終了後でも変数が使える！

![lambda実行時のヒープコピー](../images/lambda_heap_copy.png)

**図8-17: lambda呼び出し時のスタック→ヒープ変換**

この図解は**クロージャの核心メカニズム**を明示している：

**スタック領域（上部）**: 
- 一時的なローカル変数str
- rb_control_frame_t構造体

**ヒープ領域（下部）**:
- **rb_proc_t**: lambdaオブジェクト本体（is_lambdaフラグを含む）
- **rb_env_t**: 環境オブジェクト（スタックフレームの永続化コピー）
- **rb_block_t**: ブロック実行コンテキスト（iseq + EP）

**重要**: 破線の矢印は、スタックからヒープへの**環境コピー処理**を表す。これにより、メソッド終了後もローカル変数へのアクセスが可能になる。

---

## 6. 設計パラダイムとしての価値

### 従来のプログラミングパラダイム
```ruby
# 毎回全ての設定を明示的に渡す方式
log_message("エラー発生", "production", "ERROR", "MyApp")
log_message("ログイン", "production", "ERROR", "MyApp")
log_message("保存完了", "production", "ERROR", "MyApp")
```

### クロージャパラダイム
```ruby
# 設定を「封じ込む」方式
logger = create_logger("production", "ERROR", "MyApp")  # 設定フェーズ
logger.call("エラー発生")                         # 実行フェーズ
logger.call("ログイン")                           # 実行フェーズ
logger.call("保存完了")                         # 実行フェーズ
```

![設定と実行の分離](../assets/configuration-separation-diagram.txt)

### クロージャによるソフトウェア設計の改善

#### 1. 関心の分離 (Separation of Concerns)
設定管理とビジネスロジックを明確に分離

| 関心領域 | 従来の方式 | クロージャ方式 |
|----------|----------|------------|
| 設定管理 | 毎回明示 | 一回の定義で封じ込み |
| 処理実行 | コードの重複 | 再利用可能なコンポーネント |

#### 2. コードの再利用性向上
- 同一設定で異なるデータを処理
- 異なる設定で特化されたハンドラを作成

#### 3. 保守性と可読性の改善
- 設定変更の影響範囲が明確
- 呼び出しコードのシンプル化
- ビジネスロジックに集中できるインターフェース

### ソフトウェアアーキテクチャへの影響

クロージャは、現代のソフトウェア設計パターンの土台となっている：

- **コールバックパターン**: イベントドリブンアーキテクチャの基礎
- **関数型プログラミング**: map, filter, reduceなどの高階関数
- **非同期プログラミング**: Promise, async/awaitの基礎概念

![Proc参照保持](../images/proc_reference.png)

**図8-21: Procオブジェクトの内部構造**
- RTypedData構造体でのRubyオブジェクト化
- rb_proc_tが包含するrb_block_t構造体
- 関数と参照元の環境へのポインタを含むクロージャ
- VALUE ポインタによるヒープオブジェクトとしての管理

**結論**: Rubyのブロックは、数十年の歴史を持つクロージャ理論を実用的でエレガントな形で実装したものである。

---

## 5. lambda vs proc の実装差異

### 同一構造体、異なる振る舞い

両者は**同じrb_block_t構造体**を基盤としながら、`is_lambda`フラグで動作を切り替える：

```c
typedef struct rb_proc_struct {
    rb_block_t block;        // 共通のブロック構造
    VALUE envval;            // 環境オブジェクトへの参照  
    int is_lambda;           // 動作モード切り替えフラグ
} rb_proc_t;
```

### 主要な動作差異

#### 1. 引数チェックの厳密性

**lambda**: 厳密な引数チェック
```ruby
my_lambda = lambda { |x, y| puts "#{x}, #{y}" }
my_lambda.call(1)       # ArgumentError: wrong number of arguments
my_lambda.call(1, 2, 3) # ArgumentError: wrong number of arguments
```

**proc**: 柔軟な引数処理
```ruby
my_proc = proc { |x, y| puts "#{x}, #{y}" }
my_proc.call(1)       # "1, " (不足分はnil)
my_proc.call(1, 2, 3) # "1, 2" (余剰分は無視)
```

#### 2. return文の制御フロー

**lambda**: ローカルreturn（lambda自身から抜ける）
```ruby
def test_lambda
  f = lambda { return "lambda終了" }
  result = f.call     # lambdaから戻る
  puts "継続実行"      # ← 実行される
  result
end
```

**proc**: non-localreturn（メソッド全体から抜ける）
```ruby  
def test_proc
  f = proc { return "proc終了" }
  result = f.call     # メソッド全体から戻る
  puts "実行されない"   # ← 実行されない  
end
```

### 内部実装メカニズム

#### YARV仮想マシンでの処理分岐

```c
// Ruby内部でのreturn処理（簡略化）
static VALUE vm_exec_return(rb_execution_context_t *ec, VALUE val) {
    rb_proc_t *proc = GET_PROC();
    
    if (proc->is_lambda) {
        // lambda: 通常の関数returnとして処理
        return val;  
    } else {
        // proc: non-local returnとしてメソッドを終了
        rb_iter_break_value(val);
    }
}
```

#### 引数処理での分岐

```c
// 引数数チェック（簡略化）
static void check_argument_count(rb_proc_t *proc, int given, int required) {
    if (proc->is_lambda && given != required) {
        rb_raise(rb_eArgumentError, "wrong number of arguments");
    }
    // procの場合はチェックしない
}
```

### 使い分けの指針

| 用途 | 推奨 | 理由 |
|------|------|------|
| 関数型プログラミング | lambda | 厳密な引数チェック、予測可能な制御フロー |
| イテレータ実装 | proc | 柔軟な引数処理、直感的な早期リターン |
| コールバック | lambda | 独立性、デバッグ容易性 |  
| DSL構築 | proc | 自然な制御フロー、ホストメソッドとの統合 |

---

## まとめ

### Rubyブロックの正体
- **クロージャ**という1975年からの理論の現代的実装
- コード（iseq）+ 環境（EP）をセットで保存するrb_block_t構造体
- スタック→ヒープコピーによる環境永続化メカニズム

### 技術的価値
- 関心の分離：設定フェーズと実行フェーズの明確な分離
- メモリ管理：ガベージコレクション対応の自動生存期間管理
- 柔軟性：同一インターフェースでのlambda/proc切り替え

### 設計パラダイムへの影響
- 現代の非同期プログラミング、関数型プログラミングの基礎
- イベントドリブンアーキテクチャの核心技術
- Ruby on Railsなど現代フレームワークの設計哲学

**結論**: 日常的に使用するブロック機能は、数十年の研究蓄積とRuby独自の工夫が組み合わさった精密な実装である。

---

## 質疑応答

ご質問があればお気軽に！