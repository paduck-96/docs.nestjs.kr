### 컨트롤러

컨트롤러는 들어오는 **요청**을 처리하고 클라이언트에 **응답**을 반환하는 역할을 합니다.

<figure><img src="https://docs.nestjs.com/assets/Controllers_1.png" /></figure>

컨트롤러의 목적은 애플리케이션에 대한 특정 요청을 받는 것입니다. **라우팅** 메커니즘은 어떤 컨트롤러가 어떤 요청을 받을지를 제어합니다. 종종 각 컨트롤러는 여러 라우트를 가지고 있으며, 각기 다른 라우트는 다른 동작을 수행할 수 있습니다.

기본적인 컨트롤러를 만들기 위해서는 클래스와 **데코레이터**를 사용합니다. 데코레이터는 클래스와 필요한 메타데이터를 연관시키고 Nest가 라우팅 맵(요청을 해당 컨트롤러에 연결)을 생성할 수 있게 합니다.

> info **힌트** [검증](https://docs.nestjs.com/techniques/validation)이 내장된 CRUD 컨트롤러를 빠르게 생성하려면 CLI의 [CRUD 생성기](https://docs.nestjs.com/recipes/crud-generator#crud-generator)를 사용할 수 있습니다: `nest g resource [name]`.

#### 라우팅

다음 예제에서는 기본 컨트롤러를 정의하기 위해 필수적인 `@Controller()` 데코레이터를 사용할 것입니다. 여기서 `cats`라는 경로 접두사를 지정합니다. `@Controller()` 데코레이터에서 경로 접두사를 지정하면 관련된 라우트들을 쉽게 그룹화하고 반복되는 코드를 최소화할 수 있습니다. 예를 들어, 고양이 엔티티와의 상호작용을 관리하는 라우트들을 `/cats` 경로 아래에 그룹화할 수 있습니다. 이 경우, `@Controller()` 데코레이터에 `cats` 접두사를 지정하면 파일 내의 각 라우트에 대해 이 경로를 반복할 필요가 없습니다.

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

> info **힌트** CLI를 사용하여 컨트롤러를 생성하려면 `$ nest g controller [name]` 명령어를 실행하세요.

`findAll()` 메서드 앞에 있는 `@Get()` HTTP 요청 메서드 데코레이터는 특정 엔드포인트에 대한 HTTP 요청 핸들러를 생성하라고 Nest에 지시합니다. 엔드포인트는 HTTP 요청 메서드(GET)와 라우트 경로에 대응됩니다. 라우트 경로는 컨트롤러에 선언된 (선택적인) 접두사와 메서드의 데코레이터에 지정된 경로를 연결하여 결정됩니다. 위 예제에서는 모든 라우트에 대해 `cats` 접두사를 선언했고, 데코레이터에 경로 정보를 추가하지 않았으므로, Nest는 `GET /cats` 요청을 이 핸들러로 매핑합니다. 경로는 선택적인 컨트롤러 경로 접두사와 요청 메서드 데코레이터에 선언된 경로 문자열을 모두 포함합니다. 예를 들어, `cats` 접두사와 `@Get('breed')` 데코레이터를 결합하면 `GET /cats/breed` 요청에 대한 라우트 매핑이 생성됩니다.

위 예제에서는 GET 요청이 이 엔드포인트로 들어오면 Nest는 요청을 사용자 정의 `findAll()` 메서드로 라우팅합니다. 여기서 메서드 이름은 완전히 임의적입니다. 메서드를 선언해야만 라우트와 바인딩할 수 있지만, Nest는 선택된 메서드 이름에 어떤 의미도 부여하지 않습니다.

이 메서드는 200 상태 코드와 연관된 응답(이 경우 문자열)을 반환합니다. 왜 그런지 설명하자면, Nest는 응답을 조작하는 두 가지 **다른** 옵션을 사용하기 때문입니다:

<table>
  <tr>
    <td>표준(권장)</td>
    <td>
      이 내장 방법을 사용하면 요청 핸들러가 자바스크립트 객체나 배열을 반환할 때 <strong>자동으로</strong> JSON으로 직렬화됩니다. 자바스크립트 기본 타입(예: <code>string</code>, <code>number</code>, <code>boolean</code>)을 반환할 때는 직렬화를 시도하지 않고 값만 전송합니다. 응답 처리가 간단해집니다: 값만 반환하면 되고 나머지는 Nest가 처리합니다.
      <br />
      <br /> 또한, 응답의 <strong>상태 코드</strong>는 기본적으로 항상 200이며, POST 요청은 201을 사용합니다. 이 동작을 변경하려면 핸들러 수준에서 <code>@HttpCode(...)</code> 데코레이터를 추가하면 됩니다(자세한 내용은 [상태 코드](https://docs.nestjs.com/controllers#status-code)를 참조하세요).
    </td>
  </tr>
  <tr>
    <td>라이브러리 특정</td>
    <td>
      <code>@Res()</code> 데코레이터를 사용하여 메서드 핸들러 서명에 라이브러리 특정(예: Express) <a href="https://expressjs.com/en/api.html#res" rel="nofollow" target="_blank">응답 객체</a>를 주입할 수 있습니다(예: <code>findAll(@Res() response)</code>). 이 접근 방식에서는 해당 객체가 노출하는 네이티브 응답 처리 메서드를 사용할 수 있습니다. 예를 들어, Express를 사용하면 <code>response.status(200).send()</code>와 같은 코드를 사용하여 응답을 구성할 수 있습니다.
    </td>
  </tr>
</table>

> warning **경고** 핸들러가 `@Res()` 또는 `@Next()`를 사용할 때 Nest는 라이브러리 특정 옵션을 선택한 것으로 감지합니다. 두 접근 방식을 동시에 사용하면 표준 접근 방식이 **자동으로 비활성화**되며 이 라우트에서는 더 이상 정상 작동하지 않습니다. 두 접근 방식을 동시에 사용하려면(예: 쿠키/헤더만 설정하고 나머지는 프레임워크에 맡기기 위해 응답 객체를 주입하는 경우), `@Res({ passthrough: true })` 데코레이터에서 `passthrough` 옵션을 `true`로 설정해야 합니다.

#### 요청 객체

핸들러는 종종 클라이언트의 **요청** 세부 정보에 액세스해야 합니다. Nest는 기본 플랫폼(기본적으로 Express)의 [요청 객체](https://expressjs.com/en/api.html#req)에 접근할 수 있도록 합니다. 핸들러 서명에 `@Req()` 데코레이터를 추가하여 요청 객체를 주입하도록 Nest에 지시할 수 있습니다.

```typescript
import { Controller, Get, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'This action returns all cats';
  }
}
```

> info **힌트** `express` 타입을 활용하려면(위 예제에서 `request: Request` 매개변수처럼) `@types/express` 패키지를 설치하세요.

요청 객체는 HTTP 요청을 나타내며 요청 쿼리 문자열, 매개변수, HTTP 헤더 및 본문에 대한 속성을 가지고 있습니다(자세한 내용은 [여기](https://expressjs.com/en/api.html#req)에서 읽을 수 있습니다). 대부분의 경우 이러한 속성을 수동으로 가져올 필요는 없습니다. 대신 `@Body()` 또는 `@Query()`와 같은 전용 데코레이터를 사용할 수 있으며, 이는 기본적으로 사용할 수 있습니다. 아래는 제공되는 데코레이터와 해당 플랫폼 특정 객체를 나타내는 목록입니다.

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td>
    </tr>
    <tr>
      <td><code>@Response(), @Res()</code><span class="table-code-asterisk">*</span></td>
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
      <td><code>@Param(key?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[key]</code></td>
    </tr>
    <tr>
      <td><code>@Body(key?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[key]</code></td>
    </tr>
    <tr>
      <td><code>@Query(key?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[key]</code></

td>
    </tr>
    <tr>
      <td><code>@Headers(name?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[name]</code></td>
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

<sup>\* </sup>기본 HTTP 플랫폼(예: Express 및 Fastify) 간의 타이핑 호환성을 위해 Nest는 `@Res()` 및 `@Response()` 데코레이터를 제공합니다. `@Res()`는 단순히 `@Response()`의 별칭입니다. 두 데코레이터 모두 기본 플랫폼 `response` 객체 인터페이스를 직접 노출합니다. 이를 사용할 때는 기본 라이브러리의 타이핑(예: `@types/express`)을 가져와서 최대한 활용하세요. 메서드 핸들러에 `@Res()` 또는 `@Response()`를 주입하면 해당 핸들러에 대해 Nest가 **라이브러리 특정 모드**로 전환되며, 응답을 관리할 책임이 귀하에게 있습니다. 이 경우 응답 객체에서 일부 호출(예: `res.json(...)` 또는 `res.send(...)`)을 통해 응답을 발행해야 하며, 그렇지 않으면 HTTP 서버가 중단됩니다.

> info **힌트** 사용자 정의 데코레이터를 만드는 방법을 배우려면 [여기](https://docs.nestjs.com/custom-decorators) 챕터를 참조하세요.

#### 리소스

앞서, 고양이 리소스를 가져오는 엔드포인트(**GET** 라우트)를 정의했습니다. 일반적으로 새 레코드를 생성하는 엔드포인트도 제공하고자 합니다. 이를 위해 **POST** 핸들러를 만들어 보겠습니다:

```typescript
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(): string {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

Nest는 `@Get()`, `@Post()`, `@Put()`, `@Delete()`, `@Patch()`, `@Options()`, `@Head()` 등 표준 HTTP 메서드에 대한 데코레이터를 모두 제공합니다. 또한, `@All()`은 모든 메서드를 처리하는 엔드포인트를 정의합니다.

#### 라우트 와일드카드

패턴 기반 라우트도 지원됩니다. 예를 들어, 별표는 와일드카드로 사용되며 문자 조합을 매칭합니다.

```typescript
@Get('ab*cd')
findAll() {
  return 'This route uses a wildcard';
}
```

`'ab*cd'` 라우트 경로는 `abcd`, `ab_cd`, `abecd` 등을 매칭합니다. `?`, `+`, `*`, `()` 문자는 라우트 경로에서 사용할 수 있으며 정규 표현식의 하위 집합입니다. 하이픈(`-`) 및 점(`.`)은 문자열 기반 경로에서 문자 그대로 해석됩니다.

> warning **경고** 라우트 중간의 와일드카드는 express에서만 지원됩니다.

#### 상태 코드

앞서 언급했듯이, 응답 **상태 코드**는 기본적으로 항상 **200**이며, POST 요청의 경우 **201**입니다. 핸들러 수준에서 `@HttpCode(...)` 데코레이터를 추가하여 이 동작을 쉽게 변경할 수 있습니다.

```typescript
@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

> info **힌트** `HttpCode`를 `@nestjs/common` 패키지에서 가져오세요.

종종 상태 코드는 정적이지 않고 다양한 요소에 따라 달라집니다. 이 경우 라이브러리 특정 **응답** 객체(`@Res()`를 사용하여 주입)나(오류가 발생한 경우 예외를 던져서) 사용할 수 있습니다.

#### 헤더

사용자 정의 응답 헤더를 지정하려면 `@Header()` 데코레이터나 라이브러리 특정 응답 객체(`res.header()`)를 직접 호출할 수 있습니다.

```typescript
@Post()
@Header('Cache-Control', 'none')
create() {
  return 'This action adds a new cat';
}
```

> info **힌트** `Header`를 `@nestjs/common` 패키지에서 가져오세요.

#### 리디렉션

응답을 특정 URL로 리디렉션하려면 `@Redirect()` 데코레이터나 라이브러리 특정 응답 객체(`res.redirect()`)를 직접 호출할 수 있습니다.

`@Redirect()`는 `url`과 `statusCode`의 두 가지 인수를 받으며, 둘 다 선택 사항입니다. `statusCode`의 기본값은 생략 시 `302`(Found)입니다.

```typescript
@Get()
@Redirect('https://nestjs.com', 301)
```

> info **힌트** HTTP 상태 코드나 리디렉션 URL을 동적으로 결정하려는 경우가 있습니다. 이 경우 `HttpRedirectResponse` 인터페이스(`@nestjs/common`에서 가져옴)를 따르는 객체를 반환하세요.

반환된 값은 `@Redirect()` 데코레이터에 전달된 인수를 무시합니다. 예를 들어:

```typescript
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

#### 라우트 매개변수

정적 경로를 사용하는 라우트는 요청의 일부로 **동적 데이터**를 수락해야 할 때 작동하지 않습니다(예: `GET /cats/1`에서 id가 `1`인 고양이를 가져옴). 라우트 경로에 매개변수 **토큰**을 추가하여 요청 URL의 해당 위치에 동적 값을 캡처할 수 있도록 라우트를 정의할 수 있습니다. 아래 예제에서 `@Get()` 데코레이터는 이러한 사용법을 보여줍니다. 이렇게 선언된 라우트 매개변수는 메서드 서명에 `@Param()` 데코레이터를 추가하여 접근할 수 있습니다.

> info **힌트** 매개변수가 있는 라우트는 정적 경로 이후에 선언해야 합니다. 이렇게 하면 매개변수가 있는 경로가 정적 경로로 가야 할 트래픽을 가로채지 않도록 방지할 수 있습니다.

```typescript
@Get(':id')
findOne(@Param() params: any): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

`@Param()`은 메서드 매개변수(`params` 위 예제)를 데코레이트하여, 메서드 본문 내에서 장식된 매개변수의 속성으로 **라우트** 매개변수를 사용할 수 있게 합니다. 위 코드에서 보듯이 `params.id`를 참조하여 `id` 매개변수에 접근할 수 있습니다. 데코레이터에 특정 매개변수 토큰을 전달하고 메서드 본문에서 해당 라우트 매개변수를 이름으로 직접 참조할 수도 있습니다.

> info **힌트** `Param`을 `@nestjs/common` 패키지에서 가져오세요.

```typescript
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
```

#### 서브 도메인 라우팅

`@Controller` 데코레이터는 들어오는 요청의 HTTP 호스트가 특정 값과 일치하는지 확인하는 `host` 옵션을 받을 수 있습니다.

```typescript
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin page';
  }
}
```

> **경고** **Fastify**는 중첩된 라우터를 지원하지 않으므로, 서브 도메인 라우팅을 사용할 때 기본 Express 어댑터를 사용해야 합니다.

라우트 `path`와 유사하게 `hosts` 옵션은 호스트 이름의 해당 위치에 동적 값을 캡처하는 토큰을 사용할 수 있습니다. 아래 예제는 `@Controller()` 데코레이터에 호스트 매개변수 토큰을 사용하는 방법을 보여줍니다. 이렇게 선언된 호스트 매개변수는 메서드 서명에 `@HostParam()` 데코레이터를 추가하여 접근할 수 있습니다.

```typescript
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account: string) {
    return account;
  }
}
```

#### 범위

다른 프로그래밍 언어 배경을 가진 사람들에게는 Nest에서 거의 모든 것이 들어오는 요청 간에 공유된다는 것이 놀라울 수 있습니다. 데이터베이스에 대한

 연결 풀, 글로벌 상태를 가진 싱글턴 서비스 등 모두 포함됩니다. Node.js는 각 요청이 별도의 스레드에 의해 처리되는 요청/응답 멀티 스레드 무상태 모델을 따르지 않기 때문에 싱글턴 인스턴스를 사용하는 것은 애플리케이션에 대해 완전히 **안전**합니다.

그러나 요청 기반의 컨트롤러 수명이 필요한 경우도 있습니다. 예를 들어, GraphQL 애플리케이션에서 요청 기반 캐싱, 요청 추적 또는 다중 테넌시 등의 경우입니다. 범위 제어에 대한 자세한 내용은 [여기](https://docs.nestjs.com/fundamentals/injection-scopes)를 참조하세요.

#### 비동기 처리

모던 자바스크립트를 좋아하고 데이터 추출이 대부분 **비동기적**이라는 것을 알고 있습니다. 따라서 Nest는 `async` 함수를 잘 지원합니다.

> info **힌트** `async / await` 기능에 대해 자세히 알아보려면 [여기](https://kamilmysliwiec.com/typescript-2-1-introduction-async-await)를 참조하세요.

모든 비동기 함수는 `Promise`를 반환해야 합니다. 이는 Nest가 자체적으로 해결할 수 있는 지연된 값을 반환할 수 있다는 의미입니다. 예제를 통해 살펴보겠습니다:

```typescript
@Get()
async findAll(): Promise<any[]> {
  return [];
}
```

위 코드는 완전히 유효합니다. 또한, Nest 라우트 핸들러는 RxJS [observable 스트림](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html)을 반환할 수도 있습니다. Nest는 기본적으로 소스를 구독하고 스트림이 완료되면 마지막으로 발행된 값을 가져옵니다.

```typescript
@Get()
findAll(): Observable<any[]> {
  return of([]);
}
```

위 두 가지 접근 방식 모두 작동하며, 요구사항에 맞는 방식을 사용할 수 있습니다.

#### 요청 페이로드

이전의 POST 라우트 핸들러 예제는 클라이언트 매개변수를 받지 않았습니다. 이를 `@Body()` 데코레이터를 추가하여 수정해 보겠습니다.

먼저(타입스크립트를 사용하는 경우), **DTO**(데이터 전송 객체) 스키마를 정의해야 합니다. DTO는 데이터가 네트워크를 통해 전송되는 방식을 정의하는 객체입니다. **타입스크립트** 인터페이스나 간단한 클래스를 사용하여 DTO 스키마를 정의할 수 있습니다. 흥미롭게도, 여기서는 **클래스** 사용을 권장합니다. 왜냐하면 클래스는 자바스크립트 ES6 표준의 일부로 컴파일된 자바스크립트에서 실제 엔티티로 보존되기 때문입니다. 반면, 타입스크립트 인터페이스는 트랜스파일링 중에 제거되므로 Nest는 런타임에 이를 참조할 수 없습니다. 이는 **파이프**와 같은 기능이 런타임에 변수의 메타타입에 접근할 수 있을 때 추가적인 가능성을 제공하기 때문에 중요합니다.

`CreateCatDto` 클래스를 만들어 보겠습니다:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

이제 새로 만든 DTO를 `CatsController`에서 사용할 수 있습니다:

```typescript
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
```

> info **힌트** `ValidationPipe`는 메서드 핸들러에서 수신되지 않아야 하는 속성을 필터링할 수 있습니다. 이 경우 허용 목록에 수락 가능한 속성을 나열하고 허용 목록에 포함되지 않은 속성은 자동으로 결과 객체에서 제거됩니다. `CreateCatDto` 예제에서 허용 목록은 `name`, `age`, `breed` 속성입니다. 자세한 내용은 [여기](https://docs.nestjs.com/techniques/validation#stripping-properties)를 참조하세요.

#### 오류 처리

오류 처리(즉, 예외 작업)에 대한 별도의 챕터가 [여기](https://docs.nestjs.com/exception-filters) 있습니다.

#### 전체 리소스 샘플

아래 예제는 여러 데코레이터를 사용하여 기본 컨트롤러를 만드는 예제입니다. 이 컨트롤러는 내부 데이터를 액세스하고 조작하는 몇 가지 메서드를 노출합니다.

```typescript
import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
```

> info **힌트** Nest CLI는 모든 **보일러플레이트 코드**를 자동으로 생성하는 생성기(스키매틱)를 제공하여 모든 작업을 피하고 개발자 경험을 훨씬 간단하게 만듭니다. 이 기능에 대해 자세히 알아보려면 [여기](https://docs.nestjs.com/recipes/crud-generator)를 참조하세요.

#### 설정 및 실행

위의 컨트롤러를 완전히 정의했지만, `CatsController`가 존재한다는 것을 Nest는 아직 알지 못하므로 이 클래스의 인스턴스를 생성하지 않습니다.

컨트롤러는 항상 모듈에 속하므로 `@Module()` 데코레이터 내의 `controllers` 배열에 포함됩니다. 아직 루트 `AppModule` 외에는 다른 모듈을 정의하지 않았으므로 이를 사용하여 `CatsController`를 소개하겠습니다:

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';

@Module({
  controllers: [CatsController],
})
export class AppModule {}
```

모듈 클래스에 `@Module()` 데코레이터를 사용하여 메타데이터를 첨부했으며, 이제 Nest는 어떤 컨트롤러를 마운트해야 하는지 쉽게 알 수 있습니다.

#### 라이브러리 특정 접근 방식

지금까지는 응답을 조작하는 Nest 표준 방법에 대해 논의했습니다. 응답을 조작하는 두 번째 방법은 라이브러리 특정 [응답 객체](https://expressjs.com/en/api.html#res)를 사용하는 것입니다. 특정 응답 객체를 주입하려면 `@Res()` 데코레이터를 사용해야 합니다. 차이점을 보여주기 위해 `CatsController`를 다음과 같이 다시 작성해 보겠습니다:

```typescript
import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Res() res: Response) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  findAll(@Res() res: Response) {
     res.status(HttpStatus.OK).json([]);
  }
}
```

이 접근 방식은 작동하며 응답 객체(헤더 조작, 라이브러리 특정 기능 등)에 대한 전체 제어를 제공하여 어떤 면에서는 더 큰 유연성을 허용하지만, 주의해서 사용해야 합니다. 일반적으로 이 접근 방식은 덜 명확하고 몇 가지 단점이 있습니다. 주요 단점은 코드가 플랫폼에 종속되고(기본 라이브러리에 따라 응답 객체의 API가 다를 수 있음) 테스트하기 어려워진다는 것입니다(응답 객체를 모의해야 함).

또한, 위 예제에서는 Interceptors 및 `@HttpCode()` / `@Header()` 데코레이터와 같은 Nest 표준 응답 처리를 사용하는 Nest 기능과의 호환성을 잃게 됩니다. 이를 수정하려면 `passthrough` 옵션을 `true`로 설정해야 합니다:

```typescript
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return [];
}
```

이제 네이티브 응답 객체와 상호 작용할 수 있으며(예: 특정 조건에 따라 쿠키 또는 헤더 설정), 나머지는 프레임워크에 맡길 수 있습니다.
