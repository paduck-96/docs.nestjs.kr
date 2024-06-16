### Injection scopes

다양한 프로그래밍 언어 배경을 가진 사람들에게는 Nest에서는 거의 모든 것이 들어오는 요청 간에 공유된다는 사실이 예상치 못할 수 있습니다. 우리는 데이터베이스에 대한 연결 풀, 전역 상태를 가진 싱글톤 서비스 등을 사용합니다. Node.js는 각 요청이 별도의 스레드에 의해 처리되는 요청/응답 멀티 스레드 무상태 모델을 따르지 않기 때문에, 싱글톤 인스턴스를 사용하는 것이 애플리케이션에 대해 **안전**합니다.

그러나 경우에 따라 요청 기반 수명이 필요한 경우가 있을 수 있습니다. 예를 들어, GraphQL 애플리케이션에서의 요청 기반 캐싱, 요청 추적, 다중 테넌시 등이 있습니다. 주입 스코프는 원하는 provider 수명 주기 동작을 얻을 수 있는 메커니즘을 제공합니다.

#### Provider scope

provider는 다음과 같은 스코프를 가질 수 있습니다:

<table>
  <tr>
    <td><code>DEFAULT</code></td>
    <td>애플리케이션 전체에서 provider의 단일 인스턴스가 공유됩니다. 인스턴스 수명은 애플리케이션 수명 주기와 직접적으로 연결됩니다. 애플리케이션이 부트스트랩된 후 모든 싱글톤 provider가 인스턴스화됩니다. 싱글톤 스코프는 기본적으로 사용됩니다.</td>
  </tr>
  <tr>
    <td><code>REQUEST</code></td>
    <td>들어오는 각 <strong>요청</strong>마다 provider의 새 인스턴스가 생성됩니다. 요청이 처리된 후 인스턴스는 가비지 컬렉션됩니다.</td>
  </tr>
  <tr>
    <td><code>TRANSIENT</code></td>
    <td>트랜지언트 provider는 소비자 간에 공유되지 않습니다. 트랜지언트 provider를 주입하는 각 소비자는 새로 전용 인스턴스를 받게 됩니다.</td>
  </tr>
</table>

기본적으로 싱글톤 스코프를 사용하는 것이 **추천**됩니다. 소비자 간 및 요청 간에 provider를 공유하면 인스턴스를 캐시할 수 있으며 초기화는 애플리케이션 시작 시 한 번만 발생합니다.

REQUEST 스코프REQUEST 스코프REQUEST 스코프# 사용 방법

주입 스코프를 지정하려면 `@Injectable()` 데코레이터의 옵션 객체에 `scope` 속성을 전달합니다:

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {}
```

비슷하게, [커스텀 provider](/fundamentals/custom-providers)를 사용할 때는 provider 등록 시 `scope` 속성을 설정합니다:

```typescript
{
  provide: 'CACHE_MANAGER',
  useClass: CacheManager,
  scope: Scope.TRANSIENT,
}
```

싱글톤 스코프는 기본적으로 사용되며, 선언할 필요가 없습니다. 싱글톤 스코프로 provider를 선언하려면 `scope` 속성에 `Scope.DEFAULT` 값을 사용하십시오.

> Websocket Gateway는 REQUEST 스코프 provider를 사용해서는 안 됩니다. 각 게이트웨이는 실제 소켓을 캡슐화하며 여러 번 인스턴스화될 수 없습니다. 이 제한은 일부 다른 provider, 예를 들어 [_Passport strategies_](https://docs.nestjs.com/security/authentication#request-scoped-strategies)나 _Cron controllers_에도 적용됩니다.

#### Controller scope

컨트롤러는 모든 요청 메서드 핸들러에 적용되는 스코프를 가질 수 있습니다. provider 스코프와 마찬가지로 컨트롤러의 스코프는 수명을 선언합니다. REQUEST 스코프 컨트롤러의 경우, 각 인바운드 요청마다 새 인스턴스가 생성되며 요청이 완료되면 가비지 컬렉션됩니다.

컨트롤러 스코프는 `ControllerOptions` 객체의 `scope` 속성을 사용하여 선언합니다:

```typescript
@Controller({
  path: 'cats',
  scope: Scope.REQUEST,
})
export class CatsController {}
```

#### Scope 계층 구조

`REQUEST` 스코프는 주입 체인 상단으로 전파됩니다. REQUEST 스코프 provider에 의존하는 컨트롤러는 자체적으로 REQUEST 스코프가 됩니다.

다음과 같은 종속성 그래프를 상상해보십시오: `CatsController <- CatsService <- CatsRepository`. `CatsService`가 REQUEST 스코프이고 나머지는 기본 싱글톤인 경우, `CatsController`는 주입된 서비스에 의존하기 때문에 REQUEST 스코프가 됩니다. 종속되지 않은 `CatsRepository`는 싱글톤 스코프로 유지됩니다.

트랜지언트 스코프 종속성은 해당 패턴을 따르지 않습니다. 싱글톤 스코프의 `DogsService`가 트랜지언트 `LoggerService` provider를 주입하면, 새로운 인스턴스를 받게 됩니다. 그러나 `DogsService`는 싱글톤 스코프로 유지되므로, 어디서든 주입해도 새로운 `DogsService` 인스턴스로 해결되지 않습니다. 이 동작이 원하는 경우, `DogsService`도 명시적으로 `TRANSIENT`로 표시해야 합니다.

#### Request provider

HTTP 서버 기반 애플리케이션(e.g., `@nestjs/platform-express` 또는 `@nestjs/platform-fastify`를 사용하는 경우), REQUEST 스코프 provider를 사용할 때 원본 요청 객체에 대한 참조를 액세스하고자 할 수 있습니다. 이를 위해 `REQUEST` 객체를 주입할 수 있습니다.

`REQUEST` provider는 REQUEST 스코프이므로, 이 경우 명시적으로 `REQUEST` 스코프를 사용할 필요가 없습니다.

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(REQUEST) private request: Request) {}
}
```

