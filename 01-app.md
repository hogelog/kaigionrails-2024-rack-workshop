# 第1章: Rackアプリケーションをいろいろな書き方で実装してみよう

この章では、Rackを使ってWebアプリケーションを実装してみます。Rackの基本的な仕組みを理解し、シンプルなアプリケーションから始めて、徐々にRackアプリケーションとはなんなのか、理解を深めてみましょう。

---

## 1. セットアップ

RubyGemsを使ってRackをインストールします。
ここではRackアプリケーションのためのコア機能を提供するrack gemと、Rackアプリケーションを起動するrackupコマンドを提供するrackup gemをインストールします。

```console
$ gem install rack rackup
```

gemをインストールしたらrackupコマンドが実行できることを確認します。
　
```console
$ rackup --help
Usage: rackup [ruby options] [rack options] [rackup config]
...
```

これでRackアプリケーションを書く準備は完了です。

---

## 2. シンプルなRackアプリケーションの作成

まずは、最もシンプルなRackアプリケーションを作成してみましょう。

### 手順

1. 新しいファイル `app.ru` を作成し、以下のコードを記述します。
   ```ruby
   class App
     def call(env)
       [
         200,
         {},
         ["hello"],
       ]
     end
   end

   run App.new
   ```
2. `call` メソッドは、Rackアプリケーションの環境情報とHTTPリクエストの情報が格納されたHash `env` を受け取り、ステータスコード、ヘッダー、ボディの配列を返します。

### ポイント

- `call` メソッドは必ず `[status, headers, body]` の形式でレスポンスを返す必要があります。
- `body` は文字列を含む配列を返します。
- .ru という拡張子は rackup config ファイルの拡張子で、`run` などのRack独自のメソッドが定義されたRubyのDSLで記述されます。
    - 例として app.ru という名前を指定しましたが、拡張子以外の部分の名前は任意です。rackupコマンドはデフォルトだと config.ru という設定ファイルを探すので、config.ru という名前にすると rackup コマンドを実行するだけでアプリケーションが起動します。

### 実行方法

ターミナルで以下のコマンドを実行します。

```console
$ rackup app.ru
```

ブラウザやcurlコマンド等で `http://localhost:9292` にアクセスして、`hello` と表示されれば成功です。

---

## 3. envの中身を確認してみる

Rackアプリケーションが受け取る `env` には、リクエストに関するさまざまな情報が含まれています。これを確認してみましょう。

### 手順

1. `call` メソッド内に `binding.irb` を挿入します。
   ```ruby
   class App
     def call(env)
       binding.irb
       # ...
     end
   end
   ```

### 実行方法

ターミナルでRackアプリケーションを実行して `http://localhost:9292` にアクセスすると、ターミナル上でirbセッションが開始されます。`env` を調べてみましょう。

```console
$ rackup app.ru
[2024-10-20 20:43:59] INFO  WEBrick 1.8.1
[2024-10-20 20:43:59] INFO  ruby 3.3.4 (2024-07-09) [arm64-darwin23]
[2024-10-20 20:43:59] INFO  WEBrick::HTTPServer#start: pid=68048 port=9292

From: /Users/hogelog/repos/hogelog/kaigionrails/01-app/app.ru @ line 3 :

    1: class App
    2:   def call(env)
 => 3:     binding.irb
    4:     [
    5:       200,
    6:       {"content-type" => "text/plain"},
    7:       ["hello"],
    8:     ]
irb(#<App:0x0000000101952048>):001> env
=>
{"GATEWAY_INTERFACE"=>"CGI/1.1",
 "PATH_INFO"=>"/",
 "QUERY_STRING"=>"",
 "REMOTE_ADDR"=>"::1",
 "REMOTE_HOST"=>"::1",
 "REQUEST_METHOD"=>"GET",
 "REQUEST_URI"=>"http://localhost:9292/",
...
```

`env` にはリクエストに関する情報が含まれており、リクエストメソッドやパス、ヘッダー、クエリパラメータなどが確認できます。様々なパスやメソッドでアクセスして、`env` の中身が変わることを確認してみましょう。

---

## 4. レスポンスヘッダーを設定する

次はHTTPレスポンスヘッダーを設定する方法を確認してみましょう。

### 手順

1. `call` メソッドで返すヘッダーに `"content-type" => "text/plain"` を追加します。
   ```ruby
   class App
     def call(env)
       [
         200,
         {"content-type" => "text/plain"},
         ["hello"],
       ]
     end
   end
   ```

### ポイント

