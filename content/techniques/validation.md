### Validation

웹 애플리케이션에 전송되는 모든 데이터의 올바름을 검증하는 것은 모범 사례입니다. 들어오는 요청을 자동으로 검증하기 위해 Nest는 여러 파이프를 기본적으로 제공합니다:

- `ValidationPipe`
- `ParseIntPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`

`ValidationPipe`는 강력한 [class-validator](https://github.com/typestack/class-validator) 패키지와 그 선언적 검증 데코레이터를 사용합니다. `ValidationPipe`는 특정 모듈의 로컬 클래스/DTO 선언에서 간단한 애노테이션으로 선언된 검증 규칙을 모든 클라이언트 페이로드에 대해 강제하는 편리한 접근 방식을 제공합니다.

#### 개요

[Pipes](https://docs.nestjs.com/pipes) 챕터에서는 간단한 파이프를 작성하고 이를 컨트롤러, 메서드 또는 글로벌 앱에 바인딩하여 프로세스가 어떻게 작동하는지 설명했습니다. 이 챕터의 주제를 가장 잘 이해하기 위해 해당 챕터를 검토하십시오. 여기에서는 `ValidationPipe`의 다양한 **실제 사례**를 집중적으로 다루고, 몇 가지 고급 사용자 정의 기능을 사용하는 방법을 보여줍니다.

#### 내장된 ValidationPipe 사용하기

먼저 필요한 종속성을 설치합니다.

```bash
$ npm i --save class-validator class-transformer
```

> info **힌트** `ValidationPipe`는 `@nestjs/common` 패키지에서 내보내집니다.

이 파이프는 [`class-validator`](https://github.com/typestack/class-validator) 및 [`class-transformer`](https://github.com/typestack/class-transformer) 라이브러리를 사용하므로 많은 옵션을 사용할 수 있습니다. 이러한 설정은 파이프에 전달되는 구성 객체를 통해 구성할 수 있습니다. 다음은 내장된 옵션입니다:

```typescript
export interface ValidationPipeOptions extends ValidatorOptions {
  transform?: boolean;
  disableErrorMessages?: boolean;
  exceptionFactory?: (errors: ValidationError[]) => any;
}
```

이외에도, `class-validator`의 모든 옵션(ValidatorOptions 인터페이스에서 상속됨)을 사용할 수 있습니다:

<table>
  <tr>
    <th>옵션</th>
    <th>타입</th>
    <th>설명</th>
  </tr>
  <tr>
    <td><code>enableDebugMessages</code></td>
    <td><code>boolean</code></td>
    <td>설정 시, 검증기가 문제가 발생할 때 콘솔에 추가 경고 메시지를 출력합니다.</td>
  </tr>
  <tr>
    <td><code>skipUndefinedProperties</code></td>
    <td><code>boolean</code></td>
    <td>설정 시, 검증 중인 객체에서 정의되지 않은 모든 속성의 검증을 건너뜁니다.</td>
  </tr>
  <tr>
    <td><code>skipNullProperties</code></td>
    <td><code>boolean</code></td>
    <td>설정 시, 검증 중인 객체에서 null인 모든 속성의 검증을 건너뜁니다.</td>
  </tr>
  <tr>
    <td><code>skipMissingProperties</code></td>
    <td><code>boolean</code></td>
    <td>설정 시, 검증 중인 객체에서 null 또는 undefined인 모든 속성의 검증을 건너뜁니다.</td>
  </tr>
  <tr>
    <td><code>whitelist</code></td>
    <td><code>boolean</code></td>
    <td>설정 시, 검증 데코레이터가 없는 속성을 검증된(반환된) 객체에서 제거합니다.</td>
  </tr>
  <tr>
    <td><code>forbidNonWhitelisted</code></td>
    <td><code>boolean</code></td>
    <td>설정 시, 비화이트리스트 속성을 제거하는 대신 예외를 발생시킵니다.</td>
  </tr>
  <tr>
    <td><code>forbidUnknownValues</code></td>
    <td><code>boolean</code></td>
    <td>설정 시, 알 수 없는 객체의 검증 시도를 즉시 실패하게 합니다.</td>
  </tr>
  <tr>
    <td><code>disableErrorMessages</code></td>
    <td><code>boolean</code></td>
    <td>설정 시, 클라이언트에게 검증 오류 메시지를 반환하지 않습니다.</td>
  </tr>
  <tr>
    <td><code>errorHttpStatusCode</code></td>
    <td><code>number</code></td>
    <td>설정 시, 오류가 발생했을 때 사용할 예외 유형을 지정할 수 있습니다. 기본값은 <code>BadRequestException</code>입니다.</td>
  </tr>
  <tr>
    <td><code>exceptionFactory</code></td>
    <td><code>Function</code></td>
    <td>검증 오류 배열을 받고 던질 예외 객체를 반환하는 함수입니다.</td>
  </tr>
  <tr>
    <td><code>groups</code></td>
    <td><code>string[]</code></td>
    <td>객체를 검증할 때 사용할 그룹입니다.</td>
  </tr>
  <tr>
    <td><code>always</code></td>
    <td><code>boolean</code></td>
    <td>데코레이터 옵션의 기본값을 <code>always</code> 옵션으로 설정합니다. 기본값은 데코레이터 옵션에서 재정의할 수 있습니다.</td>
  </tr>
  <tr>
    <td><code>strictGroups</code></td>
    <td><code>boolean</code></td>
    <td><code>groups</code>가 주어지지 않았거나 비어 있는 경우, 적어도 하나의 그룹을 가진 데코레이터를 무시합니다.</td>
  </tr>
  <tr>
    <td><code>dismissDefaultMessages</code></td>
    <td><code>boolean</code></td>
    <td>설정 시, 검증은 기본 메시지를 사용하지 않습니다. 오류 메시지는 명시적으로 설정되지 않은 경우 항상 <code>undefined</code>가 됩니다.</td>
  </tr>
  <tr>
    <td><code>validationError.target</code></td>
    <td><code>boolean</code></td>
    <td><code>ValidationError</code>에 대상이 노출되어야 하는지 여부를 나타냅니다.</td>
  </tr>
  <tr>
    <td><code>validationError.value</code></td>
    <td><code>boolean</code></td>
    <td><code>ValidationError</code>에 검증된 값이 노출되어야 하는지 여부를 나타냅니다.</td>
  </tr>
  <tr>
    <td><code>stopAtFirstError</code></td>
    <td><code>boolean</code></td>
    <td>설정 시, 주어진 속성의 검증은 첫 번째 오류를 만나면 중지됩니다. 기본값은 false입니다.</td>
  </tr>
</table>

> info **참고** `class-validator` 패키지에 대한 자세한 정보는 [여기](https://github.com/typestack/class-validator)에서 찾을 수 있습니다.

#### 자동 검증

먼저 `ValidationPipe`를 애플리케이션 수준에서 바인딩하여 모든 엔드포인트가 잘못된 데이터를 받지 않도록 보호합니다.

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

파이프를 테스트하기 위해 기본 엔드포인트를 생성해 봅시다.

```typescript
@Post()
create(@Body() createUserDto: CreateUserDto) {
  return 'This action adds a new user';
}
```

> info **힌트** TypeScript는 **제네릭 또는 인터페이스**에 대한 메타데이터를 저장하지 않기 때문에, DTO에서 이를 사용할 때 `ValidationPipe`가 들어오는 데이터를 올바르게 검증하지 못할 수 있습니다. 따라서 DTO에서는 구체적인 클래스를 사용하는 것이 좋습니다.

> info **힌트** DTO를 가져올 때, 타입 전용 import는 런타임 시 제거되므로 사용하면 안 됩니다. 즉, `import {{ '{' }} CreateUserDto {{ '}' }}`를 사용해야 하며 `import type {{ '{' }} CreateUserDto {{ '}' }}`는 사용하지 마십시오.

이제 `CreateUserDto`에 몇 가지 검증 규칙을 추가할 수 있습니다. 이는 `class-validator` 패키지가 제공하는 데코레이터를 사용하여 수행됩니다. 자세한 내용은 [여기](https://github.com/typestack/class-validator#validation-decorators)에서 확인할 수 있습니다. 이 방식으로 `CreateUserDto`를 사용하는 모든 라우트는 자동으로 이러한 검증 규칙

을 적용받습니다.

```typescript
import { IsEmail, IsNotEmpty } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsNotEmpty()
  password: string;
}
```

이 규칙이 적용되면 요청 본문에 잘못된 `email` 속성이 포함된 요청이 엔드포인트에 도달할 경우, 애플리케이션은 자동으로 `400 Bad Request` 코드를 반환하고 다음과 같은 응답 본문을 제공합니다:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": ["email must be an email"]
}
```

`ValidationPipe`는 요청 본문뿐만 아니라 다른 요청 객체 속성에도 사용할 수 있습니다. 예를 들어, 엔드포인트 경로에서 `:id`를 받아들이고자 할 때, 이 요청 매개변수에 대해 숫자만 허용하도록 다음과 같이 구성할 수 있습니다:

```typescript
@Get(':id')
findOne(@Param() params: FindOneParams) {
  return 'This action returns a user';
}
```

`FindOneParams`는 DTO와 마찬가지로 `class-validator`를 사용하여 검증 규칙을 정의하는 클래스입니다. 다음과 같이 작성됩니다:

```typescript
import { IsNumberString } from 'class-validator';

export class FindOneParams {
  @IsNumberString()
  id: number;
}
```

#### 자세한 오류 메시지 비활성화

오류 메시지는 요청의 잘못된 부분을 설명하는 데 도움이 될 수 있습니다. 그러나 일부 프로덕션 환경에서는 자세한 오류 메시지를 비활성화하는 것을 선호합니다. 이는 `ValidationPipe`에 옵션 객체를 전달하여 수행할 수 있습니다:

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    disableErrorMessages: true,
  }),
);
```

이 결과로 인해 자세한 오류 메시지가 응답 본문에 표시되지 않습니다.

#### 속성 제거

`ValidationPipe`는 메서드 핸들러에서 수신하지 않아야 하는 속성을 필터링할 수도 있습니다. 이 경우 허용된 속성을 **화이트리스트**하고 화이트리스트에 포함되지 않은 속성은 결과 객체에서 자동으로 제거됩니다. 예를 들어, 핸들러에서 `email`과 `password` 속성을 기대하지만 요청에 `age` 속성도 포함된 경우, 이 속성을 DTO에서 자동으로 제거할 수 있습니다. 이러한 동작을 활성화하려면 `whitelist`를 `true`로 설정하십시오.

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
  }),
);
```

이렇게 설정하면, 화이트리스트에 포함되지 않은 속성(검증 클래스에 데코레이터가 없는 속성)이 자동으로 제거됩니다.

대안으로, 화이트리스트에 포함되지 않은 속성이 있는 경우 요청 처리를 중지하고 사용자에게 오류 응답을 반환할 수 있습니다. 이를 활성화하려면 `whitelist`를 `true`로 설정하고 `forbidNonWhitelisted` 옵션 속성을 `true`로 설정하십시오.

<app-banner-courses></app-banner-courses>

#### 페이로드 객체 변환

네트워크를 통해 들어오는 페이로드는 일반 JavaScript 객체입니다. `ValidationPipe`는 페이로드를 해당 DTO 클래스에 따라 타이핑된 객체로 자동 변환할 수 있습니다. 자동 변환을 활성화하려면 `transform`을 `true`로 설정하십시오. 이는 메서드 수준에서 설정할 수 있습니다:

```typescript
@Post()
@UsePipes(new ValidationPipe({ transform: true }))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

이 동작을 전역적으로 활성화하려면 글로벌 파이프에서 옵션을 설정하십시오:

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,
  }),
);
```

자동 변환 옵션이 활성화되면 `ValidationPipe`는 기본 유형의 변환도 수행합니다. 다음 예에서는 `findOne()` 메서드가 경로 매개변수에서 추출된 `id`를 인수로 사용합니다:

```typescript
@Get(':id')
findOne(@Param('id') id: number) {
  console.log(typeof id === 'number'); // true
  return 'This action returns a user';
}
```

기본적으로 모든 경로 매개변수와 쿼리 매개변수는 네트워크를 통해 `string`으로 전달됩니다. 위의 예에서, 메서드 시그니처에서 `id` 유형을 `number`로 지정했으므로 `ValidationPipe`는 문자열 식별자를 숫자로 자동 변환하려고 시도합니다.

#### 명시적 변환

위의 섹션에서는 `ValidationPipe`가 기대하는 유형에 따라 쿼리 및 경로 매개변수를 암시적으로 변환하는 방법을 설명했습니다. 그러나 이 기능은 자동 변환이 활성화되어야 합니다.

대안으로(자동 변환이 비활성화된 경우), `ParseIntPipe` 또는 `ParseBoolPipe`를 사용하여 값을 명시적으로 캐스팅할 수 있습니다(기본적으로 모든 경로 매개변수와 쿼리 매개변수는 `string`으로 전달되므로 `ParseStringPipe`는 필요하지 않습니다).

```typescript
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('sort', ParseBoolPipe) sort: boolean,
) {
  console.log(typeof id === 'number'); // true
  console.log(typeof sort === 'boolean'); // true
  return 'This action returns a user';
}
```

> info **힌트** `ParseIntPipe` 및 `ParseBoolPipe`는 `@nestjs/common` 패키지에서 내보내집니다.

#### 매핑된 타입

**CRUD**(생성/읽기/업데이트/삭제)와 같은 기능을 구축할 때 기본 엔티티 유형의 변형을 작성하는 것이 유용할 때가 많습니다. Nest는 이러한 작업을 더 편리하게 만들기 위해 여러 타입 변환 유틸리티 함수를 제공합니다.

> **경고** 애플리케이션이 `@nestjs/swagger` 패키지를 사용하는 경우, 매핑된 타입에 대한 자세한 내용은 [이 장](https://docs.nestjs.com/openapi/mapped-types)을 참조하십시오. 마찬가지로, `@nestjs/graphql` 패키지를 사용하는 경우 [이 장](https://docs.nestjs.com/graphql/mapped-types)을 참조하십시오. 두 패키지는 타입에 크게 의존하므로, 서로 다른 import를 사용해야 합니다. 따라서, `@nestjs/mapped-types`를 사용하면(적절한 패키지가 아닌 경우, `@nestjs/swagger` 또는 `@nestjs/graphql`), 문서화되지 않은 다양한 부작용이 발생할 수 있습니다.

입력 검증 유형(또는 DTO)을 작성할 때, 동일한 유형의 **생성** 및 **업데이트** 변형을 작성하는 것이 유용할 때가 많습니다. 예를 들어, **생성** 변형은 모든 필드를 요구할 수 있지만, **업데이트** 변형은 모든 필드를 선택 사항으로 만들 수 있습니다.

Nest는 이 작업을 더 쉽게 만들고 보일러플레이트를 최소화하기 위해 `PartialType()` 유틸리티 함수를 제공합니다.

`PartialType()` 함수는 입력 유형의 모든 속성을 선택 사항으로 설정하여 유형(클래스)을 반환합니다. 예를 들어, 다음과 같은 **생성** 유형이 있다고 가정해 보겠습니다:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

기본적으로 이러한 필드는 모두 필수입니다. 동일한 필드를 가진 유형을 생성하되, 각 필드를 선택 사항으로 설정하려면 `PartialType()`을 사용하여 클래스 참조(`CreateCatDto`)를 인수로 전달합니다:

```typescript
export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

