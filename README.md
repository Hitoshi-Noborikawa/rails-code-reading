# Railsのソースコード読む会
このリポジトリは「Railsのソースコード読む会」というイベントのメモです。

https://connpass.com/event/388374/

## Railsのソースコード読むときに便利なメソッド
`.source_location`

https://docs.ruby-lang.org/ja/latest/method/Method/i/source_location.html

例) クラスメソッドを探す場合
```
self.class.method(:_default_attributes).source_location

["/Users/xxx/ruby/3.3.4/lib/ruby/gems/3.3.0/gems/activerecord-8.0.2/lib/active_record/attributes.rb",
 241]
```
