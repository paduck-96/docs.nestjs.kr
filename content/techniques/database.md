### Database

Nest는 데이터베이스에 구애받지 않으며, SQL 또는 NoSQL 데이터베이스와 쉽게 통합할 수 있습니다. 선호도에 따라 여러 가지 옵션이 있습니다. 가장 일반적인 수준에서는 Express 또는 Fastify와 마찬가지로 적절한 Node.js 드라이버를 로드하여 Nest를 데이터베이스에 연결하면 됩니다.

또한 MikroORM, Sequelize, Knex.js, TypeORM, Prisma와 같은 일반적인 Node.js 데이터베이스 통합 라이브러리 또는 ORM을 직접 사용할 수 있습니다. 이들은 더 높은 수준의 추상화 작업을 제공합니다.

편의상, Nest는 TypeORM 및 Sequelize와의 통합을 위해 `@nestjs/typeorm` 및 `@nestjs/sequelize` 패키지를 기본적으로 제공하며, Mongoose와의 통합은 `@nestjs/mongoose`를 통해 제공됩니다. 이러한 통합은 모델/레포지토리 주입, 테스트 가능성, 비동기 구성 등과 같은 추가적인 NestJS 전용 기능을 제공합니다.

### TypeORM 통합

SQL 및 NoSQL 데이터베이스와 통합하기 위해 Nest는 `@nestjs/typeorm` 패키지를 제공합니다. TypeORM은 TypeScript를 위한 가장 성숙한 ORM입니다. TypeScript로 작성되었기 때문에 Nest 프레임워크와 잘 통합됩니다.

사용을 시작하려면 필요한 종속성을 먼저 설치합니다. 이 장에서는 인기 있는 MySQL 관계형 DBMS를 사용하는 방법을 시연하겠지만, TypeORM은 PostgreSQL, Oracle, Microsoft SQL Server, SQLite, MongoDB와 같은 많은 관계형 데이터베이스를 지원합니다. 이 장에서 다룰 절차는 TypeORM이 지원하는 모든 데이터베이스에 대해 동일합니다. 선택한 데이터베이스에 대한 클라이언트 API 라이브러리를 설치하기만 하면 됩니다.

```bash
$ npm install --save @nestjs/typeorm typeorm mysql2
```

설치가 완료되면 루트 `AppModule`에 `TypeOrmModule`을 가져와야 합니다.

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

> **Warning** `synchronize: true` 설정은 프로덕션에서 사용하면 안 됩니다. 그렇지 않으면 프로덕션 데이터를 잃을 수 있습니다.

`forRoot()` 메서드는 TypeORM 패키지의 `DataSource` 생성자가 노출하는 모든 구성 속성을 지원합니다. 추가로 몇 가지 추가 구성 속성이 있습니다.

<table>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>데이터베이스에 연결 시도 횟수(기본값: <code>10</code>)</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>연결 재시도 간의 지연 시간(ms)(기본값: <code>3000</code>)</td>
  </tr>
  <tr>
    <td><code>autoLoadEntities</code></td>
    <td><code>true</code>로 설정하면 엔티티가 자동으로 로드됩니다(기본값: <code>false</code>)</td>
  </tr>
</table>

