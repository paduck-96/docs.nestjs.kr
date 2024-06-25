### Pipes

Pipe는 `@Injectable()` 데코레이터로 주석이 달린 클래스이며, `PipeTransform` 인터페이스를 구현합니다.

<figure>
  <img src="https://docs.nestjs.com/assets/Pipe_1.png" />
</figure>

Pipe는 두 가지 주요 사용 사례가 있습니다:

- **변환**: 입력 데이터를 원하는 형태로 변환합니다(예: 문자열을 정수로 변환).
- **검증**: 입력 데이터를 평가하고 유효한 경우 변경 없이 그대로 통과시키며, 그렇지 않으면 예외를 발생시킵니다.

두 경우 모두, 파이프는 <a href="https://docs.nestjs.com/controllers#route-parameters">컨트롤러 라우트 핸들러</a>에서 처리되는 `arguments`에 대해 작동합니다. Nest는 메서드가 호출되기 직전에 파이프를 개입시키며, 파이프는 메서드에 전달될 인수를 받아서 작동합니다. 변환 또는 검증 작업은 이 시점에 이루어지며, 이후 라우트 핸들러는 변환된 인수와 함께 호출됩니다.

Nest는 기본적으로 사용할 수 있는 여러 내장 파이프를 제공합니다. 또한 사용자 정의 파이프를 만들 수도 있습니다. 이 장에서는 내장 파이프를 소개하고 이를 라우트 핸들러에 바인딩하는 방법을 보여줍니다. 그런 다음 사용자 정의 파이프를 몇 가지 예시로 보여주어 처음부터 파이프를 만드는 방법을 설명합니다.

