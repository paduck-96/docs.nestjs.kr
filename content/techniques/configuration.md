### Configuration

애플리케이션은 종종 **다양한 환경**에서 실행됩니다. 환경에 따라 다른 설정을 사용해야 합니다. 예를 들어, 로컬 환경에서는 로컬 DB 인스턴스에만 유효한 특정 데이터베이스 자격 증명을 사용하지만, 프로덕션 환경에서는 별도의 DB 자격 증명을 사용합니다. 구성 변수가 변경되기 때문에, 가장 좋은 방법은 [구성 변수를 환경에 저장](https://12factor.net/config)하는 것입니다.

외부에서 정의된 환경 변수는 `process.env` 글로벌을 통해 Node.js 내부에서 볼 수 있습니다. 여러 환경 문제를 해결하기 위해 각 환경에서 별도로 환경 변수를 설정할 수 있습니다. 그러나 이는 개발 및 테스트 환경에서 이러한 값을 쉽게 모킹하고/또는 변경해야 하는 경우에 빠르게 불편해질 수 있습니다.

Node.js 애플리케이션에서는 각 환경을 나타내는 `.env` 파일을 사용하여 키-값 쌍을 보유하는 것이 일반적입니다. 그런 다음 올바른 `.env` 파일을 교체하는 것만으로 다른 환경에서 애플리케이션을 실행할 수 있습니다.

이 기술을 Nest에서 사용하기 위한 좋은 접근 방식은 적절한 `.env` 파일을 로드하는 `ConfigService`를 노출하는 `ConfigModule`을 만드는 것입니다. 이러한 모듈을 직접 작성할 수도 있지만, 편의를 위해 Nest는 `@nestjs/config` 패키지를 기본적으로 제공합니다. 이번 장에서는 이 패키지를 다룹니다.

#### 설치

시작하려면 먼저 필요한 종속성을 설치합니다.

```bash
$ npm i --save @nestjs/config
```

> **Hint** `@nestjs/config` 패키지는 내부적으로 [dotenv](https://github.com/motdotla/dotenv)를 사용합니다.

> **Note** `@nestjs/config`는 TypeScript 4.1 이상이 필요합니다.

#### 시작하기

설치가 완료되면 `ConfigModule`을 임포트할 수 있습니다. 일반적으로 루트 `AppModule`에 이를 임포트하고 `.forRoot()` 정적 메서드를 사용하여 동작을 제어합니다. 이 단계에서 환경 변수 키/값 쌍이 구문 분석되고 해결됩니다. 나중에 다른 기능 모듈에서 `ConfigModule`의 `ConfigService` 클래스를 액세스하는 여러 옵션을 살펴보겠습니다.

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot()],
})
export class AppModule {}
```

위 코드는 기본 위치(프로젝트 루트 디렉터리)에서 `.env` 파일을 로드하고 구문 분석하며, `.env` 파일의 키/값 쌍을 `process.env`에 할당된 환경 변수와 병합하고, 결과를 `ConfigService`를 통해 액세스할 수 있는 개인 구조에 저장합니다. `forRoot()` 메서드는 `ConfigService` 프로바이더를 등록하며, 이는 이러한 구문 분석/병합된 구성 변수를 읽기 위한 `get()` 메서드를 제공합니다. `@nestjs/config`는 [dotenv](https://github.com/motdotla/dotenv)에 의존하므로, 환경 변수 이름의 충돌을 해결하기 위한 해당 패키지의 규칙을 사용합니다. 런타임 환경 변수(`export DATABASE_USER=test`와 같은 OS 쉘 내보내기 등)와 `.env` 파일 모두에 키가 존재하는 경우, 런타임 환경 변수가 우선합니다.

샘플 `.env` 파일은 다음과 같습니다:

```ini
DATABASE_USER=test
DATABASE_PASSWORD=test
```

#### 커스텀 env 파일 경로

기본적으로 이 패키지는 애플리케이션의 루트 디렉터리에서 `.env` 파일을 찾습니다. 다른 경로의 `.env` 파일을 지정하려면 `forRoot()`에 전달되는 옵션 객체의 `envFilePath` 속성을 설정합니다:

```typescript
ConfigModule.forRoot({
  envFilePath: '.development.env',
});
```

또한 다음과 같이 여러 경로를 지정할 수도 있습니다:

```typescript
ConfigModule.forRoot({
  envFilePath: ['.env.development.local', '.env.development'],
});
```

여러 파일에서 변수를 찾을 경우, 첫 번째 파일이 우선합니다.

#### env 변수 로딩 비활성화

`.env` 파일을 로드하지 않고 런타임 환경의 환경 변수(OS 쉘 내보내기와 같은)를 단순히 액세스하려는 경우, 옵션 객체의 `ignoreEnvFile` 속성을 `true`로 설정합니다:

```typescript
ConfigModule.forRoot({
  ignoreEnvFile: true,
});
```

#### 모듈을 전역으로 사용

다른 모듈에서 `ConfigModule`을 사용하려면 이를 임포트해야 합니다(일반적인 Nest 모듈과 동일). 또는 옵션 객체의 `isGlobal` 속성을 `true`로 설정하여 [전역 모듈](https://docs.nestjs.com/modules#global-modules)로 선언할 수 있습니다. 이 경우, 루트 모듈(`AppModule` 등)에서 로드된 후 다른 모듈에서 `ConfigModule`을 임포트할 필요가 없습니다.

```typescript
ConfigModule.forRoot({
  isGlobal: true,
});
```

#### 커스텀 구성 파일

더 복잡한 프로젝트의 경우, 중첩된 구성 객체를 반환하는 커스텀 구성 파일을 사용할 수 있습니다. 이를 통해 관련된 구성 설정을 기능별로 그룹화하고, 관련 설정을 개별 파일에 저장하여 독립적으로 관리할 수 있습니다.

커스텀 구성 파일은 구성 객체를 반환하는 팩토리 함수를 내보냅니다. 구성 객체는 임의로 중첩된 일반 JavaScript 객체일 수 있습니다. `process.env` 객체는 환경 변수 키/값 쌍(위에서 설명한 대로 `.env` 파일 및 외부 정의 변수가 해결되고 병합된 값)을 포함합니다. 반환된 구성 객체를 제어할 수 있으므로, 필요한 논리를 추가하여 값을 적절한 타입으로 캐스팅하거나 기본값을 설정할 수 있습니다. 예를 들어:

```typescript
export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT, 10) || 5432
  }
});
```

이 파일을 `forRoot()` 메서드의 옵션 객체의 `load` 속성을 사용하여 로드합니다:

```typescript
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration],
    }),
  ],
})
export class AppModule {}
```

> **Notice** `load` 속성에 할당된 값은 배열이므로 여러 구성 파일을 로드할 수 있습니다(예: `load: [databaseConfig, authConfig]`).

커스텀 구성 파일을 사용하면 YAML 파일과 같은 커스텀 파일도 관리할 수 있습니다. YAML 형식을 사용하는 구성의 예는 다음과 같습니다:

```yaml
http:
  host: 'localhost'
  port: 8080

