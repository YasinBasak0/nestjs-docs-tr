### SQL (TypeORM)

##### Bu bölüm sadece TypeScript için geçerlidir

> **Uyarı** Bu makalede, **TypeORM** paketini temel alarak sıfırdan `DatabaseModule` oluşturmayı öğreneceksiniz. Bu çözüm, kullanıma hazır ve kutudan çıkmış `@nestjs/typeorm` paketini kullanarak atlayabileceğiniz bir miktar fazlalık içerir. Daha fazla bilgi için [buraya](/docs/techniques/sql) bakın.

[TypeORM](https://github.com/typeorm/typeorm), kesinlikle node.js dünyasında bulunan en olgun Nesne İlişkisel Eşleme (ORM) aracıdır. TypeScript ile yazıldığından, Nest çerçevesi ile oldukça iyi çalışır.

#### Başlangıç

Bu kütüphaneyi kullanmaya başlamak için tüm gerekli bağımlılıkları kurmamız gerekiyor:

```bash
$ npm install --save typeorm mysql2
```

İlk yapmamız gereken şey, `typeorm` paketinden alınan `new DataSource().initialize()` sınıfını kullanarak veritabanımızla bağlantı kurmaktır. `initialize()` işlevi bir `Promise` döndürdüğü için [async sağlayıcı](/docs/fundamentals/async-components) oluşturmamız gerekiyor.

```typescript
@@filename(database.providers)
import { DataSource } from 'typeorm';

export const databaseProviders = [
  {
    provide: 'DATA_SOURCE',
    useFactory: async () => {
      const dataSource = new DataSource({
        type: 'mysql',
        host: 'localhost',
        port: 3306,
        username: 'root',
        password: 'root',
        database: 'test',
        entities: [
            __dirname + '/../**/*.entity{.ts,.js}',
        ],
        synchronize: true,
      });

      return dataSource.initialize();
    },
  },
];
```

> warning **Uyarı** `synchronize: true` ayarını üretimde kullanmamalısınız - aksi takdirde üretim verilerini kaybedebilirsiniz.

> info **İpucu** En iyi uygulamaları takip ederek, özel sağlayıcıyı, `*.providers.ts` eki olan ayrı bir dosyada bildirdik.

Daha sonra, bu sağlayıcıları **erişilebilir** hale getirmek için bunları ihraç etmemiz gerekiyor.

```typescript
@@filename(database.module)
import { Module } from '@nestjs/common';
import { databaseProviders } from './database.providers';

@Module({
  providers: [...databaseProviders],
  exports: [...databaseProviders],
})
export class DatabaseModule {}
```

Şimdi, `@Inject()` dekoratörünü kullanarak `DATA_SOURCE` nesnesini enjekte edebiliriz. `DATA_SOURCE` asenkron sağlayıcısına bağımlı olacak her sınıf, bir `Promise` çözülene kadar bekleyecektir.

#### Repository deseni

[TypeORM](https://github.com/typeorm/typeorm), bu nedenle her varlık kendi Repository'sine sahiptir. Bu depolar, veritabanından alınabilir.

Ancak önce, en az bir varlığa ihtiyacımız var. Resmi belgelerden `Photo` varlığını yeniden kullanacağız.

```typescript
@@filename(photo.entity)
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class Photo {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ length: 500 })
  name: string;

  @Column('text')
  description: string;

  @Column()
  filename: string;

  @Column('int')
  views: number;

  @Column()
  isPublished: boolean;
}
```

`Photo` varlığı, `photo` dizinine aittir. Bu dizin, `PhotoModule`'yi temsil eder. Şimdi, bir **Repository** sağlayıcısı oluşturalım:

```typescript
@@filename(photo.providers)
import { DataSource } from 'typeorm';
import { Photo } from './photo.entity';

export const photoProviders = [
  {
    provide: 'PHOTO_REPOSITORY',
    useFactory: (dataSource: DataSource) => dataSource.getRepository(Photo),
    inject: ['DATA_SOURCE'],
  },
];
```

> warning **Uyarı** Gerçek dünya uygulamalarında **sihirli dizelerden** kaçınmalısınız. Hem `PHOTO_REPOSITORY` hem de `DATA_SOURCE` ayrı `constants.ts` dosyasında tutulmalıdır.

Şimdi, `Repository<Photo>`'yi `PhotoService`'e `@Inject()` dekoratörünü kullanarak enjekte edebiliriz:

```typescript
@@filename(photo.service)
import { Injectable, Inject } from '@nestjs/common';
import { Repository } from 'typeorm';
import { Photo } from './photo.entity';

@Injectable()
export class PhotoService {
  constructor(
    @Inject('PHOTO_REPOSITORY')
    private photoRepository: Repository<Photo>,
  ) {}

  async findAll(): Promise<Photo[]> {
    return this.photoRepository.find();
  }
}
```

Veritabanı bağlantısı **asenkron** olduğundan, Nest bu süreci kullanıcının tamamen görmesini engeller. `PhotoRepository` veritabanı bağlantısını beklerken, `PhotoService` deposunun kullanıma hazır olana kadar gecikir. Tüm uygulama, her sınıf örneklendiğinde başlayabilir.

İşte nihai bir `PhotoModule`:

```typescript
@@filename(photo.module)
import { Module } from '@nestjs/common';
import { DatabaseModule } from '../database/database.module';
import { photoProviders } from './photo.providers';
import { PhotoService } from './photo.service';

@Module({
  imports: [DatabaseModule],
  providers: [
    ...photoProviders,
    PhotoService,
  ],
})
export class PhotoModule {}
```

> info **İpucu** `PhotoModule`'u kök `AppModule`'e unutmayın eklemeyi.