> info **힌트** 파이프는 예외 영역 내에서 실행됩니다. 이는 파이프가 예외를 발생시키면 해당 예외가 예외 레이어(전역 예외 필터 및 현재 컨텍스트에 적용된 [예외 필터](https://docs.nestjs.com/exception-filters))에 의해 처리됨을 의미합니다. 따라서 파이프에서 예외가 발생하면 이후의 컨트롤러 메서드는 실행되지 않습니다. 이는 외부 소스로부터 애플리케이션으로 들어오는 데이터를 검증하는 모범 사례를 제공합니다.

#### 내장 파이프

Nest는 기본적으로 사용할 수 있는 9개의 파이프를 제공합니다:

- `ValidationPipe`
- `ParseIntPipe`
- `ParseFloatPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `ParseEnumPipe`
- `DefaultValuePipe`
- `ParseFilePipe`

이들은 `@nestjs/common` 패키지에서 가져올 수 있습니다.

`ParseIntPipe`를 사용해보겠습니다. 이는 **변환** 사용 사례의 예시로, 파이프가 메서드 핸들러 매개변수를 JavaScript 정수로 변환하거나 변환이 실패하면 예외를 발생시킵니다. 이 장의 뒷부분에서는 `ParseIntPipe`의 간단한 사용자 정의 구현을 보여줍니다. 아래 예제 기술은 다른 내장 변환 파이프(`ParseBoolPipe`, `ParseFloatPipe`, `ParseEnumPipe`, `ParseArrayPipe`, `ParseUUIDPipe`)에도 적용됩니다.

#### 파이프 바인딩

파이프를 사용하려면 적절한 컨텍스트에 파이프 클래스의 인스턴스를 바인딩해야 합니다. `ParseIntPipe` 예제에서, 특정 라우트 핸들러 메서드와 파이프를 연결하고 메서드가 호출되기 전에 실행되도록 합니다. 이를 메서드 매개변수 수준에서 파이프를 바인딩하는 것으로 언급하겠습니다:

```typescript
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

이렇게 하면 `findOne()` 메서드에서 매개변수를 숫자로 받거나, 라우트 핸들러가 호출되기 전에 예외가 발생하게 됩니다.

예를 들어, 다음과 같이 호출하면:

```bash
GET localhost:3000/abc
```

Nest는 다음과 같은 예외를 발생시킵니다:

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

예외는 `findOne()` 메서드 본문이 실행되지 않도록 방지합니다.

위 예제에서는 클래스(`ParseIntPipe`)를 전달했으며, 인스턴스화는 프레임워크에 맡기고 의존성 주입을 활성화했습니다. 파이프와 가드와 마찬가지로 인스턴스를 전달할 수도 있습니다. 이는 옵션을 전달하여 내장 파이프의 동작을 사용자 정의할 때 유용합니다:

```typescript
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

다른 변환 파이프(모든 **Parse\*** 파이프) 바인딩도 유사하게 작동합니다. 이 파이프들은 모두 라우트 매개변수, 쿼리 문자열 매개변수 및 요청 본문 값의 검증 컨텍스트에서 작동합니다.

예를 들어, 쿼리 문자열 매개변수를 사용할 때:

```typescript
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

문자열 매개변수를 구문 분석하고 UUID인지 확인하기 위해 `ParseUUIDPipe`를 사용하는 예제입니다.

```typescript
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
```

> info **힌트** `ParseUUIDPipe()`를 사용할 때 UUID의 버전 3, 4 또는 5를 구문 분석하며, 특정 버전의 UUID만 필요한 경우 파이프 옵션에서 버전을 전달할 수 있습니다.

위에서 `Parse*` 패밀리의 여러 내장 파이프를 바인딩하는 예제를 보았습니다. 검증 파이프 바인딩은 약간 다릅니다. 이에 대해 다음 섹션에서 설명하겠습니다.

> info **힌트** 검증 파이프의 광범위한 예제는 [검증 기술](https://docs.nestjs.com/techniques/validation)을 참조하세요.

#### 사용자 정의 파이프

언급했듯이, 사용자 정의 파이프를 만들 수 있습니다. Nest는 강력한 내장 `ParseIntPipe`와 `ValidationPipe`를 제공하지만, 사용자 정의 버전을 처음부터 만드는 방법을 살펴보겠습니다.

간단한 `ValidationPipe`부터 시작하겠습니다. 처음에는 입력 값을 받아 그대로 반환하여 동일성 함수처럼 동작하도록 합니다.

```typescript
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

> info **힌트** `PipeTransform<T, R>`은 모든 파이프가 구현해야 하는 제네릭 인터페이스입니다. 제네릭 인터페이스는 `T`를 입력 값의 유형으로 사용하고, `R`을 `transform()` 메서드의 반환 유형으로 사용합니다.

모든 파이프는 `PipeTransform` 인터페이스 계약을 이행하기 위해 `transform()` 메서드를 구현해야 합니다. 이 메서드는 두 개의 매개변수를 가집니다:

- `value`
- `metadata`

`value` 매개변수는 현재 처리 중인 메서드 인수이며, `metadata`는 현재 처리 중인 메서드 인수의 메타데이터입니다. 메타데이터 객체에는 다음과 같은 속성이 있습니다:

```typescript
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

이 속성들은 현재 처리 중인 인수를 설명합니다.

<table>
  <tr>
    <td>
      <code>type</code>
    </td>
    <td>인수가 본문인지
      <code>@Body()</code>, 쿼리인지
      <code>@Query()</code>, 매개변수인지
      <code>@Param()</code>, 또는 사용자 정의 매개변수인지 나타냅니다 (자세한 내용은
      <a href="https://docs.nestjs.com/custom-decorators">여기</a>를 참조하세요).</td>
  </tr>
  <tr>
    <td>
      <code>metatype</code>
    </td>
    <td>
      인수의 메타타입을 제공합니다. 예를 들어,
      <code>String</code>. 메서드 서명에서 타입 선언을 생략하거나 기본 JavaScript를 사용하는 경우 값은
      <code>undefined</code>입니다.
    </td>
  </tr>
  <tr>
    <td>
      <code>data</code>
    </td>
    <td>데코레이터에 전달된 문자열입니다. 예를 들어
      <code>@Body('string')</code>. 데코레이터 괄호를 비워두면


      <code>undefined</code>입니다.</td>
  </tr>
</table>

> warning **경고** TypeScript 인터페이스는 트랜스파일 시 사라집니다. 따라서 메서드 매개변수의 타입이 클래스 대신 인터페이스로 선언된 경우 `metatype` 값은 `Object`가 됩니다.

#### 스키마 기반 검증

검증 파이프를 좀 더 유용하게 만들어 보겠습니다. `CatsController`의 `create()` 메서드를 자세히 살펴보면, 게시물 본문 객체가 유효한지 확인하고 서비스 메서드를 실행하려고 합니다.

```typescript
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

`createCatDto` 본문 매개변수에 주목하십시오. 그 타입은 `CreateCatDto`입니다:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

`create` 메서드로 들어오는 모든 요청에 유효한 본문이 포함되어 있는지 확인하고 싶습니다. 따라서 `createCatDto` 객체의 세 멤버를 검증해야 합니다. 이를 라우트 핸들러 메서드 내부에서 수행할 수도 있지만, 이는 **단일 책임 원칙**(SRP)을 위반하게 됩니다.

다른 접근 방법으로 **검증자 클래스**를 만들어 그 작업을 위임할 수 있습니다. 이 경우 단점은 각 메서드의 시작 부분에서 이 검증자를 호출해야 한다는 점입니다.

검증 미들웨어를 만드는 방법도 있습니다. 그러나 불행히도 애플리케이션 전체에서 모든 컨텍스트에서 사용할 수 있는 **일반 미들웨어**를 만들 수는 없습니다. 미들웨어는 **실행 컨텍스트**를 인식하지 못하기 때문입니다. 여기에는 호출될 핸들러와 그 매개변수가 포함됩니다.

이는 파이프가 설계된 정확한 사용 사례입니다. 따라서 검증 파이프를 구체화해 보겠습니다.

#### 객체 스키마 검증

객체 검증을 깔끔하고 [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) 방식으로 수행하는 몇 가지 접근 방법이 있습니다. 일반적인 접근 방법 중 하나는 **스키마 기반** 검증을 사용하는 것입니다. 이 접근 방식을 시도해 보겠습니다.

[Zod](https://zod.dev/) 라이브러리는 읽기 쉬운 API로 스키마를 쉽게 만들 수 있게 해줍니다. Zod 기반 스키마를 사용하는 검증 파이프를 만들어 보겠습니다.

먼저 필요한 패키지를 설치합니다:

```bash
$ npm install --save zod
```

아래 코드 샘플에서는 스키마를 `constructor` 인수로 받는 간단한 클래스를 만듭니다. 그런 다음 `schema.parse()` 메서드를 적용하여 제공된 스키마에 대해 들어오는 인수를 검증합니다.

앞서 언급했듯이, **검증 파이프**는 값을 변경하지 않고 반환하거나 예외를 발생시킵니다.

다음 섹션에서는 `@UsePipes()` 데코레이터를 사용하여 컨트롤러 메서드에 적절한 스키마를 제공하는 방법을 보여줍니다. 이를 통해 검증 파이프를 다양한 컨텍스트에서 재사용할 수 있습니다.

```typescript
import { PipeTransform, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ZodSchema  } from 'zod';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
```

#### 검증 파이프 바인딩

앞서 `ParseIntPipe` 및 다른 `Parse*` 파이프와 같은 변환 파이프를 바인딩하는 방법을 보았습니다.

검증 파이프를 바인딩하는 것도 매우 간단합니다.

이 경우 메서드 호출 수준에서 파이프를 바인딩하려고 합니다. 현재 예제에서는 `ZodValidationPipe`를 사용하려면 다음을 수행해야 합니다:

1. `ZodValidationPipe`의 인스턴스 생성
2. 파이프의 클래스 생성자에 컨텍스트별 Zod 스키마 전달
3. 메서드에 파이프 바인딩

Zod 스키마 예제:

```typescript
import { z } from 'zod';

export const createCatSchema = z
  .object({
    name: z.string(),
    age: z.number(),
    breed: z.string(),
  })
  .required();

export type CreateCatDto = z.infer<typeof createCatSchema>;
```

이를 `@UsePipes()` 데코레이터를 사용하여 다음과 같이 바인딩합니다:

```typescript
@Post()
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **힌트** `@UsePipes()` 데코레이터는 `@nestjs/common` 패키지에서 가져옵니다.

> warning **경고** `zod` 라이브러리는 `tsconfig.json` 파일에서 `strictNullChecks` 구성이 활성화되어 있어야 합니다.

#### 클래스 검증기

> warning **경고** 이 섹션의 기술은 TypeScript를 필요로 하며, 애플리케이션이 기본 JavaScript로 작성된 경우에는 사용할 수 없습니다.

검증 기술에 대한 다른 구현을 살펴보겠습니다.

Nest는 [class-validator](https://github.com/typestack/class-validator) 라이브러리와 잘 작동합니다. 이 강력한 라이브러리를 사용하면 데코레이터 기반 검증을 사용할 수 있습니다. 데코레이터 기반 검증은 특히 Nest의 **Pipe** 기능과 결합할 때 매우 강력합니다. 왜냐하면 처리 중인 속성의 `metatype`에 접근할 수 있기 때문입니다. 시작하기 전에 필요한 패키지를 설치해야 합니다:

```bash
$ npm i --save class-validator class-transformer
```

설치가 완료되면 `CreateCatDto` 클래스에 몇 가지 데코레이터를 추가할 수 있습니다. 여기서는 중요한 이점을 볼 수 있습니다: `CreateCatDto` 클래스는 게시물 본문 객체에 대한 단일 소스가 됩니다(따로 검증 클래스를 만들 필요가 없습니다).

```typescript
import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

> info **힌트** class-validator 데코레이터에 대한 자세한 내용은 [여기](https://github.com/typestack/class-validator#usage)를 참조하세요.

이제 이러한 주석을 사용하는 `ValidationPipe` 클래스를 만들 수 있습니다.

```typescript
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

> info **힌트** 다시 한 번 말하지만, 기본적으로 제공되는 `ValidationPipe`를 사용할 수 있으므로 자체적으로 일반적인 검증 파이프를 만들 필요가 없습니다. 내장된 `ValidationPipe`는 이 장에서 설명한 예제보다 더 많은 옵션을 제공합니다. 자세한 내용과 많은 예제는 [여기](https://docs.nestjs.com/techniques/validation)를 참조하세요.

> warning **알림** 위에서는 **class-validator** 라이브러리와 같은 저자가 만든 [class-transformer](https://github.com/typestack/class-transformer) 라이브러리를 사용했습니다. 결과적으로 이 두 라이브러리는 잘 함께 작동합니다.

이 코드를 살펴보겠습니다. 먼저 `transform()` 메서드가 `async`로 표시된 것을 주목하십시오. 이는 Nest가 동기 및 **비동기** 파이프를 모두 지원하기 때문에 가능합니다. 이 메서드를 `async`로 만든 이유는 일부 class-validator 검증이 [비동기적](https://github.com/typestack/class-validator#custom-validation-classes)일 수 있기 때문입니다(Promise를 사용).

다음으로, 구조 분해 할당을 사용하여 `ArgumentMetadata`의 `metatype` 필드를 추출하여 `metatype` 매개변수에 할당합니다. 이는 전체 `ArgumentMetadata`를 가져온 다음 `

metatype` 변수를 할당하는 추가 문장을 가지는 것의 약식입니다.

다음으로 `toValidate()` 도우미 함수를 주목하십시오. 이 함수는 현재 처리 중인 인수가 기본 JavaScript 타입인 경우 검증 단계를 우회하는 역할을 합니다(이들은 검증 데코레이터를 부착할 수 없으므로 검증 단계에 통과할 이유가 없습니다).

다음으로, class-transformer 함수 `plainToInstance()`를 사용하여 평범한 JavaScript 인수 객체를 타입화된 객체로 변환하여 검증을 적용할 수 있도록 합니다. 이렇게 해야 하는 이유는 네트워크 요청에서 역직렬화된 게시물 본문 객체가 **아무런 타입 정보도** 갖고 있지 않기 때문입니다(이는 Express와 같은 기본 플랫폼이 작동하는 방식입니다). class-validator는 앞서 DTO에 정의한 검증 데코레이터를 사용해야 하므로, 본문을 평범한 객체가 아닌 적절하게 주석이 달린 객체로 취급하기 위해 이 변환을 수행해야 합니다.

마지막으로, 앞서 언급했듯이, 이는 **검증 파이프**이므로 값을 변경하지 않고 반환하거나 예외를 발생시킵니다.

마지막 단계는 `ValidationPipe`를 바인딩하는 것입니다. 파이프는 매개변수 범위, 메서드 범위, 컨트롤러 범위 또는 전역 범위로 설정할 수 있습니다. 이전에 Zod 기반 검증 파이프와 함께 메서드 수준에서 파이프를 바인딩하는 예제를 보았습니다.
아래 예제에서는 파이프 인스턴스를 `@Body()` 데코레이터에 바인딩하여 파이프가 게시물 본문을 검증하도록 합니다.

```typescript
@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

매개변수 범위 파이프는 검증 논리가 지정된 매개변수에만 관련된 경우 유용합니다.

#### 전역 범위 파이프

`ValidationPipe`는 가능한 한 일반적으로 만들어졌기 때문에 전체 애플리케이션의 모든 라우트 핸들러에 적용되도록 **전역 범위** 파이프로 설정하여 그 유용성을 최대화할 수 있습니다.

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

> warning **알림** <a href="https://docs.nestjs.com/faq/hybrid-application">하이브리드 앱</a>의 경우 `useGlobalPipes()` 메서드는 게이트웨이 및 마이크로서비스에 대한 파이프를 설정하지 않습니다. "표준"(비 하이브리드) 마이크로서비스 앱의 경우 `useGlobalPipes()`는 파이프를 전역적으로 마운트합니다.

전역 파이프는 애플리케이션 전체, 모든 컨트롤러 및 모든 라우트 핸들러에서 사용됩니다.

의존성 주입 측면에서, 모듈 외부에서 등록된 전역 파이프(`useGlobalPipes()`를 사용하여 위의 예제처럼)는 모듈 외부에서 바인딩되었기 때문에 종속성을 주입할 수 없습니다. 이 문제를 해결하려면 다음 구성을 사용하여 **모듈 내에서 직접** 전역 파이프를 설정할 수 있습니다:

```typescript
import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

> info **힌트** 이 방법을 사용하여 파이프에 대한 의존성 주입을 수행할 때, 이 구성이 사용되는 모듈에 상관없이 파이프는 실제로 전역적입니다. 어디에서 이 작업을 수행해야 할까요? 파이프(`ValidationPipe`가 정의된 모듈)를 선택하세요. 또한 `useClass`는 사용자 정의 provider 등록을 처리하는 유일한 방법이 아닙니다. 자세한 내용은 [여기](https://docs.nestjs.com/fundamentals/custom-providers)를 참조하세요.

#### 내장 ValidationPipe

다시 한 번 말하지만, 자체적으로 일반적인 검증 파이프를 만들 필요는 없습니다. Nest는 기본적으로 `ValidationPipe`를 제공하기 때문입니다. 내장된 `ValidationPipe`는 이 장에서 설명한 예제보다 더 많은 옵션을 제공합니다. 자세한 내용과 많은 예제는 [여기](https://docs.nestjs.com/techniques/validation)를 참조하세요.

#### 변환 사용 사례

검증은 사용자 정의 파이프의 유일한 사용 사례가 아닙니다. 이 장의 시작 부분에서 언급했듯이, 파이프는 입력 데이터를 원하는 형식으로 **변환**할 수도 있습니다. 이는 `transform` 함수에서 반환된 값이 인수의 이전 값을 완전히 덮어쓰므로 가능합니다.

이것이 언제 유용할까요? 때때로 클라이언트에서 전달된 데이터는 라우트 핸들러 메서드에서 올바르게 처리되기 전에 일부 변경을 거쳐야 할 때가 있습니다. 예를 들어 문자열을 정수로 변환하는 경우가 있습니다. 또한 일부 필수 데이터 필드가 누락될 수 있으며 기본값을 적용하고자 할 수 있습니다. **변환 파이프**는 클라이언트 요청과 요청 핸들러 사이에 처리 함수를 개입시킴으로써 이러한 기능을 수행할 수 있습니다.

다음은 문자열을 정수 값으로 구문 분석하는 책임이 있는 간단한 `ParseIntPipe`입니다. (위에서 언급했듯이, Nest에는 더 정교한 내장 `ParseIntPipe`가 있습니다. 여기서는 간단한 사용자 정의 변환 파이프의 예로 포함했습니다).

```typescript
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

이 파이프를 선택한 매개변수에 다음과 같이 바인딩할 수 있습니다:

```typescript
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id) {
  return this.catsService.findOne(id);
}
```

다른 유용한 변환 사례는 요청에서 제공된 id를 사용하여 데이터베이스에서 **기존 사용자** 엔터티를 선택하는 것입니다:

```typescript
@Get(':id')
findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
  return userEntity;
}
```

이 파이프의 구현은 독자에게 맡기지만, 다른 변환 파이프와 마찬가지로 입력 값(id)을 받고 출력 값(UserEntity 객체)을 반환한다는 점에 유의하세요. 이를 통해 핸들러의 보일러플레이트 코드를 일반 파이프로 추상화하여 더 선언적이고 [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)하게 만들 수 있습니다.

#### 기본값 제공

`Parse*` 파이프는 매개변수의 값이 정의되어 있어야 합니다. `null` 또는 `undefined` 값을 받으면 예외를 발생시킵니다. 쿼리 문자열 매개변수 값이 누락된 경우 기본값을 제공하여 `Parse*` 파이프가 이러한 값에 대해 작동하기 전에 주입할 수 있습니다. `DefaultValuePipe`는 이러한 목적을 수행합니다. 아래와 같이 `@Query()` 데코레이터에서 관련 `Parse*` 파이프 전에 `DefaultValuePipe`를 인스턴스화하면 됩니다:

```typescript
@Get()
async findAll(
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
) {
  return this.catsService.findAll({ activeOnly, page });
}
```
