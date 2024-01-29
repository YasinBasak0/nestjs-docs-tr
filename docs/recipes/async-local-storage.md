### Async Local Storage

`AsyncLocalStorage`, `async_hooks` API tabanlı bir [Node.js API](https://nodejs.org/api/async_context.html#async_context_class_asynclocalstorage) sağlar ve uygulama içinde yerel durumu açıkça bir fonksiyon parametresi olarak iletmeye gerek olmadan iletebileceğiniz alternatif bir yol sunar. Diğer dillerdeki bir thread-local depolamaya benzer.

Async Local Storage'ın ana fikri, bir fonksiyon çağrısını `AsyncLocalStorage#run` çağrısıyla _sarabilmemizdir_. Sarılan çağrı içinde çağrılan tüm kod, aynı `store`'a erişim sağlar ve bu, her çağrı zinciri için benzersiz olacaktır.

NestJS bağlamında, bir isteğin yaşam döngüsü içinde kodun geri kalanını sarmak için bir yer bulabilirsek, yalnızca bu isteğe özgü olarak görünen ve REQUEST kapsamlı sağlayıcıların ve bazı sınırlamalarının alternatifi olarak hizmet edebilecek bir duruma erişim ve değiştirme yeteneğine sahip olacağız.

Alternatif olarak, ALS'yi bir sistemin bir kısmı için (örneğin _işlem_ nesnesi) bağımsız bir şekilde açıkça servisler arasında iletmeksizin bağlamı iletebiliriz, bu da izolasyonu ve kapsüllemeyi artırabilir.

#### Özel İmplementasyon

NestJS kendisi için `AsyncLocalStorage` için herhangi bir yerleşik soyutlama sağlamaz, bu yüzden konseptin tam anlayışını elde etmek için en basit HTTP durumu için nasıl uygulayabileceğimizi inceleyelim:

> info **info** Hazır [özel bir paket](recipes/async-local-storage#nestjs-cls) için okumaya devam etmek için aşağıya bakın.

1. İlk olarak, `AsyncLocalStorage`'ın yeni bir örneğini bir paylaşılan kaynak dosyasında oluşturun. NestJS kullanıyorsak, ayrıca bunu özel bir sağlayıcı ile bir modül haline getirelim.

```ts
@@filename(als.module)
@Module({
  providers: [
    {
      provide: AsyncLocalStorage,
      useValue: new AsyncLocalStorage(),
    },
  ],
  exports: [AsyncLocalStorage],
})
export class AlsModule {}
```
>  info **Hint** `AsyncLocalStorage` `async_hooks` tarafından içe aktarılmıştır.

2. Yalnızca HTTP ile ilgileniyoruz, bu nedenle bir ort yazılımını, `next` fonksiyonunu `AsyncLocalStorage#run` ile saran bir ort yazılımını kullanalım. Bir ort yazılımı isteğin vurduğu ilk şey olduğundan, bu `store`'u tüm geliştirmelere ve sistemin geri kalanına uygun hale getirecektir.

```ts
@@filename(app.module)
@Module({
  imports: [AlsModule]
  providers: [CatService],
  controllers: [CatController],
})
export class AppModule implements NestModule {
  constructor(
    // AsyncLocalStorage'ı modül kurucu fonksiyonuna enjekte edin,
    private readonly als: AsyncLocalStorage
  ) {}

  configure(consumer: MiddlewareConsumer) {
    // ort yazılımı bağlayın,
    consumer
      .apply((req, res, next) => {
        // depoyu, talebe dayalı olarak
        // bazı varsayılan değerlerle doldurun,
        const store = {
          userId: req.headers['x-user-id'],
        };
        // ve "next" fonksiyonunu geri çağırma
        // ile birlikte "als.run" yöntemine depoyu geçirin.
        this.als.run(store, () => next());
      })
      // ve tüm rotalar için (Fastify kullanılıyorsa '(.*)' kullanın)
      .forRoutes('*');
  }
}
@@switch
@Module({
  imports: [AlsModule]
  providers: [CatService],
  controllers: [CatController],
})
@Dependencies(AsyncLocalStorage)
export class AppModule {
  constructor(als) {
    // AsyncLocalStorage'ı modül kurucu fonksiyonuna enjekte edin,
    this.als = als
  }

  configure(consumer) {
    // ort yazılımı bağlayın,
    consumer
      .apply((req, res, next) => {
        // depoyu, talebe dayalı olarak
        // bazı varsayılan değerlerle doldurun,
        const store = {
          userId: req.headers['x-user-id'],
        };
        // ve "next" fonksiyonunu geri çağırma
        // ile birlikte "als.run" yöntemine depoyu geçirin.
        this.als.run(store, () => next());
      })
      // ve tüm rotalar için (Fastify kullanılıyorsa '(.*)' kullanın)
      .forRoutes('*');
  }
}
```

3. Şimdi, bir isteğin yaşam döngüsü içinde herhangi bir yerde yerel depo örneğine erişebiliriz.

```ts
@@filename(cat.service)
@Injectable()
export class CatService {
  constructor(
    // Sağlanan ALS örneğini enjekte edebiliriz.
    private readonly als: AsyncLocalStorage,
    private readonly catRepository: CatRepository,
  ) {}

  getCatForUser() {
    // "getStore" yöntemi her zaman
    // belirli bir taleple ilişkilendirilen depo örneğini döndürecektir.
    const userId = this.als.getStore()["userId"] as number;
    return this.catRepository.getForUser(userId);
  }
}
@@switch
@Injectable()
@Dependencies(AsyncLocalStorage, CatRepository)
export class CatService {
  constructor(als, catRepository) {
    // Sağlanan ALS örneğini enjekte edebiliriz.
    this.als = als
    this.catRepository = catRepository
  }

  getCatForUser() {
    // "getStore" yöntemi her zaman
    // belirli bir taleple ilişkilendirilen depo örneğini döndürecektir.
    const userId = this.als.getStore()["userId"] as number;
    return this.catRepository.getForUser(userId);


  }
}
```

4. İşte bu kadar. Şimdi, tüm `REQUEST` nesnesini enjekte etme ihtiyacı olmadan isteğe özgü durumu paylaşma bir yolumuz var.

> warning **warning** Teknik birçok kullanım durumu için kullanışlı olsa da, kod akışını inherent olarak obfiske eder (örtük bağlam oluşturur), bu nedenle sorumlu bir şekilde kullanın ve özellikle bağlam dışı "[Tanrı nesneleri](https://en.wikipedia.org/wiki/God_object)" oluşturmaktan kaçının.

### NestJS CLS

[nestjs-cls](https://github.com/Papooch/nestjs-cls) paketi, düz `AsyncLocalStorage` kullanımına göre birkaç DX (developer experience) geliştirmesi sunar (`CLS`, _continuation-local storage_ teriminin kısaltmasıdır). Bu, `store`'un farklı iletimler için (yalnızca HTTP değil) başlatılması için çeşitli yöntemler sunan bir `ClsModule` içine uygulamayı soyutlar ve güçlü bir türleme desteği sağlar.

Depoya [Proxy Sağlayıcıları](https://www.npmjs.com/package/nestjs-cls#proxy-providers) kullanarak iş mantığından tamamen soyutlayabilir veya enjekte edilebilir bir `ClsService` ile depoya erişebilirsiniz.

> info **info** `nestjs-cls`, NestJS çekirdek ekibi tarafından yönetilmeyen üçüncü taraf bir pakettir. Lütfen kütüphane ile ilgili bulduğunuz herhangi bir sorunu [ilgili depoda](https://github.com/Papooch/nestjs-cls/issues) bildirin.

#### Kurulum

`@nestjs` kütüphanelerine dayandığı için, yalnızca built-in Node.js API'yi kullanır. Diğer paketler gibi kurun.

```bash
npm i nestjs-cls
```

#### Kullanım

Yukarıda [açıklanan](#custom-implementation) gibi bir işlevsellik, `nestjs-cls` kullanılarak şu şekilde uygulanabilir:

1. Kök modülde `ClsModule`'u içe aktarın.

```ts
@@filename(app.module)
@Module({
  imports: [
    // ClsModule'u kaydedin,
    ClsModule.forRoot({
      middleware: {
        // Tüm rotalar için ClsMiddleware'yi
        // otomatik olarak bağlamak
        mount: true,
        // ve varsayılan depo değerlerini sağlamak için
        // setup yöntemini kullanmak için.
        setup: (cls, req) => {
          cls.set('userId', req.headers['x-user-id']);
        },
      },
    }),
  ],
  providers: [CatService],
  controllers: [CatController],
})
export class AppModule {}
```

2. Ardından, `ClsService`'i erişmek için kullanabilirsiniz.

```ts
@@filename(cat.service)
@Injectable()
export class CatService {
  constructor(
    // Sağlanan ClsService örneğini enjekte edebiliriz,
    private readonly cls: ClsService,
    private readonly catRepository: CatRepository,
  ) {}

  getCatForUser() {
    // ve "get" yöntemini kullanarak herhangi bir depolanan değeri alabiliriz.
    const userId = this.cls.get('userId');
    return this.catRepository.getForUser(userId);
  }
}
@@switch
@Injectable()
@Dependencies(AsyncLocalStorage, CatRepository)
export class CatService {
  constructor(cls, catRepository) {
    // Sağlanan ClsService örneğini enjekte edebiliriz,
    this.cls = cls
    this.catRepository = catRepository
  }

  getCatForUser() {
    // ve "get" yöntemini kullanarak herhangi bir depolanan değeri alabiliriz.
    const userId = this.cls.get('userId');
    return this.catRepository.getForUser(userId);
  }
}
```

3. `ClsService` tarafından yönetilen depo değerlerinin güçlü bir şekilde yazılmasını (ve aynı zamanda dize anahtarları için otomatik öneriler almayı) almak için enjekte ederken bir opsiyonel tür parametresi `ClsService<MyClsStore>` kullanabiliriz.

```ts
export interface MyClsStore extends ClsStore {
  userId: number;
}
```

> info **hint** Ayrıca, paketin otomatik olarak bir İstek Kimliği oluşturmasına izin vermek ve daha sonra `cls.getId()` kullanarak bunu erişmek ya da tüm İstek nesnesini `cls.get(CLS_REQ)` kullanarak almak mümkündür.

#### Test

`ClsService`, yalnızca başka bir enjekte edilebilir sağlayıcı gibi olduğundan, birim testlerde tamamen sahte yapılabilir.

Ancak, belirli entegrasyon testlerinde, gerçek `ClsService` uygulamasını kullanmak isteyebiliriz. Bu durumda, bağlam farkındalığı olan kodu `ClsService#run` veya `ClsService#runWith` çağrısı ile sarmamız gerekecektir.

```ts
describe('CatService', () => {
  let service: CatService
  let cls: ClsService
  const mockCatRepository = createMock<CatRepository>()

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      // Test modülünün çoğunu normalde nasıl kurarsak öyle kurun.
      providers: [
        CatService,
        {
          provide: CatRepository
          useValue: mockCatRepository
        }
      ],
      imports: [
        // Yalnızca ClsService'i sağlayan statik sürümünü içe aktarın
        // ki ClsService'i hiçbir şekilde ayarlamasın.
        ClsModule
      ],
    }).compile()

    service = module.get(CatService)

    // Ayrıca daha sonra kullanmak üzere ClsService'i alın.
    cls = module.get(ClsService)
  })

  describe('getCatForUser', () => {
    it('retrieves cat based on user id', async () => {
      const expectedUserId = 42
      mockCatRepository.getForUser.mockImplementationOnce(
        (id) => ({ userId: id })
      )

      // Test çağrısını "runWith" yöntemi içine sarmak için
      // burada elle yapılacak depo değerlerini iletebiliriz.
      const cat = await cls.runWith(
        { userId: expectedUserId },
        () => service.getCatForUser()
      )

      expect(cat.userId).toEqual(expectedUserId)
    })
  })
})
```

#### Daha Fazla Bilgi

Tam API belgeleri ve daha fazla kod örneği için [NestJS CLS GitHub Sayfası](https://github.com/Papooch/nestjs-cls)'ni ziyaret edin.