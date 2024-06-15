### 플랫폼 독립성

Nest는 플랫폼 독립적인 프레임워크입니다. 즉, **재사용 가능한 논리적 부분**을 개발하여 다양한 유형의 애플리케이션에서 사용할 수 있습니다. 예를 들어, 대부분의 구성 요소는 서로 다른 기본 HTTP 서버 프레임워크(예: Express와 Fastify)뿐만 아니라 서로 다른 **유형**의 애플리케이션(예: HTTP 서버 프레임워크, 다른 전송 계층을 사용하는 마이크로서비스, 웹 소켓)에서도 변경 없이 재사용할 수 있습니다.

#### 한 번 빌드하고 어디서나 사용

문서의 **개요** 섹션은 주로 HTTP 서버 프레임워크를 사용하는 코딩 기술을 보여줍니다(예: REST API를 제공하는 앱 또는 MVC 스타일의 서버 측 렌더링 앱 제공). 그러나 이러한 모든 빌딩 블록은 다른 전송 계층([마이크로서비스](https://docs.nestjs.com/microservices/basics) 또는 [웹 소켓](https://docs.nestjs.com/websockets/gateways)) 위에서도 사용할 수 있습니다.

또한, Nest에는 전용 [GraphQL](https://docs.nestjs.com/graphql/quick-start) 모듈이 포함되어 있습니다. REST API를 제공하는 것과 교대로 GraphQL을 API 레이어로 사용할 수 있습니다.

또한, [애플리케이션 컨텍스트](https://docs.nestjs.com/application-context) 기능은 CRON 작업 및 CLI 앱과 같은 모든 종류의 Node.js 애플리케이션을 Nest 위에 생성하는 데 도움을 줍니다.

Nest는 모듈성 및 재사용성을 높여주는 완전한 Node.js 애플리케이션 플랫폼을 목표로 합니다. 한 번 빌드하고 어디서나 사용하세요!