> **Hint** 데이터 소스 옵션에 대해 자세히 알아보려면 [여기](https://typeorm.io/data-source-options)를 참조하십시오.

이 작업이 완료되면, TypeORM의 `DataSource` 및 `EntityManager` 객체는 전체 프로젝트에서 주입할 수 있습니다(모듈을 가져올 필요 없이).

```typescript
import { DataSource } from 'typeorm';

@Module({
  imports: [TypeOrmModule.forRoot(), UsersModule],
})
export class AppModule {
  constructor(private dataSource: DataSource) {}
}
```

#### 레포지토리 패턴

TypeORM은 **레포지토리 디자인 패턴**을 지원하므로 각 엔티티는 자체 레포지토리를 가지고 있습니다. 이러한 레포지토리는 데이터 소스에서 가져올 수 있습니다.

예제를 계속하기 위해 최소한 하나의 엔티티가 필요합니다. `User` 엔티티를 정의해 보겠습니다.

```typescript
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;
}
```

> **Hint** 엔티티에 대해 자세히 알아보려면 [TypeORM 문서](https://typeorm.io/#/entities)를 참조하십시오.

`User` 엔티티 파일은 `users` 디렉토리에 위치합니다. 이 디렉토리에는 `UsersModule`과 관련된 모든 파일이 포함됩니다. 모델 파일을 어디에 보관할지는 선택할 수 있지만, 해당 모듈 디렉토리 내에 **도메인**에 가깝게 생성하는 것이 좋습니다.

`User` 엔티티를 사용하려면, 이를 모듈 `forRoot()` 메서드 옵션의 `entities` 배열에 삽입하여 TypeORM에 알려야 합니다(정적 glob 경로를 사용하지 않는 한).

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './users/user.entity';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [User],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

다음으로 `UsersModule`을 살펴보겠습니다:

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

이 모듈은 `forFeature()` 메서드를 사용하여 현재 범위에 등록된 레포지토리를 정의합니다. 이제 `@InjectRepository()` 데코레이터를 사용하여 `UsersService`에 `UsersRepository`를 주입할 수 있습니다:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  findAll(): Promise<User[]> {
    return this.usersRepository.find();
  }

  findOne(id: number): Promise<User | null> {
    return this.usersRepository.findOneBy({ id });
  }

  async remove(id: number): Promise<void> {
    await this.usersRepository.delete(id);
  }
}
```

> **Notice** `UsersModule`을 루트 `AppModule`에 가져오는 것을 잊지 마세요.

`TypeOrmModule.forFeature`를 가져오는 모듈 외부에서 레포지토리를 사용하려면 생성된 providers를 다시 내보내야 합니다. 전체 모듈을 내보내는 방법은 다음과 같습니다:

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  exports: [TypeOrmModule]
})
export class UsersModule {}
```

이제 `UsersModule`을 `UserHttpModule`에 가져오면 후자의 모듈에서 `@InjectRepository(User)`를 사용할 수 있습니다.

```typescript
import { Module } from '@nestjs/common';
import { UsersModule } from './users.module';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  imports: [UsersModule],
  providers: [UsersService],
  controllers: [UsersController]
})
export class UserHttpModule {}
```

#### 관계

관계는 두 개 이상의 테이블 간의 연결을 나타냅니다. 관계는 각 테이블의 공통 필드를 기반으로 하며, 일반적으로 기본 키와 외래 키를 포함합니다.

관계에는 세 가지 유형이 있습니다:

<table>
  <tr>
    <td><code>One-to-one</code></td>
    <td>기본 테이블의 각 행이 외래 테이블의 한 개의 관련 행과 일치합니다. 이 유형의 관계를 정의하려면 <code>@OneToOne()</code> 데코레이터를 사용합니다.</td>
  </tr>
  <tr>
    <td><code>One-to-many / Many-to-one</code></td>
    <td>기본 테이블의 각 행이 외래 테이블의 하나 이상의 관련 행과 일치합니다. 이 유형의 관계를 정의하려면 <code>@OneToMany()</code> 및 <code

>@ManyToOne()</code> 데코레이터를 사용합니다.</td>
  </tr>
  <tr>
    <td><code>Many-to-many</code></td>
    <td>기본 테이블의 각 행이 외래 테이블의 여러 관련 행과 일치하고, 외래 테이블의 각 행도 기본 테이블의 여러 관련 행과 일치합니다. 이 유형의 관계를 정의하려면 <code>@ManyToMany()</code> 데코레이터를 사용합니다.</td>
  </tr>
</table>

엔티티에서 관계를 정의하려면 해당 **데코레이터**를 사용합니다. 예를 들어, 각 `User`가 여러 사진을 가질 수 있도록 `@OneToMany()` 데코레이터를 사용합니다.

```typescript
import { Entity, Column, PrimaryGeneratedColumn, OneToMany } from 'typeorm';
import { Photo } from '../photos/photo.entity';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;

  @OneToMany(type => Photo, photo => photo.user)
  photos: Photo[];
}
```

> **Hint** TypeORM에서 관계에 대해 더 자세히 알아보려면 [TypeORM 문서](https://typeorm.io/#/relations)를 참조하십시오.

#### 엔티티 자동 로드

데이터 소스 옵션의 `entities` 배열에 엔티티를 수동으로 추가하는 것은 번거로울 수 있습니다. 또한, 루트 모듈에서 엔티티를 참조하면 애플리케이션 도메인 경계를 깨고 구현 세부 사항이 애플리케이션의 다른 부분으로 누출될 수 있습니다. 이를 해결하기 위해 대안 솔루션이 제공됩니다. 엔티티를 자동으로 로드하려면 구성 객체( `forRoot()` 메서드에 전달된)의 `autoLoadEntities` 속성을 `true`로 설정합니다:

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...
      autoLoadEntities: true,
    }),
  ],
})
export class AppModule {}
```

이 옵션을 지정하면, `forFeature()` 메서드를 통해 등록된 모든 엔티티가 구성 객체의 `entities` 배열에 자동으로 추가됩니다.

> **Warning** `autoLoadEntities` 설정을 통해 포함되지 않은 엔티티는 `forFeature()` 메서드를 통해 등록되지 않은 엔티티입니다. 관계를 통해서만 참조된 엔티티는 포함되지 않습니다.

#### 엔티티 정의 분리

엔티티와 컬럼을 모델 내에서 데코레이터를 사용하여 정의할 수 있습니다. 그러나 일부 사람들은 ["엔티티 스키마"](https://typeorm.io/#/separating-entity-definition)를 사용하여 별도의 파일에서 엔티티와 컬럼을 정의하는 것을 선호합니다.

```typescript
import { EntitySchema } from 'typeorm';
import { User } from './user.entity';

