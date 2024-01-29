### Mongo

Nest, [MongoDB](https://www.mongodb.com/) veritabanıyla entegre olmak için iki yöntemi destekler. Ya yerleşik [TypeORM](https://github.com/typeorm/typeorm) modülünü kullanabilirsiniz, bu modülde MongoDB için bir bağlayıcı bulunmaktadır ve ayrıntıları [burada](/docs/techniques/database) bulabilirsiniz, ya da MongoDB'nin en popüler nesne modelleme aracı olan [Mongoose](https://mongoosejs.com)'u kullanabilirsiniz. Bu bölümde, özel `@nestjs/mongoose` paketini kullanarak Mongoose'u açıklayacağız.

İlk olarak, [gerekli bağımlılıkları](https://github.com/Automattic/mongoose) yükleyerek başlayın:

```bash
$ npm i @nestjs/mongoose mongoose
```

Kurulum işlemi tamamlandığında, `MongooseModule`'u kök `AppModule`'a içe aktarabiliriz.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [MongooseModule.forRoot('mongodb://localhost/nest')],
})
export class AppModule {}
```

`forRoot()` yöntemi, Mongoose paketinin [burada](https://mongoosejs.com/docs/connections.html) açıklanan `mongoose.connect()` ile aynı yapılandırma nesnesini kabul eder.

#### Model Enjeksiyonu

Mongoose ile, her şey [Schema](http://mongoosejs.com/docs/guide.html)'dan türemiştir. Her şema, bir MongoDB koleksiyonuyla eşlenir ve bu koleksiyon içindeki belgelerin şeklini tanımlar. Şemalar, [Modelleri](https://mongoosejs.com/docs/models.html) tanımlamak için kullanılır. Modeller, temel MongoDB veritabanından belge oluşturma ve okuma sorumluluğuna sahiptir.

Şemalar, NestJS dekoratörleri kullanılarak veya doğrudan Mongoose ile manuel olarak oluşturulabilir. Dekoratörleri kullanarak şemaları oluşturmak, genelde fazla kod tekrarını azaltır ve genel kod okunabilirliğini artırır.

Hadi `CatSchema`'yı tanımlayalım:

```typescript
@@filename(schemas/cat.schema)
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

> info **İpucu:** Ayrıca `nestjs/mongoose` tarafından sağlanan `DefinitionsFactory` sınıfını kullanarak ham bir şema tanımı oluşturabilirsiniz. Bu, sağladığınız metadata temel alınarak oluşturulan şema tanımını manuel olarak değiştirmenize olanak tanır. Bu, dekoratörlerle her şeyin temsil edilmesinin zor olduğu bazı özel durumlar için kullanışlıdır.

