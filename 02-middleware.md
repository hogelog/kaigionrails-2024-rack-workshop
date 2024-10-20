# 第2章: Rackミドルウェアをいろいろな書き方で実装してみよう

この章では、Rackミドルウェアの基本的な仕組みを理解し、さまざまな方法でミドルウェアを実装してみます。ミドルウェアを活用することで、アプリケーションの機能を拡張したり、共通の処理を分離することができます。

---

## 1. Rackミドルウェアとは
Rackミドルウェアは、Rackアプリケーションのリクエストとレスポンスの間に介在し、リクエストを加工したり、レスポンスに処理を加えたりするコンポーネントです。ミドルウェアをチェーンのようにつなげることで、複雑な処理をシンプルに組み立てることができます。

---

## 2. シンプルなミドルウェアの作成
まずは、簡単なミドルウェアを作成してみましょう。レスポンスヘッダーにカスタムヘッダーを追加するミドルウェアを実装します。

### 手順
1. 新しいファイル `middleware.ru` を作成します。
2. 適当なRackアプリケーションと、Rackミドルウェアを定義します。
   ```ruby
   class App
     def call(env)
       [200, {}, ["hello"]]
     end
   end

   class Middleware
     def initialize(app, name)
       @app = app
       @name = name
     end
   
     def call(env)
       status, headers, body = @app.call(env)
       headers["hello"] = @name
       [status, headers, body]
     end
   end

   use Middleware, "rails"
   run App.new
   ```

### ポイント

- ミドルウェアは、`initialize` と `call` メソッドを持つクラスで定義します。
- ミドルウェアは `initialize` メソッドで、次のアプリケーションや任意の引数を受け取ります。
- `call` メソッドで、リクエストを受け取って次のアプリケーションに渡し、レスポンスを返します。
    - ここで、リクエストやレスポンスに加工を加えることができます。
- `use` キーワードを使ってミドルウェアを登録します。

### 実行方法

ターミナルで以下のコマンドを実行します。

```console
$ rackup middleware.ru
```

ブラウザやcurlで `http://localhost:9292` にアクセスし `hello` ヘッダーが追加されていることがわかります。

```console
$ curl -v http://localhost:9292
...
< HTTP/1.1 200 OK
< Hello: rails
< Content-Length: 5
< Server: WEBrick/1.8.1 (Ruby/3.3.4/2024-07-09)
< Date: Sun, 20 Oct 2024 16:17:46 GMT
< Connection: Keep-Alive
< 
* Connection #0 to host localhost left intact
hello
```

---

## 3. Rackの標準ミドルウェアを活用する
Rackが提供する標準のミドルウェアを使って機能を拡張してみましょう。ここではリクエストの処理時間を測定する `Rack::Runtime` と、Basic認証を行う `Rack::Auth::Basic` を使用します。

### 手順

1. 必要なライブラリを読み込みます。
   ```ruby
   require "rack/runtime"
   require "rack/auth/basic"
   ```
2. ミドルウェアを追加します。
   ```ruby
   use Rack::Runtime
   use Rack::Auth::Basic do |username, password|
     username == "rubyist" && password == "onrack"
   end
   ```

### ポイント
- ミドルウェアは上から順に適用されます。

### 実行方法

これまでと同様にrackupコマンドで起動し、ブラウザやcurlでアクセスしてみましょう。

```console
$ curl -v http://localhost:9292/
...
< HTTP/1.1 401 Unauthorized
...
```

認証情報なしでは401 Unauthorizedが返されます。

```console
$ curl -v http://rubyist:onrack@localhost:9292/
...
< HTTP/1.1 200 OK
< X-Runtime: 0.000057
< Hello: rails
< Content-Length: 6
< Server: WEBrick/1.8.1 (Ruby/3.3.4/2024-07-09)
< Date: Sun, 20 Oct 2024 16:24:30 GMT
< Connection: Keep-Alive
< 
* Connection #0 to host localhost left intact
hello!
```

認証情報を付与すると、`hello!` が表示されます。また、レスポンスヘッダーに `X-Runtime` ヘッダーが追加されていることがわかります。


---

## 4. まとめ

この章では、Rackミドルウェアの基本的な実装方法から、Rackの標準ミドルウェアの活用を学びました。
ミドルウェアを使うことで、共通の処理をアプリケーションから分離し、再利用性や保守性を向上させることができます。

ここではごくシンプルなRackアプリケーションに対し簡単なRackミドルウェア、Rack標準ミドルウェアのみ適用しました。しかしもちろんRackミドルウェアは非常に複雑なRailsアプリケーションなどに対しても同様に適用可能ですし、Rack標準以外にも様々なライブラリがRackミドルウェアの形で世の中に公開されています。



- **ミドルウェアの基本構造**: `initialize` と `call` メソッドを持つクラス。
- **Rackの標準ミドルウェア**: `Rack::Runtime` や `Rack::Auth::Basic` などの活用方法。
- **Rackアプリケーションとミドルウェアによる広がり**: Rackアプリケーションとミドルウェアによる機能の拡張やライブラリの広がり。

ここまでの理解により、Railsアプリケーションの開発でも触れるRackへの親しみがましたのではないでしょうか。

---

次の章では、Rackアプリケーションを起動するためのRackサーバについて学んでいきます。