> info **힌트** `PartialType()` 함수는 `@nestjs/mapped-types` 패키지에서 가져옵니다.

`PickType()` 함수는 입력 유형에서 속성 집합을 선택하여 새 유형(클래스)을 구성합니다. 예를 들어, 다음과 같은 유형이 있다고 가정해 보겠습니다:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

`PickType()` 유틸리티 함수를 사용하여 이 클래스에서 속성 집합을 선택할 수 있습니다:

```typescript
export class UpdateCatAgeDto extends PickType(CreateCatDto, ['age'] as const) {}
```

> info **힌트** `PickType()` 함수는 `@nestjs/mapped-types` 패키지에서 가져옵니다.

`OmitType()` 함수는 입력 유형에서 모든 속성을 선택한 다음 특정 키 세트를 제거하여 유형을 구성합니다. 예를 들어, 다음과 같은 유형이 있다고 가정해 보겠습니다:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

다음과 같이 `name` 속성을 제외한 모든 속성을 가진 파생 유형을 생성할 수 있습니다. 이 구성에서 `OmitType`의 두 번째 인수는 속성 이름

 배열입니다.

```typescript
export class UpdateCatDto extends OmitType(CreateCatDto, ['name'] as const) {}
```

> info **힌트** `OmitType()` 함수는 `@nestjs/mapped-types` 패키지에서 가져옵니다.

