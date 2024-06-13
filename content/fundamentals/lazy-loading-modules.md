### Lazy loading modules

기본적으로 모듈은 즉시 로드됩니다. 즉, 애플리케이션이 로드되자마자 즉시 필요하지 않은 모듈도 모두 로드됩니다. 대부분의 애플리케이션에서는 이것이 문제가 되지 않지만, **서버리스 환경**에서 실행되는 앱/작업의 경우, 시작 지연("콜드 스타트")이 중요한 병목 현상이 될 수 있습니다.

지연 로딩을 통해 특정 서버리스 함수 호출에 필요한 모듈만 로드하여 부트스트랩 시간을 줄일 수 있습니다. 또한 서버리스 함수가 "웜" 상태일 때 다른 모듈을 비동기적으로 로드하여 후속 호출의 부트스트랩 시간을 더욱 단축할 수 있습니다 (지연 모듈 등록).

> info **Hint** **[Angular](https://angular.dev/)** 프레임워크에 익숙하다면 "[지연 로딩 모듈](https://angular.dev/guide/ngmodules/lazy-loading#lazy-loading-basics)" 용어를 들어본 적이 있을 것입니다. 이 기술은 Nest에서 **기능적으로 다르며**, 유사한 명명 규칙을 공유하는 완전히 다른 기능으로 생각해야 합니다.

> warning **Warning** 지연 로드된 모듈 및 서비스에서는 [라이프사이클 후크 메서드](https://docs.nestjs.com/fundamentals/lifecycle-events)가 호출되지 않는다는 점을 유의하십시오.

#### 시작하기

필요에 따라 모듈을 로드하려면 `LazyModuleLoader` 클래스를 일반적인 방식으로 클래스에 주입할 수 있습니다:

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}
}
@@switch
@Injectable()
@Dependencies(LazyModuleLoader)
export class CatsService {
  constructor(lazyModuleLoader) {
    this.lazyModuleLoader = lazyModuleLoader;
  }
}
```

> info **Hint** `LazyModuleLoader` 클래스는 `@nestjs/core` 패키지에서 가져옵니다.

대안으로, 애플리케이션 부트스트랩 파일 (`main.ts`) 내에서 `LazyModuleLoader` provider에 대한 참조를 얻을 수 있습니다:

```typescript
// "app"은 Nest 애플리케이션 인스턴스를 나타냅니다
const lazyModuleLoader = app.get(LazyModuleLoader);
```

이제 다음 구조를 사용하여 모든 모듈을 로드할 수 있습니다:

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);
```

> info **Hint** "지연 로드" 모듈은 **첫 번째** `LazyModuleLoader#load` 메서드 호출 시 **캐시**됩니다. 즉, `LazyModule`을 로드하려는 각 후속 시도는 **매우 빠르며**, 모듈을 다시 로드하는 대신 캐시된 인스턴스를 반환합니다.
>
> ```bash
> Load "LazyModule" attempt: 1
> time: 2.379ms
> Load "LazyModule" attempt: 2
> time: 0.294ms
> Load "LazyModule" attempt: 3
> time: 0.303ms
> ```
>
> 또한, "지연 로드" 모듈은 애플리케이션 부트스트랩 시 즉시 로드된 모듈뿐만 아니라 나중에 애플리케이션에 등록된 다른 지연 모듈과 동일한 모듈 그래프를 공유합니다.

`lazy.module.ts`는 **일반 Nest 모듈**을 내보내는 TypeScript 파일입니다 (추가 변경 사항 필요 없음).

