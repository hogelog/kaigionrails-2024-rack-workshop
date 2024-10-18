# 第1章: Rackアプリケーションをいろいろな書き方で実装してみよう

この章では、RubyでWebアプリケーションを構築するためのミドルウェアであるRackを使って、さまざまな方法でアプリケーションを実装してみます。Rackの基本的な仕組みを理解し、シンプルなアプリケーションから始めて、徐々に機能を拡張していきましょう。

---

## 1. Rackとは

Rackは、RubyでWebサーバーとWebアプリケーションをつなぐインターフェースを提供するライブラリです。Rackを使うことで、Webサーバー（例えばPumaやUnicorn）とフレームワーク（例えばRailsやSinatra）との間で共通のインターフェースを持つことができます。

---

## 2. シンプルなRackアプリケーションの作成

まずは、最もシンプルなRackアプリケーションを作成してみましょう。

### 手順

1. 新しいファイル `01-plain.ru` を作成します。

2. 以下の骨組みを記述します。

   ```ruby
   class App
     def call(env)
       # ここにコードを記述
     end
   end

   run App.new
   ```

3. `call` メソッドは、環境変数 `env` を受け取り、ステータスコード、ヘッダー、ボディの配列を返します。

4. ステータスコード200、ヘッダーは空のハッシュ、ボディは文字列 `'hello'` を含む配列を返すように実装してみましょう。

### ポイント

- `call` メソッドは必ず `[status, headers, body]` の形式でレスポンスを返す必要があります。
- `body` は文字列を含む配列である必要があります。

### 実行方法

ターミナルで以下のコマンドを実行します。

```bash
rackup 01-plain.ru
```

ブラウザで `http://localhost:9292` にアクセスして、`hello` と表示されれば成功です。

---

## 3. 環境変数を確認してみる

アプリケーションが受け取る `env` には、リクエストに関するさまざまな情報が含まれています。これを確認してみましょう。

### 手順

1. 新しいファイル `02-binding.ru` を作成します。

2. `call` メソッド内で `binding.irb` を呼び出します。

   ```ruby
   class App
     def call(env)
       # ここでirbセッションを開始
     end
   end

   run App.new
   ```

3. `binding.irb` を使って、`env` の中身を確認してみましょう。

### ポイント

- `binding.irb` を挿入すると、その位置で対話型のirbセッションが始まります。
- `env` の内容を調べることで、リクエストの詳細情報を理解できます。

### 実行方法

ターミナルで以下のコマンドを実行します。

```bash
rackup 02-binding.ru
```

ブラウザで `http://localhost:9292` にアクセスすると、ターミナル上でirbセッションが開始されます。`env` を調べてみましょう。

---

## 4. レスポンスヘッダーを設定する

レスポンスヘッダーを設定して、適切なコンテンツタイプを指定してみましょう。

### 手順

1. 新しいファイル `03-header.ru` を作成します。

2. `call` メソッドで返すヘッダーに `'content-type' => 'text/plain'` を追加します。

   ```ruby
   class App
     def call(env)
       # ステータスコード、ヘッダー、ボディを返す
     end
   end

   run App.new
   ```

### ポイント

- レスポンスヘッダーを設定することで、クライアントに返すデータの種類を指定できます。
- ヘッダーはハッシュで表現し、キーと値は文字列です。

### 実行方法

```bash
rackup 03-header.ru
```

ブラウザでアクセスし、開発者ツールなどでレスポンスヘッダーを確認してみましょう。

---

## 5. ルーティングの実装

リクエストのパスやメソッドに応じて、異なるレスポンスを返すようにしてみます。

### 手順

1. 新しいファイル `04-routing.ru` を作成します。

2. `call` メソッド内でリクエストの情報を取得します。

   ```ruby
   class App
     def call(env)
       # リクエストメソッドとパスを取得
       # ルーティングの条件分岐
     end

     # 各ルートに対応するメソッドを定義
   end

   run App.new
   ```

3. ルート `/` にアクセスした場合は `It works!` を返す。

4. パス `/hello/:id` にアクセスした場合は `Hello :id` を返す。

### ポイント

- `env` から `REQUEST_METHOD` や `PATH_INFO` を取得できます。
- 正規表現を使ってパスパラメータを抽出します。
- 条件分岐には `case` 文やパターンマッチングを利用できます（Ruby 2.7以降）。

### 実行方法

```bash
rackup 04-routing.ru
```

ブラウザで `http://localhost:9292/` および `http://localhost:9292/hello/123` にアクセスして動作を確認します。

---

## 6. Rackのライブラリを活用する

Rackが提供する便利なクラスやメソッドを使って、コードを簡潔にしてみましょう。

### 手順

1. 新しいファイル `05-library.ru` を作成します。

2. `Rack::Request` と `Rack::Response` を使用します。

   ```ruby
   require 'rack'

   class App
     def call(env)
       # Rack::Requestを使ってリクエストを処理
       # ルーティングとレスポンスの生成
     end

     private

     # レスポンスを生成するヘルパーメソッドを定義
   end

   run App.new
   ```

3. リクエストパラメータの取得やレスポンスの生成をRackのクラスで行います。

### ポイント

- `Rack::Request` はリクエストに対する便利なメソッドを提供します。
- `Rack::Response` はレスポンスの組み立てを簡単にします。
- ヘルパーメソッドを定義してコードの再利用性を高めます。

