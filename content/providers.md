### Providers

Providers는 Nest의 기본 개념입니다. 많은 기본 Nest 클래스들은 providers로 취급될 수 있습니다. 예를 들어, 서비스, 리포지토리, 팩토리, 헬퍼 등이 있습니다. Providers의 주요 아이디어는 종속성으로 **주입**될 수 있다는 것입니다. 이는 객체들이 서로 다양한 관계를 맺을 수 있게 하며, 이러한 객체들을 연결하는 기능을 주로 Nest 런타임 시스템에 위임할 수 있음을 의미합니다.

<figure><img src="https://docs.nestjs.com/assets/Components_1.png" /></figure>

이전 장에서 간단한 `CatsController`를 만들었습니다. Controllers는 HTTP 요청을 처리하고 더 복잡한 작업을 **providers**에 위임해야 합니다. Providers는 [module](https://docs.nestjs.com/modules)에서 `providers`로 선언된 일반 자바스크립트 클래스입니다.

> info **힌트** Nest는 종속성을 더 객체 지향적인 방식으로 설계하고 구성할 수 있는 가능성을 제공하므로, [SOLID](https://en.wikipedia.org/wiki/SOLID) 원칙을 따르는 것을 강력히 권장합니다.

#### Services

간단한 `CatsService`를 만드는 것부터 시작해 보겠습니다. 이 서비스는 데이터 저장 및 검색을 담당하며, `CatsController`에 의해 사용될 예정이므로 provider로 정의하기에 적합합니다.

```typescript
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}
```

> info **힌트** CLI를 사용하여 서비스를 생성하려면 `$ nest g service cats` 명령어를 실행하세요.

`CatsService`는 하나의 속성과 두 개의 메서드를 가진 기본 클래스입니다. 유일한 새로운 기능은 `@Injectable()` 데코레이터를 사용하는 것입니다. `@Injectable()` 데코레이터는 `CatsService`가 Nest [IoC](https://en.wikipedia.org/wiki/Inversion_of_control) 컨테이너에 의해 관리될 수 있는 클래스임을 선언하는 메타데이터를 첨부합니다. 이 예제는 또한 다음과 같은 `Cat` 인터페이스를 사용합니다:

```typescript
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

이제 고양이를 검색하는 서비스 클래스를 만들었으니, 이를 `CatsController` 내부에서 사용해 보겠습니다:

```typescript
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

`CatsService`는 클래스 생성자를 통해 **주입**됩니다. `private` 문법을 사용하는 것을 주목하세요. 이 단축 구문은 `catsService` 멤버를 선언하고 초기화하는 것을 동시에 처리할 수 있게 합니다.

#### 의존성 주입

Nest는 **의존성 주입**으로 잘 알려진 강력한 설계 패턴을 중심으로 구축되었습니다. 이 개념에 대해 더 알고 싶다면 공식 [Angular](https://angular.dev/guide/di) 문서에서 훌륭한 글을 읽어보는 것을 권장합니다.

Nest에서는 타입스크립트의 기능 덕분에 종속성을 매우 쉽게 관리할 수 있습니다. 종속성은 타입으로만 해결됩니다. 아래 예제에서 Nest는 `CatsService`의 인스턴스를 생성하여 `catsService`를 해결하고 반환합니다(일반적으로 싱글톤의 경우, 이미 다른 곳에서 요청된 경우 기존 인스턴스를 반환). 이 종속성은 컨트롤러의 생성자에 전달되거나 지정된 속성에 할당됩니다:

```typescript
constructor(private catsService: CatsService) {}
```

#### 스코프

Providers는 일반적으로 애플리케이션 생명주기와 동기화된 수명("스코프")을 가집니다. 애플리케이션이 부트스트랩될 때 모든 종속성이 해결되어야 하며, 따라서 모든 provider가 인스턴스화되어야 합니다. 마찬가지로, 애플리케이션이 종료될 때 각 provider는 파괴됩니다. 그러나 provider의 수명을 **요청 스코프**로 만들 수 있는 방법도 있습니다. 이러한 기술에 대한 자세한 내용은 [여기](https://docs.nestjs.com/fundamentals/injection-scopes)를 참조하세요.

#### 사용자 정의 providers

Nest에는 providers 간의 관계를 해결하는 내장된 제어 역전("IoC") 컨테이너가 있습니다. 이 기능은 위에서 설명한 의존성 주입 기능의 기초가 되지만, 지금까지 설명한 것보다 훨씬 강력합니다. provider를 정의하는 몇 가지 방법이 있습니다: 일반 값, 클래스, 비동기 또는 동기 팩토리를 사용할 수 있습니다. 더 많은 예제는 [여기](https://docs.nestjs.com/fundamentals/dependency-injection)에 제공됩니다.

#### 선택적 providers

때때로 반드시 해결될 필요가 없는 종속성이 있을 수 있습니다. 예를 들어, 클래스가 **구성 객체**에 의존할 수 있지만, 전달되지 않으면 기본 값을 사용해야 합니다. 이러한 경우 종속성은 선택 사항이 되며, 구성 provider가 없어도 오류가 발생하지 않습니다.

provider가 선택적임을 나타내려면 생성자의 서명에서 `@Optional()` 데코레이터를 사용하세요.

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

위 예제에서 사용자 정의 provider를 사용하고 있으므로 `HTTP_OPTIONS` 사용자 정의 **토큰**을 포함하고 있습니다. 이전 예제에서는 생성자 기반 주입을 사용하여 생성자에서 클래스에 대한 종속성을 나타냈습니다. 사용자 정의 providers 및 관련 토큰에 대한 자세한 내용은 [여기](https://docs.nestjs.com/fundamentals/custom-providers)를 참조하세요.

#### 속성 기반 주입

지금까지 사용한 기술은 생성자 기반 주입이라고 하며, providers는 생성자 메서드를 통해 주입됩니다. 특정 경우에는 **속성 기반 주입**이 유용할 수 있습니다. 예를 들어, 최상위 클래스가 하나 이상의 providers에 의존하는 경우, 생성자에서 `super()`를 호출하여 이를 서브클래스에 전달하는 것이 번거로울 수 있습니다. 이를 피하기 위해 속성 수준에서 `@Inject()` 데코레이터를 사용할 수 있습니다.

```typescript
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

> warning **경고** 클래스가 다른 클래스를 상속하지 않는 경우 **생성자 기반** 주입을 항상 사용하는 것이 좋습니다. 생성자는 필요한 종속성을 명시적으로 나타내며, `@Inject`로 주석이 달린 클래스 속성보다 가시성이 더 좋습니다.

#### Provider 등록

이제 provider(`CatsService`)를 정의했고, 그 서비스를 사용하는 소비자(`CatsController`)를 정의했으니, 서비스를 Nest에 등록하여 주입할 수 있도록 해야 합니다. 이를 위해 모듈 파일(`app.module.ts`)을 편집하고 `@Module()` 데코레이터의 `providers` 배열에 서비스를 추가합니다.

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

이제 Nest는 `CatsController` 클래스의 종속성을 해결할 수 있습니다.

디렉터리 구조는 이제 다음과 같아야 합니다:

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
<div class="item">cats.service.ts</div>
</div>
<div class="item">app.module.ts</div>
<div class="item">main.ts</div>
</div>
</div>

#### 수동 인스턴스화

지금까지는 Nest가 대부분의 종속성 해결 세부 사항을 자동으로 처리하는 방법에 대해 논의했습니다. 특정 상황에서는 내장된 의존성 주입 시스템을 벗어나 수동으로 providers를 검색하거나 인스턴스화해야 할 수도 있습니다. 아래에서 이러한

 주제 중 두 가지를 간략히 다루겠습니다.

기존 인스턴스를 가져오거나 providers를 동적으로 인스턴스화하려면 [Module reference](https://docs.nestjs.com/fundamentals/module-ref)를 사용할 수 있습니다.

`bootstrap()` 함수 내에서 providers를 가져오려면(예: 컨트롤러가 없는 독립 실행형 애플리케이션 또는 부트스트래핑 중 구성 서비스를 활용하기 위해) [Standalone applications](https://docs.nestjs.com/standalone-applications)를 참조하세요.
