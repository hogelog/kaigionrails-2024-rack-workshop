# 第3章: 自作のRackサーバを実装してみよう
この章ではRackサーバを自前で実装してみます。既存のRackサーバ（Puma, Pitchfork, Unicorn, WEBRickなど）を使わずに自作のサーバを作ることで、Rackの内部的な仕組みやRackサーバの基本的な動作原理を理解することができます。

---

## 1. シンプルなRackサーバの作成

まずは最もシンプルなRackサーバを実装してみましょう。以下の手順に従って、サーバを作成していきます。

ここではまず、ごく一部のリクエストタイプ（GETメソッド）にしか対応していない単純なRackサーバを実装してみましょう。

### 手順
1. **必要なライブラリの読み込み**
   新しいファイル `server.ru` を作成し、以下ライブラリを追加します。
   ```ruby
   require "socket"
   require "logger"
   ```
   - `socket` はソケット通信を扱う標準ライブラリです。
       - <https://docs.ruby-lang.org/ja/latest/library/socket.html>
   - `logger` はログを記録するための標準ライブラリです。
      - <https://docs.ruby-lang.org/ja/latest/library/logger.html>
   - 必要に応じてここで指定した以外のライブラリを追加しても問題ありません。
2. **アプリケーションクラスの定義**
   ごく単純なRackアプリケーションを定義します。
   ```ruby
   class App
     def call(env)
       if env["PATH_INFO"] == "/"
         [200, {}, ["It works!"]]
       else
         [404, {}, ["Not Found"]]
       end
     end
   end
   ```
3. **サーバークラスの定義**
   自作のサーバークラス `SimpleServer` を定義します。

   ```ruby
   class SimpleServer
     def self.run(app, **options)
       new(app, options).start
     end

     def initialize(app, options)
       @app = app
       @options = options
       @logger = Logger.new($stdout)
     end

     def start
       # サーバーのメインループ
     end
   end
   ```
   - `self.run` クラスメソッドでサーバーを起動します。
   - `initialize` メソッドでアプリケーションとオプションを受け取ります。
   - `start` メソッドでクライアントからの接続を受けてレスポンスを返す、サーバーのメインループを実装します。
4. **ソケットサーバーの起動**
   `start` メソッド内で、以下の処理を行います。

   - `TCPServer` を使って指定されたポートでサーバーを起動します。
       - <https://docs.ruby-lang.org/ja/latest/class/TCPServer.html>
   - 無限ループでクライアントからの接続を待ち受けます。
   - クライアントからの接続があったら、リクエストを読み込みます。

   ```ruby
   def start
     @logger.info "SimpleServer starting..."
     server = TCPServer.new(@options[:Port].to_i)
     loop do
       client = server.accept

       # リクエストの受信と解析
       # ...
     end
   end
   ```
5. **リクエストの解析**
   - クライアントから送られてきたリクエストライン（例: `"GET /hello HTTP/1.1"`）を読み込みます。
      ```ruby
      client = server.accept

      # リクエストラインの解析
      request_line = client.gets&.chomp
      # ...
      path = # ...
      ```
     - クライアントの切断時に `nil` が返ることを考慮して、`&.` 演算子を使っています。
   - リクエストヘッダーを読み込みます。
      ```ruby
      # リクエストラインの解析
      request_line = client.gets&.chomp
      # ...
      path = # ...

      request_headers = {}
      loop do
        header_field = client.gets.chomp
        # ヘッダーの解析
        # ...
      end
      ```
   - 必要であれば、リクエストボディを読み込みます。
   - 参考: GETメソッドのリクエストはクライアントから以下のような `"リクエストライン\r\n（任意行繰り返されるヘッダフィールド\r\n\nリクエストボディ" `形式で送られてきます。
      ```
      GET /hello HTTP/1.1
      Host: localhost:9292
      User-Agent: curl/8.7.1

      ... (リクエストボディ)
      ```