export const UserSchema = new EntitySchema<User>({
  name: 'User',
  target: User,
  columns: {
    id: {
      type: Number,
      primary: true,
      generated: true,
    },
    firstName: {
      type: String,
    },
    lastName: {
      type: String,
    },
    isActive: {
      type: Boolean,
      default: true,
    },
  },
  relations: {
    photos: {
      type: 'one-to-many',
      target: 'Photo', // PhotoSchema의 이름
    },
  },
});
```

> **Warning** `target` 옵션을 제공하는 경우, `name` 옵션 값은 대상 클래스의 이름과 동일해야 합니다. `target`을 제공하지 않으면 임의의 이름을 사용할 수 있습니다.

Nest는 `Entity`가 예상되는 곳 어디에서나 `EntitySchema` 인스턴스를 사용할 수 있습니다.

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UserSchema } from './user.schema';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [TypeOrmModule.forFeature([UserSchema])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

#### TypeORM 트랜잭션

데이터베이스 트랜잭션은 데이터베이스 관리 시스템 내에서 데이터베이스에 대해 수행되는 작업 단위를 나타내며, 다른 트랜잭션과 독립적으로 일관되고 신뢰할 수 있는 방식으로 처리됩니다. 트랜잭션은 일반적으로 데이터베이스의 모든 변경을 나타냅니다 ([자세히 알아보기](https://en.wikipedia.org/wiki/Database_transaction)).

[TypeORM 트랜잭션](https://typeorm.io/#/transactions)을 처리하는 다양한 전략이 있습니다. `QueryRunner` 클래스를 사용하는 것을 추천합니다. 이 클래스는 트랜잭션에 대한 완전한 제어를 제공합니다.

먼저, 일반적인 방법으로 클래스에 `DataSource` 객체를 주입해야 합니다:

```typescript
@Injectable()
export class UsersService {
  constructor(private dataSource: DataSource) {}
}
```

> **Hint** `DataSource` 클래스는 `typeorm` 패키지에서 가져옵니다.

이제 이 객체를 사용하여 트랜잭션을 생성할 수 있습니다.

```typescript
async createMany(users: User[]) {
  const queryRunner = this.dataSource.createQueryRunner();

  await queryRunner.connect();
  await queryRunner.startTransaction();
  try {
    await queryRunner.manager.save(users[0]);
    await queryRunner.manager.save(users[1]);

    await queryRunner.commitTransaction();
  } catch (err) {
    // 오류가 발생했으므로 우리가 만든 변경 사항을 롤백합니다.
    await queryRunner.rollbackTransaction();
  } finally {
    // 수동으로 인스턴스화된 queryRunner를 해제해야 합니다.
    await queryRunner.release();
  }
}
```

> **Hint** `dataSource`는 `QueryRunner`를 생성하는 데만 사용됩니다. 그러나 이 클래스를 테스트하려면 전체 `DataSource` 객체를 모킹해야 합니다(여러 메서드를 노출함). 따라서 헬퍼 팩토리 클래스(예: `QueryRunnerFactory`)를 사용하고 트랜잭션을 유지하는 데 필요한 제한된 메서드 세트를 정의하는 인터페이스를 정의하는 것이 좋습니다. 이 기술은 이러한 메서드를 모킹하는 작업을 상당히 간단하게 만듭니다.

대안으로, `DataSource` 객체의 `transaction` 메서드를 사용하는 콜백 스타일 접근 방식을 사용할 수 있습니다([자세히 보기](https://typeorm.io/#/transactions/creating-and-using-transactions)).

```typescript
async createMany(users: User[]) {
  await this.dataSource.transaction(async manager => {
    await manager.save(users[0]);
    await manager.save(users[1]);
  });
}
```

#### 서브스크라이버

TypeORM [서브스크라이버](https://typeorm.io/#/listeners-and-subscribers/what-is-a-subscriber)를 사용하면 특정 엔티티 이벤트를 수신할 수 있습니다.

```typescript
import {
  DataSource,
  EntitySubscriberInterface,
  EventSubscriber,
  InsertEvent,
} from 'typeorm';
import { User } from './user.entity';

@EventSubscriber()
export class UserSubscriber implements EntitySubscriberInterface<User> {
  constructor(dataSource: DataSource) {
    dataSource.subscribers.push(this);
  }

  listenTo() {
    return User;
  }

  beforeInsert(event: InsertEvent<User>) {
    console.log(`BEFORE USER INSERTED: `, event.entity);
  }
}
```

> **Warning** 이벤트 서브스크라이버는 [요청 범위](/fundamentals/injection-scopes)로 설정할 수 없습니다.

이제 `UserSubscriber` 클래스를 `providers` 배열에 추가합니다:

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UserSubscriber } from './user.subscriber';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService, UserSubscriber],
  controllers: [UsersController],
})
export class UsersModule {}
```