`@Schema()` dekoratörü, bir sınıfı bir şema tanımı olarak işaretler. `Cat` sınıfımızı aynı isimde bir MongoDB koleksiyonuna, ancak sonunda ekstra bir "s" eklenmiş şekilde eşler - yani nihai mongo koleksiyon adı `cats` olacaktır. Bu dekoratör, tek bir isteğe bağlı argümanı kabul eder; bu, genellikle `mongoose.Schema` sınıfının yapılandırıcı metodunun ikinci argümanı olarak geçirdiğiniz nesne olarak düşünülebilir (örneğin, `new mongoose.Schema(_, options)`)). Kullanılabilir şema seçenekleri hakkında daha fazla bilgi için, [buraya](https://mongoosejs.com/docs/guide.html#options) bakın.

`@Prop()` dekoratörü, belgedeki bir özelliği tanımlar. Örneğin, yukarıdaki şema tanımında üç özellik tanımladık: `name`, `age` ve `breed`. Bu özelliklerin [şema türleri](https://mongoosejs.com/docs/schematypes.html), TypeScript metadata (ve reflection) yetenekleri sayesinde otomatik olarak çıkarılır. Ancak, türlerin açıkça yansıtılamadığı (örneğin, diziler veya iç içe geçmiş nesne yapıları gibi) daha karmaşık senaryolarda, türleri açıkça belirtmek gerekmektedir:

```typescript
@Prop([String])
tags: string[];
```

Alternatif olarak, `@Prop()` dekoratörü bir seçenek nesnesi argümanını ([daha fazla bilgi](https://mongoosejs.com/docs/schematypes.html#schematype-options) içeren) kabul eder. Bu sayede bir özelliğin zorunlu olup olmadığını belirleyebilir, varsayılan bir değer belirleyebilir veya onu değişmez olarak işaretleyebilirsiniz. Örneğin:

```typescript
@Prop({ required: true })
name: string;
```

Eğer başka bir modele ilişki belirtmek istiyorsanız, daha sonra popüle etme işlemi için, `@Prop()` dekoratörünü de kullanabilirsiniz. Örneğin, `Cat`'in `owners` koleksiyonunda saklanan `Owner`'ı varsa, özellik tipi ve referans belirtilmelidir. Örneğin:

```typescript
import * as mongoose from 'mongoose';
import { Owner } from '../owners/schemas/owner.schema';

// sınıf tanımının içinde
@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })
owner: Owner;
```

Birden çok sahibiniz varsa, özelliğinizin yapılandırması aşağıdaki gibi görünmelidir:

```typescript
@Prop({ type: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' }] })
owner: Owner[];
```

Son olarak, **ham** şema tanımı da dekoratöre geçilebilir. Bu, örneğin, bir özelliğin bir sınıf olarak tanımlanmamış iç içe geçmiş bir nesneyi temsil ettiği durumlar için kullanışlıdır. Bu durumda, `@nestjs/mongoose` paketindeki `raw()` fonksiyonunu kullanın:

```typescript
@Prop(raw({
  firstName: { type: String },
  lastName: { type: String }
}))
details: Record<string, any>;
```

Alternatif olarak, **dekoratörler kullanmak istemiyorsanız**, şemayı manuel olarak tanımlayabilirsiniz. Örneğin:

```typescript
export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,


  breed: String,
});
```

`cat.schema` dosyası, `cats` dizini içinde bir klasmanda bulunur ve aynı zamanda `CatsModule`'u tanımladığımız yerdir. Şemaları istediğiniz yere saklayabilirsiniz, ancak bunları ilgili **alan** nesneleriyle ilişkilendikleri yerde, uygun modül dizininde saklamanızı öneririz.

Şimdi `CatsModule`'a bir göz atalım:

```typescript
@@filename(cats.module)
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

`MongooseModule`, modülü yapılandırmak için `forFeature()` yöntemini sağlar, bu da hangi modellerin mevcut kapsamda kaydedileceğini tanımlar. Modelleri başka bir modülde de kullanmak istiyorsanız, `CatsModule`'u `exports` bölümüne ekleyin ve diğer modülde `CatsModule`'u içe aktarın.

Şemayı kaydettikten sonra, `CatsService`'e bir `Cat` modelini `@InjectModel()` dekoratörü kullanarak enjekte edebilirsiniz:

```typescript
@@filename(cats.service)
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
@@switch
import { Model } from 'mongoose';
import { Injectable, Dependencies } from '@nestjs/common';
import { getModelToken } from '@nestjs/mongoose';
import { Cat } from './schemas/cat.schema';

@Injectable()
@Dependencies(getModelToken(Cat.name))
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

#### Bağlantı

Bazı durumlarda, yerel [Mongoose Connection](https://mongoosejs.com/docs/api.html#Connection) nesnesine erişim sağlamanız gerekebilir. Örneğin, bağlantı nesnesi üzerinde yerel API çağrıları yapmak isteyebilirsiniz. Mongoose Connection'ı `@InjectConnection()` dekoratörünü kullanarak şu şekilde enjekte edebilirsiniz:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private connection: Connection) {}
}
```

#### Birden çok veritabanı

Bazı projelerde birden çok veritabanı bağlantısına ihtiyaç duyulabilir. Bu modülle bunu başarabilirsiniz. Birden çok bağlantı ile çalışmak için önce bağlantıları oluşturun. Bu durumda, bağlantı adlandırma **zorunlu** hale gelir.

```typescript
@@filename(app.module)
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

> warning **Dikkat** Lütfen aynı isme sahip veya adı olmayan birden çok bağlantınız olmamalıdır, aksi takdirde bunlar üzerine yazılacaktır.

Bu yapılandırmayla, `MongooseModule.forFeature()` fonksiyonuna hangi bağlantının kullanılması gerektiğini belirtmelisiniz.

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }], 'cats'),
  ],
})
export class CatsModule {}
```

Ayrıca, belirli bir bağlantı için `Connection`'ı enjekte edebilirsiniz:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection('cats') private connection: Connection) {}
}
```