db:
  postgres:
    url: 'localhost'
    port: 5432
    database: 'yaml-db'

  sqlite:
    database: 'sqlite.db'
```

YAML 파일을 읽고 구문 분석하려면 `js-yaml` 패키지를 사용할 수 있습니다.

```bash
$ npm i js-yaml
$ npm i -D @types/js-yaml
```

패키지를 설치한 후, `yaml#load` 함수를 사용하여 위에서 만든 YAML 파일을 로드합니다.

```typescript
import { readFileSync } from 'fs';
import * as yaml from 'js-yaml';
import { join } from 'path';

const YAML_CONFIG_FILENAME = 'config.yaml';

export default () => {
  return yaml.load(
    readFileSync(join(__dirname, YAML_CONFIG_FILENAME), 'utf8'),
  ) as Record<string, any>;
};
```

> **Note** Nest CLI는 빌드 프로세스 동안 "자산"(비-TS 파일)을 자동으로 `dist` 폴더로 이동하지 않습니다. YAML 파일이 복사되도록 하려면, `nest-cli.json` 파일의 `compilerOptions#assets` 객체에 이를 지정해야 합니다. 예를 들어, `config` 폴더가 `src` 폴더와 동일한 레벨에 있는 경우, `compilerOptions#assets`에 `"assets": [{{ '{' }}"include": "../config/*.yaml", "outDir": "./dist/config"{{ '}' }}]` 값을 추가하십시오. 자세한 내용은 [여기](https://docs.nestjs.com/cli/monorepo#assets)에서 확인하십시오.

#### `ConfigService` 사용하기

`ConfigService`에서 구성 값을 액세스하려면 먼저 `ConfigService`를 주입해야 합니다. 모든 프로바이더와 마찬가지로, 사용하려는 모듈에 해당 모듈(`ConfigModule`)을 임포트해야 합니다(옵션 객체의 `isGlobal` 속성을 `true`로 설정하지 않은 경우). 다음과 같이 기능 모듈에 이를 임포트합니다.

```typescript
@Module({
  imports: [ConfigModule],
  // ...
})
```

그런 다음 표준 생성자 주입을 사용하여 주입

할 수 있습니다:

```typescript
constructor(private configService: ConfigService) {}
```

> **Hint** `ConfigService`는 `@nestjs/config` 패키지에서 임포트합니다.

그리고 클래스에서 이를 사용합니다:

```typescript
// 환경 변수 가져오기
const dbUser = this.configService.get<string>('DATABASE_USER');

// 커스텀 구성 값 가져오기
const dbHost = this.configService.get<string>('database.host');
```

위에서 보듯이, 변수 이름을 전달하여 간단한 환경 변수를 `configService.get()` 메서드를 사용해 가져올 수 있습니다. 위와 같이 타입스크립트 타입 힌팅을 위해 타입을 전달할 수 있습니다(예: `get<string>(...)`). `get()` 메서드는 중첩된 커스텀 구성 객체(커스텀 구성 파일을 통해 생성된)를 탐색할 수도 있습니다.

인터페이스를 타입 힌트로 사용하여 전체 중첩된 커스텀 구성 객체를 가져올 수도 있습니다:

```typescript
interface DatabaseConfig {
  host: string;
  port: number;
}

const dbConfig = this.configService.get<DatabaseConfig>('database');

// 이제 `dbConfig.port` 및 `dbConfig.host`를 사용할 수 있습니다.
const port = dbConfig.port;
```

`get()` 메서드는 키가 존재하지 않을 때 반환할 기본 값을 정의하는 선택적 두 번째 인수를 받을 수도 있습니다:

```typescript
// "database.host"가 정의되지 않은 경우 "localhost"를 사용
const dbHost = this.configService.get<string>('database.host', 'localhost');
```

`ConfigService`는 두 가지 선택적 제너릭(타입 인수)을 가지고 있습니다. 첫 번째는 존재하지 않는 구성 속성에 접근하는 것을 방지하는 데 도움이 됩니다. 이를 사용하려면 다음과 같이 합니다:

```typescript
interface EnvironmentVariables {
  PORT: number;
  TIMEOUT: string;
}

// 코드의 어느 부분에서
constructor(private configService: ConfigService<EnvironmentVariables>) {
  const port = this.configService.get('PORT', { infer: true });

  // 타입스크립트 오류: URL 속성이 EnvironmentVariables에 정의되지 않았으므로 잘못된 접근입니다.
  const url = this.configService.get('URL', { infer: true });
}
```

`infer` 속성을 `true`로 설정하면, `ConfigService.get` 메서드는 인터페이스를 기반으로 속성 타입을 자동으로 추론하므로, 예를 들어, `typeof port === "number"` (타입스크립트의 `strictNullChecks` 플래그를 사용하지 않는 경우)입니다.

또한, `infer` 기능을 사용하면 점 표기법을 사용할 때도 중첩된 커스텀 구성 객체의 속성 타입을 추론할 수 있습니다:

```typescript
constructor(private configService: ConfigService<{ database: { host: string } }>) {
  const dbHost = this.configService.get('database.host', { infer: true })!;
  // typeof dbHost === "string"
  // 비-널 단언 연산자
}
```

두 번째 제너릭은 첫 번째 제너릭에 의존하여 `strictNullChecks`가 켜져 있을 때 `ConfigService`의 메서드가 반환할 수 있는 모든 `undefined` 타입을 제거하는 타입 단언으로 작동합니다. 예를 들어:

```typescript
// ...
constructor(private configService: ConfigService<{ PORT: number }, true>) {
  const port = this.configService.get('PORT', { infer: true });
  // 포트의 타입은 'number'가 되므로 더 이상 타입 단언이 필요하지 않습니다.
}
```

#### 구성 네임스페이스

`ConfigModule`을 사용하면 위에서 설명한 대로 여러 커스텀 구성 파일을 정의하고 로드할 수 있습니다. 이를 통해 중첩된 구성 객체로 복잡한 구성 객체 계층을 관리할 수 있습니다. 또는 `registerAs()` 함수를 사용하여 "네임스페이스화된" 구성 객체를 반환할 수 있습니다:

```typescript
export default registerAs('database', () => ({
  host: process.env.DATABASE_HOST,
  port: process.env.DATABASE_PORT || 5432
}));
```

커스텀 구성 파일과 마찬가지로, `registerAs()` 팩토리 함수 내에서 `process.env` 객체는 환경 변수 키/값 쌍(위에서 설명한 대로 `.env` 파일 및 외부 정의 변수가 해결되고 병합된 값)을 포함합니다.

> **Hint** `registerAs` 함수는 `@nestjs/config` 패키지에서 내보냅니다.

네임스페이스화된 구성을 `forRoot()` 메서드의 옵션 객체의 `load` 속성을 사용하여 로드합니다:

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [databaseConfig],
    }),
  ],
})
export class AppModule {}
```

이제 `database` 네임스페이스에서 `host` 값을 가져오려면 점 표기법을 사용합니다. 네임스페이스의 이름(`registerAs()` 함수의 첫 번째 인수로 전달된)과 일치하는 `'database'`를 속성 이름의 접두사로 사용합니다:

```typescript
const dbHost = this.configService.get<string>('database.host');
```

합리적인 대안으로 네임스페이스를 직접 주입할 수 있습니다. 이를 통해 강력한 타이핑의 이점을 얻을 수 있습니다:

```typescript
constructor(
  @Inject(databaseConfig.KEY)
  private dbConfig: ConfigType<typeof databaseConfig>,
) {}
```

> **Hint** `ConfigType`은 `@nestjs/config` 패키지에서 내보냅니다.

#### 환경 변수 캐싱

`process.env`에 접근하는 것이 느릴 수 있으므로, `ConfigService.get` 메서드의 성능을 높이기 위해 `ConfigModule.forRoot()`에 전달된 옵션 객체의 `cache` 속성을 설정할 수 있습니다:

```typescript
ConfigModule.forRoot({
  cache: true,
});
```

#### 부분 등록

지금까지는 루트 모듈(예: `AppModule`)에서 `forRoot()` 메서드를 사용하여 구성 파일을 처리했습니다. 더 복잡한 프로젝트 구조에서 여러 디렉터리에 기능별 구성 파일이 있는 경우, 이러한 파일을 루트 모듈에서 모두 로드하는 대신, `@nestjs/config` 패키지는 **부분 등록**이라는 기능을 제공하여 각 기능 모듈과 관련된 구성 파일만 참조합니다. 다음과 같이 기능 모듈 내에서 `forFeature()` 정적 메서드를 사용하여 이 부분 등록을 수행합니다:

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [ConfigModule.forFeature(databaseConfig)],
})
export class DatabaseModule {}
```