> **Hint** 엔티티 서브스크라이버에 대해 더 자세히 알아보려면 [여기](https://typeorm.io/#/listeners-and-subscribers/what-is-a-subscriber)를 참조하십시오.

#### 마이그레이션

[마이그레이션](https://typeorm.io/#/migrations)은 데이터베이스의 기존 데이터를 보존하면서 애플리케이션의 데이터 모델과 동기화 상태를 유지하기 위해 데이터베이스 스키마를 점진적으로 업데이트하는 방법을 제공합니다. 마이그레이션을 생성, 실행 및 되돌리기 위해 TypeORM은 전용 [CLI](https://typeorm.io/#/migrations/creating-a-new-migration)를 제공합니다.

마이그레이션 클래스는 Nest 애플리케이션 소스 코드와 분리되어 있습니다. 그 생명 주기는 TypeORM CLI에 의해 관리됩니다. 따라서 마이그레이션

에서는 종속성 주입 및 다른 Nest 전용 기능을 활용할 수 없습니다. 마이그레이션에 대해 자세히 알아보려면 [TypeORM 문서](https://typeorm.io/#/migrations/creating-a-new-migration)에 있는 가이드를 참조하십시오.

#### 여러 데이터베이스

일부 프로젝트는 여러 데이터베이스 연결이 필요합니다. 이 모듈로도 이를 달성할 수 있습니다. 여러 연결을 사용하려면 먼저 연결을 만듭니다. 이 경우 데이터 소스 이름 지정이 **필수**가 됩니다.

`Album` 엔티티가 자체 데이터베이스에 저장되어 있다고 가정해 보겠습니다.

```typescript
const defaultOptions = {
  type: 'postgres',
  port: 5432,
  username: 'user',
  password: 'password',
  database: 'db',
  synchronize: true,
};

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...defaultOptions,
      host: 'user_db_host',
      entities: [User],
    }),
    TypeOrmModule.forRoot({
      ...defaultOptions,
      name: 'albumsConnection',
      host: 'album_db_host',
      entities: [Album],
    }),
  ],
})
export class AppModule {}
```

> **Notice** 데이터 소스에 `name`을 설정하지 않으면 이름이 `default`로 설정됩니다. 동일한 이름을 가진 여러 연결이 있으면 덮어쓰게 되므로 이름이 없는 여러 연결 또는 동일한 이름을 가진 여러 연결을 사용해서는 안 됩니다.

이제 `User` 및 `Album` 엔티티가 각각 자체 데이터 소스에 등록되었습니다. 이 설정에서는 `TypeOrmModule.forFeature()` 메서드와 `@InjectRepository()` 데코레이터에 어떤 데이터 소스를 사용할지 알려야 합니다. 데이터 소스 이름을 전달하지 않으면 `default` 데이터 소스가 사용됩니다.

```typescript
@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    TypeOrmModule.forFeature([Album], 'albumsConnection'),
  ],
})
export class AppModule {}
```

지정된 데이터 소스에 대해 `DataSource` 또는 `EntityManager`를 주입할 수도 있습니다:

```typescript
@Injectable()
export class AlbumsService {
  constructor(
    @InjectDataSource('albumsConnection')
    private dataSource: DataSource,
    @InjectEntityManager('albumsConnection')
    private entityManager: EntityManager,
  ) {}
}
```

어떤 `DataSource`든 providers에 주입할 수도 있습니다:

```typescript
@Module({
  providers: [
    {
      provide: AlbumsService,
      useFactory: (albumsConnection: DataSource) => {
        return new AlbumsService(albumsConnection);
      },
      inject: [getDataSourceToken('albumsConnection')],
    },
  ],
})
export class AlbumsModule {}
```

#### 테스트

애플리케이션의 단위 테스트를 작성할 때는 일반적으로 데이터베이스 연결을 피하고 싶습니다. 이는 테스트 스위트를 독립적으로 유지하고 실행 프로세스를 가능한 한 빠르게 만들기 위함입니다. 하지만 클래스가 데이터 소스에서 가져온 레포지토리에 의존할 수 있습니다. 이를 어떻게 처리할까요? 해결책은 모의 레포지토리를 만드는 것입니다. 이를 위해 [사용자 정의 providers](/fundamentals/custom-providers)를 설정합니다. 등록된 각 레포지토리는 자동으로 `<EntityName>Repository` 토큰으로 표시됩니다. 여기서 `EntityName`은 엔티티 클래스의 이름입니다.

`@nestjs/typeorm` 패키지는 주어진 엔티티를 기반으로 준비된 토큰을 반환하는 `getRepositoryToken()` 함수를 노출합니다.

```typescript
@Module({
  providers: [
    UsersService,
    {
      provide: getRepositoryToken(User),
      useValue: mockRepository,
    },
  ],
})
export class UsersModule {}
```

이제 대체 `mockRepository`가 `UsersRepository`로 사용됩니다. 클래스가 `@InjectRepository()` 데코레이터를 사용하여 `UsersRepository`를 요청할 때마다 Nest는 등록된 `mockRepository` 객체를 사용합니다.

#### 비동기 구성

레포지토리 모듈 옵션을 정적으로 전달하는 대신 비동기적으로 전달하고 싶을 수 있습니다. 이 경우 `forRootAsync()` 메서드를 사용하여 비동기 구성을 처리하는 여러 가지 방법을 제공합니다.

한 가지 접근 방식은 팩토리 함수를 사용하는 것입니다:

```typescript
TypeOrmModule.forRootAsync({
  useFactory: () => ({
    type: 'mysql',
    host: 'localhost',
    port: 3306,
    username: 'root',
    password: 'root',
    database: 'test',
    entities: [],
    synchronize: true,
  }),
});
```

우리의 팩토리는 다른 [비동기 provider](https://docs.nestjs.com/fundamentals/async-providers)와 마찬가지로 동작합니다(예: `async`가 될 수 있으며 `inject`를 통해 종속성을 주입할 수 있습니다).

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (configService: ConfigService) => ({
    type: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    entities: [],
    synchronize: true,
  }),
  inject: [ConfigService],
});
```

대안으로 `useClass` 구문을 사용할 수 있습니다:

```typescript
TypeOrmModule.forRootAsync({
  useClass: TypeOrmConfigService,
});
```

위의 구성은 `TypeOrmModule` 내부에서 `TypeOrmConfigService`를 인스턴스화하고 `createTypeOrmOptions()`를 호출하여 옵션 객체를 제공합니다. 여기서 `TypeOrmConfigService`가 `TypeOrmOptionsFactory` 인터페이스를 구현해야 함을 의미합니다. 아래와 같이 구현합니다:

```typescript
@Injectable()
export class TypeOrmConfigService implements TypeOrmOptionsFactory {
  createTypeOrmOptions(): TypeOrmModuleOptions {
    return {
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [],
      synchronize: true,
    };
  }
}
```

`TypeOrmModule` 내부에서 `TypeOrmConfigService`를 생성하지 않고 다른 모듈에서 가져온 provider를 사용할 때는 `useExisting` 구문을 사용할 수 있습니다.

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

이 구성은 `useClass`와 동일한 방식으로 작동하지만 중요한 차이점이 있습니다. `TypeOrmModule`이 새 인스턴스를 만들지 않고 기존 `ConfigService`를 재사용하기 위해 가져온 모듈을 조회합니다.

> **Hint** `name` 속성이 `useFactory`, `useClass` 또는 `useValue` 속성 수준에서 정의되었는지 확인하십시오. 이렇게 하면 Nest가 적절한 주입 토큰 아래 데이터 소스를 올바르게 등록할 수 있습니다.

#### 사용자 정의 데이터 소스 팩토리

`useFactory`, `useClass`, `useExisting`을 사용하는 비동기 구성과 함께 `dataSourceFactory` 함수를 선택적으로 지정할 수 있습니다. 이를 통해 `TypeOrmModule`이 데이터 소스를 생성하는 대신 사용자가 직접 TypeORM 데이터 소스를 제공할 수 있습니다.

`dataSourceFactory`는 비동기 구성 중 `useFactory`, `useClass`, `useExisting`을 사용하여 구성된 TypeORM `DataSourceOptions`를 수신하고 `DataSource`를 해결하는 `Promise`를 반환합니다.

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (configService: ConfigService) => ({
    type: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    entities: [],
    synchronize: true,
  }),
  dataSourceFactory: async (options) => {
    const dataSource = await new DataSource(options).initialize();
    return dataSource;
  },
});
```

> **Hint** `DataSource` 클래스는 `typeorm` 패키지에서 가져옵니다.

#### 예제

작동하는 예제는 [여기](https://github.com/nestjs/nest/tree/master/sample/05-sql-typeorm)에서 확인할 수 있습니다.

### Sequelize 통합

TypeORM을 사용하는 대안으로, `@nestjs/sequelize` 패키지를 사용하여 [Sequelize](https://sequelize.org/) ORM을 사용할 수 있습니다. 또한, 엔티티를 선언적으로 정의하기 위한 추가 데코레이터 세트를 제공하는 [sequelize-typescript](https://github.com/RobinBuschmann/sequelize-typescript) 패키지를 활용합니다.

사용을 시작하려면 필요한 종속성을 먼저 설치합니다. 이 장에서는 인기 있는 MySQL 관계형 DBMS를 사용하는 방법을 시연하겠지만, Sequelize는 PostgreSQL, MySQL, Microsoft SQL Server, SQLite, MariaDB와 같은 많은 관계형 데이터베이스를 지원합니다. 이 장에서 다룰 절차는 Sequelize가 지원하는 모든 데이터베이스에 대해 동일합니다. 선택한 데이터베이스에 대한 클라이언

트 API 라이브러리를 설치하기만 하면 됩니다.

```bash
$ npm install --save @nestjs/sequelize sequelize sequelize-typescript mysql2
$ npm install --save-dev @types/sequelize
```

설치가 완료되면 루트 `AppModule`에 `SequelizeModule`을 가져와야 합니다.

```typescript
import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';

@Module({
  imports: [
    SequelizeModule.forRoot({
      dialect: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      models: [],
    }),
  ],
})
export class AppModule {}
```

`forRoot()` 메서드는 Sequelize 생성자가 노출하는 모든 구성 속성을 지원합니다. 추가로 몇 가지 추가 구성 속성이 있습니다.

<table>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>데이터베이스에 연결 시도 횟수(기본값: <code>10</code>)</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>연결 재시도 간의 지연 시간(ms)(기본값: <code>3000</code>)</td>
  </tr>
  <tr>
    <td><code>autoLoadModels</code></td>
    <td><code>true</code>로 설정하면 모델이 자동으로 로드됩니다(기본값: <code>false</code>)</td>
  </tr>
  <tr>
    <td><code>keepConnectionAlive</code></td>
    <td><code>true</code>로 설정하면 애플리케이션 종료 시 연결이 닫히지 않습니다(기본값: <code>false</code>)</td>
  </tr>
  <tr>
    <td><code>synchronize</code></td>
    <td><code>true</code>로 설정하면 자동으로 로드된 모델이 동기화됩니다(기본값: <code>true</code>)</td>
  </tr>
</table>

이 작업이 완료되면 `Sequelize` 객체를 전체 프로젝트에서 주입할 수 있습니다(모듈을 가져올 필요 없이).

```typescript
import { Injectable } from '@nestjs/common';
import { Sequelize } from 'sequelize-typescript';

@Injectable()
export class AppService {
  constructor(private sequelize: Sequelize) {}
}
```

#### 모델

Sequelize는 액티브 레코드 패턴을 구현합니다. 이 패턴에서는 모델 클래스를 직접 사용하여 데이터베이스와 상호 작용합니다. 예제를 계속하기 위해 최소한 하나의 모델이 필요합니다. `User` 모델을 정의해 보겠습니다.

```typescript
import { Column, Model, Table } from 'sequelize-typescript';

@Table
export class User extends Model {
  @Column
  firstName: string;

  @Column
  lastName: string;

  @Column({ defaultValue: true })
  isActive: boolean;
}
```

> **Hint** 사용 가능한 데코레이터에 대해 자세히 알아보려면 [여기](https://github.com/RobinBuschmann/sequelize-typescript#column)를 참조하십시오.

`User` 모델 파일은 `users` 디렉토리에 위치합니다. 이 디렉토리에는 `UsersModule`과 관련된 모든 파일이 포함됩니다. 모델 파일을 어디에 보관할지는 선택할 수 있지만, 해당 모듈 디렉토리 내에 **도메인**에 가깝게 생성하는 것이 좋습니다.

`User` 모델을 사용하려면 이를 모듈 `forRoot()` 메서드 옵션의 `models` 배열에 삽입하여 Sequelize에 알려야 합니다:

```typescript
import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';
import { User } from './users/user.model';

@Module({
  imports: [
    SequelizeModule.forRoot({
      dialect: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      models: [User],
    }),
  ],
})
export class AppModule {}
```

다음으로 `UsersModule`을 살펴보겠습니다:

```typescript
import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';
import { User } from './user.model';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [SequelizeModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

이 모듈은 `forFeature()` 메서드를 사용하여 현재 범위에 등록된 모델을 정의합니다. 이제 `@InjectModel()` 데코레이터를 사용하여 `UsersService`에 `UserModel`을 주입할 수 있습니다:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/sequelize';
import { User } from './user.model';

@Injectable()
export class UsersService {
  constructor(
    @InjectModel(User)
    private userModel: typeof User,
  ) {}

  async findAll(): Promise<User[]> {
    return this.userModel.findAll();
  }

  findOne(id: string): Promise<User> {
    return this.userModel.findOne({
      where: {
        id,
      },
    });
  }

  async remove(id: string): Promise<void> {
    const user = await this.findOne(id);
    await user.destroy();
  }
}
```

> **Notice** `UsersModule`을 루트 `AppModule`에 가져오는 것을 잊지 마세요.

`SequelizeModule.forFeature`를 가져오는 모듈 외부에서 레포지토리를 사용하려면 생성된 providers를 다시 내보내야 합니다. 전체 모듈을 내보내는 방법은 다음과 같습니다:

```typescript
import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';
import { User } from './user.entity';

@Module({
  imports: [SequelizeModule.forFeature([User])],
  exports: [SequelizeModule]
})
export class UsersModule {}
```

이제 `UsersModule`을 `UserHttpModule`에 가져오면 후자의 모듈에서 `@InjectModel(User)`를 사용할 수 있습니다.

```typescript
import { Module } from '@nestjs/common';
import { UsersModule } from './users.module';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  imports: [UsersModule],
  providers: [UsersService],
  controllers: [UsersController]
})
export class UserHttpModule {}
```

#### 관계

관계는 두 개 이상의 테이블 간의 연결을 나타냅니다. 관계는 각 테이블의 공통 필드를 기반으로 하며, 일반적으로 기본 키와 외래 키를 포함합니다.

관계에는 세 가지 유형이 있습니다:

<table>
  <tr>
    <td><code>One-to-one</code></td>
    <td>기본 테이블의 각 행이 외래 테이블의 한 개의 관련 행과 일치합니다.</td>
  </tr>
  <tr>
    <td><code>One-to-many / Many-to-one</code></td>
    <td>기본 테이블의 각 행이 외래 테이블의 하나 이상의 관련 행과 일치합니다.</td>
  </tr>
  <tr>
    <td><code>Many-to-many</code></td>
    <td>기본 테이블의 각 행이 외래 테이블의 여러 관련 행과 일치하고, 외래 테이블의 각 행도 기본 테이블의 여러 관련 행과 일치합니다.</td>
  </tr>
</table>

모델에서 관계를 정의하려면 해당 **데코레이터**를 사용합니다. 예를 들어, 각 `User`가 여러 사진을 가질 수 있도록 `@HasMany()` 데코레이터를 사용합니다.

```typescript
import { Column, Model, Table, HasMany } from 'sequelize-typescript';
import { Photo } from '../photos/photo.model';

@Table
export class User extends Model {
  @Column
  firstName: string;

  @Column
  lastName: string;

  @Column({ defaultValue: true })
  isActive: boolean;

  @HasMany(() => Photo)
  photos: Photo[];
}
```

> **Hint** Sequelize에서 관계에 대해 자세히 알아보려면 [여기](https://github.com/RobinBuschmann/sequelize-typescript#model-association)를 참조하십시오.

#### 모델 자동 로드

연결 옵션의 `models` 배열에 모델을 수동으로 추가하는 것은 번거로울 수 있습니다. 또한, 루트 모듈에서 모델을 참조하면 애플리케이션 도메인 경계를 깨고 구현 세부 사항이 애플리케이션의 다른 부분으로 누출될 수 있습니다. 이 문제를 해결하려면 구성 객체( `forRoot()` 메서드에 전달된)의 `autoLoadModels` 및 `synchronize` 속성을 모두 `true`로 설정하여 모델을 자동으로 로드합니다.

```typescript
import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';

@Module({
  imports: [
    SequelizeModule.forRoot({
      ...
      autoLoadModels: true,
      synchronize: true,
    }),
  ],
})
export class

 AppModule {}
```

이 옵션을 지정하면, `forFeature()` 메서드를 통해 등록된 모든 모델이 구성 객체의 `models` 배열에 자동으로 추가됩니다.

> **Warning** `forFeature()` 메서드를 통해 등록되지 않고 모델에서만 참조된 모델은 포함되지 않습니다.

#### Sequelize 트랜잭션

데이터베이스 트랜잭션은 데이터베이스 관리 시스템 내에서 데이터베이스에 대해 수행되는 작업 단위를 나타내며, 다른 트랜잭션과 독립적으로 일관되고 신뢰할 수 있는 방식으로 처리됩니다. 트랜잭션은 일반적으로 데이터베이스의 모든 변경을 나타냅니다 ([자세히 알아보기](https://en.wikipedia.org/wiki/Database_transaction)).

[Sequelize 트랜잭션](https://sequelize.org/v5/manual/transactions.html)을 처리하는 다양한 전략이 있습니다. 아래는 관리된 트랜잭션(자동 콜백)의 샘플 구현입니다.

먼저, 일반적인 방법으로 클래스에 `Sequelize` 객체를 주입해야 합니다:

```typescript
@Injectable()
export class UsersService {
  constructor(private sequelize: Sequelize) {}
}
```

> **Hint** `Sequelize` 클래스는 `sequelize-typescript` 패키지에서 가져옵니다.

이제 이 객체를 사용하여 트랜잭션을 생성할 수 있습니다.

```typescript
async createMany() {
  try {
    await this.sequelize.transaction(async t => {
      const transactionHost = { transaction: t };

      await this.userModel.create(
          { firstName: 'Abraham', lastName: 'Lincoln' },
          transactionHost,
      );
      await this.userModel.create(
          { firstName: 'John', lastName: 'Boothe' },
          transactionHost,
      );
    });
  } catch (err) {
    // 트랜잭션이 롤백되었습니다.
    // err는 트랜잭션 콜백에 반환된 프로미스 체인을 거부한 오류입니다.
  }
}
```

> **Hint** `Sequelize` 인스턴스는 트랜잭션을 시작하는 데만 사용됩니다. 그러나 이 클래스를 테스트하려면 전체 `Sequelize` 객체를 모킹해야 합니다(여러 메서드를 노출함). 따라서 헬퍼 팩토리 클래스(예: `TransactionRunner`)를 사용하고 트랜잭션을 유지하는 데 필요한 제한된 메서드 세트를 정의하는 인터페이스를 정의하는 것이 좋습니다. 이 기술은 이러한 메서드를 모킹하는 작업을 상당히 간단하게 만듭니다.

#### 마이그레이션

[마이그레이션](https://sequelize.org/v5/manual/migrations.html)은 데이터베이스의 기존 데이터를 보존하면서 애플리케이션의 데이터 모델과 동기화 상태를 유지하기 위해 데이터베이스 스키마를 점진적으로 업데이트하는 방법을 제공합니다. 마이그레이션을 생성, 실행 및 되돌리기 위해 Sequelize는 전용 [CLI](https://sequelize.org/v5/manual/migrations.html#the-cli)를 제공합니다.

마이그레이션 클래스는 Nest 애플리케이션 소스 코드와 분리되어 있습니다. 그 생명 주기는 Sequelize CLI에 의해 관리됩니다. 따라서 마이그레이션에서는 종속성 주입 및 다른 Nest 전용 기능을 활용할 수 없습니다. 마이그레이션에 대해 자세히 알아보려면 [Sequelize 문서](https://sequelize.org/v5/manual/migrations.html#the-cli)에 있는 가이드를 참조하십시오.

#### 여러 데이터베이스

일부 프로젝트는 여러 데이터베이스 연결이 필요합니다. 이 모듈로도 이를 달성할 수 있습니다. 여러 연결을 사용하려면 먼저 연결을 만듭니다. 이 경우 연결 이름 지정이 **필수**가 됩니다.

`Album` 엔티티가 자체 데이터베이스에 저장되어 있다고 가정해 보겠습니다.

```typescript
const defaultOptions = {
  dialect: 'postgres',
  port: 5432,
  username: 'user',
  password: 'password',
  database: 'db',
  synchronize: true,
};

@Module({
  imports: [
    SequelizeModule.forRoot({
      ...defaultOptions,
      host: 'user_db_host',
      models: [User],
    }),
    SequelizeModule.forRoot({
      ...defaultOptions,
      name: 'albumsConnection',
      host: 'album_db_host',
      models: [Album],
    }),
  ],
})
export class AppModule {}
```

> **Notice** 연결에 `name`을 설정하지 않으면 이름이 `default`로 설정됩니다. 동일한 이름을 가진 여러 연결이 있으면 덮어쓰게 되므로 이름이 없는 여러 연결 또는 동일한 이름을 가진 여러 연결을 사용해서는 안 됩니다.

이제 `User` 및 `Album` 모델이 각각 자체 연결에 등록되었습니다. 이 설정에서는 `SequelizeModule.forFeature()` 메서드와 `@InjectModel()` 데코레이터에 어떤 연결을 사용할지 알려야 합니다. 연결 이름을 전달하지 않으면 `default` 연결이 사용됩니다.

```typescript
@Module({
  imports: [
    SequelizeModule.forFeature([User]),
    SequelizeModule.forFeature([Album], 'albumsConnection'),
  ],
})
export class AppModule {}
```

지정된 연결에 대해 `Sequelize` 인스턴스를 주입할 수도 있습니다:

```typescript
@Injectable()
export class AlbumsService {
  constructor(
    @InjectConnection('albumsConnection')
    private sequelize: Sequelize,
  ) {}
}
```

어떤 `Sequelize` 인스턴스든 providers에 주입할 수도 있습니다:

```typescript
@Module({
  providers: [
    {
      provide: AlbumsService,
      useFactory: (albumsSequelize: Sequelize) => {
        return new AlbumsService(albumsSequelize);
      },
      inject: [getDataSourceToken('albumsConnection')],
    },
  ],
})
export class AlbumsModule {}
```

#### 테스트

애플리케이션의 단위 테스트를 작성할 때는 일반적으로 데이터베이스 연결을 피하고 싶습니다. 이는 테스트 스위트를 독립적으로 유지하고 실행 프로세스를 가능한 한 빠르게 만들기 위함입니다. 하지만 클래스가 연결 인스턴스에서 가져온 모델에 의존할 수 있습니다. 이를 어떻게 처리할까요? 해결책은 모의 모델을 만드는 것입니다. 이를 위해 [사용자 정의 providers](/fundamentals/custom-providers)를 설정합니다. 등록된 각 모델은 자동으로 `<ModelName>Model` 토큰으로 표시됩니다. 여기서 `ModelName`은 모델 클래스의 이름입니다.

`@nestjs/sequelize` 패키지는 주어진 모델을 기반으로 준비된 토큰을 반환하는 `getModelToken()` 함수를 노출합니다.

```typescript
@Module({
  providers: [
    UsersService,
    {
      provide: getModelToken(User),
      useValue: mockModel,
    },
  ],
})
export class UsersModule {}
```

이제 대체 `mockModel`이 `UserModel`로 사용됩니다. 클래스가 `@InjectModel()` 데코레이터를 사용하여 `UserModel`을 요청할 때마다 Nest는 등록된 `mockModel` 객체를 사용합니다.

#### 비동기 구성

`SequelizeModule` 옵션을 정적으로 전달하는 대신 비동기적으로 전달하고 싶을 수 있습니다. 이 경우 `forRootAsync()` 메서드를 사용하여 비동기 구성을 처리하는 여러 가지 방법을 제공합니다.

한 가지 접근 방식은 팩토리 함수를 사용하는 것입니다:

```typescript
SequelizeModule.forRootAsync({
  useFactory: () => ({
    dialect: 'mysql',
    host: 'localhost',
    port: 3306,
    username: 'root',
    password: 'root',
    database: 'test',
    models: [],
  }),
});
```

우리의 팩토리는 다른 [비동기 provider](https://docs.nestjs.com/fundamentals/async-providers)와 마찬가지로 동작합니다(예: `async`가 될 수 있으며 `inject`를 통해 종속성을 주입할 수 있습니다).

```typescript
SequelizeModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (configService: ConfigService) => ({
    dialect: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    models: [],
  }),
  inject: [ConfigService],
});
```

대안으로 `useClass` 구문을 사용할 수 있습니다:

```typescript
SequelizeModule.forRootAsync({
  useClass: SequelizeConfigService,
});
```

위의 구성은 `SequelizeModule` 내부에서 `SequelizeConfigService`를 인스턴스화하고 `createSequelizeOptions()`를 호출하여 옵션 객체를 제공합니다. 여기서 `SequelizeConfigService`가 `SequelizeOptionsFactory` 인터페이스를 구현해야 함을 의미합니다. 아래와 같이 구현합니다:

```typescript
@Injectable()
class SequelizeConfigService implements SequelizeOptionsFactory {
  createSequelizeOptions(): SequelizeModuleOptions {
    return {
      dialect: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      models: [],
    };
  }
}
```

`SequelizeModule` 내부에서 `SequelizeConfigService`를 생성하지 않고 다른 모듈에서 가져온 provider를 사용할 때는 `useExisting` 구문을 사용할 수 있습니다.

```typescript
Se

quelizeModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

이 구성은 `useClass`와 동일한 방식으로 작동하지만 중요한 차이점이 있습니다. `SequelizeModule`이 새 인스턴스를 만들지 않고 기존 `ConfigService`를 재사용하기 위해 가져온 모듈을 조회합니다.

#### 예제

작동하는 예제는 [여기](https://github.com/nestjs/nest/tree/master/sample/07-sequelize)에서 확인할 수 있습니다.
