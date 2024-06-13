## Custom route decorators

Nest는 **데코레이터**라는 언어 기능을 중심으로 구축되었습니다. 데코레이터는 많은 일반적인 프로그래밍 언어에서 잘 알려진 개념이지만, JavaScript 세계에서는 아직 비교적 새로운 개념입니다. 데코레이터가 어떻게 작동하는지 더 잘 이해하려면 [이 기사](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841)를 읽어보는 것을 권장합니다. 다음은 간단한 정의입니다:

<blockquote class="external">
  ES2016 데코레이터는 함수(타겟, 이름, 속성 기술자)를 인수로 받아서 반환하는 표현식입니다.
  데코레이터 앞에 <code>@</code> 문자를 붙이고 데코레이터할 대상의 맨 위에 배치하여 적용합니다.
  데코레이터는 클래스, 메서드 또는 속성에 대해 정의될 수 있습니다.
</blockquote>

# Param decorators

Nest는 HTTP 라우트 핸들러와 함께 사용할 수 있는 유용한 **파라미터 데코레이터** 세트를 제공합니다. 아래는 제공되는 데코레이터와 일반적인 Express(또는 Fastify) 객체를 나타낸 표입니다:

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td>
    </tr>
    <tr>
      <td><code>@Response(), @Res()</code></td>
      <td><code>res</code></td>
    </tr>
    <tr>
      <td><code>@Next()</code></td>
      <td><code>next</code></td>
    </tr>
    <tr>
      <td><code>@Session()</code></td>
      <td><code>req.session</code></td>
    </tr>
    <tr>
      <td><code>@Param(param?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[param]</code></td>
    </tr>
    <tr>
      <td><code>@Body(param?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[param]</code></td>
    </tr>
    <tr>
      <td><code>@Query(param?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[param]</code></td>
    </tr>
    <tr>
      <td><code>@Headers(param?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[param]</code></td>
    </tr>
    <tr>
      <td><code>@Ip()</code></td>
      <td><code>req.ip</code></td>
    </tr>
    <tr>
      <td><code>@HostParam()</code></td>
      <td><code>req.hosts</code></td>
    </tr>
  </tbody>
</table>

또한, **커스텀 데코레이터**를 만들 수도 있습니다. 이것이 왜 유용할까요?

Node.js 세계에서는 **request** 객체에 속성을 첨부하는 것이 일반적입니다. 그런 다음 각 라우트 핸들러에서 다음과 같은 코드를 사용하여 이를 수동으로 추출합니다:

```typescript
const user = req.user;
```

코드를 더 읽기 쉽고 투명하게 만들기 위해 `@User()` 데코레이터를 생성하고 모든 컨트롤러에서 재사용할 수 있습니다.

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

그런 다음 요구 사항에 맞는 곳에서 간단히 사용할 수 있습니다.

```typescript
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}
```

# Passing data

데코레이터의 동작이 특정 조건에 따라 달라지는 경우 `data` 매개변수를 사용하여 데코레이터의 팩토리 함수에 인수를 전달할 수 있습니다. 한 가지 사용 사례는 요청 객체에서 키로 속성을 추출하는 커스텀 데코레이터입니다. 예를 들어, 우리의 [인증 레이어](https://docs.nestjs.com/techniques/authentication#implementing-passport-strategies)가 요청을 검증하고 사용자 엔티티를 요청 객체에 첨부한다고 가정해 보겠습니다. 인증된 요청의 사용자 엔티티는 다음과 같이 생길 수 있습니다:

```json
{
  "id": 101,
  "firstName": "Alan",
  "lastName": "Turing",
  "email": "alan@email.com",
  "roles": ["admin"]
}
```

이제 속성 이름을 키로 받고 해당 값을 반환하는(또는 존재하지 않거나 `user` 객체가 생성되지 않은 경우 undefined를 반환하는) 데코레이터를 정의해 보겠습니다.

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
```

컨트롤러에서 `@User()` 데코레이터를 통해 특정 속성에 액세스하는 방법은 다음과 같습니다:

```typescript
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
```

동일한 데코레이터를 사용하여 다른 키로 다른 속성에 접근할 수 있습니다. `user` 객체가 깊거나 복잡한 경우, 이는 요청 핸들러 구현을 더 쉽게 읽고 이해할 수 있게 합니다.

# Working with pipes

Nest는 커스텀 파라미터 데코레이터를 빌트인 데코레이터(`@Body()`, `@Param()` 및 `@Query()`)와 동일하게 취급합니다. 이는 파이프가 커스텀 주석이 달린 매개변수(예: `user` 인수)에 대해서도 실행된다는 것을 의미합니다. 또한, 파이프를 커스텀 데코레이터에 직접 적용할 수 있습니다:

```typescript
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {
  console.log(user);
}
```

`validateCustomDecorators` 옵션은 true로 설정해야 합니다. `ValidationPipe`는 기본적으로 커스텀 데코레이터가 주석이 달린 인수를 검증하지 않습니다.

# Decorator composition

Nest는 여러 데코레이터를 조합하는 데 도움이 되는 메서드를 제공합니다. 예를 들어, 인증과 관련된 모든 데코레이터를 단일 데코레이터로 결합하려고 한다고 가정해 보겠습니다. 다음 구조를 사용하여 이를 수행할 수 있습니다:

```typescript
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

그런 다음 이 커스텀 `@Auth()` 데코레이터를 다음과 같이 사용할 수 있습니다:

```typescript
@Get('users')
@Auth('admin')
findAllUsers() {}
```

이렇게 하면 네 개의 데코레이터가 단일 선언으로 적용됩니다.

`@nestjs/swagger` 패키지의 `@ApiHideProperty()` 데코레이터는 조합이 불가능하며 `applyDecorators` 함수와 함께 제대로 작동하지 않습니다.
