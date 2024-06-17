### Logger

Nest에는 애플리케이션 부트스트래핑 및 여러 상황에서 사용할 수 있는 텍스트 기반 로거가 내장되어 있습니다. 이는 `@nestjs/common` 패키지의 `Logger` 클래스를 통해 제공됩니다. 로깅 시스템의 동작을 완전히 제어할 수 있으며, 다음을 포함합니다:

- 로깅을 완전히 비활성화
- 로깅 세부 정보 수준 지정 (예: 오류, 경고, 디버그 정보 등)
- 기본 로거에서 타임스탬프 재정의 (예: ISO8601 표준 사용)
- 기본 로거 완전히 재정의
- 기본 로거를 확장하여 사용자 정의
- 의존성 주입을 사용하여 애플리케이션 구성 및 테스트 단순화

또한 내장된 로거를 사용하거나 사용자 정의 구현을 만들어 애플리케이션 수준 이벤트 및 메시지를 로깅할 수 있습니다.

더 고급 로깅 기능을 위해 [Winston](https://github.com/winstonjs/winston)과 같은 Node.js 로깅 패키지를 사용하여 완전히 사용자 정의된 프로덕션급 로깅 시스템을 구현할 수 있습니다.

#### 기본 사용자 정의

로깅을 비활성화하려면, `NestFactory.create()` 메서드의 두 번째 인수로 전달된 Nest 애플리케이션 옵션 객체에서 `logger` 속성을 `false`로 설정합니다.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: false,
});
await app.listen(3000);
```

특정 로깅 수준을 활성화하려면, `logger` 속성을 표시할 로그 수준을 지정하는 문자열 배열로 설정합니다.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: ['error', 'warn'],
});
await app.listen(3000);
```

배열의 값은 `'log'`, `'fatal'`, `'error'`, `'warn'`, `'debug'`, `'verbose'`의 조합일 수 있습니다.

> info **힌트** 기본 로거의 메시지에서 색상을 비활성화하려면 `NO_COLOR` 환경 변수를 비어 있지 않은 문자열로 설정합니다.

#### 사용자 정의 구현

Nest가 시스템 로깅에 사용할 사용자 정의 로거 구현을 제공하려면, `logger` 속성 값을 `LoggerService` 인터페이스를 구현하는 객체로 설정합니다. 예를 들어, 내장된 전역 JavaScript `console` 객체를 사용할 수 있습니다.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: console,
});
await app.listen(3000);
```

자체 사용자 정의 로거를 구현하는 것은 간단합니다. 아래와 같이 `LoggerService` 인터페이스의 각 메서드를 구현하기만 하면 됩니다.

```typescript
import { LoggerService, Injectable } from '@nestjs/common';

@Injectable()
export class MyLogger implements LoggerService {
  /**
   * 'log' 수준의 로그 작성
   */
  log(message: any, ...optionalParams: any[]) {}

  /**
   * 'fatal' 수준의 로그 작성
   */
  fatal(message: any, ...optionalParams: any[]) {}

  /**
   * 'error' 수준의 로그 작성
   */
  error(message: any, ...optionalParams: any[]) {}

  /**
   * 'warn' 수준의 로그 작성
   */
  warn(message: any, ...optionalParams: any[]) {}

  /**
   * 'debug' 수준의 로그 작성
   */
  debug?(message: any, ...optionalParams: any[]) {}

  /**
   * 'verbose' 수준의 로그 작성
   */
  verbose?(message: any, ...optionalParams: any[]) {}
}
```

그런 다음 `Nest` 애플리케이션 옵션 객체의 `logger` 속성을 통해 `MyLogger`의 인스턴스를 제공합니다.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new MyLogger(),
});
await app.listen(3000);
```

이 기술은 간단하지만 `MyLogger` 클래스에 의존성 주입을 사용하지 않습니다. 이는 특히 테스트 시 일부 문제를 일으킬 수 있으며 `MyLogger`의 재사용성을 제한할 수 있습니다. 더 나은 솔루션을 위해 아래의 <a href="https://docs.nestjs.com/techniques/logger#dependency-injection">의존성 주입</a> 섹션을 참조하세요.

#### 내장 로거 확장

로거를 처음부터 작성하는 대신, 내장 `ConsoleLogger` 클래스를 확장하고 기본 구현의 선택된 동작을 재정의하여 요구 사항을 충족할 수 있습니다.

```typescript
import { ConsoleLogger } from '@nestjs/common';

export class MyLogger extends ConsoleLogger {
  error(message: any, stack?: string, context?: string) {
    // 맞춤 로직 추가
    super.error(...arguments);
  }
}
```

이와 같이 확장된 로거를 기능 모듈에서 사용할 수 있습니다. 자세한 내용은 <a href="https://docs.nestjs.com/techniques/logger#using-the-logger-for-application-logging">애플리케이션 로깅에 로거 사용</a> 섹션을 참조하세요.

