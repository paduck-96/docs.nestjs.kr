## Interceptors

인터셉터는 `@Injectable()` 데코레이터로 주석이 달린 클래스이며 `NestInterceptor` 인터페이스를 구현합니다.

<figure><img src="https://docs.nestjs.com/assets/Interceptors_1.png" /></figure>

인터셉터는 [측면 지향 프로그래밍](https://en.wikipedia.org/wiki/Aspect-oriented_programming) (AOP) 기술에서 영감을 받아 유용한 기능을 제공합니다. 이를 통해 다음과 같은 작업이 가능합니다:

- 메서드 실행 전/후에 추가 로직 바인딩
- 함수에서 반환된 결과 변환
- 함수에서 발생한 예외 변환
- 기본 함수 동작 확장
- 특정 조건에 따라 함수를 완전히 재정의 (예: 캐싱 목적)

# Basics

각 인터셉터는 두 개의 인수를 받는 `intercept()` 메서드를 구현합니다. 첫 번째 인수는 [가드](https://docs.nestjs.com/guards)와 동일한 `ExecutionContext` 인스턴스입니다. `ExecutionContext`는 `ArgumentsHost`에서 상속됩니다. `ArgumentsHost`는 예외 필터 장에서 본 것처럼 원래 핸들러에 전달된 인수의 래퍼이며, 애플리케이션의 유형에 따라 다른 인수 배열을 포함합니다. 자세한 내용은 [예외 필터](https://docs.nestjs.com/exception-filters#arguments-host) 장을 참조하십시오.

# Execution context

`ArgumentsHost`를 확장함으로써 `ExecutionContext`는 현재 실행 프로세스에 대한 추가 세부 정보를 제공하는 몇 가지 새로운 헬퍼 메서드를 추가합니다. 이러한 세부 정보는 더 광범위한 컨트롤러, 메서드 및 실행 컨텍스트에서 작동할 수 있는 더 일반적인 인터셉터를 만드는 데 유용할 수 있습니다. `ExecutionContext`에 대한 자세한 내용은 [여기](https://docs.nestjs.com/fundamentals/execution-context)에서 확인할 수 있습니다.

# Call handler

두 번째 인수는 `CallHandler`입니다. `CallHandler` 인터페이스는 `handle()` 메서드를 구현하여 인터셉터에서 어느 시점에 라우트 핸들러 메서드를 호출할 수 있게 합니다. `intercept()` 메서드에서 `handle()` 메서드를 호출하지 않으면 라우트 핸들러 메서드는 전혀 실행되지 않습니다.

이 접근 방식은 `intercept()` 메서드가 요청/응답 스트림을 효과적으로 **감싸는** 것을 의미합니다. 결과적으로 최종 라우트 핸들러의 실행 **전후**에 사용자 정의 로직을 구현할 수 있습니다. `intercept()` 메서드에서 `handle()`을 호출하기 **전에** 코드를 실행할 수 있는 것은 명백하지만, 이후에 무슨 일이 일어날지 어떻게 영향을 미칠 수 있을까요? `handle()` 메서드는 `Observable`을 반환하므로 강력한 [RxJS](https://github.com/ReactiveX/rxjs) 연산자를 사용하여 응답을 추가로 조작할 수 있습니다. 측면 지향 프로그래밍 용어를 사용하면, 라우트 핸들러의 호출(즉, `handle()` 호출)은 [Pointcut](https://en.wikipedia.org/wiki/Pointcut)이라고 하며, 이는 추가 로직이 삽입되는 지점을 나타냅니다.

예를 들어, `POST /cats` 요청이 들어온다고 가정해보겠습니다. 이 요청은 `CatsController` 내부에 정의된 `create()` 핸들러로 전달됩니다. `handle()` 메서드를 호출하지 않는 인터셉터가 중간에 호출되면 `create()` 메서드는 실행되지 않습니다. 일단 `handle()`이 호출되고 그 `Observable`이 반환되면 `create()` 핸들러가 트리거됩니다. `Observable`을 통해 응답 스트림을 수신한 후 추가 작업을 수행하고 최종 결과를 호출자에게 반환할 수 있습니다.

# Aspect interception

첫 번째 사용 사례로 사용자 상호작용을 기록하는 인터셉터를 사용해 보겠습니다 (예: 사용자 호출 저장, 비동기 이벤트 디스패치 또는 타임스탬프 계산). 아래는 간단한 `LoggingInterceptor`입니다:

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```

`NestInterceptor<T, R>`는 제네릭 인터페이스로, `T`는 `Observable<T>` 타입(응답 스트림 지원)을 나타내고, `R`은 `Observable<R>`로 래핑된 값의 타입입니다.

인터셉터는 컨트롤러, 프로바이더, 가드 등과 같이 `constructor`를 통해 **의존성을 주입**할 수 있습니다.

`handle()`은 RxJS `Observable`을 반환하므로, 스트림을 조작하기 위해 사용할 수 있는 연산자가 다양합니다. 위 예제에서는 `tap()` 연산자를 사용하여 `Observable` 스트림의 정상 종료 또는 예외 종료 시 로그 함수를 호출하지만, 응답 사이클에 다른 영향을 미치지 않습니다.

# Binding interceptors

인터셉터를 설정하려면 `@nestjs/common` 패키지에서 가져온 `@UseInterceptors()` 데코레이터를 사용합니다. [파이프](https://docs.nestjs.com/pipes) 및 [가드](https://docs.nestjs.com/guards)와 마찬가지로 인터셉터는 컨트롤러 범위, 메서드 범위 또는 전역 범위로 설정할 수 있습니다.

```typescript
@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```

위의 구성을 사용하면 `CatsController`에 정의된 각 라우트 핸들러는 `LoggingInterceptor`를 사용합니다. 누군가 `GET /cats` 엔드포인트를 호출하면 표준 출력에 다음과 같은 출력이 표시됩니다:

```typescript
Before...
After... 1ms
```

위에서는 인스턴스가 아닌 `LoggingInterceptor` 클래스를 전달하여 인스턴스화 책임을 프레임워크에 맡기고 의존성 주입을 활성화했습니다. 파이프, 가드, 예외 필터와 마찬가지로 인라인 인스턴스를 전달할 수도 있습니다:

```typescript
@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```

위 구조는 이 컨트롤러에서 선언된 모든 핸들러에 인터셉터를 첨부합니다. 인터셉터의 범위를 단일 메서드로 제한하려면 **메서드 수준**에서 데코레이터를 적용하면 됩니다.

전역 인터셉터를 설정하려면 Nest 애플리케이션 인스턴스의 `useGlobalInterceptors()` 메서드를 사용합니다:

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

전역 인터셉터는 애플리케이션 전체에서 모든 컨트롤러와 모든 라우트 핸들러에 사용됩니다. 의존성 주입 측면에서 전역 인터셉터는 모듈 외부에서 등록될 경우(위 예제에서처럼 `useGlobalInterceptors()`를 사용하여) 의존성을 주입할 수 없습니다. 이는 바인딩이 모듈의 컨텍스트 외부에서 이루어졌기 때문입니다. 이 문제를 해결하기 위해 다음 구조를 사용하여 **모듈 내부에서 직접 전역 인터셉터**를 설정할 수 있습니다:

```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

# Response mapping

`handle()`이 `Observable`을 반환한다는 사실을 이미 알고 있습니다. 스트림은 라우트 핸들러에서 **반환된** 값을 포함하므로, RxJS의 `map()` 연산자를 사용하여 이를 쉽게 변환할 수 있습니다.

응답 매핑 기능은 라이브러리별 응답 전략(`@Res()` 객체를 직접 사용하는 것)과 함께 작동하지 않습니다.

`TransformInterceptor`를 작성하여 각 응답을 간단히 수정하는 예제를 보겠습니다. 이는 RxJS의 `map()` 연산자를 사용하여 응답 객체를 새로 생성된 객체의 `data` 속성에 할당하고, 이 새로운 객체를 클라이언트에게 반환합니다.

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle

().pipe(map(data => ({ data })));
  }
}
```

Nest 인터셉터는 동기 및 비동기 `intercept()` 메서드 모두에서 작동합니다. 필요한 경우 메서드를 `async`로 전환할 수 있습니다.

위 구성을 통해 누군가 `GET /cats` 엔드포인트를 호출하면, 응답은 다음과 같이 나타납니다(라우트 핸들러가 빈 배열 `[]`을 반환한다고 가정):

```json
{
  "data": []
}
```

인터셉터는 애플리케이션 전체에 걸쳐 발생하는 요구 사항에 대한 재사용 가능한 솔루션을 만드는 데 큰 가치를 제공합니다.
예를 들어, `null` 값을 빈 문자열 `''`로 변환해야 한다고 가정해보겠습니다. 한 줄의 코드로 이를 수행하고 인터셉터를 전역으로 바인딩하여 각 등록된 핸들러가 자동으로 이를 사용하도록 할 수 있습니다.

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
```

# Exception mapping

또 다른 흥미로운 사용 사례는 RxJS의 `catchError()` 연산자를 사용하여 던져진 예외를 재정의하는 것입니다:

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
```

# Stream overriding

때때로 핸들러 호출을 완전히 방지하고 대신 다른 값을 반환해야 할 여러 가지 이유가 있습니다. 명백한 예로는 캐시를 구현하여 응답 시간을 개선하는 것입니다. 캐시에서 응답을 반환하는 간단한 **캐시 인터셉터**를 살펴보겠습니다. 현실적인 예에서는 TTL, 캐시 무효화, 캐시 크기 등과 같은 다른 요소를 고려해야 하지만, 이는 이 논의의 범위를 벗어납니다. 여기서는 기본 개념을 설명하는 간단한 예를 제공합니다.

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

`CacheInterceptor`는 하드코딩된 `isCached` 변수와 하드코딩된 응답 `[]`을 가지고 있습니다. 중요한 점은 여기서 RxJS의 `of()` 연산자를 사용하여 새 스트림을 반환하므로 라우트 핸들러가 **전혀 호출되지 않을 것**이라는 점입니다. `CacheInterceptor`를 사용하는 엔드포인트를 호출하면 하드코딩된 빈 배열 응답이 즉시 반환됩니다. 일반적인 솔루션을 만들려면 `Reflector`를 활용하고 커스텀 데코레이터를 생성할 수 있습니다. `Reflector`는 [가드](https://docs.nestjs.com/guards) 장에서 잘 설명되어 있습니다.

# More operators

RxJS 연산자를 사용하여 스트림을 조작할 수 있는 가능성은 무한합니다. 또 다른 일반적인 사용 사례를 살펴보겠습니다. 라우트 요청에서 **타임아웃**을 처리하려고 한다고 가정해보겠습니다. 엔드포인트가 일정 시간 후에 아무 것도 반환하지 않으면 오류 응답으로 종료하려고 합니다. 다음 구조는 이를 가능하게 합니다:

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
```

5초 후에 요청 처리가 취소됩니다. `RequestTimeoutException`을 던지기 전에 사용자 정의 로직(예: 리소스 해제)을 추가할 수도 있습니다.