6. **Rackアプリケーション入力 `env` の構築**
   Rackアプリケーションに渡す `env` ハッシュを作成します。
   ```ruby
   env = {
     Rack::REQUEST_METHOD    => "GET",
     Rack::SCRIPT_NAME       => "",
     Rack::PATH_INFO         => path,
     Rack::SERVER_NAME       => @options[:Host],
     Rack::SERVER_PORT       => @options[:Port].to_s,
     Rack::SERVER_PROTOCOL   => "HTTP/1.1",
     Rack::RACK_INPUT        => client,
     Rack::RACK_ERRORS       => $stderr,
     Rack::QUERY_STRING      => "",
     Rack::RACK_URL_SCHEME   => "http",
   }
   ```
   - ここでは動作に必要な最小限の値のみ設定しています。
7. **アプリケーションの呼び出しとレスポンスの送信**
   - Rackアプリケーションの `call` メソッドを呼び出し、レスポンスを取得します。
     ```ruby
     status, headers, body = @app.call(env)
     ```
   - クライアントにHTTPレスポンスとして返します。
     ```ruby
     client.puts "HTTP/1.1 #{status} #{Rack::Utils::HTTP_STATUS_CODES[status]}"
     headers.each do |key, value|
       client.puts "#{key}: #{value}"
     end
     client.puts
     body.each do |chunk|
       client.write chunk
     end
     ```
8. **ログの記録とクリーンアップ**
    - クライアントとの接続を確立しているソケットを閉じます。
      ```ruby
      client.close
      ```
   - リクエストとレスポンスの情報をログに記録します。
     ```ruby
     @logger.info "GET #{path} => #{status}"
     ```
9. **Rackハンドラーへの登録**
   自作のサーバーをRackハンドラーとして登録します。
   ```ruby
   Rackup::Handler.register "simple_server", SimpleServer
   ```
10. **アプリケーションの実行**
    最後に、`run` メソッドでアプリケーションを指定します。
    ```ruby
    run App.new
    ```

<details>
<summary>server.ru 全体コード</summary>

```ruby
require "socket"
require "logger"

class App
  def call(env)
    if env["PATH_INFO"] == "/"
      [200, {}, ["It works!"]]
    else
      [404, {}, ["Not Found"]]
    end
  end
end

class SimpleServer
  def self.run(app, **options)
    new(app, options).start
  end

  def initialize(app, options)
    @app = app
    @options = options
    @logger = Logger.new($stdout)
  end

  def start
    @logger.info "SimpleServer starting..."
    server = TCPServer.new(@options[:Port].to_i)
    loop do
      client = server.accept

      request_line = client.gets&.chomp
      %r[^GET (?<path>.+) HTTP/1.1$].match(request_line)
      path = Regexp.last_match(:path)

      unless path
        client.puts "HTTP/1.1 501 Not Implemented"
        client.close
        next
      end

      request_headers = {}
      loop do
        header_field = client.gets.chomp
        match = %r[^(?<name>[^:]+):\s+(?<value>.+)$].match(header_field)
        break unless match

        request_headers[match[:name]] = match[:value]
      end

      env = {
        Rack::REQUEST_METHOD    => "GET",
        Rack::SCRIPT_NAME       => "",
        Rack::PATH_INFO         => path,
        Rack::SERVER_NAME       => @options[:Host],
        Rack::SERVER_PORT       => @options[:Port].to_s,
        Rack::SERVER_PROTOCOL   => "HTTP/1.1",
        Rack::RACK_INPUT        => client,
        Rack::RACK_ERRORS       => $stderr,
        Rack::QUERY_STRING      => "",
        Rack::RACK_URL_SCHEME   => "http",
      }

      status, headers, body = @app.call(env)

      client.puts "HTTP/1.1 #{status} #{Rack::Utils::HTTP_STATUS_CODES[status]}"
      headers.each do |name, value|
        client.puts "#{name}: #{value}"
      end
      client.puts
      body.each do |line|
        client.puts line
      end
      client.close

      @logger.info "GET #{path} => #{status}"
    end
  end
end

Rackup::Handler.register "simple_server", SimpleServer

run App.new
```

</details>

