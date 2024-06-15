### 라이프사이클 이벤트

Nest 애플리케이션 및 모든 애플리케이션 요소는 Nest에 의해 관리되는 라이프사이클을 가지고 있습니다. Nest는 주요 라이프사이클 이벤트에 대한 가시성을 제공하고, 이러한 이벤트가 발생할 때 실행할 수 있는 코드를 모듈, provider 또는 controller에 등록할 수 있는 **라이프사이클 후크**를 제공합니다.

#### 라이프사이클 순서

다음 다이어그램은 애플리케이션이 부트스트랩된 시점부터 노드 프로세스가 종료될 때까지의 주요 애플리케이션 라이프사이클 이벤트의 순서를 나타냅니다. 전체 라이프사이클을 **초기화**, **실행**, **종료**의 세 단계로 나눌 수 있습니다. 이 라이프사이클을 사용하여 모듈 및 서비스의 적절한 초기화를 계획하고, 활성 연결을 관리하며, 종료 신호를 받을 때 애플리케이션을 우아하게 종료할 수 있습니다.

<figure><img src="https://docs.nestjs.com/assets/lifecycle-events.png" /></figure>

#### 라이프사이클 이벤트

라이프사이클 이벤트는 애플리케이션 부트스트래핑 및 종료 중에 발생합니다. Nest는 각 라이프사이클 이벤트마다 모듈, provider 및 controller에서 등록된 라이프사이클 후크 메서드를 호출합니다(아래 [애플리케이션 종료](https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown)에서 설명한 대로 **종료 후크**를 먼저 활성화해야 합니다). 위 다이어그램에서 보듯이, Nest는 연결을 시작하고 중지하기 위해 적절한 기본 메서드도 호출합니다.

다음 표에서 `onModuleDestroy`, `beforeApplicationShutdown` 및 `onApplicationShutdown`은 명시적으로 `app.close()`를 호출하거나 프로세스가 특별한 시스템 신호(예: SIGTERM)를 받았고 애플리케이션 부트스트랩 시 `enableShutdownHooks`를 올바르게 호출한 경우에만 트리거됩니다(아래 **애플리케이션 종료** 부분 참조).

| 라이프사이클 후크 메서드         | 라이프사이클 이벤트                                                                                                                            |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `onModuleInit()`                | 호스트 모듈의 종속성이 해결된 후 호출됩니다.                                                                                                   |
| `onApplicationBootstrap()`      | 모든 모듈이 초기화된 후, 그러나 연결을 수신하기 전 호출됩니다.                                                                                 |
| `onModuleDestroy()`\*           | 종료 신호(예: `SIGTERM`)를 받은 후 호출됩니다.                                                                                                 |
| `beforeApplicationShutdown()`\* | 모든 `onModuleDestroy()` 핸들러가 완료된 후(Promises가 해결되거나 거부됨);<br />완료되면(Promises가 해결되거나 거부됨), 모든 기존 연결이 닫히고(`app.close()` 호출됨). |
| `onApplicationShutdown()`\*     | 연결이 닫힌 후(`app.close()`가 해결됨) 호출됩니다.                                                                                             |

\* 이러한 이벤트의 경우, 명시적으로 `app.close()`를 호출하지 않는 한, 시스템 신호(예: SIGTERM)로 작동하도록 선택해야 합니다. 아래 [애플리케이션 종료](https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown)를 참조하세요.

> warning **경고** 위에 나열된 라이프사이클 후크는 **요청 범위** 클래스에는 트리거되지 않습니다. 요청 범위 클래스는 애플리케이션 라이프사이클과 연결되어 있지 않으며 수명이 예측할 수 없습니다. 이러한 클래스는 각 요청에 대해 독점적으로 생성되며 응답이 전송된 후 자동으로 가비지 컬렉션됩니다.

> info **힌트** `onModuleInit()` 및 `onApplicationBootstrap()`의 실행 순서는 모듈 가져오기 순서에 직접적으로 의존하며, 이전 후크를 기다립니다.

#### 사용법

