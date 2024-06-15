### Testing

자동화된 테스트는 어떤 진지한 소프트웨어 개발 노력에도 필수적인 부분으로 간주됩니다. 자동화는 개발 중에 개별 테스트나 테스트 스위트를 빠르고 쉽게 반복할 수 있게 해줍니다. 이를 통해 릴리스가 품질 및 성능 목표를 충족하도록 보장할 수 있습니다. 자동화는 커버리지를 증가시키고 개발자에게 더 빠른 피드백 루프를 제공합니다. 자동화는 개별 개발자의 생산성을 높이고 소스 코드 관리 체크인, 기능 통합, 버전 릴리스와 같은 중요한 개발 라이프사이클 시점에서 테스트가 실행되도록 보장합니다.

이러한 테스트는 종종 단위 테스트, 엔드 투 엔드(e2e) 테스트, 통합 테스트 등 다양한 유형을 포함합니다. 이점이 의심할 여지없이 많지만, 설정하는 것은 번거로울 수 있습니다. Nest는 효과적인 테스트를 포함한 개발 모범 사례를 장려하기 위해 다음과 같은 기능을 제공하여 개발자와 팀이 테스트를 구축하고 자동화할 수 있도록 합니다. Nest는 다음과 같은 기능을 제공합니다:

- 컴포넌트에 대한 기본 단위 테스트와 애플리케이션에 대한 e2e 테스트를 자동으로 생성합니다.
- 격리된 모듈/애플리케이션 로더를 빌드하는 테스트 러너와 같은 기본 도구를 제공합니다.
- 기본적으로 [Jest](https://github.com/facebook/jest) 및 [Supertest](https://github.com/visionmedia/supertest)와의 통합을 제공하면서도 테스트 도구에 구애받지 않습니다.
- 쉽게 컴포넌트를 모킹할 수 있도록 테스트 환경에서 Nest 의존성 주입 시스템을 사용할 수 있습니다.

언급했듯이, Nest는 특정 도구를 강요하지 않으므로 원하는 **테스트 프레임워크**를 사용할 수 있습니다. 필요한 요소(예: 테스트 러너)를 교체하면 Nest의 준비된 테스트 기능의 이점을 계속 누릴 수 있습니다.

#### 설치

시작하려면 먼저 필요한 패키지를 설치합니다:

```bash
$ npm i --save-dev @nestjs/testing
```

#### 단위 테스트

다음 예제에서는 `CatsController`와 `CatsService` 두 클래스를 테스트합니다. 언급했듯이 [Jest](https://github.com/facebook/jest)가 기본 테스트 프레임워크로 제공됩니다. Jest는 테스트 러너 역할을 하며 모킹, 스파잉 등과 같은 작업에 도움을 주는 어설션 함수와 테스트 더블 유틸리티를 제공합니다. 다음 기본 테스트에서는 이 클래스들을 수동으로 인스턴스화하고, 컨트롤러와 서비스가 API 계약을 충족하는지 확인합니다.

```typescript
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

> info **힌트** 테스트 파일을 테스트하는 클래스와 가까운 위치에 두세요. 테스트 파일은 `.spec` 또는 `.test` 접미사를 가져야 합니다.

위의 샘플은 간단하기 때문에 Nest 특정 사항을 테스트하지 않습니다. 실제로 우리는 의존성 주입을 사용하지 않습니다(우리는 `catsController`에 `CatsService`의 인스턴스를 전달합니다). 이 형태의 테스트는 프레임워크와 독립적이기 때문에 종종 **격리 테스트**라고 불립니다. Nest 기능을 더 많이 사용하는 애플리케이션을 테스트하는 데 도움이 되는 더 고급 기능을 소개해 보겠습니다.

#### 테스트 유틸리티

`@nestjs/testing` 패키지는 더 강력한 테스트 프로세스를 가능하게 하는 유틸리티 세트를 제공합니다. 내장된 `Test` 클래스를 사용하여 이전 예제를 다시 작성해 보겠습니다:

```typescript
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get<CatsService>(CatsService);
    catsController = moduleRef.get<CatsController>(CatsController);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

`Test` 클래스는 애플리케이션 실행 컨텍스트를 제공하는데 유용하며, 이는 전체 Nest 런타임을 모킹하지만 클래스 인스턴스를 관리하기 쉽게 하는 훅을 제공합니다. `Test` 클래스에는 `@Module()` 데코레이터에 전달하는 모듈 메타데이터 객체를 인수로 받는 `createTestingModule()` 메서드가 있습니다. 이 메서드는 `TestingModule` 인스턴스를 반환하며, 이는 몇 가지 메서드를 제공합니다. 단위 테스트의 경우 중요한 메서드는 `compile()` 메서드입니다. 이 메서드는 모듈과 그 의존성을 부트스트랩하고(일반적인 `main.ts` 파일에서 `NestFactory.create()`를 사용하여 애플리케이션을 부트스트랩하는 것과 유사), 테스트할 준비가 된 모듈을 반환합니다.

> info **힌트** `compile()` 메서드는 **비동기**이므로 기다려야 합니다. 모듈이 컴파일되면 `get()` 메서드를 사용하여 선언된 **정적** 인스턴스를 검색할 수 있습니다(컨트롤러와 프로바이더 포함).

`TestingModule`은 [모듈 참조](/fundamentals/module-ref) 클래스에서 상속되므로, 범위가 지정된 프로바이더(일시적 또는 요청 범위)를 동적으로 해결할 수 있는 기능을 갖추고 있습니다. 이는 `resolve()` 메서드를 사용하여 수행할 수 있습니다(`get()` 메서드는 정적 인스턴스만 검색할 수 있습니다).

```typescript
const moduleRef = await Test.createTestingModule({
  controllers: [CatsController],
  providers: [CatsService],
}).compile();

catsService = await moduleRef.resolve(CatsService);
```

> warning **경고** `resolve()` 메서드는 자체 **DI 컨테이너 하위 트리**에서 프로바이더의 고유 인스턴스를 반환합니다. 각 하위 트리는 고유한 컨텍스트 식별자를 갖습니다. 따라서 이 메서드를 여러 번 호출하고 인스턴스 참조를 비교하면 동일하지 않음을 알 수 있습니다.

> info **힌트** 모듈 참조 기능에 대해 자세히 알아보세요 [여기](https://docs.nestjs.com/fundamentals/module-ref).

프로덕션 버전의 프로바이더를 사용하는 대신, 테스트 목적으로 [커스텀 프로바이더](/fundamentals/custom-providers)로 대체할 수 있습니다. 예를 들어, 실제 데이터베이스에 연결하는 대신 데이터베이스 서비스를 모킹할 수 있습니다. 다음 섹션에서 오버라이드를 다루겠지만, 단위 테스트에서도 사용할 수 있습니다.

<app-banner-courses></app-banner-courses>

#### 자동 모킹

Nest는 또한 모든 누락된 종속성에 대해 모킹 팩토리를 정의할 수 있게 합니다. 이는 클래스에 많은 종속성이 있을 때 유용하며, 모든 종속성을 모킹하는 데 많은 시간과 설정이 필요한 경우에 특히 유용합니다. 이 기능을 사용하려면 `createTestingModule()`을 `useMocker()` 메서드와 연결하여 종속성 모킹에 대한 팩토리를 전달해야 합니다. 이 팩토리는 선택적인 토큰을 인수로 받을 수 있으며, 이는 인스턴스 토큰, Nest 프로바이더에 유효한 모든 토큰이 될 수 있으며 모킹 구현을 반환합니다. 아래는 [`jest-mock`](https://www.npmjs.com/package/jest-mock)을 사용하여 일반적인 모커를 생성하고 `CatsService`에 대해 `jest.fn()`을 사용하여 특정 모킹을 생성하는 예입니다.

```typescript
import { ModuleMocker, MockFunctionMetadata } from 'jest-mock';

const moduleMocker = new ModuleMocker(global);

describe('CatsController', () => {
  let controller: CatsController;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      controllers: [CatsController],
    })
      .useMocker((token) => {
        const results = ['test1', 'test2'];
        if (token === CatsService) {
          return { findAll: jest.fn().mockResolvedValue(results) };
        }
        if (typeof token === 'function') {
          const mockMetadata = moduleMocker.getMetadata(token) as MockFunctionMetadata<any, any>;
          const Mock = moduleMocker.generateFromMetadata(mockMetadata);
          return new Mock();
        }
      })
      .compile();

    controller = moduleRef.get(CatsController);
  });
});
```

이 모킹은 일반적인 커스텀 프로바이더처럼 테스트 컨테이너에서 검색할 수도 있습니다. `moduleRef.get(CatsService)`와 같이 사용할 수 있습니다.

> info **힌트** `createMock`와 같은 일반적인 모킹 팩토리도 직접 전달할 수 있습니다. [`@golevelup/ts-jest`](https://github.com/golevelup/nestjs/tree/master/packages/testing).

> info **힌트** `REQUEST` 및 `INQUIRER` 프로바이더는 이미 컨텍스트에서 미리 정의되어 있기 때문에 자동으로 모킹할 수 없습니다. 그러나 커스텀 프로바이더 구문을 사용하거나 `.overrideProvider` 메서드를 사용하여 _덮어쓸_ 수 있습니다.

#### 엔드 투 엔드 테스트

단위 테스트와 달리, 개별 모듈과 클래스에 초점을 맞춘 엔드 투 엔드(e2e) 테스트는 클래스와 모듈의 상호작용을 더 종합적으로 다룹니다. 이는 실제 사용자와 프로덕션 시스템 간의 상호작용과 유사합니다. 애플리케이션이 커지면 각 API 엔드포인트의 엔드 투 엔드 동작을 수동으로 테스트하기 어려워집니다. 자동화된 엔드 투 엔드 테스트는 시스템의 전체 동작이 올바르고 프로젝트 요구 사항을 충족하는지 확인하는 데 도움이 됩니다. e2e 테스트를 수행하려면 **단위 테스트**에서 다룬 것과 유사한 구성이 필요합니다. 또한, Nest는 [Supertest](https://github.com/visionmedia/supertest) 라이브러리를 사용하여 HTTP 요청을 시뮬레이션하기 쉽게 만듭니다.

```typescript
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

> info **힌트** [Fastify](https://docs.nestjs.com/techniques/performance)를 HTTP 어댑터로 사용하는 경우 약간 다른 구성이 필요하며, 내장된 테스트 기능이 있습니다:
>
> ```ts
> let app: NestFastifyApplication;
>
> beforeAll(async () => {
>   app = moduleRef.createNestApplication<NestFastifyApplication>(new FastifyAdapter());
>
>   await app.init();
>   await app.getHttpAdapter().getInstance().ready();
> });
>
> it(`/GET cats`, () => {
>   return app
>     .inject({
>       method: 'GET',
>       url: '/cats',
>     })
>     .then((result) => {
>       expect(result.statusCode).toEqual(200);
>       expect(result.payload).toEqual(/* expectedPayload */);
>     });
> });
>
> afterAll(async () => {
>   await app.close();
> });
> ```

이 예제에서는 이전에 설명한 개념을 기반으로 합니다. `compile()` 메서드 외에도, 이제 `createNestApplication()` 메서드를 사용하여 전체 Nest 런타임 환경을 인스턴스화합니다. 실행 중인 앱에 대한 참조를 `app` 변수에 저장하여 HTTP 요청을 시뮬레이션하는 데 사용할 수 있습니다.

`request()` 함수는 Supertest에서 제공되며, HTTP 테스트를 시뮬레이션하는 데 사용됩니다. 이러한 HTTP 요청을 실행 중인 Nest 앱으로 라우팅하고 싶기 때문에 `request()` 함수에 Nest의 HTTP 리스너에 대한 참조를 전달합니다(이는 Express 플랫폼에서 제공될 수 있음). 따라서 `request(app.getHttpServer())` 구문을 사용합니다. `request()` 호출은 Nest 앱에 연결된 래핑된 HTTP 서버를 반환하며, 실제 HTTP 요청을 시뮬레이션하는 메서드를 제공합니다. 예를 들어, `request(...).get('/cats')`를 사용하면 네트워크를 통해 들어오는 실제 HTTP 요청인 `get '/cats'`와 동일한 요청을 Nest 앱으로 전송할 수 있습니다.

이 예제에서는 `CatsService`의 대체(테스트 더블) 구현을 제공하며, 이는 테스트할 수 있는 하드코딩된 값을 반환합니다. `overrideProvider()`를 사용하여 이러한 대체 구현을 제공합니다. 마찬가지로, Nest는 `overrideModule()`, `overrideGuard()`, `overrideInterceptor()`, `overrideFilter()` 및 `overridePipe()` 메서드를 사용하여 모듈, 가드, 인터셉터, 필터 및 파이프를 재정의하는 방법을 제공합니다.

각 오버라이드 메서드(`overrideModule()` 제외)는 [커스텀 프로바이더](https://docs.nestjs.com/fundamentals/custom-providers)에 대해 설명된 것과 유사한 3가지 다른 메서드를 반환합니다:

- `useClass`: 객체(프로바이더, 가드 등)를 재정의하기 위해 인스턴스화할 클래스를 제공합니다.
- `useValue`: 객체를 재정의할 인스턴스를 제공합니다.
- `useFactory`: 객체를 재정의할 인스턴스를 반환하는 함수를 제공합니다.

한편, `overrideModule()`은 `useModule()` 메서드를 포함하는 객체를 반환하며, 이를 사용하여 원래 모듈을 재정의할 수 있습니다:

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideModule(CatsModule)
  .useModule(AlternateCatsModule)
  .compile();
```

각 오버라이드 메서드 유형은 차례로 `TestingModule` 인스턴스를 반환하며, 다른 메서드와 [유창한 스타일](https://en.wikipedia.org/wiki/Fluent_interface)로 연결할 수 있습니다. `compile()`을 이러한 체인의 끝에 사용하여 Nest가 모듈을 인스턴스화하고 초기화하도록 해야 합니다.

또한, 때때로 테스트가 실행될 때 커스텀 로거를 제공하고 싶을 수 있습니다(예: CI 서버에서). `setLogger()` 메서드를 사용하고 `LoggerService` 인터페이스를 충족하는 객체를 전달하여 테스트 중에 `TestModuleBuilder`가 로깅하는 방법을 지시할 수 있습니다(기본적으로 "error" 로그만 콘솔에 기록됨).

컴파일된 모듈은 다음 표에 설명된 몇 가지 유용한 메서드를 가지고 있습니다:

<table>
  <tr>
    <td>
      <code>createNestApplication()</code>
    </td>
    <td>
      주어진 모듈을 기반으로 Nest 애플리케이션(<code>INestApplication</code> 인스턴스)을 생성하고 반환합니다. 애플리케이션을 수동으로 초기화해야 합니다 <code>init()</code> 메서드를 사용하여.
    </td>
  </tr>
  <tr>
    <td>
      <code>createNestMicroservice()</code>
    </td>
    <td>
      주어진 모듈을 기반으로 Nest 마이크로서비스(<code>INestMicroservice</code> 인스턴스)를 생성하고 반환합니다.
    </td>
  </tr>
  <tr>
    <td>
      <code>get()</code>
    </td>
    <td>
      애플리케이션 컨텍스트에서 사용할 수 있는 컨트롤러 또는 프로바이더(가드, 필터 등 포함)의 정적 인스턴스를 검색합니다. [모듈 참조](https://docs.nestjs.com/fundamentals/module-ref) 클래스에서 상속받았습니다.
    </td>
  </tr>
  <tr>
     <td>
      <code>resolve()</code>
    </td>
    <td>
      애플리케이션 컨텍스트에서 사용할 수 있는 컨트롤러 또는 프로바이더(가드, 필터 등 포함)의 동적으로 생성된 범위가 지정된 인스턴스(요청 또는 일시적)를 검색합니다. [모듈 참조](https://docs.nestjs.com/fundamentals/module-ref) 클래스에서 상속받았습니다.
    </td>
  </tr>
  <tr>
    <td>
      <code>select()</code>
    </td>
    <td>
      모듈의 종속성 그래프를 탐색합니다. 선택한 모듈에서 특정 인스턴스를 검색하는 데 사용할 수 있습니다(엄격 모드(<code>strict: true</code>)와 함께 <code>get()</code> 메서드에서 사용).
    </td>
  </tr>
</table>

> info **힌트** e2e 테스트 파일을 `test` 디렉토리 내에 두세요. 테스트 파일은 `.e2e-spec` 접미사를 가져야 합니다.

#### 전역으로 등록된 향상 기능 재정의

전역으로 등록된 가드(또는 파이프, 인터셉터, 필터)가 있는 경우, 해당 향상 기능을 재정의하려면 몇 가지 추가 단계를 수행해야 합니다. 원래 등록은 다음과 같습니다:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

이는 `APP_*` 토큰을 통해 가드를 "멀티" 프로바이더로 등록합니다. 여기서 `JwtAuthGuard`를 대체할 수 있도록 등록은 이 슬롯에서 기존 프로바이더를 사용해야 합니다:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useExisting: JwtAuthGuard,
    // 'useClass' 대신 'useExisting' 사용
  },
  JwtAuthGuard,
],
```

> info **힌트** `useClass`를 `useExisting`으로 변경하여 Nest가 토큰 뒤에 인스턴스화하는 대신 등록된 프로바이더를 참조하도록 합니다.

이제 `JwtAuthGuard`는 Nest가 재정의할 수 있는 일반적인 프로바이더로 인식되어 `TestingModule`을 만들 때 사용할 수 있습니다:

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideProvider(JwtAuthGuard)
  .useClass(MockAuthGuard)
  .compile();
```

이제 모든 테스트에서 모든 요청에 대해 `MockAuthGuard`를 사용하게 됩니다.

#### 요청 범위 인스턴스 테스트

[요청 범위](/fundamentals/injection-scopes) 프로바이더는 각 수신 **요청**에 대해 고유하게 생성됩니다. 인스턴스는 요청 처리 완료 후 가비지 수집됩니다. 이는 테스트할 요청을 위해 생성된 종속성 주입 하위 트리에 접근할 수 없기 때문에 문제가 됩니다.

우리는 `resolve()` 메서드를 사용하여 동적으로 인스턴스화된 클래스를 검색할 수 있다는 것을 알고 있습니다. 또한 [여기](https://docs.nestjs.com/fundamentals/module-ref#resolving-scoped-providers)에 설명된 것처럼, DI 컨테이너 하위 트리의 수명을 제어하기 위해 고유한 컨텍스트 식별자를 전달할 수 있다는 것을 알고 있습니다. 이를 테스트 컨텍스트에서 어떻게 활용할 수 있을까요?

전략은 컨텍스트 식별자를 미리 생성하고 Nest가 모든 수신 요청에 대해 이 특정 ID를 사용하도록 강제하는 것입니다. 이렇게 하면 테스트할 요청에 대해 생성된 인스턴스에 접근할 수 있습니다.

이를 수행하려면 `ContextIdFactory`에서 `jest.spyOn()`을 사용합니다:

```typescript
const contextId = ContextIdFactory.create();
jest.spyOn(ContextIdFactory, 'getByRequest').mockImplementation(() => contextId);
```

이제 `contextId`를 사용하여 후속 요청에 대해 생성된 단일 DI 컨테이너 하위 트리에 접근할 수 있습니다.

```typescript
catsService = await moduleRef.resolve(CatsService, contextId);
```
