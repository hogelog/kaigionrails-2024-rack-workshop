# 第3章: 自作のRackサーバを実装してみよう

この章では、Rackサーバを自前で実装してみます。既存のRackサーバ（PumaやWEBrickなど）を使わずに、自作のサーバを作ることで、Rackの内部的な仕組みやWebサーバの基本的な動作原理を理解することができます。

---

## 1. シンプルなRackサーバの作成

まずは、最もシンプルなRackサーバを実装してみましょう。以下の手順に従って、サーバを作成していきます。

### 手順

1. **必要なライブラリの読み込み**

   新しいファイル `01-plain.ru` を作成し、以下のライブラリを読み込みます。

   ```ruby
   require "socket"
   require "logger"

   require "rack/response"
   require "rack/rewindable_input"
   require "rack/runtime"
   ```

   - `socket` はソケット通信を扱う標準ライブラリです。
   - `logger` はログを記録するための標準ライブラリです。
   - `rack` の各種クラスを利用します。

2. **アプリケーションクラスの定義**

   Rackアプリケーションである `App` クラスを定義します。

   ```ruby
   class App
     def call(env)
       # リクエストを処理してレスポンスを生成
     end
   end
   ```

   - `call` メソッドで、`env` 環境変数を受け取ります。
   - `Rack::Request` を使ってリクエストを処理します。
   - ルーティングを実装し、適切なレスポンスを返します。

3. **サーバークラスの定義**

   自作のサーバークラス `SimpleServer` を定義します。

   ```ruby
   class SimpleServer
     def self.run(app, **options)
       # サーバーを起動するクラスメソッド
     end

     def initialize(app, options)
       # 初期化処理
     end

     def start
       # サーバーのメインループ
     end
   end
   ```

   - `self.run` クラスメソッドでサーバーを起動します。
   - `initialize` メソッドで、アプリケーションとオプションを受け取ります。
   - `start` メソッドで、サーバーのメインループを実装します。

4. **ソケットサーバーの起動**

   `initialize` メソッドまたは `start` メソッド内で、以下の処理を行います。

   - `TCPServer` を使って指定されたポートでサーバーを起動します。
   - 無限ループでクライアントからの接続を待ち受けます。
   - クライアントからの接続があったら、リクエストを読み込みます。

   ```ruby
   @server = TCPServer.new(options[:Port].to_i)
   @logger = Logger.new($stdout)
   ```

5. **リクエストの受信と解析**

   - クライアントから送られてきたリクエストライン（例: `"GET / HTTP/1.1"`）を読み込みます。
   - リクエストヘッダーを読み込み、ハッシュに格納します。
   - 必要であれば、リクエストボディを読み込みます。

   ```ruby
   client = @server.accept
   request_line = client.gets.chomp
   # リクエストラインの解析
   # ヘッダーの読み込み
   ```

6. **環境変数 `env` の構築**

   Rackアプリケーションに渡す `env` ハッシュを作成します。

   ```ruby
   env = {
     # 必要なキーと値を設定
     Rack::REQUEST_METHOD    => "GET",
     Rack::SCRIPT_NAME       => "",
     Rack::PATH_INFO         => path,
     Rack::QUERY_STRING      => query_string,
     Rack::SERVER_NAME       => @options[:Host],
     Rack::SERVER_PORT       => @options[:Port],
     Rack::RACK_VERSION      => Rack::VERSION,
     Rack::RACK_INPUT        => StringIO.new(body),
     Rack::RACK_ERRORS       => $stderr,
     Rack::RACK_MULTITHREAD  => false,
     Rack::RACK_MULTIPROCESS => false,
     Rack::RACK_RUNONCE      => false,
     Rack::RACK_URL_SCHEME   => "http",
     Rack::SERVER_PROTOCOL   => "HTTP/1.1",
   }
   ```

   - 必要なキーと値を正確に設定することが重要です。
   - リクエストメソッド、パス、クエリ文字列、ヘッダーなどを適切に設定します。

7. **アプリケーションの呼び出しとレスポンスの送信**

   - アプリケーションの `call` メソッドを呼び出し、レスポンスを取得します。

     ```ruby
     status, headers, body = @app.call(env)
     ```

   - クライアントにHTTPレスポンスとして返します。

     ```ruby
     client.puts "HTTP/1.1 #{status} #{http_status_message(status)}"
     headers.each do |key, value|
       client.puts "#{key}: #{value}"
     end
     client.puts
     body.each do |chunk|
       client.write chunk
     end
     ```

   - `http_status_message` はステータスコードに対応するメッセージを返すヘルパーメソッドを自作します。