### ポイント
- 大まかに言うとクライアントからHTTPリクエストを受け取り、Rackプロトコルで定まった値を詰め込んだ `env` ハッシュを作成してRackアプリケーションに渡し、Rackアプリケーションから帰ってきた配列を元にクライアントにHTTPレスポンスを返すのがRackサーバの仕事です。
- .ru ファイルの中でRackアプリケーション、Rackサーバ実装どちらも含めた例となっています。
- ここで示したステップにはエラーハンドリングが含まれていません。期待する入力以外では正常に動かないことがあります。
- Rackアプリケーション `App`, Rackサーバ `SimpleServer` をつなぎこみ、 `SimpleServer.run` や `SimpleServer#start` を呼び出したりしてるのはrackupコマンドがしています。詳細を把握したい方は <https://github.com/rack/rackup> を参照してください。

### 実行方法

できあがった server.ru をターミナルで以下のコマンドを実行します。

```console
$ rackup --server simple_server server.ru
I, [2024-10-22T00:30:13.983048 #16660]  INFO -- : SimpleServer starting...
```

`-s simple_server` オプションで、自作のRackサーバー SimpleServer 使用を指定します。

ブラウザやcurlコマンド等で `http://localhost:9292` にアクセスして、`It works!` と表示されれば成功です。

```console
$ curl -i http://localhost:9292/
HTTP/1.1 200 OK
content-length: 9

It works!
$ curl -i http://localhost:9292/hello
HTTP/1.1 404 Not Found
content-length: 9

Not Found
```

---

## 2. fork を使った並列処理サーバの実装
SimpleServerのメインループはクライアントの接続を待ち、Rackアプリケーションにリクエストを処理させ、クライアントにレスポンスを返すというシンプルな実装です。
この実装では1つのリクエストを処理している間、他のリクエストを処理できません。ここでは`fork` を使って並列にリクエストを処理できるサーバを実装してみましょう。

### 手順
1. **ForkServerクラスの定義**
   先ほどと同様に .ru ファイルの中で `ForkServer` クラスを定義します。ただしForkServerはメインループの中で `fork` を使って並列処理を行います。
   ```ruby
   class ForkServer
     def self.run(app, **options)
       new(app, options).start
     end

     def initialize(app, options)
       @app = app
       @options = options
       @logger = Logger.new($stdout)
     end

     def start
       @logger.info "ForkServer starting..."
       server = TCPServer.new(@options[:Port].to_i)
       loop do
         client = server.accept
         # ...
       end
     end
   end
   ```
4. **並列処理の実装**
   - クライアントからの接続を受け付けたら、`fork` を使って子プロセスを生成します。
     ```ruby
     loop do
       client = server.accept
       fork do
         # 子プロセス内でリクエストを処理
       end
       client.close
     end
     ```
   - 子プロセス内でリクエストの受信、`env` の構築、アプリケーションの呼び出し、レスポンスの送信を行います。
   - 子プロセス内では不要な接続をクローズするため、fork後に `server.close` と、レスポンス送信後に `client.close` を行います。
5. **ForkServerの登録**
   `ForkServer` を `fork_server` ハンドラーとして登録します。
   ```ruby
   Rackup::Handler.register "fork_server", ForkServer
   ```

<details>

<summary>server.ru 全体 (fork版)</summary>

```ruby
require "socket"
require "logger"

class App
  def call(env)
    if env["PATH_INFO"] == "/"
      [200, {}, ["It works!"]]
    else
      [404, {}, ["Not Found"]]
    end
  end
end

class ForkServer
  def self.run(app, **options)
    new(app, options).start
  end

  def initialize(app, options)
    @app = app
    @options = options
    @logger = Logger.new($stdout)
  end

  def start
    @logger.info "ForkServer starting..."
    server = TCPServer.new(@options[:Port].to_i)
    loop do
      client = server.accept
      fork do
        server.close

        request_line = client.gets&.chomp
        %r[^GET (?<path>.+) HTTP/1.1$].match(request_line)
        path = Regexp.last_match(:path)

        unless path
          client.puts "HTTP/1.1 501 Not Implemented"
          client.close
          next
        end

        request_headers = {}
        while %r[^(?<name>[^:]+):\s+(?<value>.+)$].match(client.gets.chomp)
          request_headers[Regexp.last_match(:name)] = Regexp.last_match(:value)
        end

        env = ENV.to_hash.merge(
          Rack::REQUEST_METHOD    => "GET",
          Rack::SCRIPT_NAME       => "",
          Rack::PATH_INFO         => path,
          Rack::SERVER_NAME       => @options[:Host],
          Rack::RACK_INPUT        => client,
          Rack::RACK_ERRORS       => $stderr,
          Rack::QUERY_STRING      => "",
          Rack::REQUEST_PATH      => path,
          Rack::RACK_URL_SCHEME   => "http",
          Rack::SERVER_PROTOCOL   => "HTTP/1.1",
        )
        status, headers, body = @app.call(env)

        client.puts "HTTP/1.1 #{status} #{Rack::Utils::HTTP_STATUS_CODES[status]}"

        headers.each do |name, value|
          client.puts "#{name}: #{value}"
        end
        client.puts
        body.each do |line|
          client.puts line
        end

        @logger.info "GET #{path} => #{status}"
      ensure
        client.close
      end
      client.close
    end
  end
end

Rackup::Handler.register "fork_server", ForkServer

run App.new
```

