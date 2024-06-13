## 동적 modules

[Modules 챕터](https://docs.nestjs.com/modules)에서는 Nest 모듈의 기본 사항을 다루며 [동적 모듈](https://docs.nestjs.com/modules#dynamic-modules)에 대한 간략한 소개를 포함합니다. 이 챕터에서는 동적 모듈에 대해 자세히 설명합니다. 완료 후에는 동적 모듈이 무엇인지, 언제 어떻게 사용하는지에 대해 잘 이해할 수 있을 것입니다.

### 소개

**Overview** 섹션의 대부분의 애플리케이션 코드 예제는 일반 모듈 또는 정적 모듈을 사용합니다. 모듈은 [providers](https://docs.nestjs.com/providers)와 [controllers](https://docs.nestjs.com/controllers)와 같은 구성 요소 그룹을 정의하여 전체 애플리케이션의 모듈형 부분으로 결합합니다. 이들은 이러한 구성 요소에 대한 실행 컨텍스트 또는 범위를 제공합니다. 예를 들어, 모듈에서 정의된 providers는 이를 내보내지 않고도 모듈의 다른 멤버에게 보입니다. provider가 모듈 외부에서 보일 필요가 있을 때는 먼저 해당 모듈에서 내보내고 소비하는 모듈에 가져옵니다.

익숙한 예제를 살펴보겠습니다.

먼저, `UsersService`를 제공하고 내보내기 위해 `UsersModule`을 정의합니다. `UsersModule`은 `UsersService`의 **호스트** 모듈입니다.

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

다음으로, `UsersModule`을 가져와서 `UsersModule`의 내보내진 providers를 `AuthModule` 내에서 사용할 수 있게 하는 `AuthModule`을 정의합니다.

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

이 구조는 `AuthService`에서 예를 들어 `UsersService`를 주입할 수 있게 합니다.

```typescript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  /*
    this.usersService를 사용하는 구현
  */
}
```

이를 **정적** module 바인딩이라고 합니다. Nest가 모듈을 연결하는 데 필요한 모든 정보는 이미 호스트 모듈과 소비 모듈에 선언되어 있습니다. 이 과정에서 일어나는 일을 자세히 살펴보겠습니다. Nest는 `AuthModule` 내에서 `UsersService`를 사용할 수 있게 하기 위해 다음 작업을 수행합니다.

1. `UsersModule`을 인스턴스화하고, `UsersModule` 자체가 소비하는 다른 모듈을 전이적으로 가져오며, 모든 종속성을 전이적으로 해결합니다(참조: [Custom providers](https://docs.nestjs.com/fundamentals/custom-providers)).
2. `AuthModule`을 인스턴스화하고, `UsersModule`의 내보낸 providers를 `AuthModule`의 구성 요소에서 사용할 수 있게 합니다(마치 `AuthModule`에 선언된 것처럼).
3. `AuthService`에 `UsersService` 인스턴스를 주입합니다.

### 동적 module 사용 사례

정적 module 바인딩에서는 소비 모듈이 호스트 모듈의 providers가 어떻게 구성되는지 **영향을 미칠 수 있는** 기회가 없습니다. 왜 이것이 중요할까요? 일반적인 모듈이 다른 사용 사례에서 다르게 동작해야 하는 경우를 고려해보세요. 이는 많은 시스템에서 "플러그인" 개념과 유사하며, 일반적인 기능이 사용 전에 일부 구성으로 사용자 정의되어야 합니다.

Nest의 좋은 예로는 **구성 module**이 있습니다. 많은 애플리케이션은 구성 세부 정보를 외부화하여 구성 모듈을 사용하는 것이 유용합니다. 이렇게 하면 개발자용 개발 데이터베이스, 스테이징/테스트 환경용 스테이징 데이터베이스 등 다양한 배포에서 애플리케이션 설정을 동적으로 변경할 수 있습니다. 구성 매개변수 관리를 구성 모듈에 위임하면 애플리케이션 소스 코드는 구성 매개변수와 독립적으로 유지됩니다.

문제는 구성 module 자체가 일반적이기 때문에(마치 "플러그인"처럼) 소비 모듈에 의해 사용자 정의되어야 한다는 점입니다. 여기서 _동적 모듈_이 등장합니다. 동적 module 기능을 사용하면 소비 모듈이 가져올 때 구성 모듈의 속성과 동작을 사용자 정의할 수 있는 API를 제공할 수 있습니다.

즉, 동적 모듈은 하나의 모듈을 다른 모듈에 가져오고 가져올 때 해당 모듈을 사용자 정의하는 API를 제공합니다. 이는 지금까지 본 정적 바인딩과는 다릅니다.

### 구성 module 예제

이 섹션에서는 [구성 챕터](https://docs.nestjs.com/techniques/configuration#service)에서의 기본 예제를 사용할 것입니다. 이 장 끝에 완성된 버전은 [여기](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)에서 확인할 수 있습니다.

우리의 요구 사항은 `ConfigModule`이 사용자 정의할 수 있도록 `options` 객체를 받아들이는 것입니다. 기본 예제는 `.env` 파일의 위치를 프로젝트 루트 폴더로 하드코딩합니다. 예를 들어, 루트 폴더 아래의 `config` 폴더(즉, `src`의 형제 폴더)에 다양한 `.env` 파일을 저장하고 싶다고 가정해보세요. 다른 프로젝트에서 `ConfigModule`을 사용할 때 다른 폴더를 선택할 수 있기를 원할 것입니다.

동적 모듈을 사용하면 모듈을 가져올 때 매개변수를 전달하여 동작을 변경할 수 있습니다. 이것이 어떻게 작동하는지 살펴보겠습니다. 소비 모듈의 관점에서 어떻게 보일지 최종 목표부터 시작한 후 역으로 작업하는 것이 도움이 됩니다. 먼저 `ConfigModule`을 _정적_으로 가져오는 예제를 빠르게 검토해보겠습니다(즉, 가져온 모듈의 동작에 영향을 미칠 수 있는 접근 방식이 아닙니다). `@Module()` 데코레이터의 `imports` 배열에 주목하십시오.

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

이제 구성 객체를 전달하는 _동적 모듈_ 가져오기가 어떻게 보일지 생각해봅시다. 이 두 예제 사이의 `imports` 배열 차이를 비교하십시오.

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

위의 동적 예제에서 무엇이 일어나고 있는지 살펴봅시다. 움직이는 부분은 무엇일까요?

1. `ConfigModule`은 일반 클래스이므로 **정적 메서드** `register()`를 가지고 있어야 합니다. 이는 `ConfigModule` 클래스의 **인스턴스**가 아닌 클래스에서 호출하고 있기 때문에 정적 메서드입니다. 참고: 이 메서드는 임의의 이름을 가질 수 있지만, 관례적으로 `forRoot()` 또는 `register()`라고 부르는 것이 좋습니다.
2. `register()` 메서드는 우리가 정의하므로 원하는 모든 입력 인수를 받을 수 있습니다. 이 경우, 적합한 속성을 가진 간단한 `options` 객체를 받겠습니다.
3. `register()` 메서드는 모듈과 유사한 무언가를 반환해야 합니다. 반환 값은 지금까지 본 `imports` 목록에 나타나 있으며 module 목록을 포함합니다.

사실, `register()` 메서드가 반환할 것은 `DynamicModule`입니다. 동적 모듈은 런타임에 생성된 모듈로, 정적 모듈과 동일한 속성을 가지며, `module`이라는 추가 속성만 갖습니다. 샘플 정적 module 선언을 빠르게 검토해봅시다. 데코레이터에 전달된 module 옵션에 주목하십시오.

```typescript
@Module({
  imports: [DogsModule],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
```

동적 모듈은 `module`이라는 추가 속성을 가지며 동일한 인터페이스를 가진 객체를 반환

해야 합니다. `module` 속성은 모듈의 이름으로 사용되며, module 클래스 이름과 동일해야 합니다. 아래 예제를 참조하십시오.

동적 모듈의 경우, 모든 속성은 선택 사항이지만 `module` 속성은 필수입니다.

정적 `register()` 메서드는 무엇일까요? 이 메서드의 역할은 `DynamicModule` 인터페이스를 가진 객체를 반환하는 것입니다. 이를 호출하면, 모듈을 `imports` 목록에 제공하는 효과가 있으며, 정적 경우처럼 module 클래스 이름을 나열하여 수행합니다. 다시 말해, 동적 module API는 모듈을 반환하지만, `@Module` 데코레이터를 통해 속성을 고정하는 대신 프로그램적으로 속성을 지정합니다.

이해한 내용을 바탕으로 동적 `ConfigModule` 선언이 어떻게 생겨야 하는지 살펴보겠습니다.

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(): DynamicModule {
    return {
      module: ConfigModule,
      providers: [ConfigService],
      exports: [ConfigService],
    };
  }
}
```

이제 이 부분들이 어떻게 연결되는지 명확하게 보입니다. `ConfigModule.register(...)`를 호출하면 `@Module()` 데코레이터를 통해 지금까지 메타데이터로 제공한 속성과 유사한 속성을 가진 `DynamicModule` 객체를 반환합니다.

동적 모듈이 아직 흥미롭지 않은 이유는 사용자 정의할 수 있는 기능이 없기 때문입니다. 이를 다음으로 해결하겠습니다.

### module 구성

`ConfigModule`의 동작을 사용자 정의하는 명확한 방법은 정적 `register()` 메서드에 `options` 객체를 전달하는 것입니다. 다시 소비 모듈의 `imports` 속성을 살펴봅시다.

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

이렇게 하면 `options` 객체를 동적 모듈에 전달할 수 있습니다. 이 `options` 객체를 `ConfigModule`에서 어떻게 사용할 수 있을까요? 잠시 생각해봅시다. `ConfigModule`은 기본적으로 다른 providers에서 사용할 수 있도록 `ConfigService`를 제공하고 내보내는 호스트입니다. 실제로 `ConfigService`가 `options` 객체를 읽어 동작을 사용자 정의해야 합니다. `register()` 메서드에서 `options` 객체를 `ConfigService`로 전달하는 방법을 알았다고 가정해보겠습니다. 그런 가정하에, 몇 가지 변경을 통해 서비스의 동작을 `options` 객체의 속성에 따라 사용자 정의할 수 있습니다. (**참고**: 실제로 전달 방법을 아직 찾지 못했기 때문에 일단 `options`를 하드코딩하겠습니다. 이 부분은 곧 수정하겠습니다).

```typescript
import { Injectable } from '@nestjs/common';
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const options = { folder: './config' };

    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

이제 `ConfigService`는 `options`에 지정된 폴더에서 `.env` 파일을 찾을 수 있습니다.

남은 작업은 `register()` 단계에서 `options` 객체를 `ConfigService`에 주입하는 것입니다. 그리고 당연히 _의존성 주입_을 사용할 것입니다. 이는 중요한 포인트이므로 반드시 이해해야 합니다. `ConfigModule`은 `ConfigService`를 제공합니다. `ConfigService`는 런타임에만 제공되는 `options` 객체에 의존합니다. 따라서 런타임에 `options` 객체를 Nest IoC 컨테이너에 먼저 바인딩한 다음, Nest가 이를 `ConfigService`에 주입하도록 해야 합니다. **Custom providers** 챕터에서 providers는 단지 서비스뿐만 아니라 [어떤 값도 포함할 수 있다](https://docs.nestjs.com/fundamentals/custom-providers#non-service-based-providers)고 배웠으므로, 간단한 `options` 객체를 처리하는 데 의존성 주입을 사용하는 것이 좋습니다.

먼저 IoC 컨테이너에 options 객체를 바인딩하는 작업을 해결하겠습니다. 이는 정적 `register()` 메서드에서 수행됩니다. 동적으로 모듈을 구성하고 있으며, 모듈의 속성 중 하나가 providers 목록이기 때문에 `options` 객체를 provider로 정의해야 합니다. 이를 통해 `ConfigService`에 주입할 수 있습니다. 다음 코드에서 `providers` 배열에 주목하십시오.

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(options: Record<string, any>): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}
```

이제 `'CONFIG_OPTIONS'` provider를 `ConfigService`에 주입하여 프로세스를 완료할 수 있습니다. 클래스 토큰이 아닌 provider를 정의할 때는 [여기](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens)에 설명된 대로 `@Inject()` 데코레이터를 사용해야 합니다.

```typescript
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { Injectable, Inject } from '@nestjs/common';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor(@Inject('CONFIG_OPTIONS') private options: Record<string, any>) {
    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

마지막으로, 단순화를 위해 문자열 기반의 주입 토큰(`'CONFIG_OPTIONS'`)을 사용했지만, 모범 사례는 별도의 파일에서 상수(또는 `Symbol`)로 정의하고 이를 가져오는 것입니다. 예를 들어:

```typescript
export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';
```

### 예제

이 장의 전체 예제 코드는 [여기](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)에서 확인할 수 있습니다.

### 커뮤니티 가이드라인

`@nestjs/` 패키지 중 일부에서 `forRoot`, `register`, `forFeature` 메서드를 사용하는 것을 본 적이 있을 것입니다. 이 모든 메서드의 차이점이 무엇인지 궁금할 수 있습니다. 이에 대한 엄격한 규칙은 없지만, `@nestjs/` 패키지는 다음 지침을 따르려고 합니다.

- `register`를 사용하여 모듈을 생성할 때는 호출 모듈에서만 사용하도록 특정 구성을 가진 동적 모듈을 구성할 것으로 예상합니다. 예를 들어, Nest의 `@nestjs/axios`: `HttpModule.register({ baseUrl: 'someUrl' })`와 같이. 다른 모듈에서 `HttpModule.register({ baseUrl: 'somewhere else' })`를 사용하면 다른 구성을 가지게 됩니다. 원하는 만큼 많은 모듈에 대해 이 작업을 수행할 수 있습니다.

- `forRoot`를 사용하여 모듈을 구성할 때는 한 번만 구성하고 여러 곳에서 해당 구성을 재사용할 것으로 예상합니다(추상화되어 있기 때문에 무의식적으로). 따라서 하나의 `GraphQLModule.forRoot()`, 하나의 `TypeOrmModule.forRoot()` 등이 있습니다.

- `forFeature`를 사용하여 동적 모듈의 `forRoot` 구성을 사용하지만, 호출 모듈의 필요에 맞게 일부 구성을 수정해야 할 것으로 예상합니다(예: 이 모듈이 액세스해야 하는 저장소 또는 로거가 사용할 컨텍스트).

이들 모두는 일반적으로 `async` 대응 메서드(`registerAsync`, `forRootAsync`, `forFeatureAsync`)를 가지고 있으며, 이는 동일한 의미를 가지지만, 구성에도 Nest의 의존성 주입을 사용합니다.

### Configurable module 빌더

높은 구성 가능성을 가진 동적 모듈을 수동으로 생성하는 것은 특히 초보자에게 매우 복잡하기 때문에 Nest는 `ConfigurableModuleBuilder` 클래스를 제공하여 이 프로세스를 용이하게 하고 몇 줄의 코드로 모

듈 "청사진"을 구성할 수 있습니다.

예를 들어, 위에서 사용한 예제(`ConfigModule`)를 가져와 `ConfigurableModuleBuilder`를 사용하도록 변환해 보겠습니다. 시작하기 전에, `ConfigModule`이 받을 옵션을 나타내는 전용 인터페이스를 생성해야 합니다.

```typescript
export interface ConfigModuleOptions {
  folder: string;
}
```

이제, 기존 `config.module.ts` 파일 옆에 새 전용 파일을 생성하고 `config.module-definition.ts`로 이름을 지정합니다. 이 파일에서 `ConfigurableModuleBuilder`를 사용하여 `ConfigModule` 정의를 구성해 보겠습니다.

```typescript
import { ConfigurableModuleBuilder } from '@nestjs/common';
import { ConfigModuleOptions } from './interfaces/config-module-options.interface';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
```

이제 `config.module.ts` 파일을 열어 `ConfigurableModuleClass`를 활용하도록 구현을 수정해 보겠습니다.

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {}
```

`ConfigurableModuleClass`를 확장함으로써 `ConfigModule`은 이제 이전의 사용자 정의 구현처럼 `register` 메서드를 제공할 뿐만 아니라 `registerAsync` 메서드도 제공하게 됩니다. 이는 소비자가 비동기적으로 모듈을 구성할 수 있게 하며, 예를 들어 비동기 팩토리를 제공할 수 있습니다.

```typescript
@Module({
  imports: [
    ConfigModule.register({ folder: './config' }),
    // 또는 다음과 같이:
    // ConfigModule.registerAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...any extra dependencies...]
    // }),
  ],
})
export class AppModule {}
```

마지막으로 `ConfigService` 클래스를 업데이트하여 현재까지 사용한 `'CONFIG_OPTIONS'` 대신 생성된 module 옵션의 provider를 주입합니다.

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) { ... }
}
```

### 사용자 정의 메서드 키

기본적으로 `ConfigurableModuleClass`는 `register` 및 그 대응 메서드 `registerAsync`를 제공합니다. 다른 메서드 이름을 사용하려면 `ConfigurableModuleBuilder#setClassMethodName` 메서드를 사용합니다.

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setClassMethodName('forRoot').build();
```

이 구성은 `ConfigurableModuleBuilder`에 `forRoot` 및 `forRootAsync` 메서드를 노출하도록 지시합니다. 예:

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ folder: './config' }), // <-- "register" 대신 "forRoot" 사용
    // 또는 다음과 같이:
    // ConfigModule.forRootAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...any extra dependencies...]
    // }),
  ],
})
export class AppModule {}
```

### 사용자 정의 옵션 팩토리 클래스

`registerAsync` 메서드(`forRootAsync` 또는 구성에 따라 다른 이름)를 사용하면 소비자가 module 구성을 해결할 provider 정의를 전달할 수 있습니다. 라이브러리 소비자는 구성 객체를 구성하는 데 사용할 클래스를 제공할 수 있습니다.

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory,
    }),
  ],
})
export class AppModule {}
```

이 클래스는 기본적으로 module 구성 객체를 반환하는 `create()` 메서드를 제공해야 합니다. 그러나 라이브러리가 다른 명명 규칙을 따르는 경우, `ConfigurableModuleBuilder`에 다른 메서드를 기대하도록 지시할 수 있습니다. 예를 들어, `ConfigurableModuleBuilder#setFactoryMethodName` 메서드를 사용하여 `createConfigOptions`를 기대하도록 설정할 수 있습니다.

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setFactoryMethodName('createConfigOptions').build();
```

이제 `ConfigModuleOptionsFactory` 클래스는 `createConfigOptions` 메서드를 노출해야 합니다(`create` 대신).

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory, // <-- 이 클래스는 "createConfigOptions" 메서드를 제공해야 합니다.
    }),
  ],
})
export class AppModule {}
```