- レスポンスヘッダーはハッシュで表現します。
- HTTP/1.xではレスポンスヘッダーのキーの大文字小文字は無視されますが、Rackアプリケーションでは小文字で統一するよう定められています。

### 実行方法

これまでと同様rackupコマンドで起動し、ブラウザの開発者ツールやcurlコマンドでレスポンスヘッダーを確認してみましょう。

```console
$ curl -v http://localhost:9292/
...
< HTTP/1.1 200 OK
< Content-Type: text/plain
< Content-Length: 5
< Server: WEBrick/1.8.1 (Ruby/3.3.4/2024-07-09)
< Date: Sun, 20 Oct 2024 12:00:00 GMT
< Connection: Keep-Alive
<
* Connection #0 to host localhost left intact
hello
```

またcontent-type以外にも任意のヘッダーを設定可能なので、他のヘッダーも設定してみてください。

---

## 5. ルーティングの実装

次に、リクエストのパスやメソッドに応じて、異なるレスポンスを返すようにしてみましょう。

### 手順

1. `call` メソッド内でリクエストの情報を取得します。
   ```ruby
   class App
     def call(env)
       method = env["REQUEST_METHOD"]
       path = env["PATH_INFO"]

       # メソッド、パスに応じた処理を実装
       # ...
     end
   end

   run App.new
   ```
2. メソッド、パスに応じた処理を実装してください。
    - `GET /` リクエストが来たら `It works!` を返すよう実装。
    - `GET /hello/foobar` のようなリクエストには `Hello foobar!` を返すよう実装。

### ポイント

- `env` からリクエストメソッド `REQUEST_METHOD` やリクエストパス `PATH_INFO` を取得できます。それらの値に応じて処理を分岐してみましょう。

### 実行方法

これまでと同様rackupコマンドで起動し、ブラウザの開発者ツールやcurlコマンドでレスポンスを確認してみましょう。

```console
$ curl http://localhost:9292/
It works!
$ curl http://localhost:9292/hello/hogelog
Hello hogelog!
```

適切に実装できていれば、上記したようなレスポンスが返ってくるはずです。

---

## 6. Rackのライブラリを活用する

先ほどまではenvに含まれた値をそのまま扱って処理を実装していました。ここではRackが提供する便利なクラスやメソッドを使ってコードを簡潔に記述してみましょう。

### 手順

1. `Rack::Request` と `Rack::Response` を使用して、以下のようにリクエストとレスポンスを扱ってみます。
   ```ruby
   require "rack/request"
   require "rack/response"

   class App
     def call(env)
       request = Rack::Request.new(env)
       case [request.request_method, request.path_info]
       in ["GET", "/"]
         Rack::Response.new("It works!", 200).finish
         # ...
       end
     end
   end

   run App.new
   ```
2. 上記実装に加え、パスに応じた処理を実装してください。
    - `GET /hello/foobar` のようなリクエストには `Hello foobar!` を返すよう実装。
    - `GET /`, `GET /hello/foobar` 以外のリクエストには404ステータスコードで `Not Found` を返すよう実装。

### ポイント

- `Rack::Request` はリクエストに対する便利なメソッドを提供します。
    - `Rack::Request.new` は `call(env)` で渡される env を引数として受け取ります。
- `Rack::Response` はレスポンスの組み立てを簡単にします。
    - `Rack::Response.new` は body, status, headers を引数として受け取ります。

### 実行方法

これまでと同様rackupコマンドで起動し、ブラウザの開発者ツールやcurlコマンドでレスポンスを確認してみましょう。

```console
$ curl http://localhost:9292/
It works!
$ curl http://localhost:9292/hello/hogelog
Hello hogelog!
$ curl -v http://localhost:9292/foobar
...
< HTTP/1.1 404 Not Found
< Content-Type: text/plain
< Content-Length: 9
< Server: WEBrick/1.8.1 (Ruby/3.3.4/2024-07-09)
< Date: Sun, 20 Oct 2024 15:24:17 GMT
< Connection: Keep-Alive
<
* Connection #0 to host localhost left intact
Not Found
```

正しく実装できていれば、以上のようにリクエストに応じたレスポンスが返ってくるはずです。

---

## 7. Sinatraを使ったRackアプリケーション

次に、Rubyの軽量なWebフレームワークであるSinatraを使って、同様の機能を実装してみましょう。

### 手順

1. Sinatraのベースクラスを継承した `App` クラスを以下のように定義します。
   ```ruby
   require "sinatra/base"

   class App < Sinatra::Base
     get "/" do
       "It works!"
     end

     get "/hello/:name" do
       "Hello #{params[:name]}"
     end
   end

   run App.new
   ```

