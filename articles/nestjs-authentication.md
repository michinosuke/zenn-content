---
title: "【公式ドキュメント和訳】NestJS Authentication - 認証"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs", "typescript", "javascript", "nodejs", "authentication"]
published: false
---

:::message
この記事は、2021/11/5時点のNestJSの仕様に依存します。
NestJSのバージョンが違ったり、記事の公開から長い時間が経っている場合、正確ではない可能性があります。
:::

> NestJS: v8.0.0
> 翻訳元: [authentication.md](https://github.com/nestjs/docs.nestjs.com/blob/b0d2c893948863cc416032e417a1a94b1b8b4d01/content/security/authentication.md)
> ライセンス: [MIT License](https://github.com/nestjs/docs.nestjs.com/blob/b0d2c893948863cc416032e417a1a94b1b8b4d01/LICENSE)

# 認証

認証は、アプリケーションのキモとなる部分です。認証を実装する方法はたくさんあります。プロジェクトに採用される認証方法は、アプリケーションの要件によります。この記事では、様々なアプリケーションの要件に対応するために、いくつかの認証方法をご紹介します。

多くのプロダクトに採用されている実績のあるNode.jsの認証ライブラリの一つに、[Passport](https://github.com/jaredhanson/passport)があります。NestJSでは、このライブラリを簡単に利用するために、`@nestjs/passport`モジュールをご用意しています。Passportがやっていることをざっくりと説明すると、以下のようになります。

* [クレデンシャル](https://e-words.jp/w/%E3%82%AF%E3%83%AC%E3%83%87%E3%83%B3%E3%82%B7%E3%83%A3%E3%83%AB.html)を検証し、ユーザを認証します。クレデンシャルとは、ユーザ名とパスワードの組み合わせや、JSON Web Token ([JWT](https://qiita.com/Naoto9282/items/8427918564400968bd2b))、IDプロバイダから発行されたIDトークンなどのことです。

* JWTのようなポータブルトークンを発行したり、[Expressセッション](https://github.com/expressjs/session)を作成することで、認証状態を管理します。

* 認証ユーザの情報を`Request`オブジェクトに付与し、ルートハンドラで使用できるようにします。

Passportは、様々な認証方法に対応しています。Passportのコンセプト自体はシンプルですが、選択肢は多いです。Passportによって上述したようなステップは抽象化され、`@nestjs/passport`モジュールによってNestにラップされます。

この記事では、これらの強力で柔軟なモジュールを使用し、RESTful APIサーバでE2Eの認証を実装する方法についてご説明します。ここで説明するコンセプトをPassportで実装し、認証方法をカスタマイズできます。以下で説明するステップを踏むだけで、実際にアプリケーションを実装できます。最終的に作成されるアプリケーションの例は、[このリポジトリ](https://github.com/nestjs/nest/tree/master/sample/19-auth-jwt)をご参照ください。

## 必要なもの

必要なものを揃えましょう。今回のサンプルでは、認証をユーザ名とパスワードで行います。認証されると、サーバーはJWTを発行します。このJWTは、認証後のリクエストで、[Authorizationヘッダー](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Authorization)でベアラートークンとして送信できます。また、有効なJWTが含まれたリクエストだけがアクセス可能なルートを作成することもできます。

最初に、ユーザを認証するために必要なものを手に入れましょう。それから、JWTを発行する仕組みを追加します。最後に、リクエストに有効なJWTが含まれるかどうかチェックする保護されたルートを作成します。

では、必要なパッケージをインストールしましょう。Passportはユーザ名/パスワードによる認証を実装するために[passport-local](https://github.com/jaredhanson/passport-local)を提供しています。これから作成するアプリケーションに最適なので、これを使用します。

```sh
$ npm install --save @nestjs/passport passport passport-local
$ npm install --save-dev @types/passport-local
```

:::message
どんな認証方法でPassportを使用するときでも、`@nestjs/passport`と`passport`パッケージは必要です。この２つのパッケージに加える形で、あなたのプロジェクトに必要なPassportのパッケージ（`passport-jwt`や`passport-local`）を追加してください。また、TypeScriptを記述するときにアシストを受けられるように、上記の`@types/passport-local`のような型定義も併せてインストールしてください。
:::

## Passportによる認証の実装

認証機能を実装するための準備は整いました。Passportを使用した認証に必要な実装を始めましょう。Passportは、それ自体が小さなフレームワークであると考えてください。このフレームワークの優れた点は、認証プロセスをいくつかの基本的なステップに抽象化し、導入する認証方法に応じてカスタマイズできる点です。これがフレームワークに似ているのは、JSONオブジェクトのパラメータを変更したり、コールバック関数をカスタマイズすることで設定が行えるところです。`@nestjs/passport`モジュールはこのフレームワークをラップし、Nestアプリケーションで簡単に使用できるようにします。この記事でも`@nestjs/passport`を使用するのですが、まずPassportが[バニラ](https://ja.wikipedia.org/wiki/%E3%83%90%E3%83%8B%E3%83%A9_(%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2))の状態でどのように動作するのかを確かめましょう。

素の状態のパスポートに対して、２つの方法で認証方法を設定することができます。

1. 認証方法の固有のオプションを指定します。例えば、JWTの場合、トークンに署名するためのシークレットを指定することができます。
1. 検証のためのコールバック関数を使用して、Passportがユーザアカウントを保存しているストアと連携できるようにします。ユーザの存在を検証したり、ユーザを作成したり、クレデンシャルが有効か判別したりします。このコールバック関数は、認証が成功するとユーザを返し、失敗するとnullを返す必要があります。失敗というのは、ユーザが見つからないか、passport-localの場合においては、パスワードが一致しないということを意味します。

`@nestjs/passport`では、`PassportStrategy`クラスを継承することによって設定が可能です。サブクラス内の`super()`メソッドを呼び出し、オプションオブジェクトを渡すことで、上記の方法１でオプションを渡すことができます。方法２を使用して検証のためのコールバック関数を指定するためには、サブクラスで`validate()`メソッドを実装します。

では、`AuthModule`を生成し、その中に`AuthService`を生成します。

```sh
$ nest g module auth
$ nest g service auth
```

`AuthService`を実装するにあたって、ユーザに対する操作を`UsersService`にカプセル化すると便利なので、そのモジュールとサービスを作成しましょう。

```sh
$ nest g module users
$ nest g service users
```

生成されたファイルに、以下のコードを書き込んでください。このサンプルでは、`UsersService`にはユーザ情報のリストをハードコーディングしており、ユーザ名によってその中の一つを取得できるfindメソッドを実装しています。実際のアプリでは、ここでユーザのモデルの作成し、お好きなライブラリを使用して永続化を行ってください。（TypeORM, Sequelize, Mongooseなど）

```typescript:users/users.service.ts
import { Injectable } from '@nestjs/common';

// This should be a real class/interface representing a user entity
export type User = any;

@Injectable()
export class UsersService {
  private readonly users = [
    {
      userId: 1,
      username: 'john',
      password: 'changeme',
    },
    {
      userId: 2,
      username: 'maria',
      password: 'guess',
    },
  ];

  async findOne(username: string): Promise<User | undefined> {
    return this.users.find(user => user.username === username);
  }
}
```

`UsersModule`の中の`@Module`デコレータに`UsersService`を追加してください。これは、モジュールの外から`UsersService`を見えるようにするためです。これから`AuthService`で参照します。

```typescript:users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

`AuthService`では、ユーザを取得し、パスワードを検証する必要があります。そのために、`validateUser()`メソッドを作成しましょう。下のコードでは、ユーザオブジェクトをreturnする前に、ES6の[スプレッド構文](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Spread_syntax#%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%83%AA%E3%83%86%E3%83%A9%E3%83%AB%E3%81%A7%E3%81%AE%E3%82%B9%E3%83%97%E3%83%AC%E3%83%83%E3%83%89%E6%A7%8B%E6%96%87)を使用して、passwordプロパティを取り除いています。この`validateUser()`メソッドは、のちにPassport Localから呼び出されることになります。

:::message alert
当たり前ですが、実際のアプリケーションにおいてパスワードを平文で**保存**してはいけません。[bcrypt](https://github.com/kelektiv/node.bcrypt.js#readme)のような、ソルトつきのハッシュアルゴリズムを使用してください。そして、ハッシュ化したパスワードを保存し、保存したパスワードと渡されたパスワードをハッシュ化したものを比較します。したがって、平文のユーザパスワードを保存したり公開することは決してありません。この記事では、サンプルを簡潔にするために、その絶対的なルールを無視して平文を使用しています。絶対に実際のプロダクトでこんなことをしないでください。
:::

`AuthModule`で`UsersModule`をインポートできるように更新しましょう。

```typescript:auth/auth.module.ts
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
})
export class AuthModule {}
```


## Passport Localを実装する

Passport Localを使ってユーザ認証を実装しましょう。`auth`ディレクトリ内に`local.strategy.ts`というファイルを作成し、以下のコードを書き加えてください。

```typescript:auth/local.strategy.ts
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { AuthService } from './auth.service';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super();
  }

  async validate(username: string, password: string): Promise<any> {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

これは、前述したPassportでの認証の実装手順に基づいています。今回はPassport Localを使用するので、オプション設定は必要なく、コンストラクタで`super()`を呼び出すときもオプションのオブジェクトを渡していません。

:::message
`super`を呼び出すときにオプションのオブジェクトを渡すことで、Passportの動作をカスタマイズすることができます。今回の例のようにPassport localを使う場合、リクエストボディには`username`と`password`というプロパティが必要です。プロパティ名を変更する場合には、`super({ usernameField: 'email' })`のようにしてオプションのオブジェクトを渡します。詳しくは[Passportのドキュメント](http://www.passportjs.org/docs/configure/)をご参照ください。
:::

また、`validate()`メソッドも実装しています。`@nestjs/passport`で実装している`validate()`メソッドは、認証方法に応じたパラメータを取ります。Passport localでは、`validate()`メソッドは`validate(username: string, password:string): any`という形式になります。

ほとんどの検証は`AuthService`と`UsersService`の中で行われるので、このメソッドはとても簡潔なものです。どのPassportの認証方法を使用する場合でも`validate()`メソッドはこんな感じで、違うのはクレデンシャルの表現方法のみです。ユーザが見つかってクレデンシャルが有効な場合、ユーザが返され、Passportは`Request`オブジェクトに`user`プロパティを作成するなどのタスクを完了させます。それから、リクエストを処理するパイプラインが続行されます。ユーザが見つからなかった場合には例外が投げられ、[exceptionsレイヤ](https://docs.nestjs.com/exception-filters)によって処理されます。

認証方法ごとの`validate()`メソッドの違いは、ユーザの存在や有効性を**どうやって**判断するかという点です。例えばJWTを使用する場合、要件に応じて、デコードしたトークンに含まれる`userId`に一致するレコードがデータベースに存在するか、無効なトークンのリストに含まれているかを評価します。このようにしてサブクラスを使用し、認証方法ごとにバリデーションを行うパターンは一貫していて、簡潔かつ拡張性があります。

定義したPassportの機能を使用するために`AuthModule`を設定する必要があります。`auth.module.ts`を以下のように書き換えてください。

```typescript:auth/auth.module.ts
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { LocalStrategy } from './local.strategy';

@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
```

## Passport Gueard

