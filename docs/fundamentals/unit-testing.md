### Test

Otomatik test, ciddi bir yazılım geliştirme çabasının önemli bir parçası olarak kabul edilir. Otomasyon, geliştirme sırasında bireysel testleri veya test paketlerini hızlı ve kolay bir şekilde tekrarlamayı sağlar. Bu, sürümlerin kalite ve performans hedeflerini karşılamasına yardımcı olur. Otomasyon, kapsama alanını artırır ve geliştiricilere daha hızlı bir geri bildirim döngüsü sağlar. Otomasyon, bireysel geliştiricilerin veri tabanı kontrolü kontrol etme, özellik entegrasyonu ve sürüm yayınlama gibi kritik geliştirme yaşam döngüsü kavşaklarında testlerin çalışmasını sağlar.

Bu tür testler genellikle bir dizi türü kapsar, bunlar arasında birim testleri, uçtan uca (e2e) testleri, entegrasyon testleri vb. bulunur. Faydaları tartışılamaz olsa da, bunları kurmak sıkıcı olabilir. Nest, etkili test dahil olmak üzere geliştirme en iyi uygulamalarını teşvik etmeye çalışır, bu nedenle geliştiricilere ve ekiplere testler oluşturmak ve otomatikleştirmek için aşağıdaki gibi özellikler sunar. Nest:

