# ActiveRecord `find` / `find_by` メソッド内部の処理フロー

## 対象コード

- `activerecord/lib/active_record/core.rb`
- `def find`, `def find_by`

---

## 全体フロー

```text
find(*ids)
└─ cached_find_by
   └─ cached_find_by_statement
      └─ StatementCache.create
         ├─ Params.new (bind → Substitute.new)
         └─ BindMap.new(binds)
   └─ statement.execute
      └─ where(wheres).limit(1) の結果を返す

find_by(*args)
└─ 引数を hash に変換 (例: {"title"=>"hoge"})
└─ cached_find_by(hash.keys, hash.values)
   └─ (以降 find と同じ流れ)
```

---

## `find`

```ruby
def find(*ids)
  # ...
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/core.rb#L270

内部で `cached_find_by` を呼び出している。

特別な取得方法をしていると思っていたが、結局 `where(wheres).limit(1)` しているだけだった。

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/core.rb#L451

---

## `find_by`

```ruby
def find_by(*args)
  # ...
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/core.rb#L284

ここも結局 `cached_find_by(hash.keys, hash.values)` を呼んでいる。

途中で引数を hash に変換する処理がある。

```ruby
hash
{"title"=>"hoge"}
```

`find` と `find_by` はどちらも最終的に同じ `cached_find_by` → `where(wheres).limit(1)` という流れ。

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/core.rb#L327

---

## `cached_find_by` と `StatementCache`

`cached_find_by` の中では `cached_find_by_statement` を呼んで `StatementCache` を取得し、`statement.execute` で実行している。

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/core.rb#L454

### `StatementCache.create`

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/core.rb#L405

`StatementCache.create` の中で `Params.new` が呼ばれ、`params.bind` を呼ぶと `Substitute.new` が返る。

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/statement_cache.rb#L109-L111

### `StatementCache` の中身

`StatementCache` は対象の model と実行するクエリを格納している。

```ruby
#<ActiveRecord::StatementCache:0x...
 @bind_map=
  #<ActiveRecord::StatementCache::BindMap:0x...
   @bound_attributes=
    [#<ActiveRecord::Relation::QueryAttribute:0x...
      @name="id",
      @type=#<ActiveRecord::ConnectionAdapters::SQLite3Adapter::SQLite3Integer:0x...>,
      @value=#<ActiveRecord::StatementCache::Substitute:0x...>>,
     #<ActiveModel::Attribute::WithCastValue:0x...
      @name="LIMIT",
      @value=1>],
   @indexes=[0]>,
 @model=Book(...),
 @query_builder=
  #<ActiveRecord::StatementCache::PartialQuery:0x...
   @values=
    ["SELECT", " ", "\"books\"", ".", "*", " FROM ",
     "\"books\"", " WHERE ", "\"books\"", ".", "\"id\"", " = ",
     #<ActiveRecord::StatementCache::Substitute:0x...>,
     " ", "LIMIT ",
     #<ActiveRecord::StatementCache::Substitute:0x...>]>>
```

`@query_builder` を見ると、SQL の各パーツが配列で保持されていて、`Substitute` がプレースホルダーになっている。

---

## `Substitute` の役割

`where(wheres).limit(1)` の `wheres` は以下のような形になっている。

```ruby
{"id" => #<ActiveRecord::StatementCache::Substitute:0x...>}
```

これは SQL のバインドパラメータのようなもの。

`Book.find(2)` とすると、`Substitute` の部分に `2` が入るイメージ。

つまり `where(wheres).limit(1)` はまだ SQL を組み立てているだけで、実際に実行しているのは `statement.execute` の方。

---

## `where(wheres).limit(1)` と `statement.execute` の関係

- `where(wheres).limit(1)` → SQL のテンプレートを組み立てる（`StatementCache` に格納）
- `statement.execute` → 実際の値をバインドして SQL を実行する

`StatementCache` はクエリのテンプレートをキャッシュしておくことで、同じ形のクエリを何度も組み立て直さなくて済むようにしている仕組みっぽい。

---

## まとめ

- `find` も `find_by` も、内部的には `cached_find_by` → `where(wheres).limit(1)` → `statement.execute` という同じ流れ
- `find` は `id` で検索、`find_by` は任意のカラムで検索という違いだけ
- `StatementCache` を使って SQL テンプレートをキャッシュし、`Substitute` をプレースホルダーとしてバインドする仕組み
- 特別な取得方法ではなく、`where` + `limit` で愚直にやっている

---

## まだよくわかっていない点

### `ActiveRecord::StatementCache` はオブジェクトをキャッシュしている？

取得したオブジェクトではなく、SQL のテンプレート（クエリの構造）をキャッシュしているように見える。実行結果のキャッシュではなさそう。

### `ActiveRecord::Querying::QUERYING_METHODS`

`delegate` がめちゃくちゃ多い。Model クラスから `where` や `find_by` などを直接呼べるようにするためのもの。

### `find` の `super` が呼ばれるパターン

`core.rb` の `find` 内で `super` が呼ばれるケースがあるが、どういう条件で呼ばれるのかまだ追えていない。
