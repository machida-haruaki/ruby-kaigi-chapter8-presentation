# Rubyのしくみ 第8章
## Lispから借用したアイデア 発表資料

---

## 目次
1. ブロックとクロージャの基礎
2. Ruby におけるクロージャ実装
3. ラムダと Proc：関数を第一級市民として扱う
4. スタック vs. ヒープメモリ
5. 実験とベンチマーク
6. まとめ・質疑応答

---

## 第8章の概要
- **テーマ**: Rubyのブロックがクロージャをどう実装しているか
- **学習目標**: 
  - ブロックの内部実装（rb_block_t構造体）を理解する
  - 1975年のSchemeから借用したクロージャの概念を学ぶ
  - lambdaとProcの違いとメモリ管理を把握する
  - パフォーマンス特性を実験を通して確認する

---

## ブロックとクロージャの歴史
### コンピュータサイエンスの系譜
- **1958年**: John McCarthyがLispを作成
- **1964年**: Peter J. Landinがクロージャを発明
- **1975年**: Gerald SussmanとGuy SteeleがSchemeでクロージャを活用
- **1995年**: まつもとゆきひろがRubyでブロックを実装

### クロージャの定義（Sussman & Steele, 1975）
> ラムダ式と、そのラムダ式が引数に適用されるときに使われる環境の両方を持つデータ構造

```ruby
# Rubyのブロックはクロージャの実装
10.times do |i|
  puts "Hello #{i}"
end
```

---

## Ruby におけるクロージャ実装
### rb_block_t 構造体の中身

```c
typedef struct rb_block_struct {
    VALUE self;           // ブロック参照時のselfポインタ
    VALUE klass;          // 現在のオブジェクトクラス
    VALUE *ep;            // 環境ポインタ（重要！）
    rb_iseq_t *iseq;     // YARV命令列
    VALUE proc;           // Procオブジェクト参照
} rb_block_t;
```

### 2つの重要な要素
1. **iseq**: ブロック内のコード（YARV命令列）
2. **EP**: 親スコープの変数にアクセスする環境ポインタ

---

## ブロック実行の内部動作
### スタックフレームの変化

```ruby
str = "The quick brown fox"    # ①
10.times do                   # ②
  str2 = "jumps over the lazy dog."  # ③
  puts "#{str} #{str2}"        # ④
end
```

### 実行時のスタック構成
1. **トップレベル**: 変数`str`を保持
2. **times メソッド**: 内部Cコード実行
3. **ブロック**: 新しいスタックフレーム、EPで親スコープにアクセス

---

## ラムダと Proc：関数を第一級市民として扱う
### lambda キーワードの動作

```ruby
def message_function
  str = "The quick brown fox"
  lambda do |animal|
    puts "#{str} jumps over the lazy #{animal}."
  end
end

function_value = message_function  # ←関数をデータとして保存
function_value.call('dog')         # ←後から実行
```

### メモリ管理の仕組み
- **スタック**: 一時的な値（メソッド実行中のみ有効）
- **ヒープ**: 永続的な値（参照がある限り有効）
- **lambda呼び出し**: スタックフレームをヒープにコピー

---

## スタック vs. ヒープメモリ
### RubyのProcオブジェクト構造

```
VALUE → RTypedData → rb_proc_t → rb_block_t → iseq
                    ↓             ↓
                   envval        EP → ヒープ上のスタックフレーム
                    ↓
                 is_lambda (true/false)
```

### lambdaとprocの違い
- **lambda**: `is_lambda = true`、厳密な引数チェック
- **proc**: `is_lambda = false`、柔軟な引数処理

---

## 実験とベンチマーク
### while ループ vs. each ブロック

```ruby
# whileループ（高速）
i = 1
while i <= 10
  sum += i
  i += 1
end

# eachブロック（読みやすい）
(1..10).each do |i|
  sum += i
end
```

### パフォーマンス結果
- **whileループ**: 0.44秒（100万回実行）
- **eachブロック**: 0.76秒（100万回実行）
- **差**: 約71%の実行時間増加

### 考察
- ブロック呼び出しはオーバーヘッドがある
- 実用的なアプリケーションでは影響は限定的
- 可読性とエレガンスのメリットが大きい

---

## 興味深い動作：変数の共有
### 後から変数を変更した場合

```ruby
def message_function
  str = "The quick brown fox"
  func = lambda { puts str }
  str = "The sly brown fox"  # ←後から変更
  func
end

function_value = message_function
function_value.call  # → "The sly brown fox" が出力される！
```

### なぜこうなる？
- EPがヒープ上のスタックフレームコピーを指すようリセット
- 同じスコープの変数は共有される
- 複数のlambdaは同じ環境を再利用

---

## まとめ
### Rubyブロックの本質
- **クロージャ**: 1975年のSussmanとSteeleの定義を忠実に実装
- **2つの人格**: メソッドのような独立性 + 親スコープへのアクセス
- **エレガントな構文**: 複雑な概念をシンプルに表現

### 実装の巧妙さ
- **rb_block_t**: コードと環境の組み合わせ
- **メモリ管理**: スタックからヒープへの効率的なコピー
- **最適化**: 同一スコープでの環境共有

### 実践的な意義
- **表現力**: Cライクな言語より簡潔で読みやすい
- **抽象化**: 反復処理の洗練された表現
- **関数型プログラミング**: 関数を第一級市民として扱える

---

## 質疑応答
### よくある質問
- Q: ブロックとlambdaの使い分けは？
- Q: パフォーマンスを重視する場合の選択肢は？
- Q: 他の言語のクロージャとの違いは？

ご質問をお待ちしています！

---

## 参考資料
- 「Rubyのしくみ」第8章（Pat Shaughnessy著）
- Gerald J. Sussman and Guy L. Steele, Jr., "Scheme: An Interpreter for Extended Lambda Calculus" (1975)
- Ruby 2.0 ソースコード（vm_core.h）
- ベンチマーク実験コード（このリポジトリ内）