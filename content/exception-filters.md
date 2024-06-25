### Exception filters

Nest는 애플리케이션 전반에서 처리되지 않은 모든 예외를 처리하는 **예외 레이어**를 내장하고 있습니다. 예외가 애플리케이션 코드에서 처리되지 않으면 이 레이어에서 예외를 잡아 적절한 사용자 친화적인 응답을 자동으로 전송합니다.

<figure>
  <img src="https://docs.nestjs.com/assets/Filter_1.png" />
</figure>

기본적으로 이 작업은 `HttpException`(및 그 하위 클래스) 유형의 예외를 처리하는 내장 **전역 예외 필터**에 의해 수행됩니다. 예외가 **인식되지 않으면**(즉, `HttpException`이 아니거나 `HttpException`에서 상속된 클래스가 아닌 경우), 내장 예외 필터는 다음과 같은 기본 JSON 응답을 생성합니다:

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> info **힌트** 전역 예외 필터는 `http-errors` 라이브러리를 부분적으로 지원합니다. 기본적으로 `statusCode`와 `message` 속성을 포함하는 모든 예외는 적절하게 채워지고 응답으로 전송됩니다(인식되지 않은 예외에 대한 기본 `InternalServerErrorException` 대신).

#### 표준 예외 던지기

Nest는 `@nestjs/common` 패키지에서 내장된 `HttpException` 클래스를 제공합니다. 일반적인 HTTP REST/GraphQL API 기반 애플리케이션에서는 특정 오류 조건이 발생할 때 표준 HTTP 응답 객체를 보내는 것이 모범 사례입니다.

예를 들어, `CatsController`에는 `findAll()` 메서드(`GET` 라우트 핸들러)가 있습니다. 이 라우트 핸들러가 어떤 이유로 예외를 던진다고 가정해 보겠습니다. 이를 시연하기 위해 다음과 같이 하드 코딩합니다:

```typescript
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

> info **힌트** 여기서 `HttpStatus`를 사용했습니다. 이는 `@nestjs/common` 패키지에서 가져온 헬퍼 열거형입니다.

클라이언트가 이 엔드포인트를 호출하면 응답은 다음과 같습니다:

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

`HttpException` 생성자는 응답을 결정하는 두 개의 필수 인수를 받습니다:

- `response` 인수는 JSON 응답 본문을 정의합니다. 이는 `string` 또는 아래 설명된 대로 `object`일 수 있습니다.
- `status` 인수는 [HTTP 상태 코드](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)를 정의합니다.

기본적으로 JSON 응답 본문에는 두 가지 속성이 포함됩니다:

- `statusCode`: `status` 인수로 제공된 HTTP 상태 코드를 기본값으로 합니다.
- `message`: `status`에 기반한 HTTP 오류의 짧은 설명입니다.

JSON 응답 본문의 메시지 부분만 재정의하려면 `response` 인수에 문자열을 제공하면 됩니다. JSON 응답 본문 전체를 재정의하려면 `response` 인수에 객체를 전달합니다. Nest는 객체를 직렬화하여 JSON 응답 본문으로 반환합니다.

두 번째 생성자 인수인 `status`는 유효한 HTTP 상태 코드여야 합니다. 모범 사례는 `@nestjs/common`에서 가져온 `HttpStatus` 열거형을 사용하는 것입니다.

세 번째 생성자 인수(선택 사항)인 `options`는 오류의 [원인](https://nodejs.org/en/blog/release/v16.9.0/#error-cause)을 제공하는 데 사용할 수 있습니다. 이 `cause` 객체는 응답 객체에 직렬화되지 않지만, 로깅 목적으로 유용한 내부 오류 정보를 제공할 수 있습니다.

다음은 전체 응답 본문을 재정의하고 오류 원인을 제공하는 예제입니다:

```typescript
@Get()
async findAll() {
  try {
    await this.service.findAll()
  } catch (error) {
    throw new HttpException({
      status: HttpStatus.FORBIDDEN,
      error: 'This is a custom message',
    }, HttpStatus.FORBIDDEN, {
      cause: error
    });
  }
}
```

위 예제를 사용하면 응답은 다음과 같습니다:

```json
{
  "status": 403,
  "error": "This is a custom message"
}
```

#### 사용자 정의 예외

많은 경우 내장된 Nest HTTP 예외를 사용할 수 있으며, 이는 다음 섹션에서 설명합니다. 사용자 정의 예외를 작성해야 하는 경우, 사용자 정의 예외가 기본 `HttpException` 클래스에서 상속되도록 하는 **예외 계층 구조**를 만드는 것이 좋습니다. 이 접근 방식을 사용하면 Nest가 예외를 인식하고 오류 응답을 자동으로 처리합니다. 다음과 같이 사용자 정의 예외를 구현해 보겠습니다:

```typescript
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