</details>

### ポイント

- `fork` を使うことで、リクエストごとに新しいプロセスを生成し、並列に処理できます。
- プロセスの生成はオーバーヘッドが大きいため、大量のリクエストには不向きです。
  - 上限を設けずにリクエストの度に子プロセスを生成するような実装は、大量リクエストによりサーバがダウンするリスクがあり、現実的な実装ではありません。

### 実行方法

できあがった server.ru をターミナルで以下のコマンドを実行します。

```console
$ rackup --server fork_server server.ru
I, [2024-10-22T00:31:19.618183 #16730]  INFO -- : ForkServer starting...
```

`-s fork_server` オプションで、自作のRackサーバー SimpleServer 使用を指定します。

ブラウザやcurlコマンド等で `http://localhost:9292` にアクセスして、`It works!` と表示されれば成功です。

```console
$ curl -i http://localhost:9292/
HTTP/1.1 200 OK
content-length: 9

It works!
$ curl -i http://localhost:9292/hello
HTTP/1.1 404 Not Found
content-length: 9

Not Found
```

Rackアプリケーションの中にsleepを仕込むなどして、複数リクエストを同時に送ってみてください。SimpleServerでは同時にリクエストを処理できないのに対し、ForkServerでは同時に複数のリクエストを処理できることが確認できるでしょう。

---

## 3. 発展課題

ここまでで実装したサーバは、Rackの動作を理解するための最低限の機能のみ持つものです。
時間に余裕のある人は、以下の課題に挑戦してみてはいかがでしょうか。

- GET以外の主要なHTTPメソッドに適切に対応する。
  - POST, PUT, DELETE, PATCH, HEAD など。　
- Rackの仕様に正しく準拠する。
  - 仕様は <https://github.com/rack/rack/blob/main/SPEC.rdoc> に説明があります。
  - Rack::Lint ミドルウェアを使うことで、Rackの仕様に準拠しているかを確認できます。
- 適切なエラーハンドリングの実装。
  - クライアントからの不正なリクエストや、対応していないHTTPメソッドに対する適切なエラーレスポンスを返すなど、エラーハンドリングは不足しています。
- pre-forkingやスレッドプールを使った並列処理を実装する。
  - 上述した ForkServer はリクエストごとに新しいプロセスを生成するため、現実的に使えるアプリケーションではありません。事前にプロセスやスレッドを生成しておき、リクエストを受け付けるときにそのプロセスを使い回すような実装を考えてみましょう。
- Ractorを使ったRackサーバを実装する。
  - Ruby 3.0から導入された新しい並列処理の仕組み、Ractor を使用してみましょう。
  - Rack と Ractor、名前も似ていますしきっと相性が良いはずです。

---

## 4. まとめ
この章では、自作のRackサーバを実装することで、Rackサーバの基本的な動作やRackの内部構造について理解を深めました。

今後Rackサーバを実装する機会がなかったとしても、今回Rackサーバを自分で実装し理解を深めることはPuma, Unicorn, Pitchforkといった既存のRackサーバの挙動を読み解く際にもきっと役に立つでしょう。

---

これで本ハンズオンワークショップのテキストは終了です、お疲れさまでした。
是非ここで学んだ内容を活かして、より高度な開発に挑戦してみてください。