`IntersectionType()` 함수는 두 유형을 하나의 새 유형(클래스)으로 결합합니다. 예를 들어, 다음과 같은 두 가지 유형이 있다고 가정해 보겠습니다:

```typescript
export class CreateCatDto {
  name: string;
  breed: string;
}

export class AdditionalCatInfo {
  color: string;
}
```

두 유형의 모든 속성을 결합한 새 유형을 생성할 수 있습니다.

```typescript
export class UpdateCatDto extends IntersectionType(
  CreateCatDto,
  AdditionalCatInfo,
) {}
```

> info **힌트** `IntersectionType()` 함수는 `@nestjs/mapped-types` 패키지에서 가져옵니다.

타입 매핑 유틸리티 함수는 조합 가능(composable)합니다. 예를 들어, 다음 코드는 `CreateCatDto` 유형의 모든 속성을 가지되 `name` 속성을 제외하고, 이러한 속성을 선택 사항으로 설정한 유형(클래스)을 생성합니다:

```typescript
export class UpdateCatDto extends PartialType(
  OmitType(CreateCatDto, ['name'] as const),
) {}
```

#### 배열 파싱 및 검증

TypeScript는 제네릭 또는 인터페이스에 대한 메타데이터를 저장하지 않으므로 DTO에서 이를 사용할 때 `ValidationPipe`가 들어오는 데이터를 올바르게 검증하지 못할 수 있습니다. 예를 들어, 다음 코드에서 `createUserDtos`는 올바르게 검증되지 않습니다:

```typescript
@Post()
createBulk(@Body() createUserDtos: CreateUserDto[]) {
  return 'This action adds new users';
}
```

배열을 검증하려면 배열을 래핑하는 속성을 포함하는 전용 클래스를 생성하거나 `ParseArrayPipe`를 사용할 수 있습니다.

```typescript
@Post()
createBulk(
  @Body(new ParseArrayPipe({ items: CreateUserDto }))
  createUserDtos: CreateUserDto[],
) {
  return 'This action adds new users';
}
```

또한, `ParseArrayPipe`는 쿼리 매개변수를 파싱할 때 유용할 수 있습니다. 식별자가 쿼리 매개변수로 전달되는 `findByIds()` 메서드를 고려해 보겠습니다.

```typescript
@Get()
findByIds(
  @Query('ids', new ParseArrayPipe({ items: Number, separator: ',' }))
  ids: number[],
) {
  return 'This action returns users by ids';
}
```

이 구성은 HTTP `GET` 요청에서 다음과 같은 쿼리 매개변수를 유효성 검사합니다:

```bash
GET /?ids=1,2,3
```

#### WebSockets 및 마이크로서비스

이 챕터에서는 HTTP 스타일 애플리케이션(예: Express 또는 Fastify)을 사용하는 예제를 보여주지만, `ValidationPipe`는 사용되는 전송 방식에 관계없이 WebSockets 및 마이크로서비스에서도 동일하게 작동합니다.

#### 더 알아보기

사용자 정의 검증기, 오류 메시지 및 `class-validator` 패키지가 제공하는 사용 가능한 데코레이터에 대해 더 알아보려면 [여기](https://github.com/typestack/class-validator)에서 읽어보십시오.