Bir belirli `Connection`'ı özel bir sağlayıcıya (örneğin, fabrika sağlayıcı) enjekte etmek için, bağlantının adını bir argüman olarak geçirerek `getConnectionToken()` fonksiyonunu kullanın.

```typescript
{
  provide: CatsService,
  useFactory: (catsConnection: Connection) => {
    return new CatsService(catsConnection);
  },
  inject: [getConnectionToken('cats')],
}
```

Yalnızca adlandırılmış bir veritabanından modeli enjekte etmek istiyorsanız, `@InjectModel()` dekoratörüne ikinci bir parametre olarak bağlantı adını kullanabilirsiniz.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name, 'cats') private catModel: Model<Cat>) {}
}
@@switch
@Injectable()
@Dependencies(getModelToken(Cat.name, 'cats'))
export class CatsService {
  constructor(catModel) {
    this.catModel = catModel;
  }
}
```

#### Hooks (Middleware)

Middleware (aynı zamanda ön ve son kancalar olarak da adlandırılır), asenkron işlevlerin yürütülmesi sırasında kontrolü geçen işlevlerdir. Middleware, şema düzeyinde belirtilir ve eklentiler yazmak için kullanışlıdır ([kaynak](https://mongoosejs.com/docs/middleware.html)). Mongoose'da bir modelin derlendikten sonra `pre()` veya `post()` çağırmak çalışmaz. Bir kancayı **model kaydından önce** kaydetmek için, `MongooseModule`'un `forFeatureAsync()` yöntemini ve bir fabrika sağlayıcıyı (`useFactory` örneğin) kullanın. Bu teknikle bir şema nesnesine erişebilir ve ardından bu şema üzerinde bir kancayı kaydetmek için `pre()` veya `post()` yöntemini kullanabilirsiniz. Aşağıdaki örneğe bakın:

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatsSchema;
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

Diğer [fabrika sağlayıcıları](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) gibi, fabrika işlevimiz `async` olabilir ve `inject` üzerinden bağımlılıkları enjekte edebilir.

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        imports: [ConfigModule],
        useFactory: async (configService: ConfigService) => {
          const schema = CatsSchema;
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

#### Eklentiler (Plugins)

Bir belirli şema için [eklenti](https://mongoosejs.com/docs/plugins.html) kaydetmek için `forFeatureAsync()` yöntemini kullanın.

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatsSchema;
          schema.plugin(require('mongoose-autopopulate'));
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

Tüm şemalar için bir eklenti kaydetmek için, `Connection` nesnesinin `.plugin()` yöntemini çağırın. Modeller oluşturulmadan önce bağlantıya erişmelisiniz; bunun için `connectionFactory`'yi kullanın:

```typescript
@@filename(app.module)
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

#### Ayırt Ediciler (Discriminators)

[Ayırt ediciler (discriminators)](https://mongoosejs.com/docs/discriminators.html), bir şema miras mekanizmasıdır. Aynı temel MongoDB koleksiyonu üzerinde örtüşen şemalara sahip birden çok model oluşturmanıza olanak tanır.

Varsayalım ki tek bir koleksiyonda farklı türde olayları takip etmek istiyorsunuz. Her olayın bir zaman damgası olacak.

```typescript
@@filename(event.schema)
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

> info **Hint** Mongoose'un farklı ayırt edici modeller arasındaki farkı nasıl anladığı, varsayılan olarak `__t` olan "ayırt edici anahtar" aracılığıyla gerçekleşir. Mongoose, belgelinin bu belirli ayırt edici alt tür olduğunu takip etmek için şemalarınıza `__t` adında bir String yol ekler.
> Ayırt etme için yol belirtmek için `discriminatorKey` seçeneğini de kullanabilirsiniz.

`SignedUpEvent` ve `ClickedLinkEvent` örnekleri, genel olaylarla aynı koleksiyonda depolanacaktır.

Şimdi, `ClickedLinkEvent` sınıfını aşağıdaki gibi tanımlayalım:

```typescript
@@filename(click-link-event.schema)
@Schema()
export class ClickedLinkEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  url: string;
}

