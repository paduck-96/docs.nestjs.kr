### Middleware

미들웨어는 라우트 핸들러 **전에** 호출되는 함수입니다. 미들웨어 함수는 애플리케이션의 요청-응답 사이클에서 [request](https://expressjs.com/en/4x/api.html#req) 및 [response](https://expressjs.com/en/4x/api.html#res) 객체와 `next()` 미들웨어 함수에 접근할 수 있습니다. **next** 미들웨어 함수는 일반적으로 `next`라는 변수로 표시됩니다.

<figure><img src="https://docs.nestjs.com/assets/Middlewares_1.png" /></figure>

기본적으로 Nest 미들웨어는 [express](https://expressjs.com/en/guide/using-middleware.html) 미들웨어와 동일합니다. 공식 Express 문서에서 미들웨어의 기능을 다음과 같이 설명합니다:

<blockquote class="external">
  미들웨어 함수는 다음 작업을 수행할 수 있습니다:
  <ul>
    <li>코드를 실행합니다.</li>
    <li>요청 및 응답 객체를 수정합니다.</li>
    <li>요청-응답 사이클을 종료합니다.</li>
    <li>스택의 다음 미들웨어 함수를 호출합니다.</li>
    <li>현재 미들웨어 함수가 요청-응답 사이클을 종료하지 않으면 <code>next()</code>를 호출하여 제어를 다음 미들웨어 함수로 전달해야 합니다. 그렇지 않으면 요청이 중단됩니다.</li>
  </ul>
</blockquote>

사용자 정의 Nest 미들웨어는 함수 또는 `@Injectable()` 데코레이터가 있는 클래스에서 구현할 수 있습니다. 클래스는 `NestMiddleware` 인터페이스를 구현해야 하며, 함수는 특별한 요구 사항이 없습니다. 클래스 메서드를 사용하여 간단한 미들웨어 기능을 구현하는 것부터 시작해 보겠습니다.

> warning **경고** `Express`와 `fastify`는 미들웨어를 다르게 처리하며, 다른 메서드 시그니처를 제공합니다. 자세한 내용은 [여기](https://docs.nestjs.com/techniques/performance#middleware)를 참조하세요.

```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}
```

#### 의존성 주입

Nest 미들웨어는 의존성 주입을 완전히 지원합니다. providers 및 controllers와 마찬가지로 동일한 모듈 내에서 사용 가능한 **의존성 주입**이 가능합니다. 일반적으로 이는 `constructor`를 통해 수행됩니다.

#### 미들웨어 적용

`@Module()` 데코레이터에는 미들웨어를 배치할 곳이 없습니다. 대신, 모듈 클래스의 `configure()` 메서드를 사용하여 설정합니다. 미들웨어를 포함하는 모듈은 `NestModule` 인터페이스를 구현해야 합니다. `AppModule` 레벨에서 `LoggerMiddleware`를 설정해 보겠습니다.

```typescript
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
```

위 예제에서는 이전에 `CatsController` 내부에 정의된 `/cats` 라우트 핸들러에 대해 `LoggerMiddleware`를 설정했습니다. 또한 요청 메서드에 따라 특정 라우트로 미들웨어를 제한할 수 있습니다. 아래 예제에서는 원하는 요청 메서드 유형을 참조하기 위해 `RequestMethod` 열거형을 가져옵니다.

```typescript
import { Module, NestModule, RequestMethod, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

> info **힌트** `configure()` 메서드는 `async/await`을 사용하여 비동기적으로 만들 수 있습니다(예: `configure()` 메서드 본문 내에서 비동기 작업의 완료를 `await`할 수 있습니다).

> warning **경고** `express` 어댑터를 사용할 때, NestJS 앱은 기본적으로 `body-parser` 패키지에서 `json` 및 `urlencoded`를 등록합니다. 따라서 `MiddlewareConsumer`를 통해 해당 미들웨어를 사용자 정의하려면 `NestFactory.create()`로 애플리케이션을 생성할 때 `bodyParser` 플래그를 `false`로 설정하여 전역 미들웨어를 비활성화해야 합니다.

#### 라우트 와일드카드

패턴 기반 라우트도 지원됩니다. 예를 들어, 별표는 **와일드카드**로 사용되며, 문자 조합을 매칭합니다:

```typescript
forRoutes({ path: 'ab*cd', method: RequestMethod.ALL });
```

`'ab*cd'` 라우트 경로는 `abcd`, `ab_cd`, `abecd` 등을 매칭합니다. `?`, `+`, `*`, `()` 문자는 라우트 경로에서 사용할 수 있으며 정규 표현식의 하위 집합입니다. 하이픈(`-`) 및 점(`.`)은 문자열 기반 경로에서 문자 그대로 해석됩니다.

> warning **경고** `fastify` 패키지는 최신 버전의 `path-to-regexp` 패키지를 사용하므로 와일드카드 별표(`*`)를 더 이상 지원하지 않습니다. 대신 매개변수를 사용해야 합니다(예: `(.*)`, `:splat*`).

#### 미들웨어 컨슈머

`MiddlewareConsumer`는 도우미 클래스입니다. 미들웨어를 관리하기 위한 여러 내장 메서드를 제공합니다. 이 모든 메서드는 [유창한 스타일](https://en.wikipedia.org/wiki/Fluent_interface)로 **체이닝**할 수 있습니다. `forRoutes()` 메서드는 단일 문자열, 여러 문자열, `RouteInfo` 객체, 컨트롤러 클래스 및 여러 컨트롤러 클래스를 받을 수 있습니다. 대부분의 경우, 쉼표로 구분된 **컨트롤러** 목록을 전달하면 됩니다. 아래는 단일 컨트롤러 예제입니다:

```typescript
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
```

> info **힌트** `apply()` 메서드는 단일 미들웨어 또는 여러 미들웨어를 지정하는 여러 인수를 받을 수 있습니다. 자세한 내용은 [여기](https://docs.nestjs.com/middleware#multiple-middleware)를 참조하세요.

#### 라우트 제외

때때로 특정 라우트에 미들웨어를 적용하지 않도록 **제외**하고 싶을 때가 있습니다. `exclude()` 메서드를 사용하여 특정 라우트를 쉽게 제외할 수 있습니다. 이 메서드는 단일 문자열, 여러 문자열 또는 제외할 라우트를 식별하는 `RouteInfo` 객체를 받을 수 있습니다:

```typescript
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/(.*)',
  )
  .forRoutes(CatsController);
```

> info **힌트** `exclude()` 메서드는 [path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters) 패키지를 사용하여 와일드카드 매개변수를 지원합니다.

위 예제에서는 `LoggerMiddleware`가 `CatsController` 내부에 정의된 모든 라우트에 적용되지만, `exclude()` 메서드에 전달된 세 라우트는 제외됩니다.

#### 함수형 미들웨어

지금까지 사용한 `LoggerMiddleware` 클래스는 매우 간단합니다. 멤버도 없고, 추가 메서드도 없으며, 종속성도 없습니다. 이를 클래스 대신 간단한 함수로 정의할 수 있지 않을까요? 사실 가능합니다. 이러한 유형의 미들웨어를 **함수형 미들웨어**라고 합니다. 차이점을 설명하기 위해 로거 미들웨어를 클래스 기반에서 함수형 미들웨어로 변환해 보겠습니다:

```typescript
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request...`);
  next();
};
```

`AppModule`에서 이를 사용합니다:

```typescript
consumer
  .apply(logger)
  .forRoutes(CatsController);
```

> info **힌트** 미들

웨어에 종속성이 필요하지 않은 경우, 더 간단한 **함수형 미들웨어**를 사용하는 것이 좋습니다.

#### 다중 미들웨어

앞서 언급한 것처럼, 순차적으로 실행되는 여러 미들웨어를 바인딩하려면 `apply()` 메서드에 쉼표로 구분된 목록을 제공하기만 하면 됩니다:

```typescript
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```

#### 전역 미들웨어

모든 등록된 라우트에 한 번에 미들웨어를 바인딩하려면 `INestApplication` 인스턴스에서 제공하는 `use()` 메서드를 사용할 수 있습니다:

```typescript
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(3000);
```

> info **힌트** 전역 미들웨어에서 DI 컨테이너에 접근하는 것은 불가능합니다. 대신 `app.use()`를 사용할 때 [함수형 미들웨어](https://docs.nestjs.com/middleware#functional-middleware)를 사용할 수 있습니다. 또는 클래스 미들웨어를 사용하고 `AppModule`(또는 다른 모듈) 내에서 `.forRoutes('*')`로 소비할 수 있습니다.