각 라이프사이클 후크는 인터페이스로 표현됩니다. 인터페이스는 TypeScript 컴파일 후 존재하지 않기 때문에 기술적으로는 선택 사항입니다. 그럼에도 불구하고 강력한 타입 지정과 에디터 도구를 활용하기 위해 사용하는 것이 좋습니다. 라이프사이클 후크를 등록하려면 적절한 인터페이스를 구현합니다. 예를 들어, 특정 클래스(예: Controller, Provider 또는 Module)에서 모듈 초기화 중에 호출될 메서드를 등록하려면 `onModuleInit()` 메서드를 제공하여 `OnModuleInit` 인터페이스를 구현합니다. 아래 예제를 참조하세요:

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class UsersService implements OnModuleInit {
  onModuleInit() {
    console.log(`The module has been initialized.`);
  }
}
```

#### 비동기 초기화

`OnModuleInit` 및 `OnApplicationBootstrap` 후크는 애플리케이션 초기화 프로세스를 지연시키도록 허용합니다(메서드를 `async`로 표시하고 메서드 본문에서 비동기 메서드 완료를 `await`하거나 `Promise`를 반환).

```typescript
async onModuleInit(): Promise<void> {
  await this.fetch();
}
```

#### 애플리케이션 종료

`onModuleDestroy()`, `beforeApplicationShutdown()` 및 `onApplicationShutdown()` 후크는 종료 단계에서 호출됩니다(`app.close()`를 명시적으로 호출하거나 SIGTERM과 같은 시스템 신호를 수신한 경우). 이 기능은 컨테이너 수명을 관리하기 위해 [Kubernetes](https://kubernetes.io/)와 함께 자주 사용되며, [Heroku](https://www.heroku.com/)의 dynos 또는 유사한 서비스에 사용됩니다.

종료 후크 리스너는 시스템 리소스를 소비하므로 기본적으로 비활성화되어 있습니다. 종료 후크를 사용하려면 `enableShutdownHooks()`를 호출하여 **리스너를 활성화**해야 합니다:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 종료 후크 수신 대기 시작
  app.enableShutdownHooks();

  await app.listen(3000);
}
bootstrap();
```

> warning **경고** 고유한 플랫폼 제한으로 인해 NestJS는 Windows에서 애플리케이션 종료 후크에 대한 지원이 제한적입니다. `SIGINT`, `SIGBREAK` 및 어느 정도까지 `SIGHUP`은 작동하지만 `SIGTERM`은 Windows에서 절대 작동하지 않습니다. 작업 관리자에서 프로세스를 강제로 종료할 수 있기 때문입니다. `SIGINT`, `SIGBREAK` 등이 Windows에서 어떻게 처리되는지에 대한 자세한 내용은 libuv의 [관련 문서](https://docs.libuv.org/en/v1.x/signal.html)를 참조하십시오. 또한 Node.js의 [Process Signal Events](https://nodejs.org/api/process.html#process_signal_events) 문서를 참조하십시오.

> info **정보** `enableShutdownHooks`는 리스너를 시작하므로 메모리를 소비합니다. Jest를 사용하여 병렬 테스트를 실행할 때와 같이 단일 Node 프로세스에서 여러 Nest 앱을 실행하는 경우, Node는 과도한 리스너 프로세스에 대해 불평할 수 있습니다. 이러한 이유로 `enableShutdownHooks`는 기본적으로 활성화되지 않습니다. 단일 Node 프로세스에서 여러 인스턴스를 실행할 때 이 조건을 유의하십시오.

애플리케이션이 종료 신호를 수신하면 등록된 모든 `onModuleDestroy()`, `beforeApplicationShutdown()`, `onApplicationShutdown()` 메서드(위에 설명된 순서대로)를 해당 신호를 첫 번째 매개변수로 호출합니다. 등록된 함수가 비동기 호출을 대기하면(Promise를 반환하면) Promise가 해결되거나 거부될 때까지 Nest는 순서를 계속하지 않습니다.

```typescript
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal: string) {
    console.log(signal); // 예: "SIGINT"
  }
}
```

> info **정보** `app.close()`를 호출해도 Node 프로세스가 종료되지 않으며, 단지 `onModuleDestroy()` 및 `onApplicationShutdown()` 후크를 트리거합니다. 따라서 일부 간격, 장시간 실행되는 백그라운드 작업 등이 있으면 프로세스가 자동으로 종료되지 않습니다.
