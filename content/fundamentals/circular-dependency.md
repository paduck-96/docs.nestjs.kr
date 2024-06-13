### Circular dependency

순환 종속성은 두 클래스가 서로에게 의존할 때 발생합니다. 예를 들어, 클래스 A가 클래스 B를 필요로 하고, 클래스 B도 클래스 A를 필요로 하는 경우입니다. Nest에서는 모듈 간 및 provider 간에 순환 종속성이 발생할 수 있습니다.

순환 종속성은 가능한 피하는 것이 좋지만, 항상 피할 수 있는 것은 아닙니다. 이러한 경우, Nest는 provider 간의 순환 종속성을 해결하는 두 가지 방법을 제공합니다. 이 장에서는 **forward referencing**을 사용하는 방법과 **ModuleRef** 클래스를 사용하여 DI 컨테이너에서 provider 인스턴스를 검색하는 방법을 설명합니다.

또한 모듈 간의 순환 종속성을 해결하는 방법도 설명합니다.

> Barrel 파일(index.ts 파일)을 사용하여 그룹화할 때도 순환 종속성이 발생할 수 있습니다. 모듈/provider 클래스에서는 barrel 파일을 생략해야 합니다. 예를 들어, 같은 디렉토리 내에서 파일을 가져올 때는 barrel 파일을 사용하지 않아야 합니다. 자세한 내용은 [이 GitHub 이슈](https://github.com/nestjs/nest/issues/1181#issuecomment-430197191)를 참조하십시오.

#### Forward reference

**Forward reference**는 `forwardRef()` 유틸리티 함수를 사용하여 Nest가 아직 정의되지 않은 클래스를 참조할 수 있도록 합니다. 예를 들어, `CatsService`와 `CommonService`가 서로에게 의존하는 경우, 관계의 양쪽에서 `@Inject()`와 `forwardRef()` 유틸리티를 사용하여 순환 종속성을 해결할 수 있습니다. 그렇지 않으면 모든 필수 메타데이터가 사용할 수 없기 때문에 Nest는 이를 인스턴스화하지 않습니다. 다음은 예시입니다:

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    @Inject(forwardRef(() => CommonService))
    private commonService: CommonService,
  ) {}
}
@@switch
@Injectable()
@Dependencies(forwardRef(() => CommonService))
export class CatsService {
  constructor(commonService) {
    this.commonService = commonService;
  }
}
```

`forwardRef()` 함수는 `@nestjs/common` 패키지에서 가져옵니다.

이제 `CommonService`에서도 동일한 작업을 수행합니다:

```typescript
@@filename(common.service)
@Injectable()
export class CommonService {
  constructor(
    @Inject(forwardRef(() => CatsService))
    private catsService: CatsService,
  ) {}
}
@@switch
@Injectable()
@Dependencies(forwardRef(() => CatsService))
export class CommonService {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

> 인스턴스화 순서는 불확정적입니다. 코드가 어느 생성자가 먼저 호출되는지에 의존하지 않도록 하십시오. `Scope.REQUEST`를 가진 provider와 순환 종속성을 가지는 것은 정의되지 않은 종속성으로 이어질 수 있습니다. 자세한 내용은 [여기](https://github.com/nestjs/nest/issues/5778)를 참조하십시오.

#### ModuleRef 클래스 대안

`forwardRef()`를 사용하는 것 외에 코드를 리팩토링하고 `ModuleRef` 클래스를 사용하여 순환 관계의 한쪽에서 provider를 검색할 수도 있습니다. `ModuleRef` 유틸리티 클래스에 대해 자세히 알아보려면 [여기](https://docs.nestjs.com/fundamentals/module-ref)를 참조하십시오.

#### 모듈 순환 참조

모듈 간의 순환 종속성을 해결하려면 모듈 연결의 양쪽에서 동일한 `forwardRef()` 유틸리티 함수를 사용하십시오. 예를 들어:

```typescript
@@filename(common.module)
@Module({
  imports: [forwardRef(() => CatsModule)],
})
export class CommonModule {}
```

이제 `CatsModule`에서도 동일한 작업을 수행합니다:

```typescript
@@filename(cats.module)
@Module({
  imports: [forwardRef(() => CommonModule)],
})
export class CatsModule {}
```
