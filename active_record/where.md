# ActiveRecord `where` メソッド内部の処理フロー

## 対象コード

- `activerecord/lib/active_record/relation/query_methods.rb`
- `def where`

---

## 全体フロー

```ruby
where(opts)
└─ spawn
   └─ where!(opts)
      └─ build_where_clause(opts)
         └─ Relation::WhereClause.new(parts)
            └─ Arel::Nodes を組み立てる
```

## `where`

```ruby
def where(*args)
  if args.empty?
    WhereChain.new(spawn)
  elsif args.length == 1 && args.first.blank?
    self
  else
    spawn.where!(*args)
  end
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/relation/query_methods.rb#L1033-L1041

* 引数がなければ `WhereChain` を返す
  * `where.not(...)` のようなメソッドチェイン用
* 条件があれば `spawn.where!` を呼んでいる

---

## `spawn`

```ruby
def spawn
  already_in_scope?(model.scope_registry) ? model.all : clone
end
```

* `already_in_scope?` が絡んでいて、スコープ中かどうかで `model.all` か `clone` を返している 

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/relation/spawn_methods.rb#L9-L11

ここはまだ完全には追い切れていないが、現在の Relation をそのまま書き換えないようにしているっぽい。
where を呼び出すと内部的には `model.all` しているってことか？意外と愚直だと思った。でもそうしないと条件で絞れないのかな。

---

## `where!`

```ruby
def where!(*args)
  self.where_clause += build_where_clause(args)
  self
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/relation/query_methods.rb#L1043-L1046

* `build_where_clause(args)` で条件オブジェクトを作り、それを既存の `where_clause` に足している
* `where.where(...)` とチェインできるのは、`where_clause` が追加されていくからでは？

### `self.where_clause`