사용자 정의 로거를 시스템 로깅에 사용하도록 Nest에 지시하려면, 위의 <a href="https://docs.nestjs.com/techniques/logger#custom-logger-implementation">사용자 정의 구현</a> 섹션에서 설명한 대로 애플리케이션 옵션 객체의 `logger` 속성을 통해 인스턴스를 제공하거나 아래의 <a href="https://docs.nestjs.com/techniques/logger#dependency-injection">의존성 주입</a> 섹션에서 설명한 기술을 사용합니다. 이렇게 할 경우, Nest가 기대하는 기본 기능에 의존할 수 있도록 특정 로그 메서드 호출을 부모(내장) 클래스에 위임하기 위해 `super`를 호출해야 합니다.

#### 의존성 주입

더 고급 로깅 기능을 위해 의존성 주입을 활용해야 합니다. 예를 들어, 로거를 사용자 정의하기 위해 `ConfigService`를 로거에 주입하고, 사용자 정의 로거를 다른 컨트롤러 및/또는 프로바이더에 주입할 수 있습니다. 사용자 정의 로거에 의존성 주입을 사용하려면, `LoggerService`를 구현하는 클래스를 생성하고 해당 클래스를 모듈에서 프로바이더로 등록합니다. 예를 들어 다음과 같이 할 수 있습니다.

1. 기본 `ConsoleLogger`를 확장하거나 완전히 재정의하는 `MyLogger` 클래스를 정의합니다. `LoggerService` 인터페이스를 구현해야 합니다.
2. 다음과 같이 `MyLogger`를 제공하는 `LoggerModule`을 만듭니다.

```typescript
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```

이 구조로 다른 모듈에서 사용자 정의 로거를 사용할 수 있습니다. `MyLogger` 클래스가 모듈의 일부이므로 의존성 주입을 사용할 수 있습니다 (예: `ConfigService`를 주입). Nest가 시스템 로깅(예: 부트스트래핑 및 오류 처리)에 사용자 정의 로거를 사용하도록 지시하려면 한 가지 기술이 더 필요합니다.

애플리케이션 인스턴스화(`NestFactory.create()`)는 모듈의 컨텍스트 외부에서 발생하므로 초기화 단계의 일반적인 의존성 주입에 참여하지 않습니다. 따라서 최소한 하나의 애플리케이션 모듈이 `LoggerModule`을 가져와서 `MyLogger` 클래스의 싱글톤 인스턴스를 인스턴스화하도록 Nest에 트리거해야 합니다.

그런 다음 다음과 같은 구성을 사용하여 Nest가 동일한 싱글톤 인스턴스를 사용하도록 지시할 수 있습니다:

```typescript
const app = await NestFactory.create(AppModule, {
  bufferLogs: true,
});
app.useLogger(app.get(MyLogger));
await app.listen(3000);
```

> info **참고** 위 예제에서 `bufferLogs`를 `true`로 설정하여 모든 로그가 사용자 정의 로거(`이 경우 MyLogger`)가 연결될 때까지 버퍼링되도록 하고 애플리케이션 초기화 프로세스가 완료되거나 실패할 때까지 버퍼링되도록 합니다. 초기화 프로세스가 실패하면 Nest는 기본 `ConsoleLogger`로 복귀하여 보고된 오류 메시지를 출력합니다. 또한, `autoFlushLogs`를 `false`(기본값은 `true`)로 설정하여 로그를 수동으로 플러시할 수 있습니다 (`Logger#flush()` 메서드를 사용하여).

여기서 `NestApplication` 인스턴스에서 `get()` 메서드를 사용하여 `MyLogger` 객체의 싱글톤 인스턴스를 가져옵니다. 이 기술은 본질적으로 Nest가 사용하는 로거 인스턴스를 "주입"하는 방법입니다. `app.get()` 호출은 `MyLogger`의 싱글톤 인스턴스를 검색하며, 이는 위에서 설명한 대로 다른 모듈에서 처음 주입될 때 생성됩니다.

이 `MyLogger` 프로바이더를 기능 클래스에 주입하여 Nest 시스템 로깅 및 애플리케이션

 로깅 전반에 걸쳐 일관된 로깅 동작을 보장할 수 있습니다. 자세한 내용은 <a href="https://docs.nestjs.com/techniques/logger#using-the-logger-for-application-logging">애플리케이션 로깅에 로거 사용</a> 및 <a href="https://docs.nestjs.com/techniques/logger#injecting-a-custom-logger">사용자 정의 로거 주입</a>을 참조하세요.

#### 애플리케이션 로깅에 로거 사용

위의 여러 기술을 결합하여 Nest 시스템 로깅과 애플리케이션 이벤트/메시지 로깅 전반에 걸쳐 일관된 동작 및 형식을 제공할 수 있습니다.

좋은 실천 방법은 각 서비스에서 `@nestjs/common`의 `Logger` 클래스를 인스턴스화하는 것입니다. 서비스 이름을 `Logger` 생성자의 `context` 인수로 제공할 수 있습니다.

