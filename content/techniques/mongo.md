### Mongo

Nest는 [MongoDB](https://www.mongodb.com/) 데이터베이스와의 통합을 두 가지 방법으로 지원합니다. TypeORM의 [내장 모듈](/techniques/database)을 사용하거나, MongoDB에서 가장 인기 있는 객체 모델링 도구인 [Mongoose](https://mongoosejs.com)를 사용할 수 있습니다. 이 장에서는 전용 `@nestjs/mongoose` 패키지를 사용하여 후자의 방법을 설명합니다.

먼저 필요한 [종속성](https://github.com/Automattic/mongoose)을 설치합니다:

```bash
$ npm i @nestjs/mongoose mongoose
```

설치가 완료되면 `MongooseModule`을 루트 `AppModule`에 임포트합니다.

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [MongooseModule.forRoot('mongodb://localhost/nest')],
})
export class AppModule {}
```

`forRoot()` 메서드는 Mongoose 패키지의 `mongoose.connect()`와 동일한 구성 객체를 허용합니다. 자세한 내용은 [여기](https://mongoosejs.com/docs/connections.html)에서 확인할 수 있습니다.

#### 모델 주입

Mongoose를 사용하면 모든 것이 [Schema](http://mongoosejs.com/docs/guide.html)에서 파생됩니다. 각 스키마는 MongoDB 컬렉션에 매핑되고 해당 컬렉션 내의 문서 구조를 정의합니다. 스키마는 [모델](https://mongoosejs.com/docs/models.html)을 정의하는 데 사용됩니다. 모델은 기본 MongoDB 데이터베이스에서 문서를 생성하고 읽는 역할을 합니다.

스키마는 NestJS 데코레이터를 사용하여 생성하거나 Mongoose 자체로 수동으로 생성할 수 있습니다. 스키마 생성을 위해 데코레이터를 사용하면 보일러플레이트가 크게 줄어들고 코드 가독성이 향상됩니다.

`CatSchema`를 정의해 보겠습니다:

```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';

export type CatDocument = HydratedDocument<Cat>;

@Schema()
export class Cat {
  @Prop()
  name: string;

  @Prop()
  age: number;

  @Prop()
  breed: string;
}

export const CatSchema = SchemaFactory.createForClass(Cat);
```

`@Schema()` 데코레이터는 클래스를 스키마 정의로 표시합니다. 이는 `Cat` 클래스를 동일한 이름의 MongoDB 컬렉션에 매핑하지만 끝에 "s"가 추가됩니다. 따라서 최종 mongo 컬렉션 이름은 `cats`가 됩니다. 이 데코레이터는 스키마 옵션 객체인 단일 선택적 인수를 허용합니다. 자세한 내용은 [여기](https://mongoosejs.com/docs/guide.html#options)에서 확인할 수 있습니다.

`@Prop()` 데코레이터는 문서의 속성을 정의합니다. 예를 들어, 위의 스키마 정의에서 `name`, `age`, `breed` 세 가지 속성을 정의했습니다. 이 속성들의 [스키마 타입](https://mongoosejs.com/docs/schematypes.html)은 TypeScript 메타데이터 및 리플렉션 기능 덕분에 자동으로 유추됩니다. 그러나 배열이나 중첩된 객체 구조와 같은 더 복잡한 시나리오에서는 타입을 명시적으로 지정해야 합니다:

```typescript
@Prop([String])
tags: string[];
```

대안으로, `@Prop()` 데코레이터는 옵션 객체 인수를 허용합니다. 이를 통해 속성이 필수인지 여부를 지정하거나 기본값을 설정하거나 변경 불가능하게 표시할 수 있습니다. 예를 들어:

```typescript
@Prop({ required: true })
name: string;
```

다른 모델과의 관계를 지정하려는 경우에도 `@Prop()` 데코레이터를 사용할 수 있습니다. 예를 들어, `Cat`이 `Owner`를 가지고 있으며, 이는 `owners`라는 다른 컬렉션에 저장되어 있는 경우, 속성은 타입과 참조를 가져야 합니다. 예를 들어:

```typescript
import * as mongoose from 'mongoose';
import { Owner } from '../owners/schemas/owner.schema';

