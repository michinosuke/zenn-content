---
title: "【公式ドキュメント和訳】NestJS Guards"
emoji: "🛡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs", "typescript", "javascript", "nodejs", "authorization"]
published: true
---

:::message
この記事は、2021/11/7時点のNestJSの仕様に依存します。
NestJSのバージョンが違ったり、記事の公開から長い時間が経っている場合、正確ではない可能性があります。
:::

> NestJS: v8.0.0
> 翻訳元: [authentication.md](https://github.com/nestjs/docs.nestjs.com/blob/b0d2c893948863cc416032e417a1a94b1b8b4d01/content/security/authentication.md)
> ライセンス: [MIT License](https://github.com/nestjs/docs.nestjs.com/blob/b0d2c893948863cc416032e417a1a94b1b8b4d01/LICENSE)

# Guards

Guardは`@Injectable()`デコレータのついたクラスのことです。Guardは`CanActivate`インターフェースを実装している必要があります。

![guard-1](https://docs.nestjs.com/assets/Guards_1.png)

Guardには、一つの責務があります。それは、ランタイム時に権限やロール、[ACL](https://www.ntt.com/bizon/glossary/e-a/acl.html)などの状態から、与えられたリクエストがルートハンドラーによって処理されるべきかどうかを判断することです。Guardはよく認可に使用されます。認可は認証と似ていて一緒に採用されることも多いですが、Expressアプリケーションにおいては[ミドルウェア](https://docs.nestjs.com/middleware)で処理されます。ミドルウェアは認証に向いています。なぜなら、トークンの検証や、`request`オブジェクトにプロパティを付与するタスクは、特定のルートやそのメタデータに大きく依存することではないからです。

しかし、ミドルウェアはそのままではかなり貧弱な機能です。なぜかというと、ミドルウェア自身は、`next()`関数を呼び出した後にどのような処理が実行されるかを知らないからです。一方、**Guard**では`ExecutionContext`インスタンスにアクセス可能なので、次に何が実行されるのかを知ることができます。ExceptionフィルタやPipe、Interceptorといった機能は、リクエストとレスポンスのサイクルにロジックを宣言的に挿入できるようにデザインされたものです。これらの機能を使うことによって、DRYと宣言的な状態を維持できます。

:::message
Guardはミドルウェアの**後**、interceptorやpipeの**前**に実行されます。
:::

## Guardを使用した認可

前述した通り、認可においてはユーザが権限を持っている場合のみルートを使用できる必要があるため、Guardを利用すると便利です。ここでは、リクエストヘッダにトークンが付与されているユーザを認証済みとみなすための`AuthGuard`を実装してみましょう。トークンを取り出して検証し、そのトークンの情報を元にリクエストが引き続き処理されるべきかを判断します。

```typescript:auth.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

:::message
認証を実装した実際の例をお探しなら、[この章](https://docs.nestjs.com/security/authentication)をご覧ください。認可のサンプルをもっとみたい場合は、[このページ](https://docs.nestjs.com/security/authorization)を参照してください。
:::

`validateRequest()`関数内のロジックは、シンプルなものです。このサンプルで理解していただきたいのは、Guardがどのようにしてリクエストとレスポンスのサイクルに組み込まれているかという点です。

Guardは、`canActivate()`関数を実装する必要があります。この関数は、リクエストが許可されるかどうかを示すbooleanを返します。返す値は、同期でも、`Promise`や`Observable`を使用した非同期でも問題ありません。Nestは返された値によって以下のような動作をします。

* `true`が返されると、リクエストは引き続き処理されます。
* `false`が返されると、リクエストは拒否されます。

## 実行のコンテキスト

`canActivate()`関数は、`ExecutionContext`インスタンスを引数に取ります。`ExecutionContext`は、`ArgumentsHost`から引き継がれます。`ArgumentsHost`はExceptionフィルタの章でも登場しました。下のサンプルでは、`Request`オブジェクトを参照するために、`ArgumentsHost`で定義したヘルパーメソッドを使用します。詳しくは、[Exceptionフィルタ](https://docs.nestjs.com/exception-filters#arguments-host)の章の**Arguments host**を参照してください。

`ExecutionContext`は`ArgumentsHost`を継承しており、現在の実行プロセスの詳細を取得する新しいヘルパーメソッドが追加されています。実行プロセスの詳細な情報を取得できるので、幅広いコントローラやメソッド、実行コンテキストで汎用的に動作するGuardを作成することができます。詳しくは、[ここ](https://docs.nestjs.com/fundamentals/execution-context)の`ExecutionContext`を参照してください。

## ロールを使用した認証

特定のロールを持ったユーザのみにアクセス権限を与えるGuardを作成しましょう。基本的なGuardのテンプレートを使いましょう。実装は次の章で行います。下のテンプレートの状態だと、全てのリクエストを許可しています。

```typescript:roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
```

## Guardをバインドする

PipeやExceptionフィルタと同じく、Guardも**コントローラのスコープ**、メソッドのスコープ、グローバルのスコープを指定できます。以下では、`@UseGuards()`デコレータを使用して、コントローラのスコープのGuardを実装してみます。このデコレータは１つ引数、もしくはコンマ区切りのリストを引数に取ります。なので、一つのデコレーションで簡単にGuardを適用できます。

```typescript
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

:::message
`@UseGuards()`デコレータは`@nestjs/common`パッケージからインポートしてください。
:::

上のコードでは、インスタンスの代わりに`RolesGuard`タイプを渡し、インスタンス化をフレームワークに任せることでDI(dependency injection)を有効化しています。PipeやExceptionフィルタと同じく、インスタンスを渡すこともできます。

```typescript
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

 上の書き方だと、このコントローラ内で定義される全てのハンドラにGuardが適用されます。Guardを一つのメソッドにのみ適用したい場合は、`@UseGuards()`デコレータを**メソッドに対して**で適用してください。グローバルに対してGuardを適用したい場合は、Nestのアプリケーションインスタンスにあるメソッドの`useGlobalGuards()`を使用してください。

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

:::message
ハイブリッドアプリの場合、デフォルトで、`useGlobalGuards()`メソッドはゲートウェイとマイクロサービスに対してGuardを適用しません。（変更する方法は、[ここ](https://docs.nestjs.com/faq/hybrid-application)に書いてあります。）ハイブリッドではないスタンダードなマイクロサービスアプリケーションの場合、`useGlobalGuards()`はグローバルに機能します。
:::

グローバルなGuardはアプリケーション全体の全てのコントローラとルートハンドラに適用されます。DIに関しては、グローバルなGuardは`useGlobalGuards()`によってモジュールの外側で登録されているので、DIを使用することができません。解決策としては、以下のような構文でモジュールに直接Guardを設定する方法があります。

```typescript:app.module.ts
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

:::message
この方法を使ってGuardでDIを使用するときに気をつけなければいけないのは、これがどこのモジュールに書かれているかに関係なく、Guardがグローバルであるという点です。では、どこに書くべきなのでしょうか。答えは、Guardが定義されているモジュールです。上のサンプルの場合だと、`RolesGuard`が定義されているモジュールということになります。
また、カスタムプロバイダの登録は`useClass`以外でも可能です。詳しくは、[ここ](https://docs.nestjs.com/fundamentals/custom-providers)をご参照ください。
:::

## ハンドラ単位でロールを設定する

`RolesGuard`は問題なく動作しますが、まだ改善の余地があります。というのも、Guardには[実行コンテキスト](https://docs.nestjs.com/fundamentals/execution-context)という重要な機能があるからです。現状の`RolesGuard`は、ロールについて関与せず、それぞれのハンドラでどのようなロールが許可されているか把握していません。`CatsController`がルートごとに異なる権限を求める場合を考えてみましょう。管理者ユーザのみ使用可能なものもあれば、誰でも使用可能なものもあるでしょう。柔軟性と再利用性を保ったまま、どのようにしてルートに対してロールを割り当てれば良いでしょうか？

そこで、**カスタムメタデータ**の出番です。（詳しくは、[ここ](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata)をご参照ください）Nestでは、カスタムメタデータをルートハンドラに付与するために、`@SetMetadata()`というデコレータを用意しています。メタデータに`role`のデータを提供することで、よりスマートなGuardを実装することが可能になります。では、実際に`@SetMetadata`を使ってみましょう。

```typescript:cats.controller.ts
@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

:::message
`@SetMetadata`デコレータは、`@nestjs/common`パッケージからインポートしてください。
:::

上のコードでは、`create()`メソッドに`roles`というメタデータを付与しています。`roles`がキーで、`['admin']`が値です。これは問題なく動作しますが、ルートに対して直接`@SetMetadata()`を使用するのは推奨されません。代わりに、以下のようにデコレータを定義します。

```typescript:roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

こうすることで、可読性が上がり、より強く型付けできます。では、作成した`@Roles()`デコレータを`create()`メソッドに使用してみましょう。

```typescript:cats.controller.ts
@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

## 仕上げ

作成した`Roles`を`RolesGuard`で利用しましょう。現状では、`RolesGuard`は常に`true`を返し、全てのリクエストは許可されています。これから実装したいのは、リクエストしたユーザのロールと、ルートにアクセスする許可のあるロールを比較し、それに基づいて返す値を変える仕組みです。ルートに権限のあるロールのメタデータにアクセスするために、`@nestjs/core`パッケージに含まれている`Reflector`というヘルパークラスを使用します。

```typescript:roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

:::message
Node.jsでは、認可済みのユーザを`request`オブジェクトに付与するのが一般的です。したがって、上のサンプルコードでも、`request.user`にユーザのインスタンスとロールが含まれている前提になっています。実際のプロダクトにおいても、認証のためのGuardやミドルウェアでこのような紐付けを行う場合があるでしょう。詳しくは、[認証](https://docs.nestjs.com/security/authentication)の章を参照してください。
:::

:::message
`matchRoles()`関数のロジックは、シンプルなものでも、複雑なものでも構いません。このサンプルは、リクエストとレスポンスのサイクルにどうGuardが関わるかを説明することに重点を置いているので、`matchRoles()`関数の実装については扱いません。
:::

`Reflector`をコンテキストに応じて使用する方法の詳細については、実行コンテキストの章の[Reflectionとメタデータ](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata)の章で[Reflectionとメタデータ](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata)を参照してください。

ユーザがエンドポイントに対して必要な権限を満たしていなかった場合、Nestは自動的に以下のようなレスポンスを返します。

```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

内部的には、Guardが`false`を返したとき、フレームワークは`ForbiddenException`を投げています。他のレスポンスを返したい場合は、独自に例外を投げてください。例えば、以下のようにします。

```typescript
throw new UnauthorizedException();
```

Guardで投げられた例外は、[Exceptionレイヤ](https://docs.nestjs.com/exception-filters)で処理されます。ここでいうExceptionレイヤとは、グローバルなExceptionレイヤと、現在のコンテキストに適用されたExceptionレイヤのことです。

:::message
認可を実装した実際のサンプルをお探しなら、[認可](https://docs.nestjs.com/security/authorization)の章をご参照ください。
:::

## 修正依頼について

誤字・脱字や翻訳ミスなどを見つけた場合は、ぜひご連絡ください。
もっといい翻訳の仕方がある、等でも嬉しいです。
GitHubでのプルリクや、TwitterのDMでも受け付けております。

https://twitter.com/Michin0suke
