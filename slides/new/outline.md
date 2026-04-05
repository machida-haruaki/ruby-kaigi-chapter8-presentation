# 全体構成（outline） - FIX版

## 発表概要

- **タイトル**: Rubyのブロックとクロージャ ― 内部実装を生のコードで読む
- **時間**: 45分（発表40分 + Q&A 5分）
- **対象**: 8章を読んできた初〜中級Rubyist
- **差別化**: 本はRuby 2.0ベースで抽象的 → 実際のRuby 3.4ソースコードとメモリの動きで深掘りする

---

## 構成

| # | タイトル | 時間 | ファイル |
|---|---|---|---|
| 1 | ブロックの3つの性質（おさらい） | 4分 | section1.md |
| 2 | ブロック = クロージャ | 3分 | section2.md |
| 3 | 内部構造の地図 | 8分 | section3.md |
| 4 | 環境がなぜ生き続けるのか | 18分 | section4.md |
| 5 | 実験（A + B） | 5分 | section5.md |
| Q&A | | 7分 | - |
| **合計** | | **45分** | |

---

## 各セクションの詳細

### セクション1: ブロックの3つの性質（おさらい）
**目的**: 前提確認・導入。みんな読んできているので軽く流す。

**話すこと**:
- 外側変数アクセス
- メソッド終了後も環境保持
- 環境の共有

**話さないこと**:
- 各性質の詳細な説明（本で読んできた前提）
- クロージャとの関係（セクション2で話す）

---

### セクション2: ブロック = クロージャ
**目的**: 「クロージャ」という言葉の定義を共有する。

**話すこと**:
- クロージャの定義（コード + 環境への参照）
- Rubyブロックがクロージャであることの確認
- 一言だけ歴史（1964年 Landin）

**話さないこと**:
- 他言語のクロージャ比較
- 深い理論

---

### セクション3: 内部構造の地図
**目的**: セクション4で登場する構造体の「地図」を頭に入れる。

**話すこと**:
- 5つの構造体の名前・主要フィールド・責務
- 包含関係（誰が誰を持っているか）
- 誰がどのタイミングで登場するか（概要）
- `rb_proc_t` の `is_lambda` フラグに一言触れる（lambdaとprocはこれだけの違い）

**登場する構造体**:

```
rb_control_frame_t   実行中のスタックフレーム
rb_captured_block    クロージャの本体（コード + 環境への参照）
rb_block             ブロックの種類を抽象化するラッパー
rb_proc_t            RubyのProcオブジェクトの実体
rb_env_t             ヒープ上に逃がした変数領域
```

**包含関係**:
```
rb_proc_t
  ├─ is_lambda（これ1つでlambdaかprocかが決まる）
  └─ rb_block
       └─ rb_captured_block
            ├─ self
            ├─ ep ──→ rb_env_t（ヒープ上の変数領域）
            └─ code.iseq（実行するバイトコード）
```

**話さないこと**:
- 各フィールドの詳細な動作（セクション4で話す）
- rb_block_typeの全種類（mainはiseq）
- is_lambdaによる引数・returnの動作の詳細

---

### セクション4: 環境がなぜ生き続けるのか（メイン）
**目的**: スタック→ヒープコピーのメカニズムを、メモリ図とソースコードで理解する。

**追うコード例**:
```ruby
def create_counter
  count = 0
  lambda { count += 1 }
end

counter = create_counter
counter.call  # => 1
counter.call  # => 2
```

**話すこと（ステップ順）**:

① シンプルなブロックの場合（エスケープしない）
- `each { |i| puts i }` のような使い捨てブロック
- `rb_captured_block` がスタック上に作られ、`ep` もスタックを指す
- `vm_push_frame` で新しいフレームを作りブロック実行
- メソッド終了 → スタックごと消える（ヒープコピー不要）

② lambdaの場合（エスケープが必要）
- `lambda {}` を書いた瞬間に `vm_make_env_each` が走る
- `rb_env_t` をヒープに確保 → `count` をコピー → `ep` をヒープに付け替え → `VM_ENV_FLAG_ESCAPED` セット
- メソッド終了 → スタックフレームは消えるがヒープの `rb_env_t` は残る
- `counter.call` → `ep` 経由でヒープ上の `count` にアクセス

③ ネストしたブロックの場合（epチェーン）
- ブロックの中にブロックがある場合、epは内側→外側へとチェーンでつながる
- `vm_make_env_each` が再帰的に外側フレームまで遡ってヒープコピー
- 戻りながら `VM_FORCE_WRITE_SPECIAL_CONST` で各epのspecvalを更新

**図**:
- ①②それぞれのメモリ状態の比較（ASCIIアート）
- ②のステップごとのメモリ変化（ASCIIアート）
- ③のepチェーンと再帰処理の図

**話さないこと**:
- YJITとの連携（`rb_yjit_invalidate_ep_is_bp`）
- GCとの関係（ライトバリア等）

---

### セクション5: 実験（A + B）
**目的**: セクション4の内部実装の「答え合わせ」。シンプルに確認するだけ。

**実験A**: lambda作成後に変数を変更 → lambdaに反映される
```ruby
def message_function
  str = "The quick brown fox"
  func = lambda { |animal| puts "#{str} jumps over the lazy #{animal}." }
  str = "The sly brown fox"   # ← lambda作成後に変更
  func
end
message_function.call('dog')
# => "The sly brown fox jumps over the lazy dog."
```
→ なぜ: 同じ`rb_env_t`上の`str`を`ep`経由で参照しているから

**実験B**: 2つのlambdaが同じ変数を共有
```ruby
def create_counter
  count = 0
  increment = lambda { count += 1 }
  read      = lambda { count }
  [increment, read]
end
inc, read = create_counter
inc.call; inc.call
puts read.call  # => 2
```
→ なぜ: 2つの`rb_captured_block`が同じ`rb_env_t`を`ep`経由で指しているから

**話さないこと**:
- 実験結果の詳細な解説（セクション4で説明済みなので一言でOK）

---

## ファイル作成ルール

- `outline.md`: この構成ファイル。FIX後は変更しない
- `section1.md` 〜 `section5.md`: 各セクション独立。他のファイルに影響しない
- `final.md`: 全セクションFIX後に一度だけ統合。目次・インデックスはここで作成