ここで専用のメソッドとして定数に置かれ
```ruby
CLAUSE_METHODS = [:where, :having, :from]
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/relation.rb#L62


```ruby
Relation::VALUE_METHODS.each do |name|
  method_name, default =
    case name
    when *Relation::MULTI_VALUE_METHODS
      ["#{name}_values", "FROZEN_EMPTY_ARRAY"]
    when *Relation::SINGLE_VALUE_METHODS
      ["#{name}_value", name == :create_with ? "FROZEN_EMPTY_HASH" : "nil"]
    when *Relation::CLAUSE_METHODS
      ["#{name}_clause", name == :from ? "Relation::FromClause.empty" : "Relation::WhereClause.empty"]
    end

  class_eval <<-CODE, __FILE__, __LINE__ + 1
    def #{method_name}                     # def includes_values
      @values.fetch(:#{name}, #{default})  #   @values.fetch(:includes, FROZEN_EMPTY_ARRAY)
    end                                    # end

    def #{method_name}=(value)             # def includes_values=(value)
      assert_modifiable!                   #   assert_modifiable!
      @values[:#{name}] = value            #   @values[:includes] = value
    end                                    # end
  CODE
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/relation/query_methods.rb#L174-L176


ここの `method_name` に `where` が入り、
```ruby
@values.fetch(:where_clause, "Relation::WhereClause.empty")
```

となっていそう。`"Relation::FromClause.empty"` がいつ使われるのかはわかんない。

---

## `build_where_clause`

ここの最後で `Relation::WhereClause.new(parts)` を返している。

```ruby
def build_where_clause(opts, rest = []) # :nodoc:
  #...

  Relation::WhereClause.new(parts)
end
```

`Relation::WhereClause` は

```ruby
[#<Arel::Nodes::Equality:0x00000001234700a8
  #...
```

というような `Arel::Nodes` を

```ruby
#<ActiveRecord::Relation::WhereClause:0x00000001234fc828
 @predicates=
  [#<Arel::Nodes::Equality:0x00000001234700a8
    #...
```

というように `ActiveRecord::Relation::WhereClause` のインスタンスとしてラップしているっぽい。
そうすることで、`WhereClause` クラスにある +, -, or などのメソッドを使って `Arel::Nodes` の条件(left, right)を操作できるようにしているのでは？

---

## `where(title: "hoge")` の場合

`Arel::Nodes::Equality` だった。

```ruby
#<Arel::Nodes::Equality
  @left= ... name="title"
  @right= ... @value_for_database="hoge">
```

* `@left` はカラム側
* `@right` は値側
* `@value_for_database` に DB 向けの値が入っているみたい

つまり、`title = "hoge"` という条件は、

* 左辺: `books.title`
* 右辺: `"hoge"` を DB 用に型変換した値っぽい

という形の Arel ノードとして表現されているのでは...？
ここら辺は `Arel::Nodes` を詳しく見てみないとわかんないなぁ

---

## `where(id: ..10)` の場合

Range を渡したときは、`Arel::Nodes::LessThanOrEqual` になっていた。

```ruby
#<Arel::Nodes::LessThanOrEqual
  @left= ... name="id"
  @right= ... @value_for_database=10>
```

`where(id: ..10)` は単純な `Equality` ではなく、
**範囲に応じた比較演算のノード**に変換されていることがわかる。 

つまり ActiveRecord は入力値の型や形に応じて、適切な Arel ノードを組み立てているみたいだ。すげー

---

## `where.where(...)` はどうなるか

`where(title: "hoge").where(author: 2)` のように繋げたとき、`@predicates` に条件が2つ並んでいた。

```ruby
@predicates = [
  #<Arel::Nodes::Equality ... title ...>,
  #<Arel::Nodes::Equality ... author ...>
]
```

`where` をメソッドチェインで繋げると `WhereClause` の中に条件ノードが追加されていくみたい。

さっき見た `self.where_clause += build_where_clause(args)` の部分だね
```ruby
def where!(*args)
  self.where_clause += build_where_clause(args)
```

---

## `where.not`

`where.not(title: "hoge")` では、`Arel::Nodes::NotEqual` になった

```ruby
#<Arel::Nodes::NotEqual
  @left= ... name="title"
  @right= ... @value_for_database="hoge">
```

否定用の Arel ノードがあるんだなー。

ここの `where_clause.invert` で `Arel::Nodes::NotEqual` にしているっぽいな

```ruby
def not(opts, *rest)
  where_clause = @scope.send(:build_where_clause, opts, rest)

  @scope.where_clause += where_clause.invert

  @scope
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/relation/query_methods.rb#L49-L55

`invert` メソッドの `inverted_predicates = [ invert_predicate(predicates.first) ]` で

```ruby
def invert
  if predicates.size == 1
    inverted_predicates = [ invert_predicate(predicates.first) ]
  else
    inverted_predicates = [ Arel::Nodes::Not.new(ast) ]
  end

  WhereClause.new(inverted_predicates)
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/relation/where_clause.rb#L85-L93

`invert_predicate(predicates.first)` は `node.invert` を返して

```ruby
def invert_predicate(node)
  case node
  when NilClass
    raise ArgumentError, "Invalid argument for .where.not(), got nil."
  when String
    Arel::Nodes::Not.new(Arel::Nodes::SqlLiteral.new(node))
  else
    node.invert
  end
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/relation/where_clause.rb#L167-L176

`Arel::Nodes::NotEqual.new(left, right)` を返している！
なるほどー

```ruby
def invert
  Arel::Nodes::NotEqual.new(left, right)
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/arel/nodes/equality.rb#L10-L12

---

## まとめ

`where` は `Arel::Nodes` を使って条件式をオブジェクトとして組み立てている。
その中心にあるのが `WhereClause` で、内部には `@predicates` として条件ノードが保持されている。

* 等価比較なら `Arel::Nodes::Equality`
* 範囲なら `Arel::Nodes::LessThanOrEqual`
* 否定なら `Arel::Nodes::NotEqual`

のように、入力に応じてノードが変わる。 

なので、ActiveRecord の `where` は SQLの文字列を組み立てるというよりは、条件のASTを組み立てているとして理解するとよさそう。 

---

## まだよくわかっていない点

### `already_in_scope?(model.scope_registry)`

`spawn` のこの分岐はまだ読み切れていない。

```ruby
already_in_scope?(model.scope_registry)
```

とりあえず `model.all` と `clone` を返すことはわかったが、`scope_registry` って何？となっている