기본 플랫폼/프로토콜 차이로 인해, 마이크로서비스나 GraphQL 애플리케이션에서는 인바운드 요청을 약간 다르게 액세스합니다. [GraphQL](https://docs.nestjs.com/graphql/quick-start) 애플리케이션에서는 `REQUEST` 대신 `CONTEXT`를 주입합니다:

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT } from '@nestjs/graphql';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private context) {}
}
```

그런 다음 `context` 값을(GraphQLModule에서) `request`를 속성으로 포함하도록 구성합니다.

#### Inquirer provider

로깅 또는 메트릭 provider에서 provider가 생성된 클래스를 알고 싶다면, `INQUIRER` 토큰을 주입할 수 있습니다.

```typescript
import { Inject, Injectable, Scope } from '@nestjs/common';
import { INQUIRER } from '@nestjs/core';

@Injectable({ scope: Scope.TRANSIENT })
export class HelloService {
  constructor(@Inject(INQUIRER) private parentClass: object) {}

  sayHello(message: string) {
    console.log(`${this.parentClass?.constructor?.name}: ${message}`);
  }
}
```

다음과 같이 사용할 수 있습니다:

```typescript
import { Injectable } from '@nestjs/common';
import { HelloService } from './hello.service';

@Injectable()
export class AppService {
  constructor(private helloService: HelloService) {}

  getRoot(): string {
    this.helloService.sayHello('My name is getRoot');

    return 'Hello world!';
  }
}
```

위 예제에서 `AppService.getRoot`가 호출되면, `"AppService: My name is getRoot"`가 콘솔에 기록됩니다.

#### 성능

REQUEST 스코프 provider를 사용하면 애플리케이션 성능에 영향을 미칩니다. Nest는 가능한 많은 메타데이터를 캐시하려고 하지만, 여전히 각 요청에 대해 클래스의 인스턴스를 생성해야 합니다. 따라서 평균 응답 시간과 전체 벤치마킹 결과가 느려질 것입니다. provider가 REQUEST 스코프여야 하는 경우가 아니면, 기본 싱글톤 스코프를 사용하는 것이 강력히 권장됩니다.

#### Durable providers

REQUEST 스코프 provider는 성능 저하를 유발할 수 있습니다. REQUEST 스코프 provider가 하나라도 있으면, 컨트롤러도 REQUEST 스코프가 되어야 하며, 각 개별 요청마다 새로 생성(인스턴스화)되고 이후 가비지 컬렉션됩니다. 이는 병렬로 30,000개의 요청이 있을 때, 30,000개의 임시 컨트롤러 인스턴스(및 REQUEST 스코프 provider)가 생성됨을 의미합니다.

다중 테넌트 애플리케이션에서는 이러한 문제가 더욱 두드러집니다. 중앙 REQUEST 스코프 "데이터 소스" provider가 헤더/토큰에서 값을 가져와 테넌트별 데이터베이스 연결/스키마를 가져오는 경우, 대부분의 애플리케이션 구성 요소가 "데이터 소스" provider에 의존하므로 성능에 영향을 미칩니다.

하지만 더 나은 해결책이 있을까요? 각 테넌트에 대해 개별 [DI 서브 트리](https://docs.nestjs.com/fundamentals/module-ref#resolving-scoped-providers)를 생성할 수 있다면 어떨까요? 요청마다 서브 트리를 재생성할 필요 없이 테넌트별로 개별 트리를 가질 수 있습니다.

이제, **durable providers**가 필요합니다.

durable provider를 설정하려면 먼저 Nest가 "공통 요청 속성"을 알려주는 **전략**을 등록해야 합니다.

```typescript
import {
  HostComponentInfo,
  ContextId,
  ContextIdFactory,
  ContextIdStrategy,
} from '@nestjs/core';
import { Request } from 'express';

const tenants = new Map<string, ContextId>();

export class AggregateByTenantContextIdStrategy implements ContextIdStrategy {
  attach(contextId: ContextId, request: Request) {
    const tenantId = request.headers['x-tenant-id'] as string;
    let tenantSubTreeId: ContextId;

    if (tenants.has(tenantId)) {
      tenantSubTreeId = tenants.get(tenantId);
    } else {
      tenantSubTreeId = ContextIdFactory.create();
      tenants.set(tenantId, tenantSubTreeId);
    }

    return (info: HostComponentInfo) =>
      info.isTreeDurable ? tenantSubTreeId : contextId;
  }
}
```

이제 이 전략을 코드의 어느 곳에든지 등록하여(어쨌든 전역적으로 적용되므로), 예를 들어 `main.ts` 파일에 배치할 수 있습니다:

```typescript
ContextIdFactory.apply(new AggregateByTenantContextIdStrategy());
```

마지막으로, 일반 provider를 durable provider로 바꾸려면 `durable` 플래그를 `true`로 설정하고 `Scope.REQUEST`로 변경합니다.

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST, durable: true })
export class CatsService {}
```

비슷하게, [커스텀 provider](/fundamentals/custom-providers)를 사용할 때는 provider 등록 시 `durable` 속성을 설정합니다:

```typescript
{
  provide: 'foobar',
  useFactory: () => { ... },
  scope: Scope.REQUEST,
  durable: true,
}
```
