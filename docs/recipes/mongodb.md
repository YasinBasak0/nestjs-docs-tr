### MongoDB (Mongoose)

> **Uyarı** Bu makalede, özel bileşenler kullanarak sıfırdan **Mongoose** paketi üzerinde bir `DatabaseModule` oluşturmayı öğreneceksiniz. Sonuç olarak, bu çözüm, hazır kullanıma ve hemen kullanılmaya uygun olan özel `@nestjs/mongoose` paketini kullanarak atlayabileceğiniz birçok gereksiz detay içermektedir. Daha fazla bilgi için [buraya](/docs/techniques/mongodb) bakın.

[Mongoose](https://mongoosejs.com), [MongoDB](https://www.mongodb.org/) nesne modelleme aracının en popüleridir.

#### Başlarken

Bu kütüphane ile maceraya başlamak için tüm gerekli bağımlılıkları kurmamız gerekiyor:

```typescript
$ npm install --save mongoose
```

İlk yapmamız gereken adım, `connect()` fonksiyonu kullanarak veritabanı bağlantısını kurmaktır. `connect()` fonksiyonu bir `Promise` döndürdüğü için [asyonik sağlayıcı](/docs/fundamentals/async-components) oluşturmamız gerekiyor.

```typescript
@@filename(database.providers)
import * as mongoose from 'mongoose';

export const databaseProviders = [
  {
    provide: 'DATABASE_CONNECTION',
    useFactory: (): Promise<typeof mongoose> =>
      mongoose.connect('mongodb://localhost/nest'),
  },
];
@@switch
import * as mongoose from 'mongoose';

export const databaseProviders = [
  {
    provide: 'DATABASE_CONNECTION',
    useFactory: () => mongoose.connect('mongodb://localhost/nest'),
  },
];
```

> info **İpucu** En iyi uygulamaları takip ederek, özel sağlayıcıyı `*.providers.ts` uzantısına sahip ayrı bir dosyada bildirdik.

Ardından, bu sağlayıcıları **erişilebilir** yapmak için bu sağlayıcıları ihraç etmemiz gerekiyor.

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

Şimdi `@Inject()` dekoratörünü kullanarak `Connection` nesnesini enjekte edebiliriz. `Connection` async sağlayıcısına bağımlı olan her sınıf, bir `Promise` çözülene kadar bekleyecektir.

#### Model enjeksiyonu

Mongoose ile her şey [Schema](https://mongoosejs.com/docs/guide.html) (Şema) üzerinden türetilir. Şimdi `CatSchema`'yı tanımlayalım:

```typescript
@@filename(schemas/cat.schema)
import * as mongoose from 'mongoose';

export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,
  breed: String,
});
```

`CatSchema`, `cats` dizinine aittir. Bu dizin, `CatsModule`'yi temsil eder.

Şimdi bir **Model** sağlayıcısı oluşturma zamanı:

```typescript
@@filename(cats.providers)
import { Connection } from 'mongoose';
import { CatSchema } from './schemas/cat.schema';

export const catsProviders = [
  {
    provide: 'CAT_MODEL',
    useFactory: (connection: Connection) => connection.model('Cat', CatSchema),
    inject: ['DATABASE_CONNECTION'],
  },
];
@@switch
import { CatSchema } from './schemas/cat.schema';

export const catsProviders = [
  {
    provide: 'CAT_MODEL',
    useFactory: (connection) => connection.model('Cat', CatSchema),
    inject: ['DATABASE_CONNECTION'],
  },
];
```

> warning **Uyarı** Gerçek dünya uygulamalarında **sihirli dizelerden** kaçınmalısınız. Hem `CAT_MODEL` hem de `DATABASE_CONNECTION` ayrı `constants.ts` dosyasında tutulmalıdır.

Şimdi `CAT_MODEL`'i `CatsService`'ye `@Inject()` dekoratörü kullanarak enjekte edebiliriz:

```typescript
@@filename(cats.service)
import { Model } from 'mongoose';
import { Injectable, Inject } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(
    @Inject('CAT_MODEL')
    private catModel: Model<Cat>,
  ) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';

@Injectable()
@Dependencies('CAT_MODEL')
export class CatsService {
  constructor(catModel) {
    this.catModel = catModel;
  }

  async create(createCatDto) {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll() {
    return this.catModel.find().exec();
  }
}
```

Yukarıdaki örnekte `Cat` arayüzünü kullandık. Bu arayüz, mongoose paketinden `Document`'i genişletmektedir:

```typescript
import { Document } from 'mongoose';

export interface Cat extends Document {
  readonly name: string;
  readonly age: number;
  readonly breed: string;
}
```

Veritabanı bağlantısı **asyonik** (asenkron) olduğu için, Nest bu süreci son kullanıcı için tamamen görünmez hale getirir. `CatModel` sınıfı db bağlantısını bekler ve `CatsService`, model kullanıma hazır olduğunda gecikir. Tüm uygulama, her sınıf örneklendiğinde başlayabilir.

İşte nihai `CatsModule`:

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

> info **İpucu** `CatsModule`'i root `AppModule` içine import etmeyi unutmayın.

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/14-mongoose-base) bulunabilir.