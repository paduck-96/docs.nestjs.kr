### Guards

가드는 `@Injectable()` 데코레이터가 있는 클래스이며, `CanActivate` 인터페이스를 구현합니다.

<figure><img src="https://docs.nestjs.com/assets/Guards_1.png" /></figure>

가드는 **단일 책임**을 가지고 있습니다. 특정 조건(권한, 역할, ACL 등)에 따라 주어진 요청이 라우트 핸들러에 의해 처리될지 여부를 결정합니다. 이는 종종 **권한 부여**라고 합니다. 권한 부여(및 일반적으로 함께 작동하는 **인증**)는 전통적인 Express 애플리케이션에서 [미들웨어](https://docs.nestjs.com/middleware)로 처리되었습니다. 미들웨어는 토큰 유효성 검사와 요청 객체에 속성 부여와 같은 작업이 특정 라우트 컨텍스트(및 해당 메타데이터)와 강하게 연결되지 않기 때문에 인증에 적합한 선택입니다.

그러나 미들웨어는 본질적으로 어떤 핸들러가 `next()` 함수 호출 후 실행될지 알 수 없습니다. 반면, **가드**는 `ExecutionContext` 인스턴스에 액세스할 수 있으므로 다음에 실행될 정확한 핸들러를 알 수 있습니다. 예외 필터, 파이프 및 인터셉터와 마찬가지로 가드는 요청/응답 주기의 정확한 시점에 처리 로직을 선언적으로 삽입할 수 있도록 설계되었습니다. 이는 코드를 DRY하게 유지하고 선언적으로 만들 수 있도록 도와줍니다.

> info **힌트** 가드는 모든 미들웨어 이후에 실행되지만, 어떤 인터셉터나 파이프보다 먼저 실행됩니다.

#### 권한 부여 가드

앞서 언급했듯이 **권한 부여**는 가드의 훌륭한 사용 사례입니다. 특정 라우트는 호출자가 충분한 권한을 가졌을 때만 사용할 수 있어야 하기 때문입니다. 지금 만들 `AuthGuard`는 인증된 사용자를 가정하며, 요청 헤더에 토큰이 첨부되어 있다고 가정합니다. 이 가드는 토큰을 추출하고 유효성을 검사하며, 추출된 정보를 사용하여 요청을 계속 처리할지 여부를 결정합니다.

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

> info **힌트** 애플리케이션에서 인증 메커니즘을 구현하는 실제 예제가 필요하면 [여기](https://docs.nestjs.com/security/authentication)에서 이 장을 참조하세요. 보다 정교한 권한 부여 예제를 보려면 [여기](https://docs.nestjs.com/security/authorization) 페이지를 확인하세요.

`validateRequest()` 함수 내부의 로직은 필요에 따라 간단하거나 복잡할 수 있습니다. 이 예제의 주요 목적은 가드가 요청/응답 주기에 어떻게 맞춰져 있는지 보여주는 것입니다.

모든 가드는 `canActivate()` 함수를 구현해야 합니다. 이 함수는 현재 요청이 허용되는지 여부를 나타내는 부울 값을 반환해야 합니다. 이는 동기적으로 또는 비동기적으로(Promise 또는 Observable을 통해) 응답을 반환할 수 있습니다. Nest는 반환 값을 사용하여 다음 작업을 제어합니다:

- `true`를 반환하면 요청이 처리됩니다.
- `false`를 반환하면 Nest는 요청을 거부합니다.

#### 실행 컨텍스트

`canActivate()` 함수는 `ExecutionContext` 인스턴스를 단일 인수로 받습니다. `ExecutionContext`는 `ArgumentsHost`에서 상속됩니다. 이전에 예외 필터 장에서 `ArgumentsHost`를 보았습니다. 위 샘플에서는 `ArgumentsHost`에 정의된 동일한 도우미 메서드를 사용하여 `Request` 객체에 대한 참조를 얻고 있습니다. 이 주제에 대해 더 알고 싶다면 [예외 필터](https://docs.nestjs.com/exception-filters#arguments-host) 장의 **Arguments host** 섹션을 참조하세요.

`ArgumentsHost`를 확장함으로써 `ExecutionContext`는 현재 실행 프로세스에 대한 추가 세부 정보를 제공하는 몇 가지 새로운 도우미 메서드를 추가합니다. 이러한 세부 정보는 더 광범위한 컨트롤러, 메서드 및 실행 컨텍스트에서 작동할 수 있는 더 일반적인 가드를 만드는 데 유용할 수 있습니다. `ExecutionContext`에 대해 더 알고 싶다면 [여기](https://docs.nestjs.com/fundamentals/execution-context)를 참조하세요.

#### 역할 기반 인증

특정 역할을 가진 사용자만 접근할 수 있도록 하는 더 기능적인 가드를 만들어 보겠습니다. 기본 가드 템플릿으로 시작하고, 다음 섹션에서 이를 확장해 보겠습니다. 현재는 모든 요청을 처리하도록 허용합니다:

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
```

#### 가드 바인딩

파이프와 예외 필터처럼 가드는 **컨트롤러 범위**, 메서드 범위 또는 전역 범위로 설정할 수 있습니다. 아래에서는 `@UseGuards()` 데코레이터를 사용하여 컨트롤러 범위 가드를 설정합니다. 이 데코레이터는 단일 인수 또는 쉼표로 구분된 인수 목록을 받을 수 있습니다. 이를 통해 한 번의 선언으로 적절한 가드 세트를 쉽게 적용할 수 있습니다.

```typescript
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

> info **힌트** `@UseGuards()` 데코레이터는 `@nestjs/common` 패키지에서 가져옵니다.

위에서 `RolesGuard` 클래스를 전달했으며(인스턴스가 아닌), 인스턴스화 책임을 프레임워크에 맡기고 의존성 주입을 가능하게 했습니다. 파이프 및 예외 필터와 마찬가지로, 인스턴스를 직접 전달할 수도 있습니다:

```typescript
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

위 구성은 이 컨트롤러에 의해 선언된 모든 핸들러에 가드를 첨부합니다. 가드를 단일 메서드에만 적용하려면 **메서드 수준**에서 `@UseGuards()` 데코레이터를 적용합니다.

전역 가드를 설정하려면 Nest 애플리케이션 인스턴스의 `useGlobalGuards()` 메서드를 사용하세요:

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

> warning **알림** 하이브리드 앱의 경우 기본적으로 `useGlobalGuards()` 메서드는 게이트웨이와 마이크로서비스에 대한 가드를 설정하지 않습니다. (이 동작을 변경하는 방법에 대한 정보는 [하이브리드 애플리케이션](https://docs.nestjs.com/faq/hybrid-application)을 참조하세요). "표준"(비 하이브리드) 마이크로서비스 앱의 경우 `useGlobalGuards()`는 가드를 전역적으로 마운트합니다.

전역 가드는 애플리케이션 전체, 모든 컨트롤러 및 모든 라우트 핸들러에서 사용됩니다. 의존성 주입 측면에서, 모듈 외부에서 등록된 전역 가드(`위 예제에서처럼 useGlobalGuards()` 사용)는 모듈 외부에서 바인딩되었기 때문에 종속성을 주입할 수 없습니다. 이 문제를 해결하려면 다음 구성을 사용하여 모듈 내에서 직접 가드를 설정할 수 있습니다:

```typescript
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

> info **힌트** 이 방법을 사용하여 가드에 대한 의존성 주입을 수행할 때, 이 구성이 사용되는 모듈에 상관없이 가드는 실제로 전역적입니다. 어디에서 이 작업을 수행해야 할까요? 가드(`위 예제의 RolesGuard`)가 정의된 모듈을 선택하세요. 또한 `useClass`는 사용자 정의 provider 등록을 처리하는 유일한 방법이 아닙니다. 자세한 내용은 [여기](https://docs.nestjs.com/fundamentals/custom-providers)를 참조하세요.

#### 핸들러별 역할 설정

우리의 `RolesGuard`는 작동하지만 아직 스마트하지 않습니다. 아직 가장 중요한 가드 기능인 [실행 컨텍스트](https://docs.nestjs.com/fundamentals/execution-context)를 활용하지 않고 있습니다. 아직 역할이나 현재 처리 중인 라우트에 필요한 역할을 알지 못합니다. 예를 들어 `

CatsController`는 다른 라우트에 대해 다른 권한 체계를 가질 수 있습니다. 일부는 관리자 사용자에게만 허용될 수 있으며, 다른 일부는 모두에게 열려 있을 수 있습니다. 역할을 라우트와 유연하고 재사용 가능한 방식으로 일치시키려면 어떻게 해야 할까요?

여기서 **사용자 정의 메타데이터**가 필요합니다(자세한 내용은 [여기](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata)에서 확인하세요). Nest는 `Reflector#createDecorator` 정적 메서드를 통해 생성된 데코레이터 또는 내장 `@SetMetadata()` 데코레이터를 통해 라우트 핸들러에 사용자 정의 **메타데이터**를 첨부할 수 있는 기능을 제공합니다.

예를 들어, 핸들러에 메타데이터를 첨부할 `@Roles()` 데코레이터를 생성해 보겠습니다. `Reflector`는 프레임워크에 의해 기본적으로 제공되며 `@nestjs/core` 패키지에서 노출됩니다.

```typescript
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

여기서 `Roles` 데코레이터는 `string[]` 타입의 단일 인수를 받는 함수입니다.

이제 이 데코레이터를 사용하려면 다음과 같이 핸들러에 주석을 달기만 하면 됩니다:

```typescript
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

여기서 `Roles` 데코레이터 메타데이터를 `create()` 메서드에 첨부하여 `admin` 역할을 가진 사용자만 이 라우트에 접근할 수 있도록 지정했습니다.

또한, `Reflector#createDecorator` 메서드를 사용하는 대신 내장 `@SetMetadata()` 데코레이터를 사용할 수도 있습니다. 자세한 내용은 [여기](https://docs.nestjs.com/fundamentals/execution-context#low-level-approach)에서 확인하세요.

#### 모든 것 통합하기

이제 다시 돌아가서 `RolesGuard`와 이를 결합해 보겠습니다. 현재는 모든 요청을 허용하여 모든 요청을 처리합니다. 반환 값을 현재 처리 중인 라우트에 필요한 실제 역할과 현재 사용자에게 할당된 **역할**을 비교하여 조건부로 만들고자 합니다. 라우트의 역할(사용자 정의 메타데이터)에 접근하기 위해 다시 한 번 `Reflector` 도우미 클래스를 사용할 것입니다:

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

> info **힌트** Node.js 세계에서는 권한이 부여된 사용자를 `request` 객체에 첨부하는 것이 일반적인 관행입니다. 따라서 위의 샘플 코드에서는 `request.user`가 사용자 인스턴스와 허용된 역할을 포함하고 있다고 가정합니다. 애플리케이션에서는 사용자 정의 **인증 가드**(또는 미들웨어)에서 이 연결을 수행할 것입니다. 이 주제에 대한 자세한 내용은 [여기](https://docs.nestjs.com/security/authentication) 장을 참조하세요.

> warning **경고** `matchRoles()` 함수 내부의 로직은 필요에 따라 간단하거나 복잡할 수 있습니다. 이 예제의 주요 목적은 가드가 요청/응답 주기에 어떻게 맞춰져 있는지 보여주는 것입니다.

<a href="https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata">실행 컨텍스트</a> 장의 **반사 및 메타데이터** 섹션을 참조하여 컨텍스트에 민감한 방식으로 `Reflector`를 사용하는 방법에 대한 자세한 내용을 확인하세요.

권한이 없는 사용자가 엔드포인트를 요청하면 Nest는 자동으로 다음과 같은 응답을 반환합니다:

```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

무대 뒤에서, 가드가 `false`를 반환하면 프레임워크는 `ForbiddenException`을 던집니다. 다른 오류 응답을 반환하려면 특정 예외를 던져야 합니다. 예를 들어:

```typescript
throw new UnauthorizedException();
```

가드에 의해 던져진 모든 예외는 [예외 처리 레이어](https://docs.nestjs.com/exception-filters) (전역 예외 필터 및 현재 컨텍스트에 적용된 모든 예외 필터)에 의해 처리됩니다.

> info **힌트** 권한 부여를 구현하는 실제 예제가 필요하면 [여기](https://docs.nestjs.com/security/authorization) 장을 참조하세요.
