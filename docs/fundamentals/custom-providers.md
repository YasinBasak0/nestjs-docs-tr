### Özel Sağlayıcılar

Daha önceki bölümlerde, Nest içindeki **Bağımlılık Enjeksiyonu (DI)**'nun çeşitli yönlerine ve bir sınıfa örnek olarak genellikle servis sağlayıcılarını içe aktarmak için kullanılan [constructor tabanlı](https://docs.nestjs.com/providers#dependency-injection) bağımlılık enjeksiyonuna nasıl entegre edildiğine değindik. Nest'in temelinde Bağımlılık Enjeksiyonu'nun temel bir şekilde nasıl işlediğini öğrenmiştik. Şimdiye kadar sadece ana bir deseni keşfettik. Uygulamanız karmaşıklaştıkça, DI sisteminin tam özelliklerinden yararlanmanız gerekebilir. Bu nedenle, bunları daha ayrıntılı bir şekilde keşfetmeye devam edelim.

#### DI Temelleri

Bağımlılık enjeksiyonu, bağımlılıkların anlık olarak oluşturulmasını kendi kodunuzda belirlemeniz yerine (bizim durumumuzda NestJS çalışma zamanı sistemi), bir Tersine Çevirme Kontrolü ([IoC](https://en.wikipedia.org/wiki/Inversion_of_control)) tekniğidir. Örneğin [Sağlayıcılar bölümündeki](https://docs.nestjs.com/providers) bu örnekte ne olduğuna bir göz atalım.

İlk olarak, bir sağlayıcı tanımlarız. `@Injectable()` dekoratörü, `CatsService` sınıfını bir sağlayıcı olarak işaretler.

```typescript
@@filename(cats.service)
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor() {
    this.cats = [];
  }

  findAll() {
    return this.cats;
  }
}
```

Ardından, Nest'ten sağlayıcıyı controller sınıfımıza enjekte etmesini istiyoruz:

```typescript
@@filename(cats.controller)
import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
@@switch
import { Controller, Get, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

Son olarak, `CatsService` sağlayıcısını Nest IoC konteyneri ile kaydediyoruz:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

Bu çalışmayı sağlamak için perde altında neler olduğuna dair tam olarak ne olduğuna bir göz atalım. Süreçte üç temel adım bulunmaktadır:

1. `cats.service.ts` dosyasında, `@Injectable()` dekoratörü, `CatsService` sınıfını Nest IoC konteyneri tarafından yönetilebilen bir sınıf olarak bildirir.
2. `cats.controller.ts` dosyasında, `CatsController`, constructor enjeksiyonu ile `CatsService` token'ına bağımlılığını bildirir:

```typescript
  constructor(private catsService: CatsService)
```

3. `app.module.ts` dosyasında, `CatsService` token'ını `cats.service.ts` dosyasındaki `CatsService` sınıfı ile ilişkilendiririz. Bu ilişkilendirme (aynı zamanda _kayıt_ olarak da adlandırılır) nasıl gerçekleştiğini <a href="/fundamentals/custom-providers#standard-providers">aşağıda</a> tam olarak göreceğiz.

Nest IoC konteyneri bir `CatsController` örneğini başlattığında, önce herhangi bir bağımlılığa bakar\*. `CatsService` bağımlılığını bulduğunda, `CatsService` token'ında bir arama gerçekleştirir ve `CatsService` sınıfını, kayıt adımında (#3 yukarıda) döndürür. Varsayılan davranış olan `SINGLETON` kapsamını (kapsam), Nest `CatsService` örneğini oluşturur, önbelleğe alır ve döndürür; veya zaten önbellekte varsa, mevcut örneği döndürür.

\*Bu açıklama noktasını bel

irtmek adına biraz basitleştirdik. Bağımlılıklar için kodun analiz süreci oldukça sofistike bir şekilde, uygulama başlatılırken gerçekleşir. Bir ana özellik, bağımlılık analizinin veya "bağımlılık grafi oluşturmanın", **transitif** olmasıdır. Yukarıdaki örnekte, `CatsService` kendisi bağımlılıklara sahipse, bunlar da çözülür. Bağımlılık grafi, bağımlılıkların doğru sırayla çözülmesini sağlar - temelde "aşağıdan yukarıya" doğru. Bu mekanizma, geliştiricinin böyle karmaşık bağımlılık grafiklerini yönetmek zorunda kalmamasını sağlar.

<app-banner-courses></app-banner-courses>

#### Standart Sağlayıcılar

`@Module()` dekoratörüne daha yakından bakalım. `app.module` dosyasında şunu beyan ediyoruz:

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

`providers` özelliği bir dizi `sağlayıcı` alır. Şimdiye kadar, bu sağlayıcıları bir sınıf adları listesiyle sağladık. Aslında, `providers: [CatsService]` sözdizimi, daha tam sözdizimi için kısaltmadır:

```typescript
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

Bu açık yapıyı gördüğümüzde, kayıt sürecini anlayabiliriz. Burada, açıkça `CatsService` token'ını `CatsService` sınıfı ile ilişkilendiriyoruz. Kısaltılmış nota, token'ın aynı adlı bir sınıfın bir örneğini talep ettiği en yaygın kullanım durumunu basitleştirmek için bir kolaylık sadece.

#### Özel Sağlayıcılar

Gereksinimleriniz, _Standart Sağlayıcılar_ tarafından sunulanlardan daha fazlasını gerektirdiğinde ne olur? İşte birkaç örnek:

- Bir sınıfın Nest tarafından örneklenmesi yerine özel bir örnek oluşturmak istersiniz
- Varolan bir sınıfı ikinci bir bağımlılıkta yeniden kullanmak istersiniz
- Bir sınıfı test etmek için bir mock sürümü ile değiştirmek istersiniz

Nest, bu durumları yönetmek için Özel Sağlayıcılar tanımlamanıza izin verir. Özel sağlayıcıları tanımlamak için birkaç yol sunar. Onları birlikte inceleyelim.

> info **İpucu** Bağımlılık çözümleme ile ilgili sorunlar yaşıyorsanız `NEST_DEBUG` ortam değişkenini ayarlayabilir ve başlatma sırasında ek bağımlılık çözümleme günlükleri alabilirsiniz.

#### Değer Sağlayıcıları: `useValue`

`useValue` sözdizimi, bir sabit değeri enjekte etmek, bir harici kitaplığı Nest konteynerine koymak veya gerçek bir uygulamayı bir mock nesne ile değiştirmek için kullanışlıdır. Diyelim ki test amaçları için Nest'e bir mock `CatsService` kullanmasını zorlamak istiyorsunuz.

```typescript
import { CatsService } from './cats.service';

const mockCatsService = {
  /* mock implementation
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

Bu örnekte, `CatsService` token'ı, `mockCatsService` mock nesnesine çözümlenecektir. `useValue`, bir değeri gerektirir - bu durumda, yerine geçen `CatsService` sınıfı ile aynı arabirime sahip olan bir harf nesne. TypeScript'in [yapısal yazım](https://www.typescriptlang.org/docs/handbook/type-compatibility.html) özelliğinden dolayı, aynı arabirimlere sahip herhangi bir nesneyi kullanabilirsiniz, bunlar arasında harf bir nesne veya `new` ile başlatılmış bir sınıf örneği bulunabilir.

#### Sınıf tabanlı olmayan sağlayıcı token'ları

Şimdiye kadar, sağlayıcı token'larımızı (bir sağlayıcının `provide` özelliğindeki değer) sınıf adları olarak kullandık. Bu, [constructor based injection](https://docs.nestjs.com/providers#dependency-injection) ile kullanılan standart desenle eşleşir, burada token da bir sınıf adıdır. (Bu kavram tamamen net değilse, bu konsepti anımsatmak için <a href="/fundamentals/custom-providers#di-fundamentals">DI Temelleri</a>ne geri dönün). Bazen, DI token'ı olarak dizgileri veya sembolleri kullanma esnekliğine sahip olmak isteyebiliriz. Örneğin:

```typescript
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

Bu örnekte, bir dize değerli token (`'CONNECTION'`) ile dış bir dosyadan içe aktardığımız mevcut bir `connection` nesnesini ilişkilendiriyoruz.

> warning **Dikkat** Dize değerlerini token değeri olarak kullanmanın yanı sıra, JavaScript [sembolleri](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) veya TypeScript [enum'ları](https://www.typescriptlang.org/docs/handbook/enums.html) da kullanabilirsiniz.

Daha önce standart [constructor based injection](https://docs.nestjs.com/providers#dependency-injection) deseni kullanarak bir sağlayıcıyı nasıl enjekte edeceğimizi gördük. Bu desen, bağımlılığın bir sınıf adı ile bildirilmesini **gerektirir**. `'CONNECTION'` özel sağlayıcısı bir dize değerli token kullanır. Bu tür bir sağlayıcıyı nasıl enjekte edeceğimizi görelim. Bunu yapmak için `@Inject()` dekoratörünü kullanırız. Bu dekoratör, tek bir argüman alır - token.

```typescript
@@filename()
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
@@switch
@Injectable()
@Dependencies('CONNECTION')
export class CatsRepository {
  constructor(connection) {}
}
```

> info **İpucu** `@Inject()` dekoratörü, `@nestjs/common` paketinden içe aktarılır.

Yukarıdaki örneklerde dize `'CONNECTION'`'yi doğrudan kullansak da, temiz kod organizasyonu için, sembollerin veya enum'ların kendi dosyalarında tanımlandığı gibi, bunları ihtiyaç duyulan yerde içe aktarmak için genellikle en iyi uygulama pratiği bir dosyada token'ları tanımlamaktır.

#### Sınıf Sağlayıcıları: `useClass`

`useClass` sözdizimi, bir belirtecin çözümlenmesi gereken bir sınıfı dinamik olarak belirlemenizi sağlar. Örneğin, soyut (veya varsayılan) bir `ConfigService` sınıfımız olduğunu varsayalım. Mevcut ortama bağlı olarak, Nest'e farklı bir yapılandırma servisi uygulaması sağlamasını istiyoruz. Aşağıdaki kod, böyle bir stratejiyi uygular.

```typescript
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

Bu kod örneğinde birkaç ayrıntıya dikkat edelim. İlk olarak, `configServiceProvider`'ı bir literal nesne ile tanımlarız, ardından bu nesneyi modül dekoratörünün `providers` özelliğine geçiririz. Bu sadece biraz kod organizasyonu, ancak bu bölümde şimdiye kadar kullandığımız örneklerle işlevsel olarak eşdeğerdir.

Ayrıca, `ConfigService` sınıf adını belirteç olarak kullandık. `ConfigService`'e bağlı herhangi bir sınıf için, Nest, bu sınıfın örneğini (örneğin, `@Injectable()` dekoratörü ile bildirilmiş bir `ConfigService` gibi) geçerli bir uygulama olmadan bile sağlayacaktır.

#### Fabrika Sağlayıcıları: `useFactory`

`useFactory` sözdizimi, sağlayıcıları **dinamik olarak** oluşturmayı sağlar. Gerçek sağlayıcı, bir fabrika işlevinden dönen değer tarafından sağlanır. Fabrika işlevi, ihtiyaç duyulan kadar basit veya karmaşık olabilir. Basit bir fabrika, başka herhangi bir sağlayıcıya bağlı olmayabilir. Daha karmaşık bir fabrika, sonucunu hesaplamak için ihtiyaç duyduğu diğer sağlayıcıları da enjekte edebilir. Bu durum için, fabrika sağlayıcı sözdiziminin iki ilgili mekanizması bulunmaktadır:

1. Fabrika işlevi (isteğe bağlı) argümanları kabul edebilir.
2. (isteğe bağlı) `inject` özelliği, Nest'in bu sağlayıcıları çözümleyeceği ve anında işleme sürecinde fabrika işlevine argüman olarak ileteceği bir sağlayıcılar dizisini kabul eder. Ayrıca, bu sağlayıcılar isteğe bağlı olarak işaretlenebilir. İki liste korele edilmelidir: Nest, `inject` listesinden örnekleri, aynı sıra ile fabrika işlevine argüman olarak iletecektir. Aşağıdaki örnek bunu göstermektedir.

```typescript
@@filename()
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider, optionalProvider?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \_____________/            \__________________/
  //        This provider              The provider with this
  //        is mandatory.              token can resolve to `undefined`.
};

@Module({
  providers: [
    connectionProvider,
    OptionsProvider,
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
@@switch
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider, optionalProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \_____________/            \__________________/
  //        This provider              The provider with this
  //        is mandatory.              token can resolve to `undefined`.
};

@Module({
  providers: [
    connectionProvider,
    OptionsProvider,
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
```

#### Takma Ad Sağlayıcıları: `useExisting`

`useExisting` sözdizimi, mevcut sağlayıcılar için takma adlar oluşturmanıza olanak tanır. Bu, aynı sağlayıcıya iki farklı şekilde erişim sağlar. Aşağıdaki örnekte, (dize tabanlı) `AliasedLoggerService` belirteci, (sınıf tabanlı) `LoggerService` belirteci için bir takma addır. `'AliasedLoggerService'` ve `LoggerService` için iki farklı bağımlılığımız olduğunu varsayalım. Her iki bağımlılık da `SINGLETON` kapsamıyla belirlenirse, her ikisi de aynı örneğe çözümlenecektir.

```typescript
@Injectable()
class LoggerService {
  /* implementation details */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

#### Hizmet Tabanlı Olmayan Sağlayıcılar

Sağlayıcılar genellikle hizmetler sağlar, ancak bununla sınırlı değildir. Bir sağlayıcı **herhangi bir** değeri sağlayabilir. Örneğin, bir sağlayıcı mevcut ortama dayalı olarak bir dizi yapılandırma nesnesi sağlayabilir, aşağıdaki örnekte olduğu gibi:

```typescript
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```

#### Özel Sağlayıcıyı Dışa Aktarma

Herhangi bir sağlayıcı gibi, özel bir sağlayıcı da bildirildiği modülle sınırlıdır. Diğer modüller tarafından görünür olması için dışa aktarılması gerekir. Özel bir sağlayıcıyı dışa aktarmak için, ya belirteci ya da tam sağlayıcı nesnesini kullanabiliriz.

Aşağıdaki örnek, belirteci kullanarak dışa aktarmayı göstermektedir:

```typescript
@@filename()
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
@@switch
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

Alternatif olarak, tam sağlayıcı nesnesi ile dışa aktarım:

```typescript
@@filename()
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
@@switch
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```
