### Model-View-Controller

Nest는 기본적으로 [Express](https://github.com/expressjs/express) 라이브러리를 사용합니다. 따라서 Express에서 MVC(Model-View-Controller) 패턴을 사용하는 모든 기술은 Nest에도 적용됩니다.

먼저, [CLI](https://github.com/nestjs/nest-cli) 도구를 사용하여 간단한 Nest 애플리케이션을 구성해봅시다:

```bash
$ npm i -g @nestjs/cli
$ nest new project
```

MVC 애플리케이션을 만들기 위해서는 HTML 뷰를 렌더링할 [템플릿 엔진](https://expressjs.com/en/guide/using-template-engines.html)이 필요합니다:

```bash
$ npm install --save hbs
```

여기서는 `hbs`([Handlebars](https://github.com/pillarjs/hbs#readme)) 엔진을 사용하였지만, 요구 사항에 맞는 다른 엔진을 사용할 수 있습니다. 설치가 완료되면 다음 코드를 사용하여 Express 인스턴스를 구성합니다:

```typescript
import { NestFactory } from '@nestjs/core';
import { NestExpressApplication } from '@nestjs/platform-express';
import { join } from 'path';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);

  app.useStaticAssets(join(__dirname, '..', 'public'));
  app.setBaseViewsDir(join(__dirname, '..', 'views'));
  app.setViewEngine('hbs');

  await app.listen(3000);
}
bootstrap();
```

여기서 [Express](https://github.com/expressjs/express)에 `public` 디렉토리를 정적 자산 저장소로, `views` 디렉토리를 템플릿 저장소로 사용하도록 설정하고, `hbs` 템플릿 엔진을 사용하여 HTML 출력을 렌더링하도록 설정했습니다.

#### 템플릿 렌더링

이제 `views` 디렉토리와 `index.hbs` 템플릿을 생성합니다. 템플릿에서는 컨트롤러에서 전달된 `message`를 출력합니다:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>App</title>
  </head>
  <body>
    {{ message }}
  </body>
</html>
```

다음으로, `app.controller` 파일을 열고 `root()` 메서드를 다음 코드로 교체합니다:

```typescript
import { Get, Controller, Render } from '@nestjs/common';

@Controller()
export class AppController {
  @Get()
  @Render('index')
  root() {
    return { message: 'Hello world!' };
  }
}
```

여기서 `@Render()` 데코레이터에서 사용할 템플릿을 지정하고, 라우트 핸들러 메서드의 반환 값을 템플릿에 전달하여 렌더링합니다. 반환 값은 `message` 속성을 가진 객체이며, 이는 템플릿에서 생성한 `message` 플레이스홀더와 일치합니다.

애플리케이션이 실행 중일 때, 브라우저를 열고 `http://localhost:3000`으로 이동하면 `Hello world!` 메시지가 표시됩니다.

#### 동적 템플릿 렌더링

애플리케이션 로직이 렌더링할 템플릿을 동적으로 결정해야 하는 경우, `@Render()` 데코레이터 대신 `@Res()` 데코레이터를 사용하여 라우트 핸들러에서 뷰 이름을 제공합니다:

> info **힌트** Nest는 `@Res()` 데코레이터를 감지하면 라이브러리 특정 `response` 객체를 주입합니다. 이 객체를 사용하여 동적으로 템플릿을 렌더링할 수 있습니다. `response` 객체 API에 대한 자세한 내용은 [여기](https://expressjs.com/en/api.html)에서 확인하세요.

```typescript
import { Get, Controller, Res, Render } from '@nestjs/common';
import { Response } from 'express';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private appService: AppService) {}

  @Get()
  root(@Res() res: Response) {
    return res.render(this.appService.getViewName(), { message: 'Hello world!' });
  }
}
```

#### 예제

작동하는 예제는 [여기](https://github.com/nestjs/nest/tree/master/sample/15-mvc)에서 확인할 수 있습니다.

#### Fastify

이 [챕터](https://docs.nestjs.com/techniques/performance)에서 언급했듯이, Nest는 호환되는 HTTP 제공자와 함께 사용할 수 있습니다. 그 중 하나는 [Fastify](https://github.com/fastify/fastify)입니다. Fastify로 MVC 애플리케이션을 만들기 위해 다음 패키지를 설치해야 합니다:

```bash
$ npm i --save @fastify/static @fastify/view handlebars
```

설치 과정이 완료되면 `main.ts` 파일을 열고 다음과 같이 내용을 업데이트합니다:

```typescript
import { NestFactory } from '@nestjs/core';
import { NestFastifyApplication, FastifyAdapter } from '@nestjs/platform-fastify';
import { AppModule } from './app.module';
import { join } from 'path';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter(),
  );
  app.useStaticAssets({
    root: join(__dirname, '..', 'public'),
    prefix: '/public/',
  });
  app.setViewEngine({
    engine: {
      handlebars: require('handlebars'),
    },
    templates: join(__dirname, '..', 'views'),
  });
  await app.listen(3000);
}
bootstrap();
```

Fastify API는 약간 다르지만 이러한 메서드 호출의 최종 결과는 동일합니다. Fastify의 경우 `@Render()` 데코레이터에 전달되는 템플릿 이름에 파일 확장자가 포함되어야 합니다.

```typescript
import { Get, Controller, Render } from '@nestjs/common';

@Controller()
export class AppController {
  @Get()
  @Render('index.hbs')
  root() {
    return { message: 'Hello world!' };
  }
}
```

애플리케이션이 실행 중일 때, 브라우저를 열고 `http://localhost:3000`으로 이동하면 `Hello world!` 메시지가 표시됩니다.

#### 예제

작동하는 예제는 [여기](https://github.com/nestjs/nest/tree/master/sample/17-mvc-fastify)에서 확인할 수 있습니다.
