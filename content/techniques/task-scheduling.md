### 작업 스케줄링

작업 스케줄링은 고정된 날짜/시간에, 반복되는 간격으로, 또는 특정 간격 후에 한 번 임의의 코드(메서드/함수)를 실행하도록 예약할 수 있게 해줍니다. 리눅스 환경에서는 주로 [cron](https://en.wikipedia.org/wiki/Cron)과 같은 패키지를 사용하여 OS 수준에서 이 작업을 처리합니다. Node.js 앱에서는 cron과 유사한 기능을 에뮬레이트하는 여러 패키지가 있습니다. Nest는 인기 있는 Node.js [cron](https://github.com/kelektiv/node-cron) 패키지와 통합되는 `@nestjs/schedule` 패키지를 제공합니다. 이 장에서는 이 패키지를 다룹니다.

#### 설치

사용을 시작하려면 필요한 종속성을 설치합니다.

```bash
$ npm install --save @nestjs/schedule
```

작업 스케줄링을 활성화하려면 `ScheduleModule`을 루트 `AppModule`에 가져오고 `forRoot()` 정적 메서드를 다음과 같이 실행합니다:

```typescript
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot()
  ],
})
export class AppModule {}
```

`.forRoot()` 호출은 스케줄러를 초기화하고 앱 내에서 존재하는 선언적 <a href="https://docs.nestjs.com/techniques/task-scheduling#declarative-cron-jobs">cron 작업</a>, <a href="https://docs.nestjs.com/techniques/task-scheduling#declarative-timeouts">타임아웃</a> 및 <a href="https://docs.nestjs.com/techniques/task-scheduling#declarative-intervals">간격</a>을 등록합니다. 등록은 모든 모듈이 로드되고 예약된 작업을 선언한 후 `onApplicationBootstrap` 생명 주기 후크가 발생할 때 이루어집니다.

#### 선언적 cron 작업

cron 작업은 임의의 함수(메서드 호출)를 자동으로 실행하도록 예약합니다. cron 작업은 다음과 같이 실행할 수 있습니다:

- 지정된 날짜/시간에 한 번.
- 반복 간격으로; 반복 작업은 지정된 간격 내의 특정 시점(예: 매 시간 한 번, 매주 한 번, 매 5분마다 한 번)에 실행됩니다.

`@Cron()` 데코레이터를 메서드 정의 앞에 붙여 다음과 같이 cron 작업을 선언합니다:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron('45 * * * * *')
  handleCron() {
    this.logger.debug('현재 초가 45일 때 호출됩니다');
  }
}
```

이 예제에서 `handleCron()` 메서드는 현재 초가 `45`일 때마다 호출됩니다. 즉, 이 메서드는 매 분 45초 마다 한 번 실행됩니다.

`@Cron()` 데코레이터는 모든 표준 [cron 패턴](http://crontab.org/)을 지원합니다:

- 별표 (예: `*`)
- 범위 (예: `1-3,5`)
- 단계 (예: `*/2`)

위 예제에서는 데코레이터에 `45 * * * * *`을 전달했습니다. 다음 키는 cron 패턴 문자열의 각 위치가 어떻게 해석되는지를 보여줍니다:

<pre class="language-javascript"><code class="language-javascript">
* * * * * *
| | | | | |
| | | | | 요일
| | | | 월
| | | 일
| | 시간
| 분
초 (선택 사항)
</code></pre>

몇 가지 샘플 cron 패턴은 다음과 같습니다:

<table>
  <tbody>
    <tr>
      <td><code>* * * * * *</code></td>
      <td>매 초마다</td>
    </tr>
    <tr>
      <td><code>45 * * * * *</code></td>
      <td>매 분마다, 45초에</td>
    </tr>
    <tr>
      <td><code>0 10 * * * *</code></td>
      <td>매 시간마다, 10분에</td>
    </tr>
    <tr>
      <td><code>0 */30 9-17 * * *</code></td>
      <td>오전 9시부터 오후 5시까지 매 30분마다</td>
    </tr>
   <tr>
      <td><code>0 30 11 * * 1-5</code></td>
      <td>월요일부터 금요일까지 오전 11시 30분에</td>
    </tr>
  </tbody>
</table>

`@nestjs/schedule` 패키지는 일반적으로 사용되는 cron 패턴을 포함하는 편리한 열거형을 제공합니다. 이 열거형을 다음과 같이 사용할 수 있습니다:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron(CronExpression.EVERY_30_SECONDS)
  handleCron() {
    this.logger.debug('매 30초마다 호출됩니다');
  }
}
```