### 추가 옵션

모듈이 특정 방식으로 동작해야 하는 추가 옵션을 필요로 하는 경우가 있습니다(예: `isGlobal` 플래그). 이러한 옵션은 module 옵션 객체에 포함되지 않아야 합니다(예: `ConfigService`는 모듈이 글로벌 모듈로 등록되었는지 여부를 알 필요가 없습니다).

이런 경우, `ConfigurableModuleBuilder#setExtras` 메서드를 사용할 수 있습니다. 다음 예제를 참조하십시오.

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } = new ConfigurableModuleBuilder<ConfigModuleOptions>()
  .setExtras(
    {
      isGlobal: true,
    },
    (definition, extras) => ({
      ...definition,
      global: extras.isGlobal,
    }),
  )
  .build();
```

위 예제에서 `setExtras` 메서드에 전달된 첫 번째 인수는 "추가" 속성의 기본 값을 포함하는 객체입니다. 두 번째 인수는 자동 생성된 module 정의(`provider`, `exports` 등)와 소비자가 지정한 또는 기본 값을 나타내는 `extras` 객체를 취하는 함수입니다. 이 함수의 반환 값은 수정된 module 정의입니다. 이 특정 예제에서는 `extras.isGlobal` 속성을 가져와 module 정의의 `global` 속성에 할당하고 있습니다(이는 모듈이 글로벌인지 여부를 결정합니다. 자세한 내용은 [여기](https://docs.nestjs.com/modules#dynamic-modules)에서 확인할 수 있습니다).

이제 이 모듈을 사용할 때, 추가 `isGlobal` 플래그를 다음과 같이 전달할 수 있습니다.

```typescript
@Module({
  imports: [
    ConfigModule.register({
      isGlobal: true,
      folder: './config',
    }),
  ],
})
export class AppModule {}
```

그러나 `isGlobal`이 "추가" 속성으로 선언되었기 때문에 `MODULE_OPTIONS_TOKEN` provider에는 포함되지 않습니다.

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) {
    // "options" 객체에는 "isGlobal" 속성이 포함되지 않습니다.
    // ...
  }
}
```

### 자동 생성된 메서드 확장

자동 생성된 정적 메서드(`register`, `registerAsync` 등)는 필요한 경우 다음과 같이 확장할 수 있습니다.

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass, ASYNC_OPTIONS_TYPE, OPTIONS_TYPE } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {
  static register(options: typeof OPTIONS_TYPE): DynamicModule {
    return {
      // 사용자 정의 로직
      ...super.register(options),
    };
  }

  static registerAsync(options: typeof ASYNC_OPTIONS_TYPE): DynamicModule {
    return {
      // 사용자 정의 로직
      ...super.registerAsync(options),
    };
  }
}
```

module 정의 파일에서 내보내야 하는 `OPTIONS_TYPE` 및 `ASYNC_OPTIONS_TYPE` 유형을 사용하는 것에 주목하십시오.

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN, OPTIONS_TYPE, ASYNC_OPTIONS_TYPE } = new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
```