`ForbiddenException`은 기본 `HttpException`을 확장하므로 내장 예외 핸들러와 원활하게 작동하며, 따라서 `findAll()` 메서드 내에서 이를 사용할 수 있습니다.

```typescript
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

#### 내장된 HTTP 예외

Nest는 기본 `HttpException`을 상속하는 표준 예외 세트를 제공합니다. 이들은 `@nestjs/common` 패키지에서 가져올 수 있으며, 가장 일반적인 HTTP 예외를 나타냅니다:

- `BadRequestException`
- `UnauthorizedException`
- `NotFoundException`
- `ForbiddenException`
- `NotAcceptableException`
- `RequestTimeoutException`
- `ConflictException`
- `GoneException`
- `HttpVersionNotSupportedException`
- `PayloadTooLargeException`
- `UnsupportedMediaTypeException`
- `UnprocessableEntityException`
- `InternalServerErrorException`
- `NotImplementedException`
- `ImATeapotException`
- `MethodNotAllowedException`
- `BadGatewayException`
- `ServiceUnavailableException`
- `GatewayTimeoutException`
- `PreconditionFailedException`

모든 내장 예외는 오류 `cause` 및 오류 설명을 `options` 매개변수를 사용하여 제공할 수 있습니다:

```typescript
throw new BadRequestException('Something bad happened', { cause: new Error(), description: 'Some error description' })
```

위 예제를 사용하면 응답은 다음과 같습니다:

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400,
}
```

#### 예외 필터

기본(내장) 예외 필터는 많은 경우를 자동으로 처리할 수 있지만, 예외 레이어에 대한 **완전한 제어**를 원할 수 있습니다. 예를 들어, 로깅을 추가하거나 동적 요소를 기반으로 다른 JSON 스키마를 사용하고자 할 수 있습니다. **예외 필터**는 이러한 목적을 위해 설계되었습니다. 예외 필터를 사용하면 제어 흐름과 클라이언트에 다시 전송되는 응답의 내용을 정확하게 제어할 수 있습니다.

`HttpException` 클래스의 인스턴스인 예외를 잡고 이를 위한 사용자 정의 응답 로직을 구현하는 예외 필터를 만들어 보겠습니다. 이를 위해 기본 플랫폼 `Request` 및 `Response` 객체에 접근해야 합니다. `Request` 객체에 접근하여 원래 `url`을 가져와 로깅 정보에 포함할 것입니다. `Response` 객체를 사용하여 `response.json()` 메서드를 통해 전송되는 응답을 직접 제어할 것입니다.

```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

> info **힌트** 모든 예외 필터는 일반 `ExceptionFilter<T>` 인터페이스를 구현해야 합니다. 이를 통해 `catch(exception: T, host: ArgumentsHost)` 메서드를 해당 시그니처와 함께 제공해야 합니다. `T`는 예외 유형을 나타냅니다.

> warning **경고** `@nestjs/platform-fastify`를 사용하는 경우 `response.json()` 대신 `response.send()`를 사용할 수 있습니다. 올바른 타입을 `fastify`에서 가져오는 것을 잊지 마세요.

`@Catch(HttpException)` 데코레이터는 필요한 메타데이터를 예외 필터에 바인딩하여 이 필터가 `HttpException` 유형의 예외만 찾도록 합니다. `@Catch()` 데코레이터는 단일 매개변수 또는 쉼표로 구분된 목록을 받을 수 있습니다. 이를 통해 여러 유형의 예외에 대해 필터를 한 번에 설정할 수 있습니다.



#### Arguments host

`catch()` 메서드의 매개변수를 살펴보겠습니다. `exception` 매개변수는 현재 처리 중인 예외 객체입니다. `host` 매개변수는 `ArgumentsHost` 객체입니다. `ArgumentsHost`는 강력한 유틸리티 객체로, [실행 컨텍스트 장](https://docs.nestjs.com/fundamentals/execution-context)에서 더 자세히 살펴볼 것입니다. 이 코드 샘플에서는 원래 요청 핸들러에 전달되는 `Request` 및 `Response` 객체에 대한 참조를 얻기 위해 사용됩니다(예외가 발생한 컨트롤러에서). 이 코드 샘플에서는 `ArgumentsHost`의 몇 가지 도우미 메서드를 사용하여 원하는 `Request` 및 `Response` 객체를 얻었습니다. `ArgumentsHost`에 대해 자세히 알아보려면 [여기](https://docs.nestjs.com/fundamentals/execution-context)를 참조하세요.

#### 필터 바인딩

새로운 `HttpExceptionFilter`를 `CatsController`의 `create()` 메서드에 연결해 보겠습니다.

```typescript
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

> info **힌트** `@UseFilters()` 데코레이터는 `@nestjs/common` 패키지에서 가져옵니다.

