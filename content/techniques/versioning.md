### 버전 관리

> info **힌트** 이 장은 HTTP 기반 애플리케이션에만 관련이 있습니다.

버전 관리를 통해 동일한 애플리케이션 내에서 **다른 버전**의 컨트롤러 또는 개별 경로를 실행할 수 있습니다. 애플리케이션은 자주 변경되며 이전 버전을 지원해야 하는 경우에도 중단적인 변경이 필요할 때가 많습니다.

NestJS는 다음과 같은 네 가지 유형의 버전 관리를 지원합니다:

<table>
  <tr>
    <td><a href='https://docs.nestjs.com/techniques/versioning#uri-versioning-type'><code>URI 버전 관리</code></a></td>
    <td>요청 URI 내에 버전이 전달됩니다 (기본값)</td>
  </tr>
  <tr>
    <td><a href='https://docs.nestjs.com/techniques/versioning#header-versioning-type'><code>헤더 버전 관리</code></a></td>
    <td>사용자 정의 요청 헤더가 버전을 지정합니다</td>
  </tr>
  <tr>
    <td><a href='https://docs.nestjs.com/techniques/versioning#media-type-versioning-type'><code>미디어 타입 버전 관리</code></a></td>
    <td>요청의 <code>Accept</code> 헤더가 버전을 지정합니다</td>
  </tr>
  <tr>
    <td><a href='https://docs.nestjs.com/techniques/versioning#custom-versioning-type'><code>사용자 정의 버전 관리</code></a></td>
    <td>요청의 모든 측면을 사용하여 버전을 지정할 수 있습니다. 사용자 정의 함수를 제공하여 해당 버전을 추출합니다.</td>
  </tr>
</table>

#### URI 버전 관리 타입

URI 버전 관리는 `https://example.com/v1/route` 및 `https://example.com/v2/route`와 같이 요청의 URI 내에 전달된 버전을 사용합니다.

> warning **알림** URI 버전 관리를 사용할 때 버전은 <a href="https://docs.nestjs.com/faq/global-prefix">글로벌 경로 접두사</a>가 존재하는 경우 URI에 자동으로 추가되며, 컨트롤러 또는 경로 경로 앞에 위치하게 됩니다.

애플리케이션에서 URI 버전 관리를 활성화하려면 다음을 수행합니다:

```typescript
const app = await NestFactory.create(AppModule);
// 또는 "app.enableVersioning()"
app.enableVersioning({
  type: VersioningType.URI,
});
await app.listen(3000);
```

> warning **알림** URI의 버전은 기본적으로 `v`로 접두사가 붙지만, 접두사 값을 `prefix` 키를 설정하여 원하는 접두사로 설정하거나 `false`로 설정하여 비활성화할 수 있습니다.

> info **힌트** `VersioningType` 열거형은 `@nestjs/common` 패키지에서 가져와서 `type` 속성에 사용할 수 있습니다.

#### 헤더 버전 관리 타입

헤더 버전 관리는 사용자 정의 요청 헤더를 사용하여 버전을 지정하며, 헤더의 값이 요청에 사용할 버전이 됩니다.

헤더 버전 관리를 애플리케이션에서 활성화하려면 다음을 수행합니다:

```typescript
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.HEADER,
  header: 'Custom-Header',
});
await app.listen(3000);
```

`header` 속성은 요청의 버전을 포함하는 헤더의 이름이어야 합니다.

> info **힌트** `VersioningType` 열거형은 `@nestjs/common` 패키지에서 가져와서 `type` 속성에 사용할 수 있습니다.

#### 미디어 타입 버전 관리 타입

미디어 타입 버전 관리는 요청의 `Accept` 헤더를 사용하여 버전을 지정합니다.

`Accept` 헤더 내에서 버전은 세미콜론(`;`)으로 미디어 타입과 구분됩니다. 그런 다음 요청에 사용할 버전을 나타내는 키-값 쌍을 포함해야 합니다. 예를 들어, `Accept: application/json;v=2`와 같이 지정할 수 있습니다. 키는 버전을 결정할 때 키와 구분 기호를 포함하도록 구성됩니다.

미디어 타입 버전 관리를 애플리케이션에서 활성화하려면 다음을 수행합니다:

```typescript
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key: 'v=',
});
await app.listen(3000);
```

`key` 속성은 버전을 포함하는 키-값 쌍의 키와 구분 기호여야 합니다. 예를 들어, `Accept: application/json;v=2`의 경우 `key` 속성은 `v=`로 설정됩니다.

> info **힌트** `VersioningType` 열거형은 `@nestjs/common` 패키지에서 가져와서 `type` 속성에 사용할 수 있습니다.

#### 사용자 정의 버전 관리 타입

사용자 정의 버전 관리는 요청의 모든 측면을 사용하여 버전(들)을 지정합니다. 들어오는 요청은 문자열 또는 문자열 배열을 반환하는 `extractor` 함수를 사용하여 분석됩니다.

