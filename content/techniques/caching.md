### 캐싱

캐싱은 앱의 성능을 향상시키는 훌륭하고 간단한 **기법**입니다. 캐싱은 임시 데이터 저장소로 작동하여 고성능 데이터 액세스를 제공합니다.

#### 설치

먼저 필요한 패키지를 설치합니다:

```bash
$ npm install @nestjs/cache-manager cache-manager
```

> warning **경고** `cache-manager` 버전 4는 `TTL (Time-To-Live)`에 초 단위를 사용합니다. 현재 `cache-manager` (v5) 버전은 밀리초 단위로 변경되었습니다. NestJS는 값을 변환하지 않으며, 제공한 ttl을 그대로 라이브러리에 전달합니다. 즉:
> * `cache-manager` v4를 사용하는 경우 ttl을 초 단위로 제공하십시오.
> * `cache-manager` v5를 사용하는 경우 ttl을 밀리초 단위로 제공하십시오.
> * 문서는 버전 4의 cache-manager를 기준으로 작성되었기 때문에 초 단위를 참조합니다.

#### 메모리 내 캐시

Nest는 다양한 캐시 저장소 공급자를 위한 통합 API를 제공합니다. 기본 제공되는 것은 메모리 내 데이터 저장소입니다. 그러나 Redis와 같은 보다 포괄적인 솔루션으로 쉽게 전환할 수 있습니다.

캐싱을 활성화하려면 `CacheModule`을 가져와 `register()` 메서드를 호출합니다.

```typescript
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { AppController } from './app.controller';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
})
export class AppModule {}
```

#### 캐시 저장소와 상호작용

캐시 관리자 인스턴스와 상호작용하려면 `CACHE_MANAGER` 토큰을 사용하여 클래스로 주입합니다:

```typescript
constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}
```

> info **힌트** `Cache` 클래스는 `cache-manager`에서 가져오고, `CACHE_MANAGER` 토큰은 `@nestjs/cache-manager` 패키지에서 가져옵니다.

`cache-manager` 패키지의 `Cache` 인스턴스에서 `get` 메서드를 사용하여 캐시에서 항목을 검색합니다. 캐시에 항목이 없으면 `null`이 반환됩니다.

```typescript
const value = await this.cacheManager.get('key');
```

항목을 캐시에 추가하려면 `set` 메서드를 사용합니다:

```typescript
await this.cacheManager.set('key', 'value');
```

캐시의 기본 만료 시간은 5초입니다.

특정 키에 대해 TTL(만료 시간)을 수동으로 지정할 수 있습니다:

```typescript
await this.cacheManager.set('key', 'value', 1000);
```

캐시의 만료를 비활성화하려면 `ttl` 구성 속성을 `0`으로 설정합니다:

```typescript
await this.cacheManager.set('key', 'value', 0);
```

캐시에서 항목을 제거하려면 `del` 메서드를 사용합니다:

```typescript
await this.cacheManager.del('key');
```

전체 캐시를 지우려면 `reset` 메서드를 사용합니다:

```typescript
await this.cacheManager.reset();
```

#### 응답 자동 캐싱

