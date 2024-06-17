### 직렬화

직렬화는 객체가 네트워크 응답으로 반환되기 전에 발생하는 프로세스입니다. 이는 반환할 데이터를 변환하고 정리하는 규칙을 제공하기에 적절한 위치입니다. 예를 들어, 비밀번호와 같은 민감한 데이터는 항상 응답에서 제외되어야 합니다. 또는 엔티티의 속성 중 일부만 전송해야 하는 경우도 있습니다. 이러한 변환을 수동으로 수행하는 것은 번거롭고 오류가 발생하기 쉽습니다. 또한 모든 경우를 다루었는지 확신하기 어렵습니다.

#### 개요

Nest는 이러한 작업이 간단하게 수행될 수 있도록 도와주는 내장 기능을 제공합니다. `ClassSerializerInterceptor` 인터셉터는 강력한 [class-transformer](https://github.com/typestack/class-transformer) 패키지를 사용하여 객체를 선언적이고 확장 가능한 방식으로 변환할 수 있게 해줍니다. 기본적으로 이 인터셉터는 메서드 핸들러에서 반환된 값을 가져와 [class-transformer](https://github.com/typestack/class-transformer)의 `instanceToPlain()` 함수를 적용합니다. 이렇게 하면 아래 설명된 대로 엔티티/DTO 클래스에 있는 `class-transformer` 데코레이터로 표현된 규칙을 적용할 수 있습니다.

> info **힌트** 직렬화는 [StreamableFile](https://docs.nestjs.com/techniques/streaming-files#streamable-file-class) 응답에 적용되지 않습니다.

#### 속성 제외

사용자 엔티티에서 `password` 속성을 자동으로 제외하려고 한다고 가정해 보겠습니다. 엔티티를 다음과 같이 주석으로 표시합니다:

```typescript
import { Exclude } from 'class-transformer';

export class UserEntity {
  id: number;
  firstName: string;
  lastName: string;

  @Exclude()
  password: string;

  constructor(partial: Partial<UserEntity>) {
    Object.assign(this, partial);
  }
}
```

이제 이 클래스의 인스턴스를 반환하는 메서드 핸들러가 있는 컨트롤러를 고려해 보겠습니다.

```typescript
@UseInterceptors(ClassSerializerInterceptor)
@Get()
findOne(): UserEntity {
  return new UserEntity({
    id: 1,
    firstName: 'Kamil',
    lastName: 'Mysliwiec',
    password: 'password',
  });
}
```

> **경고** 클래스의 인스턴스를 반환해야 합니다. 예를 들어, `{{ '{' }} user: new UserEntity() {{ '}' }}`와 같은 평범한 JavaScript 객체를 반환하면 객체가 올바르게 직렬화되지 않습니다.

> info **힌트** `ClassSerializerInterceptor`는 `@nestjs/common`에서 가져옵니다.

이 엔드포인트가 요청되면 클라이언트는 다음과 같은 응답을 받습니다:

```json
{
  "id": 1,
  "firstName": "Kamil",
  "lastName": "Mysliwiec"
}
```

인터셉터는 애플리케이션 전체에 적용될 수 있습니다(자세한 내용은 [여기](https://docs.nestjs.com/interceptors#binding-interceptors)를 참조하십시오). 인터셉터와 엔티티 클래스 선언의 조합은 `UserEntity`를 반환하는 **모든** 메서드가 `password` 속성을 제거하도록 합니다. 이를 통해 이 비즈니스 규칙을 중앙에서 강제할 수 있습니다.

#### 속성 노출

`@Expose()` 데코레이터를 사용하여 속성에 별칭을 제공하거나 속성 값을 계산하는 함수를 실행할 수 있습니다(예: **getter** 함수와 유사).

```typescript
@Expose()
get fullName(): string {
  return `${this.firstName} ${this.lastName}`;
}
```

#### 변환

`@Transform()` 데코레이터를 사용하여 추가 데이터 변환을 수행할 수 있습니다. 예를 들어, 다음 구문은 전체 객체를 반환하는 대신 `RoleEntity`의 name 속성을 반환합니다.

```typescript
@Transform(({ value }) => value.name)
role: RoleEntity;
```

#### 옵션 전달

기본 변환 함수의 동작을 수정하고 싶을 수 있습니다. 기본 설정을 재정의하려면 `@SerializeOptions()` 데코레이터와 함께 옵션 객체를 전달합니다.

```typescript
@SerializeOptions({
  excludePrefixes: ['_'],
})
@Get()
findOne(): UserEntity {
  return new UserEntity();
}
```

> info **힌트** `@SerializeOptions()` 데코레이터는 `@nestjs/common`에서 가져옵니다.

`@SerializeOptions()`를 통해 전달된 옵션은 기본 `instanceToPlain()` 함수의 두 번째 인수로 전달됩니다. 이 예에서는 `_` 접두사로 시작하는 모든 속성을 자동으로 제외하고 있습니다.

#### 예제

작동하는 예제는 [여기](https://github.com/nestjs/nest/tree/master/sample/21-serializer)에서 확인할 수 있습니다.

#### WebSockets 및 마이크로서비스

이 장에서는 HTTP 스타일 애플리케이션(e.g., Express 또는 Fastify)을 사용한 예제를 보여주지만, `ClassSerializerInterceptor`는 사용되는 전송 방법에 관계없이 WebSockets 및 마이크로서비스에서도 동일하게 작동합니다.

#### 더 알아보기

`class-transformer` 패키지에서 제공하는 사용 가능한 데코레이터 및 옵션에 대한 자세한 내용은 [여기](https://github.com/typestack/class-transformer)에서 확인하십시오.