이 예제에서는 `handleCron()` 메서드가 매 `30`초마다 호출됩니다.

또한, `@Cron()` 데코레이터에 JavaScript `Date` 객체를 제공할 수 있습니다. 이렇게 하면 지정된 날짜에 한 번만 작업이 실행됩니다.

> info **힌트** 현재 날짜를 기준으로 작업을 예약하려면 JavaScript 날짜 산술을 사용하세요. 예를 들어, 앱 시작 후 10초 후에 작업을 실행하려면 `@Cron(new Date(Date.now() + 10 * 1000))`를 사용합니다.

또한, `@Cron()` 데코레이터에 두 번째 매개 변수로 옵션 객체를 제공할 수 있습니다.

<table>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>선언된 후 cron 작업에 접근하고 제어하는 데 유용합니다.</td>
    </tr>
    <tr>
      <td><code>timeZone</code></td>
      <td>실행할 시간대를 지정합니다. 이것은 실제 시간대를 기준으로 시간을 수정합니다. 시간대가 유효하지 않으면 오류가 발생합니다. 모든 시간대는 <a href="http://momentjs.com/timezone/">Moment Timezone</a> 웹사이트에서 확인할 수 있습니다.</td>
    </tr>
    <tr>
      <td><code>utcOffset</code></td>
      <td><code>timeZone</code> 매개 변수를 사용하지 않고 시간대 오프셋을 지정할 수 있습니다.</td>
    </tr>
    <tr>
      <td><code>disabled</code></td>
      <td>작업이 실행될지 여부를 나타냅니다.</td>
    </tr>
  </tbody>
</table>

```typescript
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class NotificationService {
  @Cron('* * 0 * * *', {
    name: 'notifications',
    timeZone: 'Europe/Paris',
  })
  triggerNotifications() {}
}
```

선언된 후 cron 작업에 접근하거나, 동적으로 cron 작업을 생성하려면 (예: cron 패턴이 런타임에 정의되는 경우) <a href="/techniques/task-scheduling#dynamic-schedule-module-api">동적 API</a>를 사용하여 cron 작업에 접근할 수 있습니다. 선언적 cron 작업에 API를 통해 접근하려면, 데코레이터의 두 번째 인수로 옵션 객체에서 `name` 속성을 전달하여 작업에 이름을 연결해야 합니다.

#### 선언적 간격

지정된 간격으로 (반복적으로) 메서드가 실행되도록 선언하려면, 메서드 정의 앞에 `@Interval()` 데코레이터를 붙입니다. 데코레이터에 간격 값을 밀리초 단위로 전달합니다. 예시는 다음과 같습니다:

```typescript
@Interval(10000)
handleInterval() {
  this.logger.debug('매 10초마다 호출됩니다');
}
```

> info **힌트** 이 메커니즘은 내부적으로 JavaScript `setInterval()` 함수를 사용합니다. 반복 작업을 예약하려면 cron 작업도 사용할 수 있습니다.

선언적 간격을 선언하는 클래스 외부에서 제어하려면 <a href="/techniques/task-scheduling#dynamic-schedule-module-api">동적 API</a>를 통해 간격에 이름을 연결합니다:

```typescript
@Interval('notifications', 2500)
handleInterval() {}
```

<a href="https://docs.nestjs.com/techniques/task-scheduling#dynamic-intervals">동적 API</a>는 **동적 간격** 생성, 간격 속성 런타임 정의, 간격 나열 및 삭제도 가능합니다.

<app-banner-enterprise></app-banner-enterprise>

#### 선언적 타임아웃

지정된 시간(한 번)에 메

서드가 실행되도록 선언하려면, 메서드 정의 앞에 `@Timeout()` 데코레이터를 붙입니다. 데코레이터에 애플리케이션 시작 후 상대 시간 오프셋(밀리초)을 전달합니다. 예시는 다음과 같습니다:

```typescript
@Timeout(5000)
handleTimeout() {
  this.logger.debug('5초 후 한 번 호출됩니다');
}
```

> info **힌트** 이 메커니즘은 내부적으로 JavaScript `setTimeout()` 함수를 사용합니다.

선언적 타임아웃을 선언하는 클래스 외부에서 제어하려면 <a href="/techniques/task-scheduling#dynamic-schedule-module-api">동적 API</a>를 통해 타임아웃에 이름을 연결합니다:

```typescript
@Timeout('notifications', 2500)
handleTimeout() {}
```