- bileşenler için varsayılan birim testleri ve uygulamalar için e2e testleri otomatik olarak iskeleler
- izole bir modül/uygulama yükleyicisi oluşturan bir test çalıştırıcı sağlar
- [Jest](https://github.com/facebook/jest) ve [Supertest](https://github.com/visionmedia/supertest) ile entegrasyon sağlar, ancak test araçlarına karşı tarafsız kalır
- Nest bağımlılık enjeksiyon sistemini test ortamında bileşenleri kolayca taklit etmek için kullanılabilir hale getirir

Bahsedildiği gibi, Nest herhangi bir **test çerçevesini** kullanabilirsiniz çünkü belirli bir araç takımı dayatmaz. Sadece ihtiyaç duyulan unsurları (örneğin, test çalıştırıcısı) değiştirin ve yine de Nest'in hazır test olanaklarının avantajlarını kullanacaksınız.

#### Kurulum

Başlamak için önce gerekli paketi yükleyin:

```bash
$ npm i --save-dev @nestjs/testing
```

#### Birim test

Aşağıdaki örnekte, iki sınıfı test ediyoruz: `CatsController` ve `CatsService`. [Jest](https://github.com/facebook/jest) öntanımlı bir test çerçevesi olarak sağlanmıştır. Hem bir test çalıştırıcısı olarak görev yapar hem de sınamak, casusluk vb. konularda yardımcı olan assert işlevleri ve test çifti araçları sağlar. Aşağıdaki temel testte, bu sınıfları manuel olarak örnekleriz ve kontrolcü ve servisin API sözleşmesini yerine getirip getirmediğini sağlarız.

```typescript
@@filename(cats.controller.spec)
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
@@switch
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController;
  let catsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats',

 async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

> bilgi **İpucu** Test dosyalarınızı test ettikleri sınıfların yanına yerleştirin. Test dosyalarının `.spec` veya `.test` uzantısına sahip olması gerekir.

Yukarıdaki örnek basit olduğu için, aslında Nest'e özgü hiçbir şeyi test etmiyoruz. Gerçekten de bağımlılık enjeksiyonu kullanmıyoruz (test dosyamıza `CatsService` örneğini geçirdiğimize dikkat edin). Bu tür bir test - test edilen sınıfları manuel olarak örneklendirdiğimiz yer - genellikle **izole test** olarak adlandırılır, çünkü çerçeveden bağımsızdır. Daha fazla Nest özelliği kullanan uygulamaları test etmenize yardımcı olan daha gelişmiş yetenekleri tanıtalım.

#### Test yardımcıları

`@nestjs/testing` paketi, daha güçlü bir test sürecini mümkün kılan bir dizi yardımcı sağlar. Önceki örneği yerleşik `Test` sınıfını kullanarak tekrar yazalım:

```typescript
@@filename(cats.controller.spec)
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get<CatsService>(CatsService);
    catsController = moduleRef.get<CatsController>(CatsController);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
@@switch
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController;
  let catsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get(CatsService);
    catsController = moduleRef.get(CatsController);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

`Test` sınıfı, genel olarak Nest çalışma zamanını taklit eden bir uygulama yürütme bağlamı sağlamak için kullanışlıdır, ancak sınıf örneklerini, casuslamayı ve geçersiz kılmayı yönetmeyi kolaylaştıran kancalar sunar. `Test` sınıfının bir `createTestingModule()` yöntemi vardır ve bu yöntem bir modül meta veri nesnesini argüman olarak alır (bu nesneyi `@Module()` dekoratörüne geçirdiğiniz nesne ile aynı). Bu yöntem, test için hazır bir modül döndüren bir `TestingModule` örneği döndürür. Birim testler için önemli olan, bu yöntemin içerdiği `compile()` yöntemidir. Bu yöntem, bağımlılıklarıyla bir modülü başlatır (uygulamanın geleneksel `main.ts` dosyasında `NestFactory.create()` kullanılarak uygulamanın başlatılma şekliyle benzer), ve test için hazır bir modülü döndürür.

> info **İpucu** `compile()` yöntemi **asenkron** olduğundan dolayı beklenmelidir. Modül derlendikten sonra, `get()` yöntemini kullanarak herhangi bir **statik** örneği alabilirsiniz.

`TestingModule`, [modül referansı](/docs/fundamentals/module-reference) sınıfından miras alır ve bu nedenle dinamik olarak çözümlenebilen kapsamlı sağlayıcıları (geçici veya istek tabanlı) çözme yeteneğine sahiptir. Bunu `resolve()` yöntemi ile yapın (`get()` yöntemi yalnızca statik örnekleri alabilir).

```typescript
const moduleRef = await Test.createTestingModule({
  controllers: [CatsController],
  providers: [CatsService],
}).compile();

catsService = await moduleRef.resolve(CatsService);
```

> warning **Uyarı** `resolve()` yöntemi, sağlayıcının kendi **DI konteyner alt-ağacından** benzersiz bir örnek döndürür. Her alt-ağacın benzersiz bir bağlam tanımlayıcısı vardır. Bu nedenle bu yöntemi birden fazla kez çağırırsanız ve örnek referanslarını karşılaştırırsanız, eşit olmadıklarını göreceksiniz.

> info **İpucu** Modül referansı özelliklerini [buradan](/docs/fundamentals/module-reference) daha fazla öğrenin.

Herhangi bir sağlayıcının üretim sürümü yerine, test amaçlı bir [özel sağlayıcı](/docs/fundamentals/custom-providers) ile geçersiz kılabilirsiniz. Örneğin, bir canlı veritabanına bağlanmak yerine bir veritabanı servisini taklit edebilirsiniz. Override'ları bir sonraki bölümde ele alacağız, ancak bunlar birim testleri için de mevcuttur.

#### Auto mocking

Nest ayrıca, tüm eksik bağımlılıklarınıza uygulanacak bir mock fabrikası tanımlamanıza olanak tanır. Bu, bir sınıfta çok sayıda bağımlılığınızın olduğu durumlar için yararlıdır ve hepsini mocklamak uzun zaman alabilir ve çok fazla kurulum gerektirebilir. Bu özelliği kullanmak için `createTestingModule()` yöntemi, bağımlılık mock'ları için bir fabrika ile zincirlenmelidir. Bu fabrika, isteğe bağlı bir belirteç alabilir, bu belirteç bir Nest sağlayıcısı için geçerli olan herhangi bir belirteç olabilir ve bir mock uygulaması döndürür. Aşağıda, [`jest-mock`](https://www.npmjs.com/package/jest-mock) ve `jest.fn()` kullanarak genel bir mocker oluşturma ve `CatsService` için özel bir mock örneği oluşturma örneği bulunmaktadır.

```typescript
// ...
import { ModuleMocker, MockFunctionMetadata } from 'jest-mock';

const moduleMocker = new ModuleMocker(global);

describe('CatsController', () => {
  let controller: CatsController;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      controllers: [CatsController],
    })
      .useMocker((token) => {
        const results = ['test1', 'test2'];
        if (token === CatsService) {
          return { findAll: jest.fn().mockResolvedValue(results) };
        }
        if (typeof token === 'function') {
          const mockMetadata = moduleMocker.getMetadata(token) as MockFunctionMetadata<any, any>;
          const Mock = moduleMocker.generateFromMetadata(mockMetadata);
          return new Mock();
        }
      })
      .compile();

    controller = moduleRef.get(CatsController);
  });
});
```

Bu mock'ları, genellikle özel sağlayıcılar gibi, test konteynerinden çıkartabilirsiniz, `moduleRef.get(CatsService)`.

> info **İpucu** `@golevelup/ts-jest`'ten `createMock` gibi genel bir mock fabrikası da doğrudan iletilmesi mümkündür.

> info **İpucu** `REQUEST` ve `INQUIRER` sağlayıcıları otomatik olarak mocklanamaz çünkü zaten bağlam içinde önceden tanımlanmışlardır. Ancak, özel sağlayıcı sözdizimini kullanarak veya `.overrideProvider` yöntemini kullanarak bunlar _üzerine yazılabilir_.

#### End-to-end testing (uçtan uca test)

Birleşik testler (e2e) tek başına modüllere ve sınıflara odaklanan birim testlerin aksine, modüllerin ve sınıfların etkileşimini daha yüksek bir seviyede kapsar. Bu, bir uygulama büyüdükçe her API uç noktasının bir araya getirilmiş davranışını manuel olarak test etmenin zor olacağı anlamına gelir. Otomatik birleşik testler, sistemin genel davranışının doğru olduğunu ve proje gereksinimlerini karşıladığını sağlamamıza yardımcı olur. Birleşik testler için, yukarıda **birim testi** için kapsadığımız konfigürasyonu kullanıyoruz. Ayrıca, Nest, HTTP isteklerini simüle etmek için [Supertest](https://github.com/visionmedia/supertest) kütüphanesini kullanmayı kolaylaştırır.

```typescript
@@filename(cats.e2e-spec)
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
@@switch
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

> info **İpucu** HTTP adaptörü olarak [Fastify](/docs/techniques/performance) kullanıyorsanız, bir miktar farklı bir yapılandırmaya ihtiyaç duyar ve yerleşik test yeteneklerine sahiptir:
>
> ```ts
> let app: NestFastifyApplication;
>
> beforeAll(async () => {
>   app = moduleRef.createNestApplication<NestFastifyApplication>(new FastifyAdapter());
>
>   await app.init();
>   await app.getHttpAdapter().getInstance().ready();
> });
>
> it(`/GET cats`, () => {
>   return app
>     .inject({
>       method: 'GET',
>       url: '/cats',
>     })
>     .then((result) => {
>       expect(result.statusCode).toEqual(200);
>       expect(result.payload).toEqual(/docs/* expectedPayload */);
>     });
> });
>
> afterAll(async () => {
>   await app.close();
> });
> ```

Bu örnekte, önceki bölümlerde açıklanan bazı kavramlara dayanmaktayız. Daha önce kullandığımız `compile()` yönteminin yanı sıra, şimdi tam bir Nest çalışma ortamını anlatan `createNestApplication()` yöntemini kullanıyoruz. Çalışan uygulamamıza referansı `app` değişkeninde saklıyoruz, böylece onu HTTP isteklerini simüle etmek için kullanabiliriz.

HTTP testlerini, Supertest'ten gelen `request()` fonksiyonunu kullanarak simüle ediyoruz. Bu HTTP isteklerinin çalışan Nest uygulamamıza yönlendirilmesini istiyoruz, bu nedenle `request()` fonksiyonuna Nest'in temelinde yatan HTTP dinleyicisine bir referans geçiriyoruz (ki bu da Express platformu tarafından sağlanmış olabilir). Bu nedenle `request(app.getHttpServer())` yapısını kullanırız. `request()` çağrısı bize, şimdi Nest uygulamasına bağlı bir şekilde çalışan bir HTTP Sunucusu üzerine sarılmış bir nesne verir, bu da gerçek bir HTTP isteği gibi ağ üzerinden gelen `get '/cats'` gibi bir isteği simüle etmek için kullanılabilen yöntemleri açığa çıkarır.

Bu örnekte, ayrıca `CatsService`'in alternatif (test-double) bir uygulamasını sağlı

klı bir şekilde test edebileceğimiz sabit bir değeri döndüren bir test-double uygulamasını da sağlıyoruz. Bu tür bir alternatif uygulama sağlamak için `overrideProvider()`'ı kullanın. Benzer şekilde, Nest, `overrideModule()`, `overrideGuard()`, `overrideInterceptor()`, `overrideFilter()` ve `overridePipe()` yöntemleri ile modülleri, koruyucuları, interceptor'ları, filtreleri ve boruları sırasıyla geçersiz kılmak için yöntemler sağlar.

Her geçersiz kılma yöntemi ( `overrideModule()` hariç), [özel sağlayıcılar](https://docs.nestjs.com/fundamentals/custom-providers) için açıklanan 3 farklı yöntemi içeren bir nesne döndürür:

- `useClass`: Nesnenin (sağlayıcı, koruyucu, filtre, vb.) yerine geçecek bir örneği sağlayacak bir sınıfı sağlarsınız.
- `useValue`: Nesnenin (sağlayıcı, koruyucu, filtre, vb.) yerine geçecek bir örneği sağlarsınız.
- `useFactory`: Nesnenin (sağlayıcı, koruyucu, filtre, vb.) yerine geçecek bir örneği döndüren bir işlevi sağlarsınız.

Öte yandan, `overrideModule()` yöntemi, özgün modülü geçersiz kılacak bir modül sağlamak için kullanılabilen `useModule()` yöntemine sahip bir nesne döndürür:

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideModule(CatsModule)
  .useModule(AlternateCatsModule)
  .compile();
```

Her geçersiz kılma yöntemi türü, sırasıyla `TestingModule` örneğini döndürür ve bu nedenle [akıcı stil](https://en.wikipedia.org/wiki/Fluent_interface) içinde diğer yöntemlerle zincirlenebilir. Bu tür bir zincirin sonunda Nest'in modülü başlatmasını ve başlatmasını sağlamak için `compile()` yöntemini kullanmalısınız.

Ayrıca, bazen testler çalıştırıldığında (örneğin, bir CI sunucusunda) özel bir günlükçü sağlamak isteyebilirsiniz. Nasıl günlük tutulacağını belirtmek için `LoggerService` arayüzünü yerine getiren bir nesneyi `setLogger()` yöntemini kullanın (varsayılan olarak sadece "hata" günlükleri konsola kaydedilecektir).

Derlenmiş modülün birkaç kullanışlı yöntemi bulunmaktadır, bunlar aşağıdaki tabloda açıklanmıştır:

<table>
  <tr>
    <td>
      <code>createNestApplication()</code>
    </td>
    <td>
      Verilen modüle dayalı bir Nest uygulaması (<code>INestApplication</code> örneği) oluşturur ve döndürür.
      Uygulamayı manuel olarak <code>init()</code> yöntemi kullanılarak başlatmanız gerektiğini unutmayın.
    </td>
  </tr>
  <tr>
    <td>
      <code>createNestMicroservice()</code>
    </td>
    <td>
      Verilen modüle dayalı bir Nest mikroservisi (<code>INestMicroservice</code> örneği) oluşturur ve döndürür.
    </td>
  </tr>
  <tr>
    <td>
      <code>get()</code>
    </td>
    <td>
      Uygulama bağlamında mevcut olan bir denetleyici veya sağlayıcının (koruyucular, filtreler vb. dahil) statik bir örneğini alır. [modül referansı](/docs/fundamentals/module-reference) sınıfından miras alınmıştır.
    </td>
  </tr>
  <tr>
     <td>
      <code>resolve()</code>
    </td>
    <td>
      Uygulama bağlamında mevcut olan bir denetleyici veya sağlayıcının (koruyucular, filtreler vb. dahil) dinamik olarak oluşturulmuş bir örneğini alır. [modül referansı](/docs/fundamentals/module-reference) sınıfından miras alınmıştır.
    </td>
  </tr>
  <tr>
    <td>
      <code>select()</code>
    </td>
    <td>
      Modülün bağımlılık grafiği üzerinden gezinir; seçilen modülden belirli bir örneği almak için kullanılabilir ( `get()` yöntemi içinde strict modu (<code>strict: true</code>) ile birlikte kullanılır).
    </td>
  </tr>
</table>

> info **İpucu** Birleşik test dosyalarınızı `test` dizini içinde tutun. Test dosyalarının `.e2e-spec` uzantısına sahip olması gerekmektedir.

#### Global Olarak Kayıtlı Enhancer'ları Geçersiz Kılma

Eğer global olarak kayıtlı bir koruyucu (veya boru, aracı veya filtre) varsa, bu enhancer'ı geçersiz kılmak için birkaç ek adım atmanız gerekebilir. Orijinal kayıt şöyle görünüyor:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

Bu, koruyucuyu "multi"-sağlayıcı olarak `APP_*` belirteci aracılığıyla kaydediyor. Bu noktada `JwtAuthGuard`'ı burada değiştirmek için, kayıt bu yuva içinde mevcut bir sağlayıcıyı kullanmalıdır:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useExisting: JwtAuthGuard,
    // ^^^^^^^^ 'useClass' yerine 'useExisting' kullanımına dikkat edin
  },
  JwtAuthGuard,
],
```

> info **İpucu** `useClass`'ı `useExisting` ile değiştirerek, Nest'in sağlayıcıyı belirteç arkasında yaratmak yerine kaydedilen bir sağlayıcıya referans yapmasını sağlayın.

Şimdi `JwtAuthGuard`, Nest'e, `TestingModule` oluşturulurken geçersiz kılınabilecek düzenli bir sağlayıcı olarak görünüyor:

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideProvider(JwtAuthGuard)
  .useClass(MockAuthGuard)
  .compile();
```

Artık tüm testleriniz her bir istekte `MockAuthGuard`'ı kullanacaktır.

#### Request Kapsamlı Örneklerin Test Edilmesi

[Request kapsamlı](/docs/fundamentals/injection-scopes) sağlayıcılar, her gelen **istek** için benzersiz bir şekilde oluşturulur. Örnek, istek işleme tamamlandıktan sonra çöp toplama tarafından temizlenir. Bu bir sorun oluşturur, çünkü test edilen bir istek için özel olarak oluşturulan bağımlılık enjeksiyon alt ağını erişemeyiz.

Yukarıdaki bölümlere dayanarak, `resolve()` yönteminin dinamik olarak oluşturulan bir sınıfı almak için kullanılabileceğini biliyoruz. Ayrıca, <a href="https://docs.nestjs.com/fundamentals/module-ref#resolving-scoped-providers">burada</a> açıklandığı gibi, bir DI konteyner alt ağının yaşam döngüsünü kontrol etmek için benzersiz bir bağlam tanımlayabiliriz. Bu nasıl bir test bağlamında kullanılır?

Strateji, önceden bir bağlam tanımlayıcı oluşturmak ve Nest'in tüm gelen istekler için bu belirli kimliği kullanmaya zorlamaktır. Bu şekilde test edilen bir istek için oluşturulan örnekleri alabiliriz.

Bunu başarmak için, `ContextIdFactory` üzerinde `jest.spyOn()` kullanın:

```typescript
const contextId = ContextIdFactory.create();
jest.spyOn(ContextIdFactory, 'getByRequest').mockImplementation(() => contextId);
```

Şimdi `contextId`'yi kullanarak herhangi bir sonraki istek için oluşturulan tek bir DI konteyner alt ağını erişebiliriz.

```typescript
catsService = await moduleRef.resolve(CatsService, contextId);
```