> warning **경고** [GraphQL](https://docs.nestjs.com/graphql/quick-start) 애플리케이션에서는 필드 리졸버마다 인터셉터가 별도로 실행됩니다. 따라서 응답을 캐시하는 `CacheModule`은 제대로 작동하지 않을 수 있습니다.

응답 자동 캐싱을 활성화하려면 캐시 데이터를 캐시할 위치에 `CacheInterceptor`를 연결하기만 하면 됩니다.

```typescript
@Controller()
@UseInterceptors(CacheInterceptor)
export class AppController {
  @Get()
  findAll(): string[] {
    return [];
  }
}
```

> warning **경고** 오직 `GET` 엔드포인트만 캐시됩니다. 또한 네이티브 응답 객체(`@Res()`)를 주입하는 HTTP 서버 라우트는 Cache Interceptor를 사용할 수 없습니다. 자세한 내용은 [응답 매핑](https://docs.nestjs.com/interceptors#response-mapping)을 참조하십시오.

필요한 보일러플레이트의 양을 줄이기 위해 `CacheInterceptor`를 전역적으로 모든 엔드포인트에 바인딩할 수 있습니다:

```typescript
import { Module } from '@nestjs/common';
import { CacheModule, CacheInterceptor } from '@nestjs/cache-manager';
import { AppController } from './app.controller';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: CacheInterceptor,
    },
  ],
})
export class AppModule {}
```

#### 캐시 사용자 정의

모든 캐시된 데이터는 자체 만료 시간([TTL](https://en.wikipedia.org/wiki/Time_to_live))을 가집니다. 기본값을 사용자 정의하려면 옵션 객체를 `register()` 메서드에 전달합니다.

```typescript
CacheModule.register({
  ttl: 5, // 초 단위
  max: 10, // 캐시 항목 최대 개수
});
```

#### 모듈을 전역으로 사용

다른 모듈에서 `CacheModule`을 사용하려면 이를 가져와야 합니다(Nest 모듈에서 표준으로 사용됨). 또는 아래와 같이 옵션 객체의 `isGlobal` 속성을 `true`로 설정하여 [전역 모듈](https://docs.nestjs.com/modules#global-modules)로 선언할 수 있습니다. 이 경우, 루트 모듈(e.g., `AppModule`)에 한 번 로드되면 다른 모듈에서 `CacheModule`을 가져올 필요가 없습니다.

```typescript
CacheModule.register({
  isGlobal: true,
});
```

#### 전역 캐시 재정의

전역 캐시가 활성화된 상태에서 캐시 항목은 라우트 경로를 기반으로 자동 생성된 `CacheKey` 아래에 저장됩니다. 메서드별로 특정 캐시 설정(`@CacheKey()` 및 `@CacheTTL()`)을 재정의하여 개별 컨트롤러 메서드에 맞는 맞춤형 캐싱 전략을 사용할 수 있습니다. 이는 [다른 캐시 저장소](https://docs.nestjs.com/techniques/caching#different-stores)를 사용할 때 특히 유용할 수 있습니다.

```typescript
@Controller()
export class AppController {
  @CacheKey('custom_key')
  @CacheTTL(20)
  findAll(): string[] {
    return [];
  }
}
```

> info **힌트** `@CacheKey()` 및 `@CacheTTL()` 데코레이터는 `@nestjs/cache-manager` 패키지에서 가져옵니다.

`@CacheKey()` 데코레이터는 해당하는 `@CacheTTL()` 데코레이터와 함께 또는 없이 사용할 수 있습니다. 하나만 선택하여 재정의할 수 있습니다. 재정의되지 않은 설정은 전역적으로 등록된 기본값을 사용합니다(자세한 내용은 [캐시 사용자 정의](https://docs.nestjs.com/techniques/caching#customize-caching)를 참조하십시오).

#### WebSockets 및 마이크로서비스

`CacheInterceptor`를 WebSocket 구독자와 마이크로서비스 패턴(사용되는 전송 방식에 관계없이)에 적용할 수도 있습니다.

```typescript
@CacheKey('events')
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
```

그러나 캐시된 데이터를 저장하고 검색하는 데 사용되는 키를 지정하려면 추가 `@CacheKey()` 데코레이터가 필요합니다. 또한 **모든 것을 캐시해서는 안 됩니다.** 데이터를 단순히 쿼리하는 대신 비즈니스 작업을 수행하는 작업은 절대 캐시하지 않아야 합니다.

또한 전역 기본 TTL 값을 재정의하는 `@CacheTTL()` 데코레이터를 사용하여 캐시 만료 시간(TTL)을 지정할 수 있습니다.

```typescript
@CacheTTL(10)
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
```

> info **힌트** `@CacheTTL()` 데코레이터는 해당하는 `@CacheKey()` 데코레이터와 함께 또는 없이 사용할 수 있습니다.

#### 추적 조정

기본적으로 Nest는 HTTP 앱에서는 요청 URL을 사용하고, 웹소켓 및 마이크로서비스 앱에서는 캐시 키(즉, `@CacheKey()` 데코레이터를 통해 설정)를 사용하여 캐시 레코드를 엔드포인트와 연결합니다. 그러나 때로는 HTTP 헤더(e.g., `Authorization`을 사용하여 `profile` 엔드포인트를 올바르게 식별)와 같은 다른 요소를 기반으로 추적을 설정하고 싶을 수 있습니다.

이를 위해 `CacheInterceptor`의 하위 클래스를 생성하고 `trackBy()` 메서드를 재정의합니다.

```typescript
@Injectable()
class HttpCacheInterceptor extends CacheInterceptor {
  trackBy(context: ExecutionContext): string | undefined {
    return 'key';
  }
}
```

#### 다른 저장소

이 서비스는 내부적으로 [cache-manager](https://github.com

/node-cache-manager/node-cache-manager)를 사용합니다. `cache-manager` 패키지는 [Redis 저장소](https://github.com/dabroek/node-cache-manager-redis-store)와 같은 다양한 유용한 저장소를 지원합니다. 지원되는 저장소의 전체 목록은 [여기](https://github.com/node-cache-manager/node-cache-manager#store-engines)에서 확인할 수 있습니다. Redis 저장소를 설정하려면 패키지와 해당 옵션을 `register()` 메서드에 전달하기만 하면 됩니다.

```typescript
import type { RedisClientOptions } from 'redis';
import * as redisStore from 'cache-manager-redis-store';
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { AppController } from './app.controller';

@Module({
  imports: [
    CacheModule.register<RedisClientOptions>({
      store: redisStore,

      // 저장소별 구성:
      host: 'localhost',
      port: 6379,
    }),
  ],
  controllers: [AppController],
})
export class AppModule {}
```

> warning **경고** `cache-manager-redis-store`는 Redis v4를 지원하지 않습니다. `ClientOpts` 인터페이스가 올바르게 존재하고 작동하려면 최신 `redis` 3.x.x 메이저 릴리스를 설치해야 합니다. 이 업그레이드 진행 상황을 추적하려면 [이 문제](https://github.com/dabroek/node-cache-manager-redis-store/issues/40)를 참조하십시오.

#### 비동기 구성

모듈 옵션을 컴파일 시점에 정적으로 전달하는 대신 비동기적으로 전달하려는 경우 `registerAsync()` 메서드를 사용합니다. 이 경우 비동기 구성을 처리하는 여러 가지 방법을 제공합니다.

하나의 접근법은 팩토리 함수를 사용하는 것입니다:

```typescript
CacheModule.registerAsync({
  useFactory: () => ({
    ttl: 5,
  }),
});
```

우리의 팩토리는 모든 다른 비동기 모듈 팩토리와 동일하게 동작합니다(비동기일 수 있으며 `inject`를 통해 종속성을 주입할 수 있습니다).

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    ttl: configService.get('CACHE_TTL'),
  }),
  inject: [ConfigService],
});
```

대안으로 `useClass` 메서드를 사용할 수 있습니다:

```typescript
CacheModule.registerAsync({
  useClass: CacheConfigService,
});
```

위의 구성은 `CacheModule` 내에서 `CacheConfigService`를 인스턴스화하고 옵션 객체를 얻는 데 사용합니다. `CacheConfigService`는 구성 옵션을 제공하기 위해 `CacheOptionsFactory` 인터페이스를 구현해야 합니다:

```typescript
@Injectable()
class CacheConfigService implements CacheOptionsFactory {
  createCacheOptions(): CacheModuleOptions {
    return {
      ttl: 5,
    };
  }
}
```

다른 모듈에서 가져온 기존 구성 공급자를 사용하려면 `useExisting` 구문을 사용하십시오:

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

이 구문은 `useClass`와 동일하게 동작하지만 한 가지 중요한 차이점이 있습니다. `CacheModule`은 자체 인스턴스를 만들지 않고 이미 생성된 `ConfigService`를 재사용하기 위해 가져온 모듈을 조회합니다.

> info **힌트** `CacheModule#register` 및 `CacheModule#registerAsync`와 `CacheOptionsFactory`에는 선택적인 제네릭(유형 인수)이 있어 저장소별 구성 옵션을 좁혀서 타입 안전성을 보장합니다.

#### 예제

작동하는 예제는 [여기](https://github.com/nestjs/nest/tree/master/sample/20-cache)에서 확인할 수 있습니다.
