### 비동기 providers

때때로 애플리케이션 시작을 하나 이상의 **비동기 작업**이 완료될 때까지 지연시켜야 합니다. 예를 들어, 데이터베이스와의 연결이 설정될 때까지 요청을 받지 않도록 할 수 있습니다. 비동기 providers를 사용하여 이를 달성할 수 있습니다.

이를 위한 구문은 `useFactory` 구문과 함께 `async/await`를 사용하는 것입니다. 팩토리는 `Promise`를 반환하고, 팩토리 함수는 비동기 작업을 `await`할 수 있습니다. Nest는 이러한 provider를 주입하는 클래스의 인스턴스를 생성하기 전에 promise의 해결을 기다립니다.

```typescript
{
  provide: 'ASYNC_CONNECTION',
  useFactory: async () => {
    const connection = await createConnection(options);
    return connection;
  },
}
```

비동기 providers에 대해 더 알아보려면 [여기](https://docs.nestjs.com/fundamentals/custom-providers)를 참조하십시오.

### 주입

비동기 providers는 다른 provider와 마찬가지로 토큰을 통해 다른 컴포넌트에 주입됩니다. 위의 예에서는 `@Inject('ASYNC_CONNECTION')` 구문을 사용합니다.

### 예제

[TypeORM 레시피](https://docs.nestjs.com/recipes/sql-typeorm)에는 비동기 provider의 더 구체적인 예제가 나와 있습니다.