@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })
owner: Owner;
```

다수의 소유자가 있는 경우 속성 구성은 다음과 같아야 합니다:

```typescript
@Prop({ type: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' }] })
owner: Owner[];
```

마지막으로, **raw** 스키마 정의도 데코레이터에 전달할 수 있습니다. 예를 들어, 중첩된 객체를 클래스로 정의하지 않는 경우 유용합니다. 이를 위해 `@nestjs/mongoose` 패키지의 `raw()` 함수를 사용합니다:

```typescript
@Prop(raw({
  firstName: { type: String },
  lastName: { type: String }
}))
details: Record<string, any>;
```

데코레이터를 사용하지 않으려면 수동으로 스키마를 정의할 수도 있습니다. 예를 들어:

```typescript
export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,
  breed: String,
});
```

`cat.schema` 파일은 `cats` 디렉토리의 폴더에 위치하며, 여기에서 `CatsModule`도 정의합니다. 스키마 파일을 어디에 저장할지 선택할 수 있지만, 관련된 **도메인** 객체와 가까운 적절한 모듈 디렉토리에 저장하는 것이 좋습니다.

`CatsModule`을 살펴보겠습니다:

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { Cat, CatSchema } from './schemas/cat.schema';

@Module({
  imports: [MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }])],
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

`MongooseModule`은 현재 범위에서 등록할 모델을 정의하는 `forFeature()` 메서드를 제공합니다. 다른 모듈에서도 모델을 사용하려면 `CatsModule`의 `exports` 섹션에 MongooseModule을 추가하고 해당 모듈에서 `CatsModule`을 임포트합니다.

스키마를 등록한 후, `@InjectModel()` 데코레이터를 사용하여 `CatsService`에 `Cat` 모델을 주입할 수 있습니다:

```typescript
import { Model } from 'mongoose';
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Cat } from './schemas/cat.schema';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name) private catModel: Model<Cat>) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }
}
```

#### 연결

때때로 기본 [Mongoose Connection](https://mongoosejs.com/docs/api.html#Connection) 객체에 액세스해야 할 수 있습니다. 예를 들어, 연결 객체에서 기본 API 호출을 수행하려는 경우가 있습니다. `@InjectConnection()` 데코레이터를 사용하여 Mongoose 연결을 주입할 수 있습니다:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private connection: Connection) {}
}
```

#### 여러 데이터베이스

일부 프로젝트는 여러 데이터베이스 연결이 필요합니다. 이 모듈을 사용하여 이를 달성할 수 있습니다. 여러 연결을 사용하려면 먼저 연결을 만듭니다. 이 경우 연결 이름 지정이 **필수**가 됩니다.

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionName: 'cats',
    }),
    MongooseModule.forRoot('mongodb://localhost/users', {
      connectionName: 'users',
    }),
  ],
})
export class AppModule {}
```

> **Notice** 여러 이름이 없는 연결 또는 동일한 이름을 가진 여러 연결을 사용해서는 안 됩니다. 그렇지 않으면 덮어쓰게 됩니다.

이 설정으로 `MongooseModule.forFeature()` 함수에 어떤 연결을 사용할지 알려야 합니다.

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }], 'cats'),
  ],
})
export class CatsModule {}
```

지정된 연결에 대해 `Connection`을 주입할 수도 있습니다:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection('cats') private connection: Connection) {}
}
```

지정된 `Connection`을 커스텀 provider에 주입하려면(예: 팩토리 provider), `getConnectionToken()` 함수를 사용

하여 연결 이름을 인수로 전달합니다.

```typescript
{
  provide: CatsService,
  useFactory: (catsConnection: Connection) => {
    return new CatsService(catsConnection);
  },
  inject: [getConnectionToken('cats')],
}
```

단순히 이름이 지정된 데이터베이스에서 모델을 주입하려는 경우, `@InjectModel()` 데코레이터의 두 번째 매개변수로 연결 이름을 사용할 수 있습니다.

```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { Cat } from './schemas/cat.schema';

