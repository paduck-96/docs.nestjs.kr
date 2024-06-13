## Custom providers

이전 장에서 **의존성 주입(DI)** 의 다양한 측면과 Nest에서 어떻게 사용되는지 살펴봤습니다. 그 예로 [생성자 기반](https://docs.nestjs.com/providers#dependency-injection) 의존성 주입을 통해 클래스에 인스턴스(종종 서비스 providers)를 주입하는 방법이 있습니다. 의존성 주입은 Nest 코어에 기본적으로 내장되어 있습니다. 애플리케이션이 더 복잡해짐에 따라 DI 시스템의 전체 기능을 활용할 필요가 있을 수 있으므로 자세히 살펴보겠습니다.

### DI 기본 사항

의존성 주입은 [제어 역전(IoC)](https://en.wikipedia.org/wiki/Inversion_of_control) 기술로, 의존성의 인스턴스화를 코드에서 명령적으로 수행하는 대신 IoC 컨테이너(우리의 경우 NestJS 런타임 시스템)에 위임합니다. [Providers 장](https://docs.nestjs.com/providers)의 예제를 통해 이를 자세히 살펴보겠습니다.

먼저, 제공자를 정의합니다. `@Injectable()` 데코레이터는 `CatsService` 클래스를 제공자로 표시합니다.

```typescript
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
```

그런 다음 Nest가 제공자를 컨트롤러 클래스에 주입하도록 요청합니다:

```typescript
import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

마지막으로 제공자를 Nest IoC 컨테이너에 등록합니다:

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

이 작업이 작동하기 위해서는 다음과 같은 세 가지 주요 단계가 있습니다:

1. `cats.service.ts`에서 `@Injectable()` 데코레이터는 `CatsService` 클래스를 Nest IoC 컨테이너에서 관리할 수 있는 클래스로 선언합니다.
2. `cats.controller.ts`에서 `CatsController`는 생성자 주입으로 `CatsService` 토큰에 대한 의존성을 선언합니다:

```typescript
constructor(private catsService: CatsService)
```

3. `app.module.ts`에서 `CatsService` 토큰을 `cats.service.ts` 파일의 `CatsService` 클래스와 연결합니다. 이 연결(또는 등록) 방법은 [아래](https://docs.nestjs.com/fundamentals/custom-providers#standard-providers)에서 자세히 설명합니다.

Nest IoC 컨테이너가 `CatsController`를 인스턴스화할 때 먼저 의존성을 찾습니다. `CatsService` 의존성을 찾으면 `CatsService` 토큰에 대한 조회를 수행하여 등록 단계(#3)에서 `CatsService` 클래스를 반환합니다. `SINGLETON` 범위(기본 동작)를 가정하면 Nest는 `CatsService` 인스턴스를 생성하여 캐시하고 반환하거나 이미 캐시된 인스턴스가 있으면 해당 인스턴스를 반환합니다.

### 표준 providers

`@Module()` 데코레이터를 자세히 살펴보겠습니다. `app.module`에서 다음과 같이 선언합니다:

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

`providers` 속성은 `providers`의 배열을 받습니다. 지금까지 우리는 클래스 이름을 통해 제공자를 제공했습니다. 사실, `providers: [CatsService]` 구문은 다음과 같은 더 완전한 구문의 단축 표현입니다:

```typescript
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

이제 이 명시적 구성을 보면 등록 과정을 이해할 수 있습니다. 여기서 우리는 `CatsService` 토큰을 `CatsService` 클래스와 명확하게 연결하고 있습니다. 단축 표기법은 토큰을 동일한 이름의 클래스 인스턴스로 요청하는 가장 일반적인 사용 사례를 단순화하기 위한 편의 기능입니다.

### 사용자 정의 providers

표준 제공자가 제공하는 기능을 초과하는 요구 사항이 있을 때 사용자 정의 제공자를 정의할 수 있습니다. 다음은 몇 가지 예입니다:

- Nest가 클래스를 인스턴스화하는 대신 사용자 정의 인스턴스를 만들고 싶을 때
- 기존 클래스를 두 번째 의존성에서 재사용하고 싶을 때
- 테스트를 위해 클래스를 모의 버전으로 재정의하고 싶을 때

Nest는 이러한 경우를 처리하기 위해 사용자 정의 제공자를 정의할 수 있는 여러 가지 방법을 제공합니다.

### Value providers: `useValue`

`useValue` 구문은 상수 값을 주입하거나 외부 라이브러리를 Nest 컨테이너에 넣거나 실제 구현을 모의 객체로 교체하는 데 유용합니다. 예를 들어, 테스트 목적으로 `CatsService`를 모의 서비스로 사용하도록 Nest를 강제하고 싶을 때 다음과 같이 할 수 있습니다:

```typescript
import { CatsService } from './cats.service';

const mockCatsService = {
  /* mock implementation
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

이 예제에서 `CatsService` 토큰은 `mockCatsService` 모의 객체로 해결됩니다. `useValue`는 값이 필요하며, 이 경우 교체할 `CatsService` 클래스와 동일한 인터페이스를 가진 리터럴 객체를 사용합니다. TypeScript의 [구조적 타이핑](https://www.typescriptlang.org/docs/handbook/type-compatibility.html) 덕분에 호환 가능한 인터페이스를 가진 모든 객체를 사용할 수 있습니다.

### 비클래스 기반 providers 토큰

지금까지 우리는 providers 토큰으로 클래스 이름을 사용했습니다. 때로는 DI 토큰으로 문자열이나 심볼을 사용하는 유연성을 원할 수 있습니다. 예를 들어:

```typescript
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

이 예제에서는 문자열 값 토큰(`'CONNECTION'`)을 외부 파일에서 가져온 기존 `connection` 객체와 연결하고 있습니다.

`@Inject()` 데코레이터를 사용하여 이러한 제공자를 주입할 수 있습니다. 이 데코레이터는 하나의 인수(토큰)를 받습니다.

```typescript
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
```

### 클래스 providers: `useClass`

`useClass` 구문을 사용하면 토큰이 해결해야 하는 클래스를 동적으로 결정할 수 있습니다. 예를 들어, `ConfigService`라는 추상(또는 기본) 클래스가 있고 현재 환경에 따라 다른 구현을 제공하고자 할 때 다음과 같이 할 수 있습니다:

```typescript
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

여기서 `ConfigService` 클래스 이름을 토큰으로 사용하고 있습니다. `ConfigService`에 의존하는 클래스에 대해 Nest는 제공된 클래스(`DevelopmentConfigService` 또는 `ProductionConfigService`)의 인스턴스를 주입합니다.

### 팩토리 providers: `useFactory`

`useFactory` 구문은 제공자를 **동적으로** 생성할 수 있습니다. 실제 제공자는 팩토리 함수에서 반환된 값에 의해 제공됩니다. 팩토리 함수는 간단할 수도 있고 복잡할 수도 있으며, 다른 제공자를 주입하여 결과를 계산할 수 있습니다. 다음 예제는 이를 설명합니다:

```typescript
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider, optionalProvider?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
};

@Module({
  providers: [
    connectionProvider,
    OptionsProvider,
  ],
})
export class AppModule {}
```

### 별칭 providers: `useExisting`

`useExisting` 구문을 사용하면 기존 제공자에 대한 별칭을 만들 수 있습니다. 예를 들어, 문자열 기반 토큰 `'AliasedLoggerService'`는 클래스 기반 토큰 `LoggerService`에 대한 별칭입니다:

```typescript
@Injectable()
class LoggerService {
  /* implementation details */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

### 비서비스

 기반 providers

제공자는 종종 서비스를 제공하지만, 그 사용은 이에 국한되지 않습니다. 제공자는 **어떤** 값도 제공할 수 있습니다. 예를 들어, 현재 환경에 따라 구성 객체 배열을 제공하는 예는 다음과 같습니다:

```typescript
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```

### 사용자 정의 providers 내보내기

사용자 정의 제공자도 다른 모듈에서 사용하려면 내보내야 합니다. 토큰이나 전체 providers 객체를 사용하여 내보낼 수 있습니다. 다음 예제는 토큰을 사용하여 내보내는 방법을 보여줍니다:

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

또는 전체 providers 객체로 내보낼 수도 있습니다:

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```

이 내용은 사용자 정의 제공자를 설정하고 사용하는 다양한 방법을 포함하고 있으며, 복잡한 애플리케이션에서 의존성을 관리하는 데 유용한 정보를 제공합니다.
