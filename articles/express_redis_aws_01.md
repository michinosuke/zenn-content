---
title: "ExpressとRedisでセッションを実装してAWS Fargateにデプロイしよう-ローカル編"
emoji: "🦋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Nodejs","Express","AWS","Redis","ElastiCache"]
published: true
---



# はじめに

この記事では、こんなWebアプリケーションを作成します。

![](https://storage.googleapis.com/zenn-user-upload/eac052d18519-20211126.gif)

内容としては、以下のようなことを扱います。

* Expressアプリケーションの作成
* Dockerイメージの作成
* Docker ComposeによるExpressとRedisの連携



# 予備知識

## Expressとは？

Node.js(JavaScript)のWebアプリケーションフレームワークです。

## Redisとは？

Express.jsなど多くのWebフレームワークでは、セッション情報を保存するためにインメモリのセッションストアを使用します。

なので、アプリケーションのアップデートや再起動のたびに、全てのユーザのセッションは破棄され、強制的にログアウトさせられます。

また、複数のコンテナ間でセッションを共有できません。

これを回避するため、Redisやmemcachedなど、外部のインメモリキャッシュシステムを使用するのが一般的です。



# STEP 1 : Expressアプリケーションを作る

## 1-1 : Expressの雛形を作る

簡単にExpressアプリケーションの雛形を作れる`express-generator`を使います。

プロジェクトを作成したいディレクトリに移動して、以下のコマンドを実行してください。

```sh
$ npm i -g express-generator
$ express express-redis-sample
$ cd express-redis-sample/
$ npm i
```

これだけで終わりです。



Expressアプリケーションを立ち上げるときは、以下のコマンドを実行します。

```sh
$ npm start
```



`localhost:3000`にアクセスして、以下のような画面が表示されることを確認してください。

![npm startした後にブラウザに表示される画像](https://storage.googleapis.com/zenn-user-upload/b211c62fc111-20211125.png)



:::details この時点でのファイル構造
```sh
$ tree -I node_modules
.
├── app.js
├── bin
│   └── www
├── package-lock.json
├── package.json
├── public
│   ├── images
│   ├── javascripts
│   └── stylesheets
│       └── style.css
├── routes
│   ├── index.js
│   └── users.js
└── views
    ├── error.jade
    ├── index.jade
    └── layout.jade
```
:::



# STEP 2 : Docker Composeを使う

## 2-1 : Dockerfileを作る

開発環境用のDockerfileを作成します。

```sh
$ touch Dockerfile_dev
```



下の内容を書き込みます。

```dockerfile:Dockerfile_dev
ARG VARIANT="14-buster"
FROM mcr.microsoft.com/vscode/devcontainers/javascript-node:0-${VARIANT}
WORKDIR /app
COPY . .
RUN npm install
```



## 2-2 : docker-compose.ymlを作る

`docker-compose.yml`を作成します。

```sh
$ touch docker-compose.yml
```



下の内容を書き込みます。

```yaml
version: '3'
services:
  express:
    build:
      context: .
      dockerfile: Dockerfile_dev
    volumes:
      - /app/node_modules
      - type: bind
        source: ./
        target: /app
    tty: true
    ports:
      - 3000:3000
    environment:
      - PORT=3000
      - SESSION_SECRET=hogehoge
      - REDIS_HOST=redis_db
      - REDIS_PORT=6379
    depends_on:
      - redis_db
  
  redis_db:
    image: redis:latest
    ports:
      - 6379:6379
```

簡単に説明すると、`services`直下の`express`が作成しているExpressアプリケーションの設定で、`redis_db`がセッションデータを保存するRedisの設定です。

`express`の中の`environment`には、今後Expressアプリケーションで使用する環境変数を記述しています。

* `PORT` : Expressアプリケーションを実行するポート
* `SESSION_SECRET` : Cookieに保存するセッションIDの署名と暗号化に使用します。開発環境では適当でいいですが、本番環境では適度に長いランダムな文字列に、秘密にする必要があります。定期的に変更することが推奨されます。
* `REDIS_HOST` : Redisに接続するときに使用するホスト名です。Docker Composeでは、このようにアプリケーション名をホスト名として使用することができます。
* `REDIS_PORT` : Redisに接続するときに使用するポート番号です。Redisではデフォルトで`6379`番ポートを使用します。



## 2-3 : VSCode Remote Containersを使う

VSCode Remote Containersは、コンテナの中なのにローカルと同じように開発ができるツールです。

`.devcontainer.json`を作成します。

```sh
touch .devcontainer.json
```



下の内容を書き込みます。

```json
{
	"name": "express-redis-example",
  "dockerComposeFile": ["docker-compose.yml"],
  "service": "express",
  "workspaceFolder": "/app",
	"remoteUser": "root",
	"postCreateCommand": ["node", "./bin/www"]
}
```

`postCreateCommand`は、コンテナの中に入ったときに実行されるコマンドになります。STEP 2-1で作ったDockerfileに`CMD`がなかったのは、ここでアプリケーションの実行を指定しているためです。



:::message

もしVSCodeのRemote Containers拡張をインストールしていない場合は、以下の拡張機能をインストールしてください。もちろん、Dockerもインストール済である必要があります。

https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers

:::



あとは、`Command+Shift+P`をおして、`Reopen in Container`をクリックすると、コンテナが立ち上がります。（左下の緑色のところをクリックしてもできます。）



しばらくして、`localhost:3000`にアクセスして前と同じ画面が表示できたら、成功です。これで、コンテナ開発環境が整いました！

![npm startした後にブラウザに表示される画像](https://storage.googleapis.com/zenn-user-upload/b211c62fc111-20211125.png)



:::details この時点でのファイル構造

```sh
$ tree -I node_modules
.
├── Dockerfile_dev
├── docker-compose.yml
├── .devcontainer.json
├── app.js
├── bin
│   └── www
├── package-lock.json
├── package.json
├── public
│   ├── images
│   ├── javascripts
│   └── stylesheets
│       └── style.css
├── routes
│   ├── index.js
│   └── users.js
└── views
    ├── error.jade
    ├── index.jade
    └── layout.jade
```

:::



# STEP 3 : セッションを実装する

## 3-1 : 必要なものをインストールする

以下のコマンドで、必要なnpmパッケージをインストールします。

```sh
$ npm i express express-session redis connect-redis
```



## 3-2 : Redisと接続する

`app.js`を開いて、5行目くらいのところ（`require`たくさんしてるところ）に以下のコードを追加します。

```javascript:app.js
var session = require('express-session');
var RedisStore = require('connect-redis')(session)
var Redis = require('ioredis');
var redisClient = new Redis(process.env.REDIS_PORT, process.env.REDIS_HOST);
```



すると、16行目くらいに`var app = express();`っていう行があると思いますが、その下に以下のコードを追加します。

```javascript:app.js
app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: true,
  rolling: true,
  cookie: {
    maxAge: 60000,
  },
  store: new RedisStore({
    client: redisClient,
    prefix: 'sid:',
  }),
}))
```



これも簡単に説明すると、最初の方は、必要なパッケージを`require`してるだけです。`redisClient`のところがキモで、ここでRedisと接続しています。AWSにデプロイしたあとに、Redisのホスト名と環境変数が違ったりすると、ここでエラーがでます。

下の`app.use`のコードは、アプリケーションでセッションを使用するように設定しています。これによって、今後出てくる`req`というオブジェクトに`session`が追加されるようになります。それぞれの設定項目も簡単にご紹介します。

* `secret` : セッションの署名や暗号化に使用されるシークレットです。ここでは、Docker Composeで指定した環境変数を読み込んでいます。
* `resave` : アクセスのたびにセッションストアにデータを保存し直すかどうかです。基本的に`false`を指定しておけば問題ないとドキュメントにも書いてあります。
* `saveUninitialized` : セッションが形成されてないアクセスがあったときに、セッションストアを初期化するかどうかみたいなやつです。容量の圧迫にもなるので、基本的に`false`で問題ないとドキュメントに書いてあります。
* `rolling` : これは分かりやすくするために`true`にしてありますが、本番環境では`false`の方がいい可能性があります。`true`だとセッションの有効期限が**セッション作成日時から**になりますが、`false`だと**最終アクセス日時から**になります。アプリケーション完成後に変えてみると分かりやすいです。
* `cookie.maxAge` : Cookieの有効期限です。ただ、Redisの有効期限にもこの値が使用されます。ミリ秒なので、この例だと１分でセッションが失効してしまうことになります。
* `store` : ここに使用するセッションストアを指定します。これがないと、デフォルトでメモリ内にセッション情報は保存されます。その場合、`npm start`するたびにセッションが失効します。
* `prefix` : redisのキーにつけるプレフィックスです。一つのRedisを複数のアプリケーションで使用することもできるので、その場合にキーが重複しないようにするためです。



> 詳しいオプションを知りたい方は、`express-session`の[公式ドキュメント](https://github.com/expressjs/session#options)をご参照ください。



## 3-3 : アプリケーション構成を考える

こんな感じでホワイトボードに書いてみました。

![](https://storage.googleapis.com/zenn-user-upload/ad343f4b5db7-20211125.jpg)

ログインと言いつつ、ユーザ名をセッションに保存するだけなので、パスワードとかはありません。

これを実装していきましょう。



## 3-4 : ファイルをつくる

JavaScriptファイルを３つ、Jadeファイルを１つ作成します。

```sh
$ touch routes/login.js routes/save-session.js routes/delete-session.js views/login.jade
```

`users.js`は使わないので、削除します。

```sh
$ rm routes/users.js
```

ここから、以下６つのファイルの中身を書き換えます。

* `routes/index.js`
* `routes/login.js`
* `routes/save-session.js`
* `routes/delete-session.js`
* `views/login.jade`
* `views/login.jade`

### routes/index.js

```js:routes/index.js
var express = require('express');
var router = express.Router();

router.get('/', function(req, res, next) {
  if (req.session.userName) {
    res.render('index', {
      userName: req.session.userName,
      sessionId: req.session.id,
      maxAge: req.session.cookie.maxAge,
    })
  } else {
    res.redirect('login');
  }
});

module.exports = router;
```



### routes/login.js

```js:routes/login.js
var express = require('express');
var router = express.Router();

router.get('/', function(req, res, next) {
  if (req.session.userName) {
    res.redirect('/');
    return;
  }

  res.render('login');
});

module.exports = router;
```



### routes/save-session.js

```js:save-session.js
var express = require('express');
var router = express.Router();

router.get('/', function(req, res, next) {
  if (req.query.userName) {
    req.session.userName = req.query.userName
  }
  res.redirect('/');
});

module.exports = router;
```



### routes/delete-session.js

```js:delete-session.js
var express = require('express');
var router = express.Router();

router.get('/', function(req, res, next) {
  delete req.session.userName;
  res.redirect('/');
});

module.exports = router;
```



### views/index.jade

```pug:views/index.jade
extends layout

block content
  h1 Hello, #{userName}!
  p SESSION ID : #{sessionId}
  p Expire on after #{maxAge/1000}s
  a(href="/delete-session") Logout
```



### views/login.jade

```pug:views/login.jade
extends layout

block content
  h1 Login

  form(action="/save-session")
    input(name="userName")
    input(type="submit")
```



やってることは説明するまでもないくらいシンプルです。`login.jade`から`save-session.js`には、普通に`form`要素を使ってGETでユーザ名を渡しています。



### app.js

作ったファイルを読み込ませる必要があるので、`app.js`も書き換えます。

下のように、`require`してるところと、`app.use`してるところの２箇所を変更します。

```diff:app.js
 var indexRouter = require('./routes/index');
-var usersRouter = require('./routes/users');
+var loginRouter = require('./routes/login');
+var deleteSessionRouter = require('./routes/delete-session');
+var saveSessionRouter = require('./routes/save-session');
 
 
 app.use('/', indexRouter);
-app.use('/users', usersRouter);
+app.use('/login', loginRouter);
+app.use('/delete-session', deleteSessionRouter);
+app.use('/save-session', saveSessionRouter);
```



## 3-5 : 確認

再度、STEP 2と同じようにRemote Containerを実行して、`localhost:3000`にアクセスできるかを確認します。

初期状態ではセッションが保存されていないので、`/login`にリダイレクトされます。

![/loginの画像](https://storage.googleapis.com/zenn-user-upload/9c2a563720b2-20211126.png)

任意のユーザ名を入力して`送信`を押すと、`/save-session?userName=入力したユーザ名`にリダイレクトされ、セッションにユーザ名が保存されて、`/`に再度リダイレクトされます。

![/の画像](https://storage.googleapis.com/zenn-user-upload/43a3cd431450-20211126.png)

すると、ユーザ名と、Cookieに保存されたセッションID、セッションが保存されてからの秒数が表示されます。

この秒数が何度アクセスしてもずっと減っていくのは、STEP 3-2で`rolling: true`と設定したからです。`rolling: false`に設定すると、アクセスするたびに延長されます。



:::details この時点でのファイル構造

```sh
$ tree -I node_modules
.
├── Dockerfile_dev
├── app.js
├── bin
│   └── www
├── docker-compose.yml
├── package-lock.json
├── package.json
├── public
│   ├── images
│   ├── javascripts
│   └── stylesheets
│       └── style.css
├── routes
│   ├── delete-session.js
│   ├── index.js
│   ├── login.js
│   └── save-session.js
└── views
    ├── error.jade
    ├── index.jade
    ├── layout.jade
    └── login.jade
```

:::



# STEP 4 : 本番用Dockerfileを作成する

## 4-1 : Dockerfileを作成する

Dockerfileを作成します。

```sh
$ touch Dockerfile
```



以下の内容を書き込みます。

```dockerfile:Dockerfile
FROM node:latest
WORKDIR /app
COPY . .
RUN npm install
ENTRYPOINT ["node", "./bin/www"]
```



## 4-2 : 実行用スクリプトを作成する

簡単に実行できるように、イメージ作成用の`docker-build.sh`と、イメージ実行用の`docker-run.sh`を作成します。

```sh
$ touch docker-build.sh docker-run.sh
```

以下の内容を書き込みます。



* `docker-build.sh`

```sh:docker-build.sh
docker build -t express-redis-example .
```



* `docker-run.sh`

```sh:docker-run.sh
docker run -it -d \
-e REDIS_HOST=host.docker.internal \
-e PORT=3000 \
-e SESSION_SECRET=hogehoge \
-e REDIS_PORT=6379 \
-p 3000:3000 \
express-redis-example-image bash
```



`REDIS_HOST`に指定した`host.docker.internal`というのは、コンテナの外の`localhost`のことです。

## 4-3 : Redisを実行する

Docker Composeを使用せずに動かすので、普通にローカルでRedisを動かしましょう。

Homebrewを使っています。他の方法でも構いません。

```sh
$ brew install redis
$ brew services start redis
$ redis-cli
Could not connect to Redis at 127.0.0.1:6379: Connection refused
not connected>
```



## 4-4 : 本番用Dockerを実行する

ここからは、Remote Container外で実行してください。

```sh
$ bash docker-build.sh
$ bash docker-run.sh
```



そして、STEP 3と同じく`localhost:3000`でアクセスできたら成功です！



:::details この時点でのファイル構造

```sh
$ tree -I node_modules
.
├── Dockerfile
├── Dockerfile_dev
├── app.js
├── bin
│   └── www
├── docker-build.sh
├── docker-compose.yml
├── docker-run.sh
├── package-lock.json
├── package.json
├── public
│   ├── images
│   ├── javascripts
│   └── stylesheets
│       └── style.css
├── routes
│   ├── delete-session.js
│   ├── index.js
│   ├── login.js
│   └── save-session.js
└── views
    ├── error.jade
    ├── index.jade
    ├── layout.jade
    └── login.jade
```

:::



次回、この作成したイメージをAWSにデプロイします。
