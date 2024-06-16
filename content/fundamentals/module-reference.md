### Module reference

Nest는 `ModuleRef` 클래스를 제공하여 내부 provider 목록을 탐색하고 주입 토큰을 조회 키로 사용하여 모든 provider에 대한 참조를 얻을 수 있게 합니다. `ModuleRef` 클래스는 정적 및 스코프가 지정된 provider를 동적으로 인스턴스화할 수 있는 방법도 제공합니다. `ModuleRef`는 일반적인 방법으로 클래스에 주입할 수 있습니다:

```typescript
import { Injectable, ModuleRef } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor(private moduleRef: ModuleRef) {}
}
```

`ModuleRef` 클래스는 `@nestjs/core` 패키지에서 가져옵니다.

#### 인스턴스 검색

`ModuleRef` 인스턴스는 `get()` 메서드를 가지고 있습니다. 이 메서드는 현재 모듈에 존재하는 provider, controller 또는 injectable (예: guard, interceptor 등)을 주입 토큰/클래스 이름을 사용하여 검색합니다.

```typescript
import { Injectable, ModuleRef, OnModuleInit } from '@nestjs/common';

@Injectable()
export class CatsService implements OnModuleInit {
  private service: Service;
  constructor(private moduleRef: ModuleRef) {}

  onModuleInit() {
    this.service = this.moduleRef.get(Service);
  }
}
```

스코프가 지정된 provider (일시적 또는 요청 스코프)는 `get()` 메서드로 검색할 수 없습니다. 대신, 아래에서 설명하는 기술을 사용하십시오. 스코프 제어 방법에 대해 알아보려면 [여기](https://docs.nestjs.com/fundamentals/injection-scopes)를 참조하십시오.

글로벌 컨텍스트에서 provider를 검색하려면 (예: 다른 모듈에 주입된 provider인 경우), `get()` 메서드의 두 번째 인수로 `{ strict: false }` 옵션을 전달하십시오.

```typescript
this.moduleRef.get(Service, { strict: false });
```

#### 스코프가 지정된 provider 해결

스코프가 지정된 provider (일시적 또는 요청 스코프)를 동적으로 해결하려면 provider의 주입 토큰을 인수로 전달하여 `resolve()` 메서드를 사용하십시오.

```typescript
import { Injectable, ModuleRef, OnModuleInit } from '@nestjs/common';

@Injectable()
export class CatsService implements OnModuleInit {
  private transientService: TransientService;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.transientService = await this.moduleRef.resolve(TransientService);
  }
}
```

`resolve()` 메서드는 자체 **DI 컨테이너 서브 트리**에서 provider의 고유한 인스턴스를 반환합니다. 각 서브 트리는 고유한 **컨텍스트 식별자**를 가집니다. 따라서 이 메서드를 여러 번 호출하고 인스턴스 참조를 비교하면 서로 다름을 알 수 있습니다.

```typescript
import { Injectable, ModuleRef, OnModuleInit } from '@nestjs/common';

@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService),
      this.moduleRef.resolve(TransientService),
    ]);
    console.log(transientServices[0] === transientServices[1]); // false
  }
}
```

여러 `resolve()` 호출에서 단일 인스턴스를 생성하고 동일한 생성된 DI 컨테이너 서브 트리를 공유하도록 하려면 컨텍스트 식별자를 `resolve()` 메서드에 전달할 수 있습니다. `ContextIdFactory` 클래스를 사용하여 컨텍스트 식별자를 생성합니다. 이 클래스는 적절한 고유 식별자를 반환하는 `create()` 메서드를 제공합니다.

```typescript
import { Injectable, ModuleRef, OnModuleInit, ContextIdFactory } from '@nestjs/common';

@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const contextId = ContextIdFactory.create();
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService, contextId),
      this.moduleRef.resolve(TransientService, contextId),
    ]);
    console.log(transientServices[0] === transientServices[1]); // true
  }
}
```

`ContextIdFactory` 클래스는 `@nestjs/core` 패키지에서 가져옵니다.

#### `REQUEST` provider 등록

수동으로 생성된 컨텍스트 식별자 (`ContextIdFactory.create()` 사용)는 Nest 종속성 주입 시스템에 의해 인스턴스화되고 관리되지 않으므로 `REQUEST` provider가 `undefined`로 설정된 DI 서브 트리를 나타냅니다.

수동으로 생성된 DI 서브 트리에 사용자 정의 `REQUEST` 객체를 등록하려면 다음과 같이 `ModuleRef.registerRequestByContextId()` 메서드를 사용하십시오:

```typescript
const contextId = ContextIdFactory.create();
this.moduleRef.registerRequestByContextId(/* YOUR_REQUEST_OBJECT */, contextId);
```

#### 현재 서브 트리 가져오기

때로는 **요청 컨텍스트** 내에서 요청 스코프 provider 인스턴스를 해결해야 할 수 있습니다. 예를 들어, `CatsService`가 요청 스코프이고 요청 스코프 provider로 표시된 `CatsRepository` 인스턴스를 해결하려는 경우, 동일한 DI 컨테이너 서브 트리를 공유하려면 새 컨텍스트 식별자를 생성하는 대신 현재 컨텍스트 식별자를 얻어야 합니다 (예: 위에 표시된 `ContextIdFactory.create()` 함수 사용). 현재 컨텍스트 식별자를 얻으려면 `@Inject()` 데코레이터를 사용하여 요청 객체를 주입하십시오.

```typescript
import { Injectable, Inject, REQUEST } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor(
    @Inject(REQUEST) private request: Record<string, unknown>,
  ) {}
}
```

요청 provider에 대해 자세히 알아보려면 [여기](https://docs.nestjs.com/fundamentals/injection-scopes#request-provider)를 참조하십시오.

이제 `ContextIdFactory` 클래스의 `getByRequest()` 메서드를 사용하여 요청 객체를 기반으로 컨텍스트 ID를 생성하고 이를 `resolve()` 호출에 전달하십시오:

```typescript
const contextId = ContextIdFactory.getByRequest(this.request);
const catsRepository = await this.moduleRef.resolve(CatsRepository, contextId);
```

#### 사용자 정의 클래스 동적 인스턴스화

**이전에 등록되지 않은** 클래스를 동적으로 인스턴스화하려면 모듈 참조의 `create()` 메서드를 사용하십시오.

```typescript
import { Injectable, ModuleRef, OnModuleInit } from '@nestjs/common';

@Injectable()
export class CatsService implements OnModuleInit {
  private catsFactory: CatsFactory;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.catsFactory = await this.moduleRef.create(CatsFactory);
  }
}
```

이 기술을 사용하면 프레임워크 컨테이너 외부에서 다양한 클래스를 조건부로 인스턴스화할 수 있습니다.
