### Veritabanı

Nest, herhangi bir SQL veya NoSQL veritabanıyla kolayca entegre olmanıza olanak tanıyan veritabanı bağımsız bir yapıya sahiptir. Tercihlerinize bağlı olarak bir dizi seçeneğiniz bulunmaktadır. Genelde, Nest'i bir veritabanına bağlamak, [Express](https://expressjs.com/en/guide/database-integration.html) veya Fastify ile olduğu gibi uygun bir Node.js sürücüsünü yüklemekle ilgilidir.

Ayrıca, [MikroORM](https://mikro-orm.io/) (bkz. [MikroORM rehberi](/docs/recipes/mikroorm)), [Sequelize](https://sequelize.org/) (bkz. [Sequelize entegrasyonu](/docs/techniques/database#sequelize-integration)), [Knex.js](https://knexjs.org/) (bkz. [Knex.js öğretici](https://dev.to/nestjs/build-a-nestjs-module-for-knex-js-or-other-resource-based-libraries-in-5-minutes-12an)), [TypeORM](https://github.com/typeorm/typeorm) ve [Prisma](https://www.github.com/prisma/prisma) gibi genel amaçlı Node.js veritabanı entegrasyon kütüphaneleri veya ORM'leri doğrudan kullanabilirsiniz. Bu, daha yüksek bir soyutlama seviyesinde çalışmanıza olanak tanır.

Pratiklik açısından, Nest, `@nestjs/typeorm` ve `@nestjs/sequelize` paketleri aracılığıyla TypeORM ve Sequelize ile sıkı entegrasyon sağlar. Bu entegrasyonlar, model/repository enjeksiyonu, test edilebilirlik ve seçtiğiniz veritabanına erişimi daha da kolaylaştırmak için özel NestJS özellikleri sağlar.

### TypeORM Entegrasyonu

SQL ve NoSQL veritabanları ile entegrasyon için Nest, `@nestjs/typeorm` paketini sağlar. [TypeORM](https://github.com/typeorm/typeorm), TypeScript için mevcut olan en olgun Nesne İlişkisel Haritalayıcı (ORM) 'dir. TypeScript'te yazıldığından, Nest çerçevesiyle iyi entegre olur.

Onu kullanmaya başlamak için önce gerekli bağımlılıkları yükleriz. Bu bölümde, popüler [MySQL](https://www.mysql.com/) İlişkisel Veritabanı Yönetim Sistemi'ni kullanmayı göstereceğiz, ancak TypeORM, PostgreSQL, Oracle, Microsoft SQL Server, SQLite ve hatta MongoDB gibi birçok ilişkisel veritabanını destekler. Bu bölümde anlattığımız işlem, TypeORM tarafından desteklenen herhangi bir veritabanı için aynı olacaktır. Sadece seçtiğiniz veritabanı için ilişkili istemci API kitaplıklarını yüklemeniz gerekecektir.

```bash
$ npm install --save @nestjs/typeorm typeorm mysql2
```

Yükleme işlemi tamamlandıktan sonra, `TypeOrmModule`'u ana `AppModule` içine alabiliriz.

```typescript
@@filename(app.module)
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

> uyarı **Uyarı** `synchronize: true` ayarının üretimde kullanılmaması gerekir - aksi takdirde üretim verilerinizi kaybedebilirsiniz.

`forRoot()` yöntemi, [TypeORM](https://typeorm.io/data-source-options#common-data-source-options) paketindeki `DataSource` kurucusu tarafından açıklanan tüm yapılandırma özelliklerini destekler. Ayrıca aşağıda açıklanan birkaç ek yapılandırma özelliği bulunmaktadır.

<table>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>Veritabanına bağlanma denemelerinin sayısı (varsayılan: <code>10</code>)</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>Veritabanına bağlanma denemeleri arasındaki gecikme süresi (ms) (varsayılan: <code>3000</code>)</td>
  </tr>
  <tr>
    <td><code>autoLoadEntities</code></td>
    <td>Eğer <code>true</code> ise, varlıklar otomatik olarak yüklenecektir (varsayılan: <code>false</code>)</td>
  </tr>
</table>

> info **İpucu** Veri kaynağı seçenekleri hakkında daha fazla bilgi için [buraya](https://typeorm.io/data-source-options) bakın.

Bunu yaptıktan sonra, TypeORM `DataSource` ve `EntityManager` nesneleri projenin tamamına (herhangi bir modülü içe aktarmadan) enjekte edilebilir hale gelir, örneğin:

```typescript
@@filename(app.module)
import { DataSource } from 'typeorm';

@Module({
  imports: [TypeOrmModule.forRoot(), UsersModule],
})
export class AppModule {
  constructor(private dataSource: DataSource) {}
}
@@switch
import { DataSource } from 'typeorm';

@Dependencies(DataSource)
@Module({
  imports: [TypeOrmModule.forRoot(), UsersModule],
})
export class AppModule {
  constructor(dataSource) {
    this.dataSource = dataSource;
  }
}
```

#### Repository Deseni

[TypeORM](https://github.com/typeorm/typeorm), her varlık için kendi repository'sini destekler. Bu repository'ler, veritabanı veri kaynağından elde edilebilir.

Örneğe devam etmek için en az bir varlık (entity) gerekli. Hadi `User` varlığını tanımlayalım.

```typescript
@@filename(user.entity)
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

> info **İpucu** [TypeORM belgelerinde](https://typeorm.io/#/entities) varlıklar hakkında daha fazla bilgi edinin.

`User` varlık dosyası, `users` dizininde bulunmaktadır. Bu dizin, `UsersModule` ile ilgili tüm dosyaları içerir. Model dosyalarınızı nereye koyacağınıza siz karar verebilirsiniz, ancak genellikle onları **domain** (alan)larına yakın, ilgili modül dizininde oluşturmanızı öneririz.

`User` varlığını kullanmaya başlamak için, onu modül `forRoot()` metodunun seçenekleri içindeki `entities` dizisine (eğer statik bir glob yolunu kullanmıyorsanız) ekleyerek TypeORM'a bildirmemiz gerekmektedir:

```typescript
@@filename(app.module)
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

Daha sonra, `UsersModule`'a bakalım:

```typescript
@@filename(users.module)
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

Bu modül, hangi repository'lerin mevcut kapsamda kaydedildiğini tanımlamak için `forFeature()` metodunu kullanır. Bu yerine getirildiğinde, `@InjectRepository()` dekoratörünü kullanarak `UsersService` içine `UsersRepository`'yi enjekte edebiliriz:

```typescript
@@filename(users.service)
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
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { getRepositoryToken } from '@nestjs/typeorm';
import { User } from './user.entity';

@Injectable()
@Dependencies(getRepositoryToken(User))
export class UsersService {
  constructor(usersRepository) {
    this.usersRepository = usersRepository;
  }

  findAll() {
    return this.usersRepository.find();
  }

  findOne(id) {
    return this.usersRepository.findOneBy({ id });
  }

  async remove(id) {
    await this.usersRepository.delete(id);
  }
}
```

> warning **Dikkat** `UsersModule`'u root `AppModule` içine eklemeyi unutmayın.

`TypeOrmModule.forFeature`'i içe aktaran modül dışında repository'yi kullanmak istiyorsanız, bu modül tarafından oluşturulan sağlayıcıları yeniden ihraç etmeniz gerekir.
Bunu şu şekilde yapabilirsiniz:

```typescript
@@filename(users.module)
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  exports: [TypeOrmModule]
})
export class UsersModule {}
```

Şimdi `UserHttpModule` içinde `UsersModule`'u içe aktarırsak, bu modülün sağlayıcılarında `@InjectRepository(User)`'yi kullanabiliriz.

```typescript
@@filename(users-http.module)
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

#### İlişkiler

İlişkiler, iki veya daha fazla tablo arasında kurulan bağlantılardır. İlişkiler, genellikle her iki tablodan gelen anahtarlarla ilgili olarak gerçekleşir.

Üç tür ilişki vardır:

<table>
  <tr>
    <td><code>One-to-one</code></td>
    <td>Primary (ana) tablodaki her satır, yalnızca bir ve yalnızca bir ilişkili satıra sahiptir. Bu tür bir ilişkiyi tanımlamak için <code>@OneToOne()</code> dekoratörünü kullanın.</td>
  </tr>
  <tr>
    <td><code>One-to-many / Many-to-one</code></td>
    <td>Primary (ana) tablodaki her satır, ilişkili (foreign) tabloda bir veya daha fazla ilgili satıra sahiptir. Bu tür bir ilişkiyi tanımlamak için <code>@OneToMany()</code> ve <code>@ManyToOne()</code> dekoratörlerini kullanın.</td>
  </tr>
  <tr>
    <td><code>Many-to-many</code></td>
    <td>Primary (ana) tablodaki her satır, ilişkili (foreign) tabloda birçok ilgili satıra sahiptir ve ilişkili (foreign) tablodaki her kayıt, birçok ilgili satıra sahiptir. Bu tür bir ilişkiyi tanımlamak için <code>@ManyToMany()</code> dekoratörünü kullanın.</td>
  </tr>
</table>

Varlıklarda ilişkileri tanımlamak için ilgili **dekoratörleri** kullanın. Örneğin, her `User`'ın birden fazla fotoğrafı olabileceğini tanımlamak için `@OneToMany()` dekoratörünü kullanın.

```typescript
@@filename(user.entity)
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

> info **İpucu** TypeORM'deki ilişkiler hakkında daha fazla bilgi için [TypeORM belgelerini](https://typeorm.io/#/relations) ziyaret edin.

#### Otomatik varlık yükleme

Varlıkları veri kaynağı seçeneklerinin `entities` dizisine manuel olarak eklemek zahmetli olabilir. Ayrıca, varlıkları kök modülden referans vermek, uygulama alanı sınırlarını bozar ve diğer uygulama alanlarına uygulama detaylarını sızdırır. Bu sorunu çözmek için alternatif bir çözüm sunulmaktadır. Varlıkları otomatik olarak yüklemek için, yapılandırma nesnesinin (`forRoot()` yöntemine iletilen) `autoLoadEntities` özelliğini `true` olarak ayarlayın, aşağıdaki gibi:

```typescript
@@filename(app.module)
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

Bu seçenek belirtildiğinde, `forFeature()` yöntemi aracılığıyla kaydedilen her varlık otomatik olarak yapılandırma nesnesinin `entities` dizisine eklenir.

> warning **Uyarı** `forFeature()` yöntemi aracılığıyla kaydedilmeyen, ancak varlık tarafından (bir ilişki aracılığıyla) sadece referans edilen varlıklar, `autoLoadEntities` ayarından dolayı dahil edilmez.

#### Varlık tanımını ayırma

Dekoratörler kullanarak bir varlığı ve sütunlarını model içinde doğrudan tanımlayabilirsiniz. Ancak bazı kişiler varlıkları ve sütunlarını ["varlık şemaları"](https://typeorm.io/#/separating-entity-definition) adlı ayrı dosyalarda tanımlamayı tercih eder.

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
      target: 'Photo', // PhotoSchema'nın adı
    },
  },
});
```

> warning error **Uyarı** `target` seçeneğini sağlarsanız, `name` seçeneği değeri hedef sınıfın adıyla aynı olmalıdır.
> `target` sağlamazsanız herhangi bir adı kullanabilirsiniz.

Nest, bir `EntitySchema` örneğini beklenen her yerde bir `Entity` kullanmanıza izin verir, örneğin:

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

#### TypeORM İşlemleri

Bir veritabanı işlemi, bir veritabanı yönetim sistemi içinde bir veritabanına karşı gerçekleştirilen ve diğer işlemlerden bağımsız olarak tutarlı ve güvenilir bir şekilde işlenen bir çalışma birimini simgeler. Bir işlem genellikle bir veritabanında [değişiklik](https://en.wikipedia.org/wiki/Database_transaction) anlamına gelir.

[TypeORM işlemlerini](https://typeorm.io/#/transactions) yönetmek için birçok farklı strateji bulunmaktadır. `QueryRunner` sınıfını kullanmanızı öneririz çünkü bu sınıf işlem üzerinde tam kontrol sağlar.

İlk olarak, `DataSource` nesnesini normal bir şekilde bir sınıfa enjekte etmemiz gerekir:

```typescript
@Injectable()
export class UsersService {
  constructor(private dataSource: DataSource) {}
}
```

> info **İpucu** `DataSource` sınıfı, `typeorm` paketinden alınmıştır.

Şimdi, bu nesneyi bir işlem oluşturmak için kullanabiliriz.

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
    // Hatalarımız olduğundan işlemleri geri alalım
    await queryRunner.rollbackTransaction();
  } finally {
    // Manuel olarak önceden oluşturulmuş bir queryRunner'ı serbest bırakmanız gerekir
    await queryRunner.release();
  }
}
```

> info **İpucu** `dataSource` yalnızca `QueryRunner`'ı oluşturmak için kullanılır. Ancak, bu sınıfı test etmek için tam `DataSource` nesnesini taklit etmek gerektiğinden, genellikle bu işlemleri sürdürmek için gereken yöntemlerin sınırlı bir kümesine sahip bir yardımcı fabrika sınıfı (örneğin, `QueryRunnerFactory`) kullanmanızı öneririz. Bu teknik, bu yöntemleri taklit etmeyi oldukça basitleştirir.

<app-banner-devtools></app-banner-devtools>

Alternatif olarak, `DataSource` nesnesinin `transaction` yöntemiyle geri çağırma yaklaşımını kullanabilirsiniz ([daha fazlasını okuyun](https://typeorm.io/#/transactions/creating-and-using-transactions)).

```typescript
async createMany(users: User[]) {
  await this.dataSource.transaction(async manager => {
    await manager.save(users[0]);
    await manager.save(users[1]);
  });
}
```

#### Aboneler

TypeORM [aboneleri](https://typeorm.io/#/listeners-and-subscribers/what-is-a-subscriber) kullanarak belirli varlık olaylarını dinleyebilirsiniz.

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

> error **Uyarı** Olay aboneleri [istek kapsamında olamazlar](/docs/fundamentals/injection-scopes).

Şimdi, `UserSubscriber` sınıfını `providers` dizisine ekleyin:

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

> info **İpucu** Varlık aboneleri hakkında daha fazla bilgi için [buraya](https://typeorm.io/#/listeners-and-subscribers/what-is-a-subscriber) bakın.

#### Migrations

[Migrations](https://typeorm.io/#/migrations), veritabanı şemasını uygulamanın veri modeli ile senkronize bir şekilde tutarak, mevcut verileri veritabanında korurken, şemayı aşamalı olarak güncellemenin bir yolunu sağlar. Migrations oluşturmak, çalıştırmak ve geri almak için TypeORM, ayrılmış bir [CLI](https://typeorm.io/#/migrations/creating-a-new-migration) sunar.

Migration sınıfları, Nest uygulaması kaynak kodundan ayrıdır. Yaşamları TypeORM CLI tarafından yönetilir. Bu nedenle, migrations ile bağımlılık enjeksiyonu ve diğer Nest özel özelliklerini kullanamazsınız. Migrations hakkında daha fazla bilgi edinmek için [TypeORM belgelerindeki](https://typeorm.io/#/migrations/creating-a-new-migration) rehberi takip edin.

#### Birden Fazla Veritabanı

Bazı projeler, birden fazla veritabanı bağlantısı gerektirebilir. Bu modülle bunu başarabilirsiniz. Birden fazla bağlantı ile çalışmak için önce bağlantıları oluşturun. Bu durumda, data source isimlendirmesi **zorunludur**.

Diyelim ki `Album` varlığını kendi veritabanında saklıyorsunuz.

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

> warning **Uyarı** Eğer bir data source için `name` belirlemezseniz, onun adı `default` olarak ayarlanır. Birden fazla bağlantıya veya aynı ada sahip bağlantılara sahip olmamalısınız, aksi takdirde bunlar üzerine yazılacaktır.

> warning **Uyarı** Eğer `TypeOrmModule.forRootAsync` kullanıyorsanız, data source adını `useFactory` dışında **ayrıca** belirtmelisiniz. Örneğin:
>
> ```typescript
> TypeOrmModule.forRootAsync({
>   name: 'albumsConnection',
>   useFactory: ...,
>   inject: ...,
> }),
> ```
>
> Daha fazla bilgi için [bu konuyu](https://github.com/nestjs/typeorm/issues/86) inceleyin.

Bu noktada, `User` ve `Album` varlıklarını kendi data source'ları ile kaydettiniz. Bu kurulum ile, `TypeOrmModule.forFeature()` yöntemi ve `@InjectRepository()` dekoratörü hangi data source'un kullanılacağını belirtmeniz gerekmektedir. Eğer herhangi bir data source adı vermezseniz, `default` data source kullanılır.

```typescript
@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    TypeOrmModule.forFeature([Album], 'albumsConnection'),
  ],
})
export class AppModule {}
```

Ayrıca, belirli bir data source için `DataSource` veya `EntityManager`'ı enjekte edebilirsiniz:

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

Ayrıca, herhangi bir `DataSource`'ı provider'lara enjekte etmek mümkündür:

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

#### Test

Uygulamayı birim test etme konusunda genellikle bir veritabanı bağlantısı yapmaktan kaçınmak istiyoruz, test süitlerimizi bağımsız tutmak ve yürütme süreçlerini mümkün olduğunca hızlı tutmak istiyoruz. Ancak sınıflarımız, data source (bağlantı) örneğinden çekilen repository'lere bağlı olabilir. Bu durumu nasıl ele alırız? Çözüm, mock repository'ler oluşturmaktır. Bunu başarmak için [özel sağlayıcıları](/docs/fundamentals/custom-providers) kurarız. Her kayıtlı repository, `<EntityName>Repository` adında bir token ile otomatik olarak temsil edilir, burada `EntityName`, varlık sınıfınızın adıdır.

`@nestjs/typeorm` paketi, belirli bir varlık temel alınarak hazırlanan bir belirteci döndüren `getRepositoryToken()` işlevini açıklar.

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

Şimdi `mockRepository`, `UsersRepository` olarak kullanılacaktır. Herhangi bir sınıf, `@InjectRepository()` dekoratörünü kullanarak `UsersRepository` istediğinde, Nest, kayıtlı `mockRepository` nesnesini kullanacaktır.

#### Asenkron Yapılandırma

Repository modül seçeneklerinizi statik olarak değil, asenkron olarak iletmek isteyebilirsiniz. Bu durumda, async yapılandırma ile başa çıkmak için `forRootAsync()` yöntemini kullanın. Bu yöntem, async yapılandırmayla başa çıkmanın birkaç yolunu sunar.

Bir yaklaşım, bir fabrika işlevi kullanmaktır:

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

Fabrikamız, diğer [asenkron sağlayıcılar](https://docs.nestjs.com/fundamentals/async-providers) gibi davranır (örneğin, `async` olabilir ve bağımlılıkları `inject` aracılığıyla enjekte edebilir).

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

Alternatif olarak, `useClass` sözdizimini kullanabilirsiniz:

```typescript
TypeOrmModule.forRootAsync({
  useClass: TypeOrmConfigService,
});
```

Yukarıdaki yapı, `TypeOrmModule` içinde `TypeOrmConfigService`'yi örnekleyecek ve `createTypeOrmOptions()`'ı çağırarak bir seçenek nesnesi sağlamak için kullanacaktır. Unutmayın ki bu, `TypeOrmConfigService`'nin `TypeOrmOptionsFactory` arabirimini uygulaması gerektiği anlamına gelir, aşağıda gösterildiği gibi:

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

`TypeOrmConfigService`'nin `TypeOrmModule` içinde oluşturulmasını önlemek ve başka bir modülden alınan bir sağlayıcıyı kullanmak için, `useExisting` sözdizimini kullanabilirsiniz.

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

Bu yapı, `useClass` ile aynı şekilde çalışır, ancak kritik bir farkla - `TypeOrmModule`, `ConfigService`'yi yeni bir tane örneklemek yerine alıntılanmış modüllere bakacak ve var olan bir `ConfigService`'yi yeniden kullanacaktır.

#### Custom DataSource Factory

`useFactory`, `useClass` veya `useExisting` kullanarak async yapılandırmayı kullanarak birleştirildiğinde, ayrıca `dataSourceFactory` işlevini belirleyebilir ve `TypeOrmModule`'un data source oluşturmasına izin vermek yerine kendi TypeORM data source'unuzu sağlama olanağına sahip olabilirsiniz.

`dataSourceFactory`, `useFactory`, `useClass` veya `useExisting` kullanarak yapılandırılan TypeORM `DataSourceOptions`'ı alır ve bir TypeORM `DataSource`'ı çözen bir `Promise` döndürür.

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  // Use useFactory, useClass, or useExisting
  // to configure the DataSourceOptions.
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
  // dataSource, yapılandırılan DataSourceOptions'ı alır
  // ve bir Promise<DataSource> çözer.
  dataSourceFactory: async (options) => {
    const dataSource = await new DataSource(options).initialize();
    return dataSource;
  },
});
```

> info **Hint** `DataSource` sınıfı, `typeorm` paketinden içe aktarılır.

#### Örnek

Çalışan bir örnek [burada bulunabilir](https://github.com/nestjs/nest/tree/master/sample/05-sql-typeorm).

<app-banner-enterprise></app-banner-enterprise>

### Sequelize Entegrasyonu

TypeORM kullanmanın alternatifleri arasında, `@nestjs/sequelize` paketi ile [Sequelize](https://sequelize.org/) ORM'yi kullanmaktır. Ayrıca, [sequelize-typescript](https://github.com/RobinBuschmann/sequelize-typescript) paketini kullanıyoruz, bu da varlıkları deklaratif olarak tanımlamak için bir dizi ek dekoratör sağlar.

Kullanmaya başlamak için önce gerekli bağımlılıkları yükleriz. Bu bölümde popüler [MySQL](https://www.mysql.com/) İlişkisel DBMS'yi kullanmayı göstereceğiz, ancak Sequelize, PostgreSQL, MySQL, Microsoft SQL Server, SQLite ve MariaDB gibi birçok ilişkisel veritabanı için destek sağlar. Bu bölümde işlem sırası, Sequelize tarafından desteklenen herhangi bir veritabanı için aynı olacaktır. Seçtiğiniz veritabanı için ilişkili istemci API kütüphanelerini yüklemeniz yeterlidir.

```bash
$ npm install --save @nestjs/sequelize sequelize sequelize-typescript mysql2
$ npm install --save-dev @types/sequelize
```

Kurulum işlemi tamamlandığında, `SequelizeModule`'u root `AppModule` içine alabiliriz.

```typescript
@@filename(app.module)
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

`forRoot()` yöntemi, Sequelize kurucusu tarafından açığa çıkarılan tüm yapılandırma özelliklerini destekler ([daha fazlasını oku](https://sequelize.org/v5/manual/getting-started.html#setting-up-a-connection)). Ayrıca, aşağıda açıklanan birkaç ek yapılandırma özelliği bulunmaktadır.

<table>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>Veritabanına bağlanma girişim sayısı (varsayılan: <code>10</code>)</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>Bağlantı tekrar deneme girişimleri arasındaki gecikme (ms) (varsayılan: <code>3000</code>)</td>
  </tr>
  <tr>
    <td><code>autoLoadModels</code></td>
    <td><code>true</code> ise, modeller otomatik olarak yüklenecektir (varsayılan: <code>false</code>)</td>
  </tr>
  <tr>
    <td><code>keepConnectionAlive</code></td>
    <td><code>true</code> ise, bağlantı uygulama kapanışında kapatılmaz (varsayılan: <code>false</code>)</td>
  </tr>
  <tr>
    <td><code>synchronize</code></td>
    <td><code>true</code> ise, otomatik olarak yüklenen modeller senkronize edilir (varsayılan: <code>true</code>)</td>
  </tr>
</table>

Bunu yaptıktan sonra, `Sequelize` nesnesi, projenin genelinde (herhangi bir modülü içe aktarmadan) enjekte edilebilir hale gelir, örneğin:

```typescript
@@filename(app.service)
import { Injectable } from '@nestjs/common';
import { Sequelize } from 'sequelize-typescript';

@Injectable()
export class AppService {
  constructor(private sequelize: Sequelize) {}
}
@@switch
import { Injectable } from '@nestjs/common';
import { Sequelize } from 'sequelize-typescript';

@Dependencies(Sequelize)
@Injectable()
export class AppService {
  constructor(sequelize) {
    this.sequelize = sequelize;
  }
}
```

#### Modeller

Sequelize, Active Record tasarım desenini uygular. Bu desenle, model sınıflarını doğrudan veritabanı ile etkileşimde bulunmak için kullanırsınız. Örneği sürdürmek için en az bir modelimize ihtiyacımız var. `User` modelini tanımlayalım.

```typescript
@@filename(user.model)
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

> info **Hint** Kullanılabilir dekoratörler hakkında daha fazla bilgi için [buraya](https://github.com/RobinBuschmann/sequelize-typescript#column) bakın.

`User` model dosyası `users` dizininde bulunuyor. Bu dizin, `UsersModule` ile ilgili tüm dosyaları içerir. Model dosyalarını nereye koyacağınıza karar verebilirsiniz, ancak genellikle onları ilgili modül dizinine, yani karşılık gelen modül dizinine oluşturmanızı öneririz.

`User` modelini kullanmaya başlamak için, onu modül `forRoot()` yöntemi seçeneklerindeki `models` dizisine ekleyerek Sequelize'a bildirmemiz gerekiyor:

```typescript
@@filename(app.module)
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

Ardından, `UsersModule`'e bakalım:

```typescript
@@filename(users.module)
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

Bu modül, şu anki kapsamda hangi modellerin kaydedildiğini belirlemek için `forFeature()` yöntemini kullanır. Bu yerine getirildiğinde, `@InjectModel()` dekoratörünü kullanarak `UsersService` içine `UserModel`'i enjekte edebiliriz:

```typescript
@@filename(users.service)
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
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { getModelToken } from '@nestjs/sequelize';
import { User } from './user.model';

@Injectable()
@Dependencies(getModelToken(User))
export class UsersService {
  constructor(usersRepository) {
    this.usersRepository = usersRepository;
  }

  async findAll() {
    return this.userModel.findAll();
  }

  findOne(id) {
    return this.userModel.findOne({
      where: {
        id,
      },
    });
  }

  async remove(id) {
    const user = await this.findOne(id);
    await user.destroy();
  }
}
```

> warning **Notice** `UsersModule`'u root `AppModule` içine eklemeyi unutmayın.

Eğer `SequelizeModule.forFeature`'i içe aktaran modül dışında repository'yi kullanmak istiyorsanız, bu tarafından oluşturulan sağlayıcıları yeniden ihraç etmeniz gerekir.
Bunu, bütün modülü ihraç ederek yapabilirsiniz, örneğin:

```typescript
@@filename(users.module)
import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';
import { User } from './user.entity';

@Module({
  imports: [SequelizeModule.forFeature([User])],
  exports: [SequelizeModule]
})
export class UsersModule {}
```

Şimdi `UserHttpModule` içinde `UsersModule`'i içe aktarırsak, ikinci modülün sağlayıcılarında `@InjectModel(User)`'i kullanabiliriz.

```typescript
@@filename(users-http.module)
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

#### İlişkiler

İlişkiler, iki veya daha fazla tablo arasında kurulan bağlantılardır. İlişkiler, genellikle her tablodan ortak alanlara dayanır ve genellikle birincil ve dış anahtarları içerir.

Üç tür ilişki bulunmaktadır:

<table>
  <tr>
    <td><code>One-to-one</code></td>
    <td>Temel tablodaki her satırın yalnızca bir ve yalnızca bir ilişkili satırı vardır.</td>
  </tr>
  <tr>
    <td><code>One-to-many / Many-to-one</code></td>
    <td>Temel tablodaki her satırın, dış tabloda bir veya daha fazla ilişkili satırı vardır.</td>
  </tr>
  <tr>
    <td><code>Many-to-many</code></td>
    <td>Temel tablodaki her satırın, dış tabloda birçok ilişkili satırı vardır ve dış tablodaki her kayıdın, temel tabloda birçok ilişkili satırı vardır.</td>
  </tr>
</table>

Modellerde ilişkileri tanımlamak için ilgili **dekoratörleri** kullanın. Örneğin, her `User`'ın birden çok fotoğrafı olabileceğini belirtmek için `@HasMany()` dekoratörünü kullanın.

```typescript
@@filename(user.model)
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

> info **Hint** Sequelize'deki ilişkiler hakkında daha fazla bilgi için [bu](https://github.com/RobinBuschmann/sequelize-typescript#model-association) bölümü inceleyebilirsiniz.

#### Modelleri Otomatik Yükleme

Modelleri manuel olarak bağlantı seçeneklerinin `models` dizisine eklemek zaman alıcı olabilir. Ayrıca, modelleri kök modülden referans vermek, uygulama alanı sınırlarını bozar ve uygulamanın diğer kısımlarına uygulama ayrıntılarını sızdırır. Bu sorunu çözmek için, aşağıda gösterildiği gibi yapılandırma nesnesine (`forRoot()` yöntemine iletilen) `autoLoadModels` ve `synchronize` özelliklerini `true` olarak ayarlayarak modelleri otomatik olarak yükleyin:

```typescript
@@filename(app.module)
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
export class AppModule {}
```

Bu seçenek belirlendiğinde, `forFeature()` yöntemi aracılığıyla kaydedilen her model otomatik olarak yapılandırma nesnesinin `models` dizisine eklenir.

> warning **Uyarı** `forFeature()` yöntemi aracılığıyla kaydedilmeyen, ancak yalnızca modelden (bir ilişki aracılığıyla) referans alınan modeller dahil edilmez.

#### Sequelize İşlemleri

Veritabanı işlemi, bir veritabanı yönetim sistemi içinde bir veritabanına karşı gerçekleştirilen iş birimini temsil eder ve diğer işlemlerden bağımsız, tutarlı ve güvenilir bir şekilde işlenir. Bir işlem genellikle bir veritabanındaki herhangi bir değişikliği temsil eder ([daha fazla bilgi](https://en.wikipedia.org/wiki/Database_transaction)).

[Sequelize işlemleri](https://sequelize.org/v5/manual/transactions.html) ile başa çıkmak için birkaç farklı strateji vardır. İşte yönetilen bir işlem (otomatik geri çağrı) örneğinin bir uygulaması:

İlk olarak, bu sınıfa `Sequelize` nesnesini normal şekilde enjekte etmemiz gerekiyor:

```typescript
@Injectable()
export class UsersService {
  constructor(private sequelize: Sequelize) {}
}
```

> info **İpucu** `Sequelize` sınıfı, `sequelize-typescript` paketinden içe aktarılır.

Şimdi, bu nesneyi bir işlem oluşturmak için kullanabiliriz.

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
    // İşlem geri alındı
    // err, işlem geri arama zincirini reddeden her şeydir ve işlem geri çağrı fonksiyonuna geri döndürülür
  }
}
```

> info **İpucu** `Sequelize` örneği, yalnızca işlemi başlatmak için kullanılır. Ancak bu sınıfı test etmek için tüm `Sequelize` nesnesini (birçok yöntemi açığa çıkarır) taklit etmek gerektiğinden, bu yöntemi kullanmanızı öneririz. Bu teknik, bu yöntemleri sürdürmek için gereken sınırlı bir dizi yöntemi tanımlayan bir ara yardımcı sınıf (örneğin, `TransactionRunner`) kullanmanızı sağlar ve bu yöntemleri taklit etmeyi oldukça basitleştirir.

#### Göçler (Migrations)

[Göçler](https://sequelize.org/v5/manual/migrations.html), veritabanı şemasını uygulamanın veri modeli ile senkronize tutarak var olan veriyi koruma amacını taşır. Göçleri oluşturmak, çalıştırmak ve geri almak için Sequelize özel bir [CLI](https://sequelize.org/v5/manual/migrations.html#the-cli) sağlar.

Göç sınıfları, Nest uygulama kaynak kodundan ayrıdır. Bunların yaşam döngüsü Sequelize CLI tarafından yönetilir. Bu nedenle, göçlerle bağımlılık enjeksiyonu ve diğer Nest özel özelliklerini kullanamazsınız. Göçlerle ilgili daha fazla bilgi için [Sequelize belgelerindeki](https://sequelize.org/v5/manual/migrations.html#the-cli) rehberi takip edin.

<app-banner-courses></app-banner-courses>

#### Birden Fazla Veritabanı

Bazı projeler birden fazla veritabanı bağlantısı gerektirebilir. Bu durumda, bağlantı adlandırma **zorunludur**.

Varsayalım ki kendi veritabanında saklanan bir `Album` varlığınız var.

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

> uyarı **Dikkat** Bir bağlantı için `name`'i belirtmezseniz, adı `default` olarak ayarlanır. Lütfen aynı ada sahip veya ada belirtmeksizin birden çok bağlantınızın olmamasına dikkat edin, aksi takdirde bunlar geçersiz kılınacaktır.

Bu aşamada, `User` ve `Album` modelleri kendi bağlantılarıyla kaydedilmiştir. Bu kurulumda, `SequelizeModule.forFeature()` yöntemine ve `@InjectModel()` dekoratörüne hangi bağlantının kullanılması gerektiğini belirtmelisiniz. Eğer herhangi bir bağlantı adı geçmezseniz, varsayılan bağlantı kullanılır.

```typescript
@Module({
  imports: [
    SequelizeModule.forFeature([User]),
    SequelizeModule.forFeature([Album], 'albumsConnection'),
  ],
})
export class AppModule {}
```

Ayrıca, belirli bir bağlantı için `Sequelize` örneğini enjekte edebilirsiniz:

```typescript
@Injectable()
export class AlbumsService {
  constructor(
    @InjectConnection('albumsConnection')
    private sequelize: Sequelize,
  ) {}
}
```

Ayrıca, sağlayıcılara herhangi bir `Sequelize` örneğini enjekte etmek de mümkündür:

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

#### Test

Bir uygulamayı birim test etme konusunda genellikle veritabanı bağlantısı kurmaktan kaçınmak, test paketlerimizi bağımsız tutmak ve yürütme süreçlerini mümkün olduğunca hızlı tutmak isteriz. Ancak sınıflarımız, bağlantı örneğinden çekilen modellere bağlı olabilir. Bu durumu nasıl ele alabiliriz? Çözüm, sahte modeller oluşturmaktır. Bunu başarmak için [özel sağlayıcılar](/docs/fundamentals/custom-providers) kurarız. Her kayıtlı model, otomatik olarak `<ModelAdı>Model` belirteci ile temsil edilir, burada `ModelAdı` model sınıfınızın adıdır.

`@nestjs/sequelize` paketi, belirli bir modele dayalı olarak hazırlanmış bir belirteç döndüren `getModelToken()` işlevini sunar.

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

Şimdi bir yedek olan `mockModel`, `UserModel` olarak kullanılacaktır. Herhangi bir sınıf, bir `@InjectModel()` dekoratörü kullanarak `UserModel` istediğinde, Nest, kayıtlı `mockModel` nesnesini kullanacaktır.

#### Async (Asenkron) Yapılandırma

`SequelizeModule` seçeneklerinizi statik olarak değil, asenkron olarak iletmek isteyebilirsiniz. Bu durumda, async yapılandırma ile başa çıkmak için çeşitli yöntemler sunan `forRootAsync()` yöntemini kullanın.

Bir yaklaşım, bir fabrika işlevi kullanmaktır:

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

Fabrikamız, diğer [asenkron sağlayıcılar](https://docs.nestjs.com/fundamentals/async-providers) gibi davranır (örneğin, `async` olabilir ve bağımlılıkları `inject` aracılığıyla enjekte edebilir).

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

Alternatif olarak, `useClass` sözdizimini kullanabilirsiniz:

```typescript
SequelizeModule.forRootAsync({
  useClass: SequelizeConfigService,
});
```

Yukarıdaki yapım, `SequelizeModule` içinde `SequelizeConfigService`'yi örnekler ve `createSequelizeOptions()`'ı çağırarak bir seçenek nesnesi sağlamak için kullanır. Bu, `SequelizeConfigService`'nin `SequelizeOptionsFactory` arabirimini uygulaması gerektiği anlamına gelir, aşağıda gösterildiği gibi:

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

`SequelizeConfigService`'nin `SequelizeModule` içinde oluşturulmasını önlemek ve farklı bir modülden alınan bir sağlayıcıyı kullanmak için, `useExisting` sözdizimini kullanabilirsiniz.

```typescript
SequelizeModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

Bu yapı, `useClass` ile aynı şekilde çalışır, ancak kritik bir farkla - `SequelizeModule`, yeni bir tane oluşturmak yerine, mevcut bir `ConfigService`'yi yeniden kullanmak için içe aktarılan modülleri arayacaktır.

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/07-sequelize) bulunabilir.