```typescript
import { Logger, Injectable } from '@nestjs/common';

@Injectable()
class MyService {
  private readonly logger = new Logger(MyService.name);

  doSomething() {
    this.logger.log('무언가를 수행 중...');
  }
}
```

기본 로거 구현에서는 `context`가 대괄호 안에 출력됩니다. 예를 들어, 아래의 `NestFactory`와 같이 출력됩니다:

```bash
[Nest] 19096   - 12/08/2019, 7:12:59 AM   [NestFactory] Starting Nest application...
```

사용자 정의 로거를 `app.useLogger()`를 통해 제공하면 Nest 내부에서 실제로 사용됩니다. 즉, 코드는 구현에 독립적이며, `app.useLogger()`를 호출하여 기본 로거를 사용자 정의 로거로 쉽게 대체할 수 있습니다.

이렇게 하면 이전 섹션의 단계를 따르고 `app.useLogger(app.get(MyLogger))`를 호출하면, `MyService`의 `this.logger.log()` 호출이 `MyLogger` 인스턴스의 `log` 메서드 호출로 이어집니다.

이 방법은 대부분의 경우에 적합합니다. 그러나 더 많은 사용자 정의가 필요한 경우(예: 사용자 정의 메서드 추가 및 호출) 다음 섹션으로 이동하세요.

#### 사용자 정의 로거 주입

시작하려면 다음과 같이 기본 로거를 확장합니다. 이 예에서는 `ConsoleLogger` 메서드(예: `log()`, `warn()` 등)를 확장하지 않지만, 필요에 따라 이를 확장할 수 있습니다.

```typescript
import { Injectable, Scope, ConsoleLogger } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class MyLogger extends ConsoleLogger {
  customLog() {
    this.log('고양이에게 먹이를 주세요!');
  }
}
```

그런 다음 다음과 같은 `LoggerModule`을 만듭니다:

```typescript
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```

그런 다음 기능 모듈에서 `LoggerModule`을 가져옵니다. 기본 `Logger`를 확장했으므로 `setContext` 메서드를 사용하는 편리함이 있습니다. 이렇게 하면 컨텍스트 인식 사용자 정의 로거를 사용할 수 있습니다:

```typescript
import { Injectable } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  constructor(private myLogger: MyLogger) {
    // 일시적 범위 덕분에 CatsService는 MyLogger의 고유한 인스턴스를 가지며,
    // 여기서 컨텍스트를 설정해도 다른 서비스의 인스턴스에 영향을 미치지 않습니다.
    this.myLogger.setContext('CatsService');
  }

  findAll(): Cat[] {
    // 기본 메서드를 모두 호출할 수 있습니다.
    this.myLogger.warn('고양이를 반환하려고 합니다!');
    // 사용자 정의 메서드도 호출할 수 있습니다.
    this.myLogger.customLog();
    return this.cats;
  }
}
```

마지막으로, 다음과 같이 `main.ts` 파일에서 사용자 정의 로거의 인스턴스를 사용하도록 Nest에 지시합니다. 물론 이 예제에서는 실제로 로거 동작을 사용자 정의하지 않았으므로(예: `Logger` 메서드인 `log()`, `warn()` 등을 확장하지 않았으므로) 이 단계는 실제로 필요하지 않습니다. 그러나 해당 메서드에 사용자 정의 로직을 추가하고 Nest가 동일한 구현을 사용하도록 하려면 이 단계가 필요합니다.

```typescript
const app = await NestFactory.create(AppModule, {
  bufferLogs: true,
});
app.useLogger(new MyLogger());
await app.listen(3000);
```

> info **힌트** 대안으로 `bufferLogs`를 `true`로 설정하는 대신 로거를 일시적으로 비활성화할 수 있습니다. `NestFactory.create`에 `logger: false`를 제공하면 `useLogger`를 호출할 때까지 아무것도 로깅되지 않으므로 일부 중요한 초기화 오류를 놓칠 수 있습니다. 초기 메시지의 일부가 기본 로거로 로깅되는 것이 괜찮다면 `logger: false` 옵션을 생략할 수 있습니다.

#### 외부 로거 사용

프로덕션 애플리케이션은 종종 고급 필터링, 형식 지정 및 중앙 집중식 로깅을 포함한 특정 로깅 요구 사항을 가지고 있습니다. Nest의 내장 로거는 Nest 시스템 동작 모니터링에 사용되며, 개발 중 기능 모듈에서 기본 형식 지정 텍스트 로깅에 유용할 수 있지만, 프로덕션 애플리케이션은 종종 [Winston](https://github.com/winstonjs/winston)과 같은 전용 로깅 모듈을 활용합니다. 모든 표준 Node.js 애플리케이션과 마찬가지로 Nest에서도 이러한 모듈을 완전히 활용할 수 있습니다.