요청자가 여러 버전을 제공하는 경우, 추출기 함수는 내림차순으로 정렬된 문자열 배열을 반환할 수 있습니다. 버전은 내림차순으로 경로와 일치합니다.

`extractor`에서 빈 문자열 또는 배열이 반환되면, 경로가 일치하지 않고 404가 반환됩니다.

예를 들어, 들어오는 요청이 `1`, `2`, `3` 버전을 지원한다고 지정하면, `extractor` **MUST**는 `[3, 2, 1]`을 반환해야 합니다. 이는 가능한 한 높은 버전의 경로가 먼저 선택되도록 보장합니다.

버전 `[3, 2, 1]`이 추출되지만 버전 `2` 및 `1`에 대한 경로만 있는 경우, 버전 `2`에 일치하는 경로가 선택됩니다(버전 `3`은 자동으로 무시됩니다).

> warning **알림** 요청에서 제공된 버전 목록 중 가장 높은 버전을 선택하는 기능은 Express 어댑터의 설계 제한으로 인해 제대로 작동하지 않습니다. 단일 버전(문자열 또는 1개의 요소가 포함된 배열)은 Express에서 잘 작동합니다. Fastify는 가장 높은 일치 버전 선택과 단일 버전 선택을 모두 올바르게 지원합니다.

애플리케이션에서 사용자 정의 버전 관리를 활성화하려면 `extractor` 함수를 작성하고 다음과 같이 애플리케이션에 전달합니다:

```typescript
// 사용자 정의 헤더에서 버전 목록을 추출하고 정렬된 배열로 변환하는 예제 추출기 함수입니다.
// 이 예제는 Fastify를 사용하지만 Express 요청도 유사하게 처리할 수 있습니다.
const extractor = (request: FastifyRequest): string | string[] =>
  [request.headers['custom-versioning-field'] ?? '']
     .flatMap(v => v.split(','))
     .filter(v => !!v)
     .sort()
     .reverse()

const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.CUSTOM,
  extractor,
});
await app.listen(3000);
```

#### 사용법

버전 관리를 통해 컨트롤러, 개별 경로에 버전을 설정할 수 있으며 특정 리소스가 버전 관리를 선택 해제할 수 있는 방법도 제공합니다. 애플리케이션이 사용하는 버전 관리 타입과 관계없이 버전 관리의 사용법은 동일합니다.

> warning **알림** 애플리케이션에 대해 버전 관리가 활성화되었지만 컨트롤러 또는 경로에 버전이 지정되지 않은 경우 해당 컨트롤러/경로에 대한 모든 요청은 `404` 응답 상태로 반환됩니다. 유사하게, 요청에 대해 지정된 버전이 해당하는 컨트롤러 또는 경로가 없는 경우에도 `404` 응답 상태로 반환됩니다.

#### 컨트롤러 버전

버전은 컨트롤러에 적용되어 컨트롤러 내의 모든 경로에 대해 버전을 설정할 수 있습니다.

컨트롤러에 버전을 추가하려면 다음을 수행합니다:

```typescript
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1';
  }
}
```

#### 경로 버전

버전은 개별 경로에 적용될 수 있습니다. 이 버전은 컨트롤러 버전과 같은 경로에 영향을 미치는 다른 버전을 무시합니다.

개별 경로에 버전을 추가하려면 다음을 수행합니다:

```typescript
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1(): string {
    return 'This action returns all cats for version 1';
  }

  @Version('2')
  @Get('cats')
  findAllV2(): string {
    return 'This action returns all cats for version 

2';
  }
}
```

#### 다중 버전

여러 버전을 컨트롤러 또는 경로에 적용할 수 있습니다. 여러 버전을 사용하려면 버전을 배열로 설정합니다.

여러 버전을 추가하려면 다음을 수행합니다:

```typescript
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1 or 2';
  }
}
```

#### 버전 "중립"

일부 컨트롤러 또는 경로는 버전에 상관없이 동일한 기능을 가질 수 있습니다. 이를 위해 버전을 `VERSION_NEUTRAL` 심볼로 설정할 수 있습니다.

들어오는 요청은 요청에 지정된 버전에 관계없이 `VERSION_NEUTRAL` 컨트롤러 또는 경로와 일치하며, 요청에 버전이 전혀 포함되지 않은 경우에도 일치합니다.

> warning **알림** URI 버전 관리를 사용할 때 `VERSION_NEUTRAL` 리소스는 URI에 버전이 포함되지 않습니다.

버전 중립 컨트롤러 또는 경로를 추가하려면 다음을 수행합니다:

```typescript
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats regardless of version';
  }
}
```

#### 글로벌 기본 버전

각 컨트롤러 또는 개별 경로에 버전을 제공하지 않으려는 경우 또는 버전이 지정되지 않은 모든 컨트롤러/경로에 대해 특정 버전을 기본 버전으로 설정하려는 경우 `defaultVersion`을 다음과 같이 설정