### 実行方法

```bash
rackup 05-library.ru
```

ブラウザで前と同様にアクセスし、動作を確認します。

---

## 7. Sinatraを使ったRackアプリケーション

次に、Rubyの軽量なWebフレームワークであるSinatraを使って、同じ機能を実装してみましょう。

### 手順

1. 新しいファイル `06-sinatra.ru` を作成します。

2. Sinatraのベースクラスを継承した `App` クラスを定義します。

   ```ruby
   require 'sinatra/base'

   class App < Sinatra::Base
     # ルートの定義
   end

   run App.new
   ```

3. 以下のルートを定義します。

   - `get "/"` で `It works!` を返す。
   - `get "/hello/:id"` で `Hello :id` を返す。

### ポイント

- Sinatraは、`Sinatra::Base` を継承したクラス内でDSLを使ってルートを定義します。
- `params` ハッシュを使って、URLパラメータを取得できます。
- SinatraアプリケーションもRackアプリケーションとして扱えるため、`run App.new` で実行できます。

### 実行方法

```bash
rackup 06-sinatra.ru
```

ブラウザで `http://localhost:9292/` および `http://localhost:9292/hello/456` にアクセスして動作を確認します。

### 補足

- Sinatraを使用するために、`gem install sinatra` でSinatraをインストールしておいてください。
- Sinatraは内部的にRackを使用しているため、Rackの知識がSinatraの理解にも役立ちます。

---

## 8. Railsを使ったRackアプリケーション

最後に、RubyのフルスタックWebフレームワークであるRailsを使って、同じ機能を実装してみましょう。Railsは大規模なアプリケーション開発に適していますが、最小限の構成でシンプルなRackアプリケーションとして動作させることもできます。

### 手順

1. 新しいファイル `07-rails.ru` を作成します。

2. 必要なライブラリを読み込みます。

   ```ruby
   require 'action_controller/railtie'
   ```

3. Railsアプリケーションを定義します。

   ```ruby
   class App < Rails::Application
     config.logger = Logger.new($stdout)
     Rails.logger  = config.logger

     routes.draw do
       root 'apps#index'
       resources :apps, only: :show, path: 'hello'
     end
   end
   ```

4. コントローラーを定義します。

   ```ruby
   class AppsController < ActionController::Base
     def index
       render plain: 'It works!'
     end

     def show
       render plain: "Hello #{params[:id]}"
     end
   end
   ```

5. アプリケーションを実行します。

   ```ruby
   run Rails.application
   ```

### ポイント

- `require 'action_controller/railtie'` でRailsの必要最小限の機能をロードします。
- `App < Rails::Application` でアプリケーションの設定を行います。
- `routes.draw` ブロック内でルーティングを定義します。
  - `root 'apps#index'` はルートパス（`/`）にアクセスしたときに `AppsController` の `index` アクションを呼び出します。
  - `resources :apps, only: :show, path: 'hello'` は `/hello/:id` へのGETリクエストを `show` アクションにルーティングします。
- `AppsController` は `ActionController::Base` を継承し、`index` と `show` アクションを定義します。
- `render plain: '...'` でプレーンテキストのレスポンスを返します。
- `run Rails.application` でアプリケーションを実行します。

### 実行方法

Railsを使用するために、必要なgemをインストールしておきます。

```bash
gem install rails
```

`07-rails.ru` ファイルを保存したら、以下のコマンドでアプリケーションを起動します。

```bash
rackup 07-rails.ru
```

ブラウザで以下のURLにアクセスして動作を確認します。

- `http://localhost:9292/` : `It works!` と表示されます。
- `http://localhost:9292/hello/789` : `Hello 789` と表示されます。

### 補足

- この方法でRailsアプリケーションを動かすと、Railsの全機能を使うことはできませんが、最小限の構成でシンプルなアプリケーションを作成できます。
- Railsは内部的にRackを使用しているため、Rackの知識がRailsの理解にも役立ちます。

---

## 9. まとめ

この章では、Rackアプリケーションの基本的な構造から始めて、環境変数の確認、レスポンスヘッダーの設定、ルーティングの実装、Rackライブラリの活用、SinatraやRailsを使った実装までを学びました。これらの基礎を押さえることで、さまざまなフレームワークやライブラリを使ったWebアプリケーションの構築に役立てることができます。

これまでの内容を通じて、以下のポイントを理解できたと思います。

- **Rackの基本構造**: `call` メソッドで `[status, headers, body]` を返す。
- **環境変数 `env` の利用**: リクエストに関する情報を取得できる。
- **レスポンスヘッダーの設定**: ヘッダーを適切に設定してクライアントに情報を伝える。
- **ルーティングの実装**: パスやメソッドに応じて処理を分岐する。
- **Rackのライブラリの活用**: `Rack::Request` や `Rack::Response` を使ってコードを簡潔に。
- **SinatraやRailsとの組み合わせ**: フレームワークを使って同じ機能を実装し、Rackとの関係を理解する。

これらの知識を基に、より複雑なWebアプリケーションやフレームワークを使った開発に挑戦してみてください。

---

次の章では、Rackミドルウェアの作成や、ミドルウェアを使った機能拡張について学んでいきます。