> **Warning** 특정 상황에서는 `onModuleInit()` 훅을 사용하여 부분 등록을 통해 로드된 속성에 접근해야 할 수도 있습니다. 이는 `forFeature()` 메서드가 모듈 초기화 중에 실행되며, 모듈 초기화 순서가 불확정적이기 때문입니다. 구성에 의존하는 값을 생성자에서 액세스하면, 구성에 의존하는 모듈이 아직 초기화되지 않았을 수 있습니다. `onModuleInit()` 메서드는 의존하는 모든 모듈이 초기화된 후에만 실행되므로 이 기술은 안전합니다.

#### 스키마 검증

필수 환경 변수가 제공되지 않거나 특정 검증 규칙을 충족하지 않는 경우 애플리케이션 시작 시 예외를 던지는 것이 표준 관행입니다. `@nestjs/config` 패키지는 이를 수행할 수 있는 두 가지 방법을 제공합니다:

- [Joi](https://github.com/sideway/joi) 내장 검증기. Joi를 사용하면 객체 스키마를 정의하고 이에 대해 JavaScript 객체를 검증할 수 있습니다.
- 환경 변수를 입력으로 받는 커스텀 `validate()` 함수.

Joi를 사용하려면 Joi 패키지를 설치해야 합니다:

```bash
$ npm install --save joi
```

이제 Joi 검증 스키마를 정의하고 이를 `forRoot()` 메서드의 옵션 객체의 `validationSchema` 속성을 통해 전달할 수 있습니다:

```typescript
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().port().default(3000),
      }),
    }),
  ],
})
export class AppModule {}
```

기본적으로 모든 스키마 키는 선택 사항으로 간주됩니다. 여기에서는 환경(`.env` 파일 또는 프로세스 환경)에서 이러한 변수를 제공하지 않으면 `NODE_ENV` 및 `PORT`에 대한 기본 값을 설정합니다. 대신, `required()` 검증 메서드를 사용하여 환경에서 변수(예: `.env` 파일 또는 프로세스 환경)를 제공해야 한다고 요구할 수 있습니다. 이 경우, 환경에서 변수를 제공하지 않으면 검증 단계에서 예외가 발생합니다. 검증 스키마를 구성하는 방법에 대한 자세한 내용은 [Joi 검증 메서드](https://joi.dev/api/?v=17.3.0#example)를 참조하십시오.

기본적으로 알려지지 않은 환경 변수(스키마에 키가 없는 환경 변수)는 허용되며 검증 예외를 발생시키지 않습니다. 기본적으로 모든 검증 오류가 보고됩니다. 이러한 동작을 변경하려면 `validationOptions` 키를 통해 전달되는 옵션 객체를 사용하여 조정할 수 있습니다. 이 옵션 객체는 [Joi 검증 옵션](https://joi.dev/api/?v=17.3.0#anyvalidatevalue-options)에서 제공하는 표준 검증 옵션 속성 중 하나를 포함할 수 있습니다. 예를 들어, 위의 두 가지 설정을 반대로 하려면 다음과 같이 옵션을 전달합니다:

```typescript
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().port().default(3000),
      }),
      validationOptions: {
        allowUnknown: false,
        abortEarly: true,
      },
    }),
  ],
})
export class AppModule {}
```

`@nestjs/config` 패키지는 기본 설정으로 다음을 사용합니다:

- `allowUnknown`: 환경 변수에 알려지지 않은 키를 허용할지 여부를 제어합니다. 기본값은 `true`입니다.
- `abortEarly`: `true`이면 첫 번째 오류에서 검증을 중지하고, `false`이면 모든 오류를 반환합니다. 기본값은 `false`입니다.

`validationOptions` 객체를 전달하기로 결정한 경우, 명시적으로 전달하지 않은 모든 설정은 `@nestjs/config` 기본값이 아닌 `Joi` 표준 기본값으로 설정됩니다. 예를 들어, 커스텀 `validationOptions` 객체에서 `allowUnknowns`를 지정하지 않으면 `Joi` 기본값인 `false`가 됩니다. 따라서 이 설정을 명시적으로 지정하는 것이 가장 안전합니다.

#### 커스텀 validate 함수

대안으로, `.env` 파일 및 프로세스의 환경 변수를 입력으로 받아 필요한 경우 이를 변환/변경하는 객체를 반환하는 **동기** `validate` 함수를 지정할 수 있습니다. 함수가 오류를 던지면 애플리케이션이 부트스트랩되지 않습니다.

이 예제에서는 `class-transformer` 및 `class-validator` 패키지를 사용합니다. 먼저 다음을 정의해야 합니다:

- 검증 제약 조건이 있는 클래스,
- `plainToInstance` 및 `validateSync` 함수를 사용하는 validate 함수.

```typescript
import { plainToInstance } from 'class-transformer';
import { IsEnum, IsNumber, Max, Min, validateSync } from 'class-validator';

enum Environment {
  Development = "development",
  Production = "production",
  Test = "test",
  Provision = "provision",
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsNumber()
  @Min(0)
  @Max(65535)
  PORT: number;
}

export function validate(config: Record<string, unknown>) {
  const validatedConfig = plainToInstance(
    EnvironmentVariables,
    config,
    { enableImplicitConversion: true },
  );
  const errors = validateSync(validatedConfig, { skipMissingProperties: false });

  if (errors.length > 0) {
    throw new Error(errors.toString());
  }
  return validatedConfig;
}
```

이제 `ConfigModule`의 구성 옵션으로 `validate` 함수를 사용합니다:

```typescript
import { validate } from './env.validation';

@Module({
  imports: [
    ConfigModule.forRoot({
      validate,
    }),
  ],
})
export class AppModule {}
```

#### 커스텀 getter 함수

`ConfigService`는 키로 구성 값을 가져오기 위해 `get()` 메서드를 정의합니다. 더 자연스러운 코딩 스타일을 위해 `getter` 함수를 추가할 수도 있습니다:

```typescript
@Injectable()
export class ApiConfigService {
  constructor(private configService: ConfigService) {}

  get isAuthEnabled(): boolean {
    return this.configService.get('AUTH_ENABLED') === 'true';
  }
}
```

이제 getter 함수를 다음과 같이 사용할 수 있습니다:

```typescript
@Injectable()
export class AppService {
  constructor(apiConfigService: ApiConfigService) {
    if (apiConfigService.isAuthEnabled) {
      // 인증이 활성화됨
    }
  }
}
```

#### 환경 변수 로드 후크

모듈 구성이 환경 변수에 의존하고, 이러한 변수가 `.env` 파일에서 로드되는 경우, `.env` 파일이 로드되기 전에 `process.env` 객체와 상호 작용하는 것을 방지하기 위해 `ConfigModule.envVariablesLoaded` 후크를 사용할 수 있습니다. 다음 예제를 참조하십시오:

```typescript
export async function getStorageModule() {
  await ConfigModule.envVariablesLoaded;
  return process.env.STORAGE === 'S3' ? S3StorageModule : DefaultStorageModule;
}
```

이 구성은 `ConfigModule.envVariablesLoaded` 프라미스가 해결된 후에 모든 구성 변수가 로드되었음을 보장합니다.

#### 조건부 모듈 구성

때때로 환경 변수에 조건을 지정하여 모듈을 조건부로 로드하고자 할 때가 있습니다. 다행히도 `@nestjs/config`는 이를 수행할 수 있는 `ConditionalModule`을 제공합니다.

```typescript
@Module({
  imports: [ConfigModule.forRoot(), ConditionalModule.registerWhen(FooModule, 'USE_FOO')],
})
export class AppModule {}
```

위 모듈은 `.env` 파일에 `USE_FOO` 환경 변수가 `false` 값이 아닌 경우에만 `FooModule`을 로드합니다. 또한, `process.env` 참조를 받는 함수와 부울 값을 반환해야 하는 조건을 직접 전달할 수도 있습니다:

```typescript
@Module({
  imports: [ConfigModule.forRoot(), ConditionalModule.registerWhen(FooBarModule, (env: NodeJS.ProcessEnv) => !!env['foo'] && !!env['bar'])],
})
export class AppModule {}
```

`ConditionalModule`을 사용할 때는 `ConfigModule`이 애플리케이션에 로드되어 있어야 하며, `ConfigModule.envVariablesLoaded` 후크를 올바르게 참조하고 사용할 수 있어야 합니다. 후크가 5초(또는 사용자가 `registerWhen` 메서드의 세 번째 옵션 매개변수로 설정한 타임아웃) 내에 true로 전환되지 않으면, `ConditionalModule`은 오류를 발생시키고 Nest는 애플리케이션 시작을 중단합니다.

#### 확장 가능한 변수

`@nestjs/config` 패키지는 환경 변수 확장을 지원합니다. 이 기술을 사용하면 한 변수를 다른 변수 정의 내에서 참조하는 중첩된 환경 변수를 만들 수 있습니다. 예를 들어:

```ini
APP_URL=mywebsite.com
SUPPORT_EMAIL=support@${APP_URL}
```

이 구조를 사용하면 `SUPPORT_EMAIL` 변수는 `'support@mywebsite.com'`으로 해결됩니다. `${{ '{' }}...{{ '}' }}` 구문을 사용하여 `SUPPORT_EMAIL` 정의 내에서 `APP_URL` 변수의 값을 트리거합니다.

> **Hint** 이 기능을 위해 `@nestjs/config` 패키지는 내부적으로 [dotenv-expand](https://github.com/motdotla/dotenv-expand)를 사용합니다.

환경 변수 확장을 활성화하려면, `ConfigModule`의 `forRoot()` 메서드에 전달되는 옵션 객체의 `expandVariables` 속성을 설정합니다:

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({
      // ...
      expandVariables: true,
    }),
  ],
})
export class AppModule {}
```

#### `main.ts`에서 사용하기

구성은 서비스에 저장되어 있지만, 여전히 `main.ts` 파일에서 이를 사용할 수 있습니다. 이렇게 하면 애플리케이션 포트 또는 CORS 호스트와 같은 변수를 저장하는 데 사용할 수 있습니다.

액세스하려면 서비스 참조 뒤에 `app.get()` 메서드를 사용해야 합니다:

```typescript
const configService = app.get(ConfigService);
```

그런 다음 구성 키와 함께 `get` 메서드를 호출하여 일반적으로 사용할 수 있습니다:

```typescript
const port = configService.get('PORT');
```
