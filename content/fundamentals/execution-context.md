### Execution Context

Nest는 여러 애플리케이션 컨텍스트(예: Nest HTTP 서버 기반, 마이크로서비스 및 WebSockets 애플리케이션 컨텍스트)에서 작동하는 애플리케이션을 쉽게 작성할 수 있도록 도와주는 여러 유틸리티 클래스를 제공합니다. 이러한 유틸리티는 현재 실행 컨텍스트에 대한 정보를 제공하여 다양한 컨트롤러, 메서드 및 실행 컨텍스트에서 작동할 수 있는 일반적인 [guards](https://docs.nestjs.com/guards), [filters](https://docs.nestjs.com/exception-filters) 및 [interceptors](https://docs.nestjs.com/interceptors)를 구축할 수 있게 합니다.

이 챕터에서는 `ArgumentsHost`와 `ExecutionContext`라는 두 가지 클래스를 다룹니다.

#### ArgumentsHost 클래스

`ArgumentsHost` 클래스는 핸들러에 전달되는 인수를 검색하는 메서드를 제공합니다. 이를 통해 적절한 컨텍스트(예: HTTP, RPC(마이크로서비스) 또는 WebSockets)를 선택하여 인수를 검색할 수 있습니다. 프레임워크는 `ArgumentsHost`의 인스턴스를 제공합니다. 이는 보통 `host` 매개변수로 참조되며, 액세스가 필요한 곳에서 사용됩니다. 예를 들어, [예외 필터](https://docs.nestjs.com/exception-filters#arguments-host)의 `catch()` 메서드는 `ArgumentsHost` 인스턴스와 함께 호출됩니다.

`ArgumentsHost`는 단순히 핸들러의 인수에 대한 추상화를 제공합니다. 예를 들어, HTTP 서버 애플리케이션(`@nestjs/platform-express`가 사용될 때)에서는 `host` 객체가 Express의 `[request, response, next]` 배열을 캡슐화합니다. 여기서 `request`는 요청 객체이고, `response`는 응답 객체이며, `next`는 애플리케이션의 요청-응답 사이클을 제어하는 함수입니다. 반면, [GraphQL](https://docs.nestjs.com/graphql/quick-start) 애플리케이션에서는 `host` 객체가 `[root, args, context, info]` 배열을 포함합니다.

#### 현재 애플리케이션 컨텍스트

여러 애플리케이션 컨텍스트에서 실행될 일반적인 [guards](https://docs.nestjs.com/guards), [filters](https://docs.nestjs.com/exception-filters), [interceptors](https://docs.nestjs.com/interceptors)를 작성할 때, 메서드가 현재 실행 중인 애플리케이션의 유형을 결정할 방법이 필요합니다. `ArgumentsHost`의 `getType()` 메서드를 사용하여 이를 수행할 수 있습니다:

```typescript
if (host.getType() === 'http') {
  // 일반 HTTP 요청(REST) 컨텍스트에서만 중요한 작업 수행
} else if (host.getType() === 'rpc') {
  // 마이크로서비스 요청 컨텍스트에서만 중요한 작업 수행
} else if (host.getType<GqlContextType>() === 'graphql') {
  // GraphQL 요청 컨텍스트에서만 중요한 작업 수행
}
```

> **Hint** `GqlContextType`은 `@nestjs/graphql` 패키지에서 가져옵니다.

애플리케이션 유형을 알 수 있게 되면, 아래와 같이 더 일반적인 컴포넌트를 작성할 수 있습니다.

#### 호스트 핸들러 인수

핸들러에 전달되는 인수 배열을 검색하려면 `host` 객체의 `getArgs()` 메서드를 사용할 수 있습니다.

```typescript
const [req, res, next] = host.getArgs();
```

`getArgByIndex()` 메서드를 사용하여 특정 인수를 인덱스로 선택할 수 있습니다:

```typescript
const request = host.getArgByIndex(0);
const response = host.getArgByIndex(1);
```

위 예제에서는 인덱스로 요청 및 응답 객체를 검색했지만, 이는 애플리케이션을 특정 실행 컨텍스트에 결합시키므로 권장되지 않습니다. 대신, `host` 객체의 유틸리티 메서드 중 하나를 사용하여 애플리케이션 컨텍스트에 적합한 컨텍스트로 전환하는 것이 더 견고하고 재사용 가능한 코드를 작성하는 방법입니다. 컨텍스트 전환 유틸리티 메서드는 다음과 같습니다.

```typescript
/**
 * 컨텍스트를 RPC로 전환.
 */
switchToRpc(): RpcArgumentsHost;
/**
 * 컨텍스트를 HTTP로 전환.
 */
switchToHttp(): HttpArgumentsHost;
/**
 * 컨텍스트를 WebSockets로 전환.
 */
switchToWs(): WsArgumentsHost;
```

이전 예제를 `switchToHttp()` 메서드를 사용하여 다시 작성해 보겠습니다. `host.switchToHttp()` 도우미 호출은 HTTP 애플리케이션 컨텍스트에 적합한 `HttpArgumentsHost` 객체를 반환합니다. `HttpArgumentsHost` 객체에는 원하는 객체를 추출하는 데 사용할 수 있는 두 가지 유용한 메서드가 있습니다. 이 경우 Express 타입 어설션을 사용하여 네이티브 Express 타입의 객체를 반환합니다:

```typescript
const ctx = host.switchToHttp();
const request = ctx.getRequest<Request>();
const response = ctx.getResponse<Response>();
```

유사하게 `WsArgumentsHost` 및 `RpcArgumentsHost`에는 마이크로서비스 및 WebSockets 컨텍스트에서 적절한 객체를 반환하는 메서드가 있습니다. 여기 `WsArgumentsHost`의 메서드가 있습니다:

```typescript
export interface WsArgumentsHost {
  /**
   * 데이터 객체를 반환합니다.
   */
  getData<T>(): T;
  /**
   * 클라이언트 객체를 반환합니다.
   */
  getClient<T>(): T;
}
```

다음은 `RpcArgumentsHost`의 메서드입니다:

```typescript
export interface RpcArgumentsHost {
  /**
   * 데이터 객체를 반환합니다.
   */
  getData<T>(): T;

  /**
   * 컨텍스트 객체를 반환합니다.
   */
  getContext<T>(): T;
}
```

#### ExecutionContext 클래스

`ExecutionContext`는 `ArgumentsHost`를 확장하여 현재 실행 프로세스에 대한 추가 세부 정보를 제공합니다. `ArgumentsHost`와 마찬가지로, Nest는 이를 필요로 하는 곳에서 `ExecutionContext` 인스턴스를 제공합니다. 예를 들어, [guard](https://docs.nestjs.com/guards#execution-context)의 `canActivate()` 메서드와 [interceptor](https://docs.nestjs.com/interceptors#execution-context)의 `intercept()` 메서드에서 이를 사용할 수 있습니다. `ExecutionContext`는 다음과 같은 메서드를 제공합니다:

```typescript
export interface ExecutionContext extends ArgumentsHost {
  /**
   * 현재 핸들러가 속한 컨트롤러 클래스의 유형을 반환합니다.
   */
  getClass<T>(): Type<T>;
  /**
   * 요청 파이프라인에서 다음에 호출될 핸들러(메서드)에 대한 참조를 반환합니다.
   */
  getHandler(): Function;
}
```

`getHandler()` 메서드는 호출될 핸들러에 대한 참조를 반환합니다. `getClass()` 메서드는 특정 핸들러가 속한 `Controller` 클래스의 유형을 반환합니다. 예를 들어, HTTP 컨텍스트에서 현재 처리 중인 요청이 `CatsController`의 `create()` 메서드에 바인딩된 `POST` 요청인 경우, `getHandler()`는 `create()` 메서드에 대한 참조를 반환하고, `getClass()`는 `CatsController` **클래스**(인스턴스가 아님)를 반환합니다.

```typescript
const methodKey = ctx.getHandler().name; // "create"
const className = ctx.getClass().name; // "CatsController"
```

현재 클래스와 핸들러 메서드에 대한 참조에 액세스할 수 있는 기능은 큰 유연성을 제공합니다. 가장 중요한 것은 `Reflector.createDecorator`를 통해 생성된 데코레이터 또는 내장 `@SetMetadata()` 데코레이터를 통해 설정된 메타데이터에 가드 또는 인터셉터 내에서 액세스할 수 있는 기회를 제공합니다. 이 사용 사례를 아래에서 다룹니다.

#### 리플렉션 및 메타데이터

Nest는 `Reflector.createDecorator` 메서드를 통해 생성된 데코레이터와 내장 `@SetMetadata()` 데코레이터를 통해 라우트 핸들러에 **사용자 정의 메타데이터**를 첨부할 수 있는 기능을 제공합니다. 이 섹션에서는 두 접근 방식을 비교하고 가드 또는 인터셉터 내에서 메타데이터에 액세스하는 방법을 살펴봅니다.

`Reflector.createDecorator`를 사용하여 강력한 타입의 데코레이터를 생성하려면 타입 인수를 지정해야 합니다. 예를 들어, 문자열 배열을 인수로 받는 `Roles` 데코레이터를 만들어 보겠습니다.

```typescript
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

여기서 `Roles` 데코레이터는 `string[]` 타입의 인수를 받는 함수입니다.

이제 이 데코레이터를 사용하여 핸들러를 주석 처리할 수 있습니다:

```typescript


@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

여기서 우리는 `create()` 메서드에 `Roles` 데코레이터 메타데이터를 첨부하여 관리자 역할을 가진 사용자만 이 경로에 액세스할 수 있도록 했습니다.

라우트의 역할(사용자 정의 메타데이터)에 액세스하려면 다시 `Reflector` 도우미 클래스를 사용합니다. `Reflector`는 일반적인 방식으로 클래스에 주입할 수 있습니다:

```typescript
import { Injectable, Reflector } from '@nestjs/common';

@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
```

> **Hint** `Reflector` 클래스는 `@nestjs/core` 패키지에서 가져옵니다.

이제 핸들러 메타데이터를 읽으려면 `get()` 메서드를 사용하십시오:

```typescript
const roles = this.reflector.get(Roles, context.getHandler());
```

`Reflector.get` 메서드는 데코레이터 참조와 메타데이터를 검색할 **컨텍스트**(데코레이터 대상)를 두 번째 인수로 전달하여 메타데이터에 쉽게 액세스할 수 있게 합니다. 이 예에서는 특정 **데코레이터**로 `Roles`를 지정합니다 (위의 `roles.decorator.ts` 파일을 참조). 컨텍스트는 `context.getHandler()` 호출에 의해 제공되며, 현재 처리 중인 라우트 핸들러의 메타데이터를 추출하는 결과를 가져옵니다. `getHandler()`는 라우트 핸들러 함수에 대한 **참조**를 제공합니다.

대안으로, 컨트롤러 수준에서 메타데이터를 적용하여 컨트롤러 클래스의 모든 경로에 적용할 수 있습니다.

```typescript
@Roles(['admin'])
@Controller('cats')
export class CatsController {}
```

이 경우 컨트롤러 메타데이터를 추출하려면 메타데이터 추출을 위한 컨텍스트로 컨트롤러 클래스를 제공하는 대신 `context.getClass()`를 두 번째 인수로 전달합니다:

```typescript
const roles = this.reflector.get(Roles, context.getClass());
```

여러 레벨에서 메타데이터를 제공할 수 있으므로 여러 컨텍스트에서 메타데이터를 추출하고 병합해야 할 수 있습니다. `Reflector` 클래스는 이를 돕기 위해 두 가지 유틸리티 메서드를 제공합니다. 이러한 메서드는 **컨트롤러와 메서드** 메타데이터를 한 번에 추출하고 서로 다른 방식으로 결합합니다.

다음과 같은 시나리오를 고려해 보겠습니다. 여기서 `Roles` 메타데이터를 두 레벨에서 제공했습니다.

```typescript
@Roles(['user'])
@Controller('cats')
export class CatsController {
  @Post()
  @Roles(['admin'])
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }
}
```

기본 역할로 `'user'`를 지정하고 특정 메서드에 대해 선택적으로 덮어쓰려는 경우, 아마도 `getAllAndOverride()` 메서드를 사용할 것입니다.

```typescript
const roles = this.reflector.getAllAndOverride(Roles, [context.getHandler(), context.getClass()]);
```

이 코드가 있는 가드가 `create()` 메서드 컨텍스트에서 실행되면, 위의 메타데이터와 함께 `roles`에는 `['admin']`이 포함됩니다.

메타데이터를 모두 가져와 병합하려면(이 메서드는 배열과 객체를 병합합니다) `getAllAndMerge()` 메서드를 사용합니다:

```typescript
const roles = this.reflector.getAllAndMerge(Roles, [context.getHandler(), context.getClass()]);
```

이 경우 `roles`에는 `['user', 'admin']`이 포함됩니다.

이 두 병합 메서드 모두 첫 번째 인수로 메타데이터 키를, 두 번째 인수로 메타데이터 대상 컨텍스트의 배열(즉, `getHandler()` 및/또는 `getClass())` 메서드 호출)을 전달합니다.

#### 저수준 접근 방식

앞서 언급했듯이, `Reflector.createDecorator`를 사용하는 대신 내장 `@SetMetadata()` 데코레이터를 사용하여 핸들러에 메타데이터를 첨부할 수도 있습니다.

```typescript
@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

> **Hint** `@SetMetadata()` 데코레이터는 `@nestjs/common` 패키지에서 가져옵니다.

위 구조를 사용하여 `create()` 메서드에 `roles` 메타데이터(`roles`는 메타데이터 키이고 `['admin']`은 연결된 값)를 첨부했습니다. 이는 작동하지만 경로에서 `@SetMetadata()`를 직접 사용하는 것은 좋은 방법이 아닙니다. 대신, 아래와 같이 사용자 정의 데코레이터를 만들 수 있습니다:

```typescript
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

이 접근 방식은 훨씬 더 깔끔하고 가독성이 높으며, `Reflector.createDecorator` 접근 방식과 어느 정도 유사합니다. 차이점은 `@SetMetadata`를 사용하면 메타데이터 키와 값에 대해 더 많은 제어 권한을 가지며, 더 많은 인수를 받는 데코레이터를 생성할 수 있다는 점입니다.

이제 사용자 정의 `@Roles()` 데코레이터가 있으므로 이를 사용하여 `create()` 메서드를 주석 처리할 수 있습니다.

```typescript
@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

라우트의 역할(사용자 정의 메타데이터)에 액세스하려면 다시 `Reflector` 도우미 클래스를 사용합니다:

```typescript
import { Injectable, Reflector } from '@nestjs/common';

@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
```

> **Hint** `Reflector` 클래스는 `@nestjs/core` 패키지에서 가져옵니다.

이제 핸들러 메타데이터를 읽으려면 `get()` 메서드를 사용합니다.

```typescript
const roles = this.reflector.get<string[]>('roles', context.getHandler());
```

여기서는 데코레이터 참조 대신 메타데이터 **키**를 첫 번째 인수로 전달합니다(우리의 경우 `'roles'`). 나머지는 `Reflector.createDecorator` 예제와 동일하게 유지됩니다.