### ポイント

- Sinatraは `Sinatra::Base` を継承したクラス内でDSLを使ってルートを定義します。
    - "GET /foo" のリクエストに対応する処理は `get '/foo' do ... end` のように記述します。
    - リクエストのクエリやURLパラメタなどは `params` ハッシュから取得できます。

### 実行方法

`gem install sinatra` を実行しsinatra gemをインストールした上で、これまでのようにrackupコマンドでアプリケーションを起動し、ブラウザやcurlコマンドで動作を確認してみましょう。

```console
$ curl http://localhost:9292/
It works!
$ curl http://localhost:9292/hello/hogelog
Hello hogelog!
```

Sinatraを使うことでRackアプリケーションを非常に簡潔に記述できることがわかります。

---

## 8. Railsを使ったRackアプリケーション

最後に、Railsを使って、同じ機能を実装してみましょう。Railsは起動するときなども通常rackupコマンドを使わず、意識することも少ないかもしれませんが、RailsアプリケーションがRackアプリケーションの形で提供されています。

ここではフルセットのRailsアプリケーションではなく、1ファイルのシンプルな形でRailsアプリケーション定義をしてrackupコマンドで起動してみます。

### 手順
1. 最小限のRails機能をロードするために、`action_controller/railtie` をロードします。
   ```ruby
   require "action_controller/railtie"
   ```
2. 最低限のRailsアプリケーション、コントローラ定義します。
   ```ruby
   require "action_controller/railtie"

   class App < Rails::Application
     config.secret_key_base = "secret_key_base"
     config.logger = Logger.new($stdout)
     Rails.logger  = config.logger

     routes.draw do
       root "apps#index"
       resources :apps, only: :show, path: "hello"
     end
   end

   class AppsController < ActionController::Base
     def index
       render plain: "It works!"
     end

     def show
       render plain: "Hello #{params[:id]}!"
     end
   end

   run Rails.application
   ```
3. 最後に、`run Rails.application` でアプリケーションを起動します。
   ```ruby
   run Rails.application
   ```

<details>
<summary>コード全体</summary>

```ruby
require "action_controller/railtie"

class App < Rails::Application
  config.secret_key_base = "secret_key_base"
  config.logger = Logger.new($stdout)
  Rails.logger  = config.logger

  routes.draw do
    root "apps#index"
    resources :apps, only: :show, path: "hello"
  end
end

class AppsController < ActionController::Base
  def index
    render plain: "It works!"
  end

  def show
    render plain: "Hello #{params[:id]}!"
  end
end

run Rails.application
```

</details>

### 実行方法

Railsを使用するために、必要なgemをインストールしておきます。

```console
gem install rails
```

rackupコマンドでアプリケーションを起動します。

```console
$ rackup 07-rails.ru
[2024-10-21 00:57:36] INFO  WEBrick 1.8.1
[2024-10-21 00:57:36] INFO  ruby 3.3.4 (2024-07-09) [arm64-darwin23]
[2024-10-21 00:57:36] INFO  WEBrick::HTTPServer#start: pid=74850 port=9292
```

ブラウザやcurlなどでアクセスし、以下のように正しくレスポンスを返せていることを確認してみましょう。

```console
$ curl http://localhost:9292/
It works!
$ curl http://localhost:9292/hello/hogelog
Hello hogelog!
```

このRackアプリケーションはRailsの機能のごく一部しか利用していませんが、RailsアプリケーションがRackアプリケーションを定義するものであることがわかります。

---

## 9. まとめ

この章では、Rackアプリケーションの基本的な構造から始めて、環境変数の確認、レスポンスヘッダーの設定、ルーティングの実装、Rackライブラリの活用、SinatraやRailsを使った実装までを学びました。これらの基礎を押さえることで、さまざまなフレームワークやライブラリを使ったWebアプリケーションの構築に役立てることができます。

これまでの内容を通じて、以下のポイントを理解できたと思います。

- **Rackの基本構造**: `call` メソッドで `[status, headers, body]` を返す。
- **Rackアプリケーション引数 `env` の利用**: リクエストに関する情報を取得できる。
- **Rackのライブラリの活用**: `Rack::Request` や `Rack::Response` といった便利なクラスの提供。
- **SinatraやRailsとの組み合わせ**: SinatraやRailsといったフレームワークとRackの関係性。　

---

次の章では、Rackミドルウェアの作成や、ミドルウェアを使った機能拡張について学んでいきます。