`LazyModuleLoader#load` 메서드는 주입 토큰을 조회 키로 사용하여 내부 provider 목록을 탐색하고 모든 provider에 대한 참조를 얻을 수 있는 [module reference](https://docs.nestjs.com/fundamentals/module-ref)를 반환합니다.

예를 들어, 다음 정의를 가진 `LazyModule`이 있다고 가정해 보겠습니다:

```typescript
@Module({
  providers: [LazyService],
  exports: [LazyService],
})
export class LazyModule {}
```

> info **Hint** 지연 로드된 모듈은 **전역 모듈**로 등록될 수 없습니다. 이는 지연 로드된 모듈이 온디맨드로 등록되기 때문에 전역 모듈로 등록하는 것은 의미가 없습니다. 마찬가지로 등록된 **전역 엔핸서** (guards/interceptors 등)도 올바르게 작동하지 않습니다.

이로 인해 `LazyService` provider에 대한 참조를 다음과 같이 얻을 수 있습니다:

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);

const { LazyService } = await import('./lazy.service');
const lazyService = moduleRef.get(LazyService);
```

> warning **Warning** **Webpack**을 사용하는 경우, `tsconfig.json` 파일을 업데이트해야 합니다. `compilerOptions.module`을 `"esnext"`로 설정하고 `compilerOptions.moduleResolution` 속성에 `"node"` 값을 추가하십시오:
>
> ```json
> {
>   "compilerOptions": {
>     "module": "esnext",
>     "moduleResolution": "node",
>     ...
>   }
> }
> ```
>
> 이러한 옵션이 설정되면 [코드 분할](https://webpack.js.org/guides/code-splitting/) 기능을 활용할 수 있습니다.

#### 컨트롤러, 게이트웨이 및 리졸버 지연 로딩

Nest에서 컨트롤러 (또는 GraphQL 애플리케이션의 리졸버)는 경로/쿼리/주제 (또는 쿼리/변이)를 나타내므로 `LazyModuleLoader` 클래스를 사용하여 **지연 로드할 수 없습니다**.

> error **Warning** 지연 로드된 모듈에 등록된 컨트롤러, [리졸버](https://docs.nestjs.com/graphql/resolvers) 및 [게이트웨이](https://docs.nestjs.com/websockets/gateways)는 예상대로 동작하지 않습니다. 마찬가지로, 온디맨드로 미들웨어 함수를 ( `MiddlewareConsumer` 인터페이스를 구현하여) 등록할 수 없습니다.

예를 들어, Fastify 드라이버가 내장된 REST API (HTTP 애플리케이션)를 빌드한다고 가정해 보겠습니다 (`@nestjs/platform-fastify` 패키지를 사용하여). Fastify는 애플리케이션이 준비되어 성공적으로 메시지를 수신할 때 경로를 등록할 수 없습니다. 즉, 모듈의 컨트롤러에 등록된 경로 매핑을 분석하더라도 모든 지연 로드된 경로는 런타임에 등록할 방법이 없기 때문에 접근할 수 없습니다.

마찬가지로, `@nestjs/microservices` 패키지의 일부로 제공되는 일부 전송 전략 (Kafka, gRPC 또는 RabbitMQ 포함)은 연결이 설정되기 전에 특정 주제/채널에 구독해야 합니다. 애플리케이션이 메시지를 수신하기 시작하면 프레임워크는 새 주제에 구독할 수 없습니다.

마지막으로, 코드 우선 접근 방식이 활성화된 `@nestjs/graphql` 패키지는 메타데이터를 기반으로 GraphQL 스키마를 동적으로 생성합니다. 이는 모든 클래스를 미리 로드해야 한다는 것을 의미합니다. 그렇지 않으면 적절하고 유효한 스키마를 생성할 수 없습니다.

#### 일반적인 사용 사례

가장 일반적으로, 작업자/크론 작업/람다 & 서버리스 함수/웹훅이 입력 인수 (경로/날짜/쿼리 매개변수 등)에 따라 다른 서비스를 트리거해야 할 때 지연 로드된 모듈을 볼 수 있습니다. 반면에, 지연 로딩 모듈은 모놀리식 애플리케이션의 경우 부트스트랩 시간이 그다지 중요하지 않기 때문에 큰 의미가 없을 수 있습니다.