8. **ログの記録とクリーンアップ**

   - リクエストとレスポンスの情報をログに記録します。

     ```ruby
     @logger.info "#{env[Rack::REQUEST_METHOD]} #{env[Rack::PATH_INFO]} -> #{status}"
     ```

   - クライアントソケットを閉じます。

     ```ruby
     client.close
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

### 実行方法

ターミナルで以下のコマンドを実行します。

```bash
rackup 01-plain.ru -s simple_server
```

- `-s simple_server` オプションで、自作のサーバーを使用することを指定します。
- ブラウザで `http://localhost:9292/` や `http://localhost:9292/hello` にアクセスして、動作を確認します。

---

## 2. フォークを使った並列処理サーバの実装

先ほどのサーバは1つのリクエストを処理している間、他のリクエストを待たせてしまいます。ここでは、`fork` を使って並列にリクエストを処理できるサーバを実装してみましょう。

### 手順

1. **新しいファイルの作成**

   `02-fork.ru` ファイルを作成します。

2. **アプリケーションクラスの再利用**

   前のステップで作成した `App` クラスをそのまま使用します。

3. **フォークサーバークラスの定義**

   `ForkServer` クラスを定義し、`fork` を使って並列処理を行います。

   ```ruby
   class ForkServer
     def self.run(app, **options)
       # サーバーを起動するクラスメソッド
     end

     def initialize(app, options)
       # 初期化処理
     end

     def start
       # サーバーのメインループ
     end
   end
   ```

4. **並列処理の実装**

   - クライアントからの接続を受け付けたら、`fork` を使って子プロセスを生成します。

     ```ruby
     loop do
       client = @server.accept
       fork do
         # 子プロセス内でリクエストを処理
       end
       client.close
     end
     ```

   - 子プロセス内でリクエストの受信、`env` の構築、アプリケーションの呼び出し、レスポンスの送信を行います。

5. **子プロセスの終了**

   - 子プロセス内で処理が終わったら、`exit` を呼び出してプロセスを終了します。

     ```ruby
     ensure
       client.close
       exit
     ```

6. **親プロセスでのクリーンアップ**

   - 親プロセスでは、クライアントソケットを閉じて次の接続を待ちます。

### 実行方法

ターミナルで以下のコマンドを実行します。

```bash
rackup 02-fork.ru -s fork_server
```

- ブラウザで複数のタブを開き、`http://localhost:9292/` や `http://localhost:9292/hello` にアクセスして、並列にリクエストが処理されることを確認します。

### ポイント

- `fork` を使うことで、リクエストごとに新しいプロセスを生成し、並列に処理できます。
- プロセスの生成はオーバーヘッドが大きいため、大量のリクエストには不向きです。
- 実際のWebサーバーでは、スレッドやイベントループを使って並列処理を行うことが一般的です。

---

## 3. まとめ

この章では、自作のRackサーバを実装することで、Webサーバの基本的な動作やRackの内部構造について理解を深めました。自作サーバを作ることで、以下のポイントを学ぶことができました。

- **ソケット通信の基礎**: `TCPServer` や `TCPSocket` を使ったソケット通信の基本。
- **HTTPプロトコルの理解**: リクエストラインやヘッダーの解析、レスポンスの生成方法。
- **Rack環境変数 `env` の構築**: Rackアプリケーションに渡す `env` ハッシュの中身と作り方。
- **並列処理の方法**: `fork` を使った並列処理の実装とその限界。

これらの知識は、Webサーバやミドルウェアの動作原理を理解する上で非常に重要です。さらに発展的な内容に挑戦し、以下の点を改善・拡張してみてください。

- **HTTPメソッドの対応**: `POST`、`PUT`、`DELETE` などの他のHTTPメソッドにも対応してみましょう。
- **リクエストヘッダーのパース**: `Content-Length` や `Transfer-Encoding` を考慮して、リクエストボディの読み込みを実装してみましょう。
- **エラーハンドリング**: 例外が発生した場合に、ステータスコード500で適切なレスポンスを返すようにします。
- **マルチスレッド対応**: `fork` の代わりにスレッドを使って並列処理を行うサーバーを実装してみましょう。
- **パフォーマンスの最適化**: ノンブロッキングI/Oやイベントループを使って効率的なサーバを目指してみましょう。

---

これで本ハンズオンワークショップのテキストは終了です。お疲れさまでした。ここで学んだ内容を活かして、より高度なRackアプリケーションやミドルウェアの開発に挑戦してみてください。