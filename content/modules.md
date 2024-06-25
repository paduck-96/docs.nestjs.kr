### Modules

모듈은 `@Module()` 데코레이터로 주석이 달린 클래스입니다. `@Module()` 데코레이터는 **Nest**가 애플리케이션 구조를 조직하는 데 사용하는 메타데이터를 제공합니다.

<figure><img src="https://docs.nestjs.com/assets/Modules_1.png" /></figure>

각 애플리케이션에는 최소 하나의 모듈, 즉 **루트 모듈**이 있습니다. 루트 모듈은 Nest가 **애플리케이션 그래프**를 구축하는 시작점입니다. 애플리케이션 그래프는 모듈 및 provider 관계와 종속성을 해결하는 데 사용되는 내부 데이터 구조입니다. 매우 작은 애플리케이션은 이론적으로 루트 모듈만 가질 수 있지만, 이는 일반적인 경우는 아닙니다. 모듈은 컴포넌트를 조직하는 효과적인 방법으로 **강력히** 권장됩니다. 따라서 대부분의 애플리케이션은 여러 모듈을 사용하여 각 모듈이 밀접하게 관련된 **기능** 집합을 캡슐화합니다.

`@Module()` 데코레이터는 모듈을 설명하는 속성을 가진 객체를 받습니다:

|               |                                                                                                                                                                                                          |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `providers`   | Nest 인젝터에 의해 인스턴스화되고 최소한 이 모듈에서 공유될 수 있는 providers                                                                                          |
| `controllers` | 이 모듈에서 정의된 컨트롤러 세트로, 인스턴스화되어야 합니다                                                                                                                              |
| `imports`     | 이 모듈에서 필요한 providers를 내보내는 모듈들의 목록                                                                                                                 |
| `exports`     | 이 모듈에서 제공하는 `providers`의 하위 집합으로, 이 모듈을 가져오는 다른 모듈에서 사용할 수 있어야 합니다. provider 자체 또는 해당 토큰(`provide` 값)을 사용할 수 있습니다. |

모듈은 기본적으로 providers를 **캡슐화**합니다. 이는 현재 모듈의 일부가 아니거나 가져온 모듈에서 내보내지 않은 providers를 주입하는 것이 불가능함을 의미합니다. 따라서 모듈에서 내보낸 providers를 모듈의 공개 인터페이스 또는 API로 간주할 수 있습니다.

#### 기능 모듈

`CatsController`와 `CatsService`는 동일한 애플리케이션 도메인에 속합니다. 이들이 밀접하게 관련되어 있으므로, 이를 기능 모듈로 이동하는 것이 합리적입니다. 기능 모듈은 특정 기능과 관련된 코드를 조직하여 코드를 체계적으로 유지하고 명확한 경계를 설정합니다. 이는 애플리케이션의 크기나 팀의 규모가 커짐에 따라 복잡성을 관리하고 [SOLID](https://en.wikipedia.org/wiki/SOLID) 원칙을 따르는 데 도움이 됩니다.

이를 시연하기 위해 `CatsModule`을 생성하겠습니다.

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

> info **힌트** CLI를 사용하여 모듈을 생성하려면 `$ nest g module cats` 명령어를 실행하세요.

위에서는 `cats.module.ts` 파일에 `CatsModule`을 정의하고, 이 모듈과 관련된 모든 것을 `cats` 디렉터리로 이동했습니다. 마지막으로 이 모듈을 루트 모듈(`app.module.ts` 파일에 정의된 `AppModule`)에 가져와야 합니다.

```typescript
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

이제 디렉터리 구조는 다음과 같습니다:

<div class="file-tree">
  <div class="item">src</div>
  <div class="children">
    <div class="item">cats</div>
    <div class="children">
      <div class="item">dto</div>
      <div class="children">
        <div class="item">create-cat.dto.ts</div>
      </div>
      <div class="item">interfaces</div>
      <div class="children">
        <div class="item">cat.interface.ts</div>
      </div>
      <div class="item">cats.controller.ts</div>
      <div class="item">cats.module.ts</div>
      <div class="item">cats.service.ts</div>
    </div>
    <div class="item">app.module.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

#### 공유 모듈

Nest에서 모듈은 기본적으로 **싱글톤**이며, 따라서 여러 모듈 간에 동일한 provider 인스턴스를 쉽게 공유할 수 있습니다.

<figure><img src="/assets/Shared_Module_1.png" /></figure>

모든 모듈은 자동으로 **공유 모듈**입니다. 생성된 모듈은 어떤 모듈에서든 재사용할 수 있습니다. `CatsService` 인스턴스를 여러 다른 모듈 간에 공유하려는 경우를 상상해 봅시다. 이를 위해 먼저 `exports` 배열에 `CatsService` provider를 추가하여 내보내야 합니다:

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

이제 `CatsModule`을 가져오는 모든 모듈은 `CatsService`에 접근할 수 있으며, 이를 가져오는 다른 모든 모듈과 동일한 인스턴스를 공유합니다.

#### 모듈 재수출

위에서 본 것처럼, 모듈은 내부 providers를 내보낼 수 있습니다. 또한, 가져온 모듈을 다시 내보낼 수도 있습니다. 아래 예제에서 `CommonModule`은 `CoreModule`에 가져와서 내보내져, 이를 가져오는 다른 모듈에서 사용할 수 있게 합니다.

```typescript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

#### 의존성 주입

모듈 클래스는 providers를 **주입**할 수 있습니다(예: 구성 목적).

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
```

그러나 모듈 클래스 자체는 [순환 종속성](https://docs.nestjs.com/fundamentals/circular-dependency) 때문에 provider로 주입될 수 없습니다.

#### 전역 모듈

모든 곳에서 동일한 모듈 세트를 가져와야 하는 경우가 있을 수 있습니다. Nest와 달리, [Angular](https://angular.dev)에서는 `providers`가 전역 범위에 등록됩니다. 한 번 정의되면 어디서든 사용할 수 있습니다. 그러나 Nest는 providers를 모듈 범위 내에 캡슐화합니다. 모듈을 캡슐화하지 않으면 다른 곳에서 모듈의 providers를 사용할 수 없습니다.

helpers, 데이터베이스 연결 등 어디서든 사용할 수 있어야 하는 providers 세트를 제공하려면 `@Global()` 데코레이터로 모듈을 **전역**으로 만드세요.

```typescript
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

`@Global()` 데코레이터는 모듈을 전역 범위로 만듭니다. 전역 모듈은 일반적으로 루트 또는 코어 모듈에 한 번만 등록해야 합니다. 위 예제에서는 `CatsService` provider가 어디서든 사용 가능하게 되며, 서비스를 주입하려는 모듈은 `imports` 배열에 `CatsModule`을 추가할 필요가 없습니다.

> info **힌트** 모든 것을 전역으로 만드는 것은 좋은 설계 결정이 아닙니다. 전역 모듈은 필요한 보일러플레이트 양을 줄이기 위해 존재합니다. 모듈의 API를 소비자에게 제공하는 일반적인 방법은 `imports` 배열을 사용하는 것입니다.

#### 동적 모듈

Nest 모듈 시스템에는 **동적 모듈**이라는 강력한 기능이 포함되어 있습니다. 이 기능을 사용하면 providers를 동적으로 등록하고 구성할 수 있는 맞춤형 모듈을 쉽게 만들 수 있습니다. 동적 모듈에 대한 자세한 내용은 [여기](https://docs.nestjs.com/fundamentals/dynamic-modules)에서 확인할 수 있습니다. 이 장에서는 모듈에 대한 소개를 완료하기 위해 간략하게 설명합니다.

다음은 `DatabaseModule`에 대한 동적 모듈 정의 예제입니다:

```typescript
import { Module,

 DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

> info **힌트** `forRoot()` 메서드는 동적 모듈을 동기적 또는 비동기적으로 반환할 수 있습니다(즉, `Promise`를 통해).

이 모듈은 기본적으로 `Connection` provider를 정의하지만(`@Module()` 데코레이터 메타데이터에서), `forRoot()` 메서드에 전달된 `entities` 및 `options` 객체에 따라 예를 들어, 리포지토리와 같은 providers의 컬렉션을 추가로 노출합니다. 동적 모듈이 반환하는 속성은 `@Module()` 데코레이터에 정의된 기본 모듈 메타데이터를 **확장**합니다(재정의하지 않음). 따라서 정적으로 선언된 `Connection` provider와 동적으로 생성된 리포지토리 providers가 모듈에서 내보내집니다.

동적 모듈을 전역 범위로 등록하려면 `global` 속성을 `true`로 설정하세요.

```typescript
{
  global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

> warning **경고** 앞서 언급한 것처럼, 모든 것을 전역으로 만드는 것은 **좋은 설계 결정이 아닙니다**.

`DatabaseModule`은 다음과 같이 가져와서 구성할 수 있습니다:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

동적 모듈을 다시 내보내려면 `exports` 배열에서 `forRoot()` 메서드 호출을 생략할 수 있습니다:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```

[동적 모듈](https://docs.nestjs.com/fundamentals/dynamic-modules) 장에서는 이 주제를 더 자세히 다루며, [작동하는 예제](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)를 포함하고 있습니다.

> info **힌트** `ConfigurableModuleBuilder`를 사용하여 고도로 맞춤화된 동적 모듈을 만드는 방법은 [이 장](https://docs.nestjs.com/fundamentals/dynamic-modules#configurable-module-builder)에서 확인할 수 있습니다.