@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name, 'cats') private catModel: Model<Cat>) {}
}
```

#### 후크(미들웨어)

미들웨어(프리 및 포스트 후크라고도 함)는 비동기 함수 실행 중에 제어를 전달하는 함수입니다. 미들웨어는 스키마 수준에서 지정되며 플러그인 작성에 유용합니다([source](https://mongoosejs.com/docs/middleware.html)). 모델을 컴파일한 후 `pre()` 또는 `post()`를 호출하는 것은 Mongoose에서 작동하지 않습니다. 모델 등록 **전에** 후크를 등록하려면 팩토리 provider(i.e., `useFactory`)와 함께 `MongooseModule`의 `forFeatureAsync()` 메서드를 사용합니다. 이 기술을 사용하면 스키마 객체에 액세스한 다음 `pre()` 또는 `post()` 메서드를 사용하여 해당 스키마에 후크를 등록할 수 있습니다. 예를 들어:

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatSchema;
          schema.pre('save', function () {
            console.log('Hello from pre save');
          });
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

다른 [팩토리 provider](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory)와 마찬가지로, 팩토리 함수는 `async`가 될 수 있으며 `inject`를 통해 종속성을 주입할 수 있습니다.

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => {
          const schema = CatSchema;
          schema.pre('save', function() {
            console.log(
              `${configService.get('APP_NAME')}: Hello from pre save`,
            ),
          });
          return schema;
        },
        inject: [ConfigService],
      },
    ]),
  ],
})
export class AppModule {}
```

#### 플러그인

주어진 스키마에 [플러그인](https://mongoosejs.com/docs/plugins.html)을 등록하려면 `forFeatureAsync()` 메서드를 사용합니다.

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatSchema;
          schema.plugin(require('mongoose-autopopulate'));
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

한 번에 모든 스키마에 플러그인을 등록하려면 `Connection` 객체의 `.plugin()` 메서드를 호출합니다. 모델이 생성되기 전에 연결에 액세스해야 합니다. 이를 위해 `connectionFactory`를 사용합니다:

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionFactory: (connection) => {
        connection.plugin(require('mongoose-autopopulate'));
        return connection;
      }
    }),
  ],
})
export class AppModule {}
```

#### 디스크리미네이터

[디스크리미네이터](https://mongoosejs.com/docs/discriminators.html)는 스키마 상속 메커니즘입니다. 이를 통해 동일한 기본 MongoDB 컬렉션 위에 중복되는 스키마를 가진 여러 모델을 가질 수 있습니다.

단일 컬렉션에서 다양한 유형의 이벤트를 추적하려고 한다고 가정해 보겠습니다. 모든 이벤트에는 타임스탬프가 있습니다.

```typescript
@Schema({ discriminatorKey: 'kind' })
export class Event {
  @Prop({
    type: String,
    required: true,
    enum: [ClickedLinkEvent.name, SignUpEvent.name],
  })
  kind: string;

  @Prop({ type: Date, required: true })
  time: Date;
}

export const EventSchema = SchemaFactory.createForClass(Event);
```

`SignedUpEvent`와 `ClickedLinkEvent` 인스턴스는 일반 이벤트와 동일한 컬렉션에 저장됩니다.

이제 `ClickedLinkEvent` 클래스를 다음과 같이 정의해 보겠습니다:

```typescript
@Schema()
export class ClickedLinkEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  url: string;
}

export const ClickedLinkEventSchema = SchemaFactory.createForClass(ClickedLinkEvent);
```

그리고 `SignUpEvent` 클래스:

```typescript
@Schema()
export class SignUpEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  user: string;
}