여기서는 `@UseFilters()` 데코레이터를 사용했습니다. `@Catch()` 데코레이터와 마찬가지로 단일 필터 인스턴스 또는 쉼표로 구분된 필터 인스턴스 목록을 받을 수 있습니다. 여기서는 `HttpExceptionFilter` 인스턴스를 즉시 생성했습니다. 또는 클래스 자체를 전달하여 인스턴스화를 프레임워크에 맡기고 **의존성 주입**을 활성화할 수 있습니다.

```typescript
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

> info **힌트** 가능하면 인스턴스 대신 클래스를 사용하여 필터를 적용하는 것이 좋습니다. 이는 Nest가 전체 모듈에 걸쳐 동일한 클래스의 인스턴스를 쉽게 재사용할 수 있기 때문에 **메모리 사용량**을 줄일 수 있습니다.

위 예제에서는 `HttpExceptionFilter`가 단일 `create()` 라우트 핸들러에만 적용되어 메서드 범위가 됩니다. 예외 필터는 메서드 범위의 컨트롤러/리졸버/게이트웨이, 컨트롤러 범위 또는 전역 범위로 설정할 수 있습니다. 예를 들어, 필터를 컨트롤러 범위로 설정하려면 다음과 같이 합니다:

```typescript
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

이 구성은 `CatsController` 내부에 정의된 모든 라우트 핸들러에 대해 `HttpExceptionFilter`를 설정합니다.

전역 범위 필터를 생성하려면 다음과 같이 합니다:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```

> warning **경고** `useGlobalFilters()` 메서드는 게이트웨이 또는 하이브리드 애플리케이션에 대한 필터를 설정하지 않습니다.

전역 범위 필터는 애플리케이션 전체에서 모든 컨트롤러와 모든 라우트 핸들러에 사용됩니다. 의존성 주입 측면에서, 전역 필터는 모듈 외부에서 등록된 경우(`useGlobalFilters()`를 사용하여 위의 예제처럼) 종속성을 주입할 수 없습니다. 이 문제를 해결하려면 다음과 같은 구성을 사용하여 **모듈 내에서 직접** 전역 범위 필터를 등록할 수 있습니다:

```typescript
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

> info **힌트** 이 방법을 사용하여 필터에 대해 의존성 주입을 수행할 때, 이 구성이 사용되는 모듈에 상관없이 필터는 실제로 전역적입니다. 어디에서 이 작업을 수행해야 할까요? 필터(`HttpExceptionFilter`가 정의된 모듈)를 선택하세요. 또한 `useClass`는 사용자 정의 provider 등록을 처리하는 유일한 방법이 아닙니다. 자세한 내용은 [여기](https://docs.nestjs.com/fundamentals/custom-providers)를 참조하세요.

이 기술을 사용하여 필요한 만큼 많은 필터를 추가할 수 있습니다. 단순히 providers 배열에 각 필터를 추가하세요.

#### 모든 예외 잡기

**모든** 처리되지 않은 예외(예외 유형에 상관없이)를 잡으려면 `@Catch()` 데코레이터의 매개변수 목록을 비워 둡니다. 예: `@Catch()`.

아래 예제는 [HTTP 어댑터](https://docs.nestjs.com/faq/http-adapter)를 사용하여 응답을 제공하므로 플랫폼 비종속적인 코드입니다.

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // 특정 상황에서 `httpAdapter`는 생성자 메서드에서 사용할 수 없으므로
    // 여기서 해결해야 합니다.
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

> warning **경고** 특정 유형에 바인딩된 필터와 모든 예외를 잡는 필터를 결합할 때, "모든 예외를 잡는" 필터를 먼저 선언하여 특정 필터가 바인딩된 유형을 올바르게 처리할 수 있도록 해야 합니다.

#### 상속

일반적으로 애플리케이션 요구 사항을 충족하기 위해 완전히 맞춤화된 예외 필터를 만듭니다. 그러나 기본 기본 **전역 예외 필터**를 확장하고 특정 요소에 따라 동작을 재정의하려는 경우가 있을 수 있습니다.

예외 처리를 기본 필터에 위임하려면 `BaseExceptionFilter`를 확장하고 상속된 `catch()` 메서드를 호출해야 합니다.

```typescript
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```

> warning **경고** `BaseExceptionFilter`를 확장하는 메서드 범위 및 컨트롤러 범위 필터는 `new`로 인스턴스화해서는 안 됩니다. 대신 프레임워크가 자동으로 인스턴스화하도록 해야 합니다.

전역 필터는 기본 필터를 확장할 **수 있습니다**. 이는 두 가지 방법으로 수행할 수 있습니다.

첫 번째 방법은 사용자 정의 전역 필터를 인스턴스화할 때 `HttpAdapter` 참조를 주입하는 것입니다:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(3000);
}
bootstrap();
```

두 번째 방법은 [여기](https://docs.nestjs.com/exception-filters#binding-filters)에서 설명한 대로 `APP_FILTER` 토큰을 사용하는 것입니다.
