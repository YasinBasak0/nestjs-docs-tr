### SQL (Sequelize)

##### Bu bölüm sadece TypeScript için geçerlidir

> **Uyarı** Bu makalede, **Sequelize** paketini temel alarak sıfırdan `DatabaseModule` oluşturmayı öğreneceksiniz. Bu teknik, kullanımı daha kolay olan ve önceden yapılandırılmış olan `@nestjs/sequelize` paketini kullanarak önemli ölçüde tasarruf edebileceğiniz bir miktarda fazlalık içerir. Daha fazla bilgi için [buraya](/docs/techniques/database#sequelize-integration) bakın.

[Sequelize](https://github.com/sequelize/sequelize), vanilya JavaScript'te yazılmış popüler bir Nesne İlişkisel Eşlemci (ORM) dir. Ancak, bir dizi dekoratör ve diğer ekstraları sağlayan bir TypeScript sarmalayıcısı olan [sequelize-typescript](https://github.com/RobinBuschmann/sequelize-typescript) de bulunmaktadır.

#### Başlangıç

Bu kütüphaneyle maceraya başlamak için aşağıdaki bağımlılıkları kurmamız gerekiyor:

```bash
$ npm install --save sequelize sequelize-typescript mysql2
$ npm install --save-dev @types/sequelize
```

İlk adım olarak, bir seçenek nesnesini içeren bir yapılandırma dosyasına gönderilen bir **Sequelize** örneği oluşturmalıyız. Ayrıca tüm modelleri (alternatif olarak `modelPaths` özelliğini kullanabilirsiniz) eklemeli ve veritabanı tablolarımızı `sync()` metodu ile senkronize etmeliyiz.

```typescript
@@filename(database.providers)
import { Sequelize } from 'sequelize-typescript';
import { Cat } from '../cats/cat.entity';

export const databaseProviders = [
  {
    provide: 'SEQUELIZE',
    useFactory: async () => {
      const sequelize = new Sequelize({
        dialect: 'mysql',
        host: 'localhost',
        port: 3306,
        username: 'root',
        password: 'password',
        database: 'nest',
      });
      sequelize.addModels([Cat]);
      await sequelize.sync();
      return sequelize;
    },
  },
];
```

> info **İpucu** En iyi uygulamaları takip ederek, özel sağlayıcıyı, `*.providers.ts` eki olan ayrı bir dosyada bildirdik.

Daha sonra, bu sağlayıcıları **erişilebilir** hale getirmek için bunları ihraç etmemiz gerekiyor.

```typescript
import { Module } from '@nestjs/common';
import { databaseProviders } from './database.providers';

@Module({
  providers: [...databaseProviders],
  exports: [...databaseProviders],
})
export class DatabaseModule {}
```

Şimdi, `@Inject()` dekoratörünü kullanarak `Sequelize` nesnesini enjekte edebiliriz. `Sequelize` ile bağımlı olacak her sınıf, bir `Promise` çözülene kadar bekleyecektir.

#### Model Enjeksiyonu

[Sequelize](https://github.com/sequelize/sequelize)'de **Model**, veritabanındaki bir tabloyu tanımlar. Bu sınıfın örnekleri bir veritabanı satırını temsil eder. İlk olarak, en az bir varlık oluşturmamız gerekiyor:

```typescript
@@filename(cat.entity)
import { Table, Column, Model } from 'sequelize-typescript';

@Table
export class Cat extends Model {
  @Column
  name: string;

  @Column
  age: number;

  @Column
  breed: string;
}
```

`Cat` varlığı, `cats` dizinine aittir. Bu dizin, `CatsModule`'yi temsil eder. Şimdi bir **Repository** sağlayıcısı oluşturmanın zamanı geldi:

```typescript
@@filename(cats.providers)
import { Cat } from './cat.entity';

export const catsProviders = [
  {
    provide: 'CATS_REPOSITORY',
    useValue: Cat,
  },
];
```

> warning **Uyarı** Gerçek dünya uygulamalarında **sihirli dizelerden** kaçınmalısınız. Hem `CATS_REPOSITORY` hem de `SEQUELIZE` ayrı `constants.ts` dosyasında tutulmalıdır.

Sequelize'de veriyi manipüle etmek için statik yöntemleri kullanırız ve bu nedenle burada bir **takma ad** oluşturduk.

Şimdi, `@Inject()` dekoratörünü kullanarak `CatsService`'e `CATS_REPOSITORY`'i enjekte edebiliriz:

```typescript
@@filename(cats.service)
import { Injectable, Inject } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { Cat } from './cat.entity';

@Injectable()
export class CatsService {
  constructor(
    @Inject('CATS_REPOSITORY')
    private catsRepository: typeof Cat
  ) {}

  async findAll(): Promise<Cat[]> {
    return this.catsRepository.findAll<Cat>();
  }
}
```

Veritabanı bağlantısı **asenkron** olduğundan, Nest bu süreci kullanıcının tamamen görmesini engeller. `CATS_REPOSITORY` sağlayıcısı, db bağlantısı beklerken bekler ve `CatsService`, depo kullanıma hazır olduğunda gecikir. Tüm uygulama, her sınıf örneklendiğinde başlayabilir.

İşte nihai bir `CatsModule`:

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { catsProviders } from './cats.providers';
import { DatabaseModule } from '../database/database.module';

@Module({
  imports: [DatabaseModule],
  controllers: [CatsController],
  providers: [
    CatsService,
    ...catsProviders,
  ],
})
export class CatsModule {}
```

> info **İpucu** `CatsModule`'u kök `AppModule`'e unutmayın eklemeyi.