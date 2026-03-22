# Railsのソースコード読む会 #1 saveメソッド
イベント: https://connpass.com/event/388348/

discord のチャンネルは #save-reading


# ActiveRecord `save` メソッド内部の処理フロー

## 対象コード

* `activerecord/lib/active_record/persistence.rb`
* `def save`

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/persistence.rb#L390-L394

---

# save の全体フロー

```text
save
└─ create_or_update
   ├─ _raise_readonly_record_error
   ├─ destroyed?
   ├─ _create_record
   │   ├─ attributes_for_create
   │   │   ├─ column_names
   │   │   ├─ delete_if
   │   │   └─ virtual?
   │   ├─ with_connection
   │   ├─ _returning_columns_for_insert
   │   └─ _insert_record
   │       ├─ from_database
   │       └─ Arel.insert
   │      _write_attribute
   │       ├─ write_from_user
   └─ _update_record
```

---

# create_or_update

## `_raise_readonly_record_error`

```ruby
_raise_readonly_record_error
```

* 更新できない属性(readonly)がある場合に例外を発生させるみたい
* `attr_readonly` で指定されたカラムなどが対象かと

---

## `destroyed?`

```ruby
destroyed?
```

### 役割

* レコードが削除済みかどうかを確認
* DB上では削除されているが、Rubyのインスタンスが残っているケースがあるためチェックしているみたい
  

---

# _create_record

```ruby
_create_record
```

レコード新規作成時の処理。

---

## `_` が付いたメソッドについて

```ruby
_create_record
_update_record
_insert_record
```

### 意味

* Rubyの **privateメソッド的な意味合い**
* Rails内部実装用のメソッドであることを示す慣習

Ruby言語仕様ではなく慣例見たい

---

# attributes_for_create

```ruby
attributes_for_create
```

INSERT対象のカラムを決定する処理。

---

## `column_names`

```ruby
column_names
```

### 役割

* テーブルのカラム一覧を取得

```ruby
Book.column_names
```

```ruby
["id", "title", "author", "created_at", "updated_at"]
```

---

## `delete_if`

```ruby
Array#delete_if
```

https://docs.ruby-lang.org/ja/latest/method/Array/i/delete_if.html

条件に合致するカラムを除外する。`reject!`と似ているが、配列を返すところが異なるみたい。

---

## `virtual?`

```ruby
column.virtual?
```

https://github.com/rails/rails/blob/f55588587d0ffcf1f940411de191ebbc4fbf5aad/activerecord/lib/active_record/connection_adapters/postgresql/column.rb#L28-L29

---

### 仮説

`virtual?` は：

* **DBに実在しないカラム**
* 仮想カラム(generated column など)

を判定している可能性がある。

つまり：

```text
DBに存在しないカラムは
INSERT対象から除外するためのチェック
```

と考えられる。たぶん。
検証はできていない。

---

# with_connection

```ruby
with_connection
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/connection_handling.rb#L311-L313

Connection Pool ってよくわかっていないけど、DB adapter に接続しているってこと...？

---

### 役割

* DB接続を取得してブロック内で利用する
* Connection Pool から接続を取り出す

```ruby
with_connection do |connection|
  ...
end
```

---

# _returning_columns_for_insert

```ruby
_returning_columns_for_insert
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/model_schema.rb#L436-L444

渡されたカラム名とスキーマにあるカラム名が合致しているかチェックしているっぽい。
合致しているものはinsert対象

---

### 仮説

* DBのschemaにあるカラムと
* 実際にINSERT後に取得したいカラム

を決定している処理。

```text
insert後に id を取得するため
returning id を生成しているはず
```

---

# _insert_record


```ruby
_insert_record
```

PKがある場合は、`_default_attributes`をセットしているっぽい

`_default_attributes`は`ActiveModel::Attribute.from_database`を使ってDBに保存されているカラムを整形してattributeに保存しているみたい

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/attributes.rb#L252-L264

---

## from_database

```ruby
from_database
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activemodel/lib/active_model/attribute.rb#L8-L10

---

### 役割

DBから取得した値をattributeに変換しているっぽい

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activemodel/lib/active_model/attribute.rb#L173

---

# Arel

```text
Arel
```

### 状態

* Rails内部のSQL構築ライブラリ
* Public APIなのか...？

* https://techracho.bpsinc.jp/kazz/2022_12_08/125126
* https://qiita.com/jnchito/items/630b9f038c87298b5756

---

## insert

```ruby
Arel::InsertManager
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/arel/insert_manager.rb#L21-L38

---

### 役割

SQLのINSERT文を組み立てているっぽい

こんな感じ
```sql
INSERT INTO books (title)
VALUES ('Rails Guide')
```

内部的に `insert_all` を呼び出している。知らなかった

---

## insert_all

```ruby
insert_all
```

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/relation.rb#L733-L735

---

### 役割

* 複数レコードのINSERTを行う
* bulk insert用API

---

## _write_attribute

attributesにも保存しているみたい

https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activerecord/lib/active_record/attribute_methods/write.rb#L41-L43

`write_from_user`
https://github.com/rails/rails/blob/c0c715b66e98dee1436f06ef682c42ab1de0c7e1/activemodel/lib/active_model/attribute_set.rb#L58-L62

---

# まとめ

```text
save
 ├─ readonlyチェック
 ├─ destroyedチェック
 ├─ INSERT or UPDATE 判定
 ├─ SQL生成(Arel)
 ├─ DBへカラムを保存
 └─ attributeを保存(存在すれば)
```
