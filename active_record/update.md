# ActiveRecord `update` メソッド内部の処理フロー

## 対象コード

* `activerecord/lib/active_record/persistence.rb`
* `def update`

---

## 全体フロー

```text
update(attributes)
└─ with_transaction_returning_status
   ├─ assign_attributes(attributes)
   └─ save
      └─ create_or_update
         └─ _update_record
            └─ _update_row
               └─ self.class._update_record(values, constraints)
                  └─ predicate_builder で WHERE 条件を組み立てる
                  └─ Arel::UpdateManager で UPDATE 文を組み立てる
                  └─ connection.update(...)
                     └─ query cache を汚す処理
                     └─ super
                        └─ DatabaseStatements#update
                           └─ to_sql_and_binds
                           └─ exec_update
```

`update(attributes)` は `assign_attributes(attributes)` のあとに `save` を呼び、その `save` が `create_or_update` を経由して、更新系では `_update_record` に進む。

---

## `update(attributes)`

```ruby
def update(attributes)
  with_transaction_returning_status do
    assign_attributes(attributes)
    save
  end
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/persistence.rb#L563-L570

---

## `create_or_update`

```ruby
def create_or_update(**, &block)
  _raise_readonly_record_error if readonly?
  return false if destroyed?

  result = new_record? ? _create_record(&block) : _update_record(&block)
  result != false
end
```

ここの `_update_record` が呼び出される

---

## `_update_record`

```ruby
def _update_record(attribute_names = self.attribute_names)
  attribute_names = attributes_for_update(attribute_names)

  if attribute_names.empty?
    affected_rows = 0
    @_trigger_update_callback = true
  else
    affected_rows = _update_row(attribute_names)
    @_trigger_update_callback = affected_rows == 1
  end

  @previously_new_record = false

  yield(self) if block_given?

  affected_rows
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/persistence.rb#L900-L916

* 更新対象がなければ SQL を流さない
* 更新対象があれば `_update_row` を呼んでいる

---

## `_update_row`

```ruby
def _update_row(attribute_names, attempted_action = "update")
  self.class._update_record(
    attributes_with_values(attribute_names),
    _query_constraints_hash
  )
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/persistence.rb#L884-L889

---

## クラス側の `_update_record(values, constraints)`

```ruby
def _update_record(values, constraints) # :nodoc:
  constraints = constraints.map { |name, value| predicate_builder[name, value] }

  default_constraint = build_default_constraint
  constraints << default_constraint if default_constraint

  if current_scope = self.global_current_scope
    constraints << current_scope.where_clause.ast
  end

  um = Arel::UpdateManager.new(arel_table)
  um.set(values.transform_keys { |name| arel_table[name] })
  um.wheres = constraints

  with_connection do |c|
    c.update(um, "#{self} Update")
  end
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/persistence.rb#L263-L280

* WHERE 句に使う条件を `predicate_builder` で Arel の条件式に変換
* `Arel::UpdateManager` で UPDATE 文を組み立てているっぽい
* 最後に `connection.update` している。

---

## `predicate_builder`

```ruby
constraints = constraints.map { |name, value| predicate_builder[name, value] }
```

* Arel の WHERE 条件に変換しているっぽい

---

## `Arel::UpdateManager`

```ruby
um = Arel::UpdateManager.new(arel_table)
um.set(values.transform_keys { |name| arel_table[name] })
um.wheres = constraints
```

* `UPDATE users SET ... WHERE ...` のような SQL を Arel オブジェクトとして組み立てているっぽい

active_record は Arel::xxxManager を結構みるな。Arel が肝っぽいよなぁ

---

## `connection.update`

```ruby
with_connection do |c|
  c.update(um, "#{self} Update")
end
```

* Arel オブジェクトを DB アダプタに渡して実行しているっぽい

ただし、ここで直接 `DatabaseStatements#update` に行くわけではなく、query cache のラッパーが入る。実行速度の問題だろう。
メソッド grep しても見つからないよー。ってなっていた。

---

## `dirties_query_cache`

```ruby
def dirties_query_cache(base, *method_names)
  method_names.each do |method_name|
    base.class_eval <<-end_code, __FILE__, __LINE__ + 1
      def #{method_name}(...)
        if pool.dirties_query_cache
          ActiveRecord::Base.clear_query_caches_for_current_thread
        end
        super
      end
    end_code
  end
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/connection_adapters/abstract/query_cache.rb#L20-L31

* `update` 実行前に `ActiveRecord::Base.clear_query_caches_for_current_thread` で キャッシュをクリアしているっぽい
* その後 `super` で `update` 処理を呼んでいる

なんのキャッシュが残っているのだろう？

---

## `ActiveRecord::ConnectionAdapters::DatabaseStatements#update`

```ruby
def update(arel, name = nil, binds = [])
  sql, binds = to_sql_and_binds(arel, binds)
  exec_update(sql, name, binds)
end
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/connection_adapters/abstract/database_statements.rb#L206-L209

### 役割

* Arel を SQL 文字列と bind に変換している
* `exec_update` で DB に UPDATE を実行させているみたい

---

## まとめ

ざっくりとした `update(attributes)` の流れ

1. `save` が `create_or_update` を呼ぶ
2. 既存レコードなのでインスタンスメソッドの `_update_record` に進む
3. `_update_row` が更新値と WHERE 条件を取り出す
4. クラス側の `_update_record(values, constraints)` が Arel で UPDATE 文を組み立てる
5. `connection.update` が query cache をクリアしつつ SQL を実行する
6. `DatabaseStatements#update` が `to_sql_and_binds` → `exec_update` で DB に SQL を実行する

---