<a href="https://docs.nestjs.com/techniques/task-scheduling#dynamic-timeouts">동적 API</a>는 **동적 타임아웃** 생성, 타임아웃 속성 런타임 정의, 타임아웃 나열 및 삭제도 가능합니다.

#### 동적 스케줄 모듈 API

`@nestjs/schedule` 모듈은 선언적 <a href="https://docs.nestjs.com/techniques/task-scheduling#declarative-cron-jobs">cron 작업</a>, <a href="https://docs.nestjs.com/techniques/task-scheduling#declarative-timeouts">타임아웃</a> 및 <a href="https://docs.nestjs.com/techniques/task-scheduling#declarative-intervals">간격</a>을 관리하는 동적 API를 제공합니다. 이 API는 속성이 런타임에 정의된 **동적** cron 작업, 타임아웃 및 간격을 생성하고 관리하는 기능도 제공합니다.

#### 동적 cron 작업

`SchedulerRegistry` API를 사용하여 코드 어디에서나 이름으로 `CronJob` 인스턴스를 참조합니다. 먼저 표준 생성자 주입을 사용하여 `SchedulerRegistry`를 주입합니다:

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

> info **힌트** `SchedulerRegistry`는 `@nestjs/schedule` 패키지에서 가져옵니다.

그런 다음 클래스를 다음과 같이 사용합니다. cron 작업이 다음과 같이 선언되었다고 가정합니다:

```typescript
@Cron('* * 8 * * *', {
  name: 'notifications',
})
triggerNotifications() {}
```

이 작업에 접근하려면 다음을 수행합니다:

```typescript
const job = this.schedulerRegistry.getCronJob('notifications');

job.stop();
console.log(job.lastDate());
```

`getCronJob()` 메서드는 이름이 지정된 cron 작업을 반환합니다. 반환된 `CronJob` 객체는 다음 메서드를 가집니다:

- `stop()` - 실행될 예정인 작업을 중지합니다.
- `start()` - 중지된 작업을 다시 시작합니다.
- `setTime(time: CronTime)` - 작업을 중지하고, 새로운 시간을 설정한 후, 작업을 시작합니다.
- `lastDate()` - 작업이 마지막으로 실행된 날짜의 `DateTime` 표현을 반환합니다.
- `nextDate()` - 작업이 다음으로 실행될 예정인 날짜의 `DateTime` 표현을 반환합니다.
- `nextDates(count: number)` - 작업 실행을 트리거할 다음 날짜 세트에 대한 `DateTime` 표현 배열(size `count`)을 제공합니다. `count`는 기본적으로 0으로 설정되어 빈 배열을 반환합니다.

> info **힌트** `DateTime` 객체를 JavaScript Date로 렌더링하려면 `toJSDate()`를 사용하세요.

**새로운** cron 작업을 동적으로 생성하려면 `SchedulerRegistry.addCronJob` 메서드를 사용합니다:

```typescript
addCronJob(name: string, seconds: string) {
  const job = new CronJob(`${seconds} * * * * *`, () => {
    this.logger.warn(`작업 ${name}이(가) ${seconds}초마다 실행됩니다!`);
  });

  this.schedulerRegistry.addCronJob(name, job);
  job.start();

  this.logger.warn(
    `작업 ${name}이(가) 매 분 ${seconds}초마다 추가되었습니다!`,
  );
}
```

이 코드에서 우리는 `cron` 패키지의 `CronJob` 객체를 사용하여 cron 작업을 생성합니다. `CronJob` 생성자는 cron 패턴(마치 `@Cron()` <a href="https://docs.nestjs.com/techniques/task-scheduling#declarative-cron-jobs">데코레이터</a>처럼)을 첫 번째 인수로, cron 타이머가 실행될 때 실행할 콜백을 두 번째 인수로 받습니다. `SchedulerRegistry.addCronJob` 메서드는 `CronJob`의 이름과 `CronJob` 객체 자체 두 가지 인수를 받습니다.

> warning **경고** `SchedulerRegistry`에 접근하기 전에 주입하는 것을 잊지 마세요. `CronJob`은 `cron` 패키지에서 가져옵니다.

**이름이 지정된** cron 작업을 삭제하려면 `SchedulerRegistry.deleteCronJob` 메서드를 다음과 같이 사용합니다:

```typescript
deleteCron(name: string) {
  this.schedulerRegistry.deleteCronJob(name);
  this.logger.warn(`작업 ${name}이(가) 삭제되었습니다!`);
}
```

모든 cron 작업을 **목록화**하려면 `SchedulerRegistry.getCronJobs` 메서드를 다음과 같이 사용합니다:

```typescript
getCrons() {
  const jobs = this.schedulerRegistry.getCronJobs();
  jobs.forEach((value, key, map) => {
    let next;
    try {
      next = value.nextDate().toJSDate();
    } catch (e) {
      next = '오류: 다음 실행 날짜가 과거에 있습니다!';
    }
    this.logger.log(`작업: ${key} -> 다음: ${next}`);
  });
}
```

`getCronJobs()` 메서드는 `map`을 반환합니다. 이 코드에서 우리는 map을 반복하고 각 `CronJob`의 `nextDate()` 메서드에 접근하려고 시도합니다. `CronJob` API에서, 작업이 이미 실행되었고 미래 실행 날짜가 없는 경우, 예외가 발생합니다.

#### 동적 간격

`SchedulerRegistry.getInterval` 메서드를 사용하여 간격에 대한 참조를 얻습니다. 위와 같이, 표준 생성자 주입을 사용하여 `SchedulerRegistry`를 주입합니다:

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

그리고 다음과 같이 사용합니다:

```typescript
const interval = this.schedulerRegistry.getInterval('notifications');
clearInterval(interval);
```

새로운 **간격을 동적으로** 생성하려면 `SchedulerRegistry.addInterval` 메서드를 사용합니다:

```typescript
addInterval(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`간격 ${name}이(가) ${milliseconds}밀리초마다 실행됩니다!`);
  };

  const interval = setInterval(callback, milliseconds);
  this.schedulerRegistry.addInterval(name, interval);
}
```

이 코드에서 우리는 표준 JavaScript 간격을 생성한 다음 이를 `SchedulerRegistry.addInterval` 메서드에 전달합니다. 이 메서드는 간격의 이름과 간격 자체 두 가지 인수를 받습니다.

**이름이 지정된** 간격을 삭제하려면 `SchedulerRegistry.deleteInterval` 메서드를 다음과 같이 사용합니다:

```typescript
deleteInterval(name: string) {
  this.schedulerRegistry.deleteInterval(name);
  this.logger.warn(`간격 ${name}이(가) 삭제되었습니다!`);
}
```

모든 간격을 **목록화**하려면 `SchedulerRegistry.getIntervals` 메서드를 다음과 같이 사용합니다:

```typescript
getIntervals() {
  const intervals = this.schedulerRegistry.getIntervals();
  intervals.forEach(key => this.logger.log(`간격: ${key}`));
}
```

#### 동적 타임아웃

`SchedulerRegistry.getTimeout` 메서드를 사용하여 타임아웃에 대한 참조를 얻습니다. 위와 같이, 표준 생성자 주입을 사용하여 `SchedulerRegistry`를 주입합니다:

```typescript
constructor(private readonly schedulerRegistry: SchedulerRegistry) {}
```

그리고 다음과 같이 사용합니다:

```typescript
const timeout = this.schedulerRegistry.getTimeout('notifications');
clearTimeout(timeout);
```

새로운 **타임아웃을 동적으로** 생성하려면 `SchedulerRegistry.addTimeout` 메서드를 사용합니다:

```typescript
addTimeout(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`타임아웃 ${name}이(가) ${milliseconds}밀리초 후에 실행됩니다!`);
  };

  const timeout = setTimeout(callback, milliseconds);
  this.schedulerRegistry.addTimeout(name, timeout);
}
```

이 코드에서 우리는 표준 JavaScript 타임아웃을 생성한 다음 이를 `SchedulerRegistry.addTimeout` 메서드에 전달합니다. 이 메서드는 타임아웃의 이름과 타임아웃 자체 두 가지 인수를 받습니다.

**이름이 지정된** 타임아웃을 삭제하려면 `SchedulerRegistry.deleteTimeout` 메서드를 다음과 같이 사용합니다:

```typescript
deleteTimeout(name: string) {
  this.schedulerRegistry.deleteTimeout(name);
  this.logger.warn(`

타임아웃 ${name}이(가) 삭제되었습니다!`);
}
```

모든 타임아웃을 **목록화**하려면 `SchedulerRegistry.getTimeouts` 메서드를 다음과 같이 사용합니다:

```typescript
getTimeouts() {
  const timeouts = this.schedulerRegistry.getTimeouts();
  timeouts.forEach(key => this.logger.log(`타임아웃: ${key}`));
}
```

#### 예제

작동하는 예제는 [여기](https://github.com/nestjs/nest/tree/master/sample/27-scheduling)에서 확인할 수 있습니다.