export const SignUpEventSchema = SchemaFactory.createForClass(SignUpEvent);
```

이제, `discriminators` 옵션을 사용하여 주어진 스키마에 대해 디스크리미네이터를 등록합니다. 이는 `MongooseModule.forFeature` 및 `MongooseModule.forFeatureAsync` 모두에서 작동합니다:

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: Event.name,
        schema: EventSchema,
        discriminators: [
          { name: ClickedLinkEvent.name, schema: ClickedLinkEventSchema },
          { name: SignUpEvent.name, schema: SignUpEventSchema },
        ],
      },
    ]),
  ]
})
export class EventsModule {}
```

#### 테스트

애플리케이션의 단위 테스트를 수행할 때는 일반적으로 데이터베이스 연결을 피하고 싶어합니다. 이렇게 하면 테스트 스위트를 더 간단하게 설정하고 실행 속도를 빠르게 할 수 있습니다. 그러나 우리의 클래스는 연결 인스턴스로부터 모델을 필요로 할 수 있습니다. 이러한 클래스를 어떻게 해결할 수 있을까요? 해결책은 모형 모델을 만드는 것입니다.

이를 더 쉽게 하기 위해, `@nestjs/mongoose` 패키지는 토큰 이름을 기반으로 준비된 [주입 토큰](https://docs.nestjs.com/fundamentals/custom-providers#di-fundamentals)을 반환하는 `getModelToken()` 함수를 제공합니다. 이 토큰을 사용하면 `useClass`, `useValue`, `useFactory`와 같은 표준 [사용자 정의 provider](/fundamentals/custom-providers) 기술을 사용하여 모형 구현을 쉽게 제공할 수 있습니다. 예를 들어:

```typescript
@Module({
  providers: [
    CatsService,
    {
      provide: getModelToken(Cat.name),
      useValue: catModel,
    },
  ],
})
export class CatsModule {}
```

이 예제에서, 하드코딩된 `catModel`(객체 인스턴스)이 `@InjectModel()` 데코레이터를 사용하여 `Model<Cat>`을 주입하는 모든 소비자에게 제공됩니다.

#### 비동기 구성

모듈 옵션을 정적으로 전달하는 대신 비동기로 전달해야 할 경우, `forRootAsync()` 메서드를 사용하십시오. 대부분의 동적 모듈과 마찬가지로, Nest는 비동기 구성을 처리하기 위한 여러 가지 기술을 제공합니다.

한 가지 기술은 팩토리 함수를 사용하는 것입니다:

```typescript
MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: 'mongodb://localhost/nest',
  }),
});
```

다른 [팩토리 provider](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory)와 마찬가지로, 팩토리 함수는 `async`가 될 수 있으며 `inject`를 통해 종속성을 주입할 수 있습니다.

```typescript
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    uri: configService.get<string>('MONGODB_URI'),
  }),
  inject: [ConfigService],
});
```

대안으로, 팩토리 대신 클래스를 사용하여 `MongooseModule`을 구성할 수 있습니다:

```typescript
MongooseModule.forRootAsync({
  useClass: MongooseConfigService,
});
```

위의 구성은 `MongooseModule` 내부에 `MongooseConfigService`를 인스턴스화하고, 이를 사용하여 필요한 옵션 객체를 생성합니다. 이 예제에서는 `MongooseConfigService`가 `MongooseOptionsFactory` 인터페이스를 구현해야 한다는 점을 주의하십시오. `MongooseModule`은 제공된 클래스의 인스턴스화된 객체에서 `createMongooseOptions()` 메서드를 호출합니다.

```typescript
@Injectable()
export class MongooseConfigService implements Mongoose

OptionsFactory {
  createMongooseOptions(): MongooseModuleOptions {
    return {
      uri: 'mongodb://localhost/nest',
    };
  }
}
```

`MongooseModule` 내부에서 `MongooseConfigService`를 인스턴스화하는 대신, 기존의 옵션 provider를 재사용하려면 `useExisting` 구문을 사용하십시오.

```typescript
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

#### 예제

작동 예제는 [여기](https://github.com/nestjs/nest/tree/master/sample/06-mongoose)에서 확인할 수 있습니다.