export const ClickedLinkEventSchema = SchemaFactory.createForClass(ClickedLinkEvent);
```

Ve `SignUpEvent` sınıfı:

```typescript
@@filename(sign-up-event.schema)
@Schema()
export class SignUpEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  user: string;
}

export const SignUpEventSchema = SchemaFactory.createForClass(SignUpEvent);
```

Bunu yerine getirdikten sonra, bir şema için bir ayırt edici kaydetmek için `discriminators` seçeneğini kullanın. Hem `MongooseModule.forFeature` hem de `MongooseModule.forFeatureAsync` üzerinde çalışır:

```typescript
@@filename(event.module)
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

#### Test

Bir uygulamayı birim test ettiğimizde genellikle herhangi bir veritabanı bağlantısından kaçınmak isteriz, bu da test süitlerimizi kurmak için daha basit ve daha hızlı hale getirir. Ancak sınıflarımız bağlantı örneğinden alınan modellere bağlı olabilir. Bu sınıfları nasıl çözebiliriz? Çözüm, sahte modeller oluşturmaktır.

Bunu daha kolay hale getirmek için, `@nestjs/mongoose` paketi, bir belirteç adına dayalı olarak hazırlanmış bir [enjeksiyon belirteci](https://docs.nestjs.com/fundamentals/custom-providers#di-fundamentals) döndüren `getModelToken()` fonksiyonunu ortaya çıkarır. Bu belirteç kullanılarak, `useClass`, `useValue` ve `useFactory` gibi standart [özel sağlayıcı](/docs/fundamentals/custom-providers) tekniklerini kullanarak kolayca bir sahte uygulama sağlayabilirsiniz. Örneğin:

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

Bu örnekte, herhangi bir tüketici, `@InjectModel()` dekoratörünü kullanarak bir `Model<Cat>` enjekte ettiğinde, sabit bir `catModel` (nesne örneği) sağlanacaktır.

<app-banner-courses></app-banner-courses>

#### Asenkron yapılandırma

Modül seçeneklerini statik olarak değil, asenkron olarak iletmek istediğinizde, `forRootAsync()` yöntemini kullanın. Nest, çoğu dinamik modülle ilgilenmek için asenkron yapılandırma ile başa çıkmak için birkaç teknik sağlar.

Bir teknik, bir fabrika fonksiyonu kullanmaktır:

```typescript
MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: 'mongodb://localhost/nest',
  }),
});
```

Diğer [fabrika sağlayıcıları](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) gibi, fabrika fonksiyonumuz `async` olabilir ve `inject` aracılığıyla bağımlılıkları enjekte edebilir.

```typescript
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    uri: configService.get<string>('MONGODB_URI'),
  }),
  inject: [ConfigService],
});
```

Alternatif olarak, bir sınıf kullanarak `MongooseModule`'i bir fabrika yerine yapılandırabilirsiniz, aşağıda gösterildiği gibi:

```typescript
MongooseModule.forRootAsync({
  useClass: MongooseConfigService,
});
```

Yapılandırmada gerekli seçenekleri oluşturmak için `MongooseConfigService`'yi içinde `MongooseModule` kullanarak oluşturur. Bu örnekte, `MongooseConfigService`'nin `MongooseOptionsFactory` arabirimini uygulaması gerektiğini unutmayın. `MongooseModule`, sağlanan sınıfın örneği üzerinde `createMongooseOptions()` yöntemini çağıracaktır.

```typescript
@Injectable()
export class MongooseConfigService implements MongooseOptionsFactory {
  createMongooseOptions(): MongooseModuleOptions {
    return {
      uri: 'mongodb://localhost/nest',
    };
  }
}
```

Eğer `MongooseModule` içinde özel bir kopya oluşturmak yerine mevcut bir seçenek sağlayıcısını yeniden kullanmak istiyorsanız, `useExisting` sözdizimini kullanın.

```typescript
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/06-mongoose) bulunabilir.