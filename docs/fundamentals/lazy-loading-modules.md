### Tembel Yükleme (Lazy-loading) Modülleri

Varsayılan olarak, modüller çoğu durumda otomatik olarak yüklenir. Yani, uygulama yüklendiği anda, modüller de hemen yüklenir, bunlar anında gerekli olup olmadığına bakılmaksızın. Bu çoğu uygulama için uygun olsa da, başlatma gecikmesinin ("soğuk başlatma") kritik olduğu **serverless ortamında** çalışan uygulama/worker'lar için bir engel olabilir.

Tembel yükleme, özel bir serverless fonksiyon çağrısı tarafından gereken modülleri yükleyerek önyükleme süresini azaltmaya yardımcı olabilir. Ayrıca, serverless fonksiyonunun "ısındığında" diğer modülleri de asenkron olarak yükleyebilir ve ardından daha hızlı başlatma süresi için kaydırabilirsiniz (ertelenmiş modül kaydı).

> info **İpucu** Eğer **[Angular](https://angular.dev/)** çerçevesine aşina iseniz, önce "lazy-loading modules" terimini duymuş olabilirsiniz. Ancak bu teknik, Nest'te fonksiyonel olarak farklıdır, bu nedenle bunu benzer adlandırma kurallarını paylaşan tamamen farklı bir özellik olarak düşünün.

> warning **Uyarı** Tembel yüklenen modüller ve servisler, [yaşam döngüsü kancaları yöntemleri](https://docs.nestjs.com/fundamentals/lifecycle-events) tarafından çağrılmaz.

#### Başlangıç

Modülleri gerektiğinde yüklemek için, Nest `LazyModuleLoader` sınıfını sunar ve bu sınıfı normal şekilde bir sınıfa enjekte edebilirsiniz:

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}
}
@@switch
@Injectable()
@Dependencies(LazyModuleLoader)
export class CatsService {
  constructor(lazyModuleLoader) {
    this.lazyModuleLoader = lazyModuleLoader;
  }
}
```

> info **İpucu** `LazyModuleLoader` sınıfı, `@nestjs/core` paketinden içe aktarılır.

Alternatif olarak, uygulamanızın başlatma dosyasından (`main.ts`) `LazyModuleLoader` sağlayıcısına bir referans alabilirsiniz:

```typescript
// "app" bir Nest uygulama örneğini temsil eder
const lazyModuleLoader = app.get(LazyModuleLoader);
```

Bunu sağladıktan sonra, aşağıdaki yapıyı kullanarak herhangi bir modülü yükleyebilirsiniz:

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);
```

> info **İpucu** "Tembel yüklenen" modüller, ilk `LazyModuleLoader#load` yöntemi çağrısı sırasında **önbelleğe alınır**. Bu, `LazyModule`'i yüklemek için her ardışık girişim çok **hızlı** olacak ve modülü tekrar yüklemek yerine önbelleğe alınmış bir örneği döndürecektir.
>
> ```bash
> "LazyModule" Yükleme denemesi: 1
> süre: 2.379ms
> "LazyModule" Yükleme denemesi: 2
> süre: 0.294ms
> "LazyModule" Yükleme denemesi: 3
> süre: 0.303ms
> ```
>
> Ayrıca, "tembel yüklenen" modüller, uygulama başlatma anında statik olarak kaydedilen modüllerle aynı modül grafiği paylaşır ve uygulamanıza daha sonra kaydedilen herhangi bir tembel modülle de aynı grafikte yer alır.

Burada `lazy.module.ts` bir TypeScript dosyasıdır ve herhangi bir değişiklik yapmadan **düzenli bir Nest modülü** (reguler Nest module) döndürür.

`LazyModuleLoader#load` yöntemi, [modül referansını](/docs/fundamentals/module-reference) (`LazyModule`'in) döndürür ve bu, sağlayıcıların iç listesinde gezinmenizi ve bir arama anahtarı olarak enjeksiyon anahtarını kullanarak herhangi bir sağlayıcıya referans almanızı sağlar.

Örneğin, aşağıdaki tanıma sahip bir `LazyModule`'imiz olduğunu varsayalım:

```typescript
@Module({
  providers: [LazyService],
  exports: [LazyService],
})
export class LazyModule {}
```

> info **İpucu** Tembel yüklenen modüller **global modül** olarak kaydedilemez, çünkü bu basitçe mantıklı değildir (çünkü modüller lazımlıkla, zaten statik olarak kaydedilmiş modüllerin tümü anında örneklenmiş olduktan sonra gerektiğinde kaydedilir). Aynı şekilde, **global iyileştiricileri** (guard/interceptor gibi) **düzgün çalışmaz**.

Bununla birlikte, `LazyService` sağlayıcısına bir referans alabiliriz, örneğin:

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);

const { Lazy

Service } = await import('./lazy.service');
const lazyService = moduleRef.get(LazyService);
```

> warning **Uyarı** Eğer **Webpack** kullanıyorsanız, `tsconfig.json` dosyanızı güncellediğinizden emin olun - `compilerOptions.module`'i `"esnext"` olarak ayarlama ve `compilerOptions.moduleResolution` özelliğini `"node"` olarak ekleyin:
>
> ```json
> {
>   "compilerOptions": {
>     "module": "esnext",
>     "moduleResolution": "node",
>     ...
>   }
> }
> ```
>
> Bu seçenekleri ayarladığınızda, [kod bölmeleme](https://webpack.js.org/guides/code-splitting/) özelliğini kullanabilirsiniz.

#### Tembel yüklenen controller, gateway ve resolver'lar

Nest'te controller'lar (veya GraphQL uygulamalarındaki resolver'lar), rotaları/yolları/konuları temsil ettiği için, bu tür bir **tembel yükleme yapılamaz**.

> error **Uyarı** Tembel yüklenen modüller içinde kayıtlı controller'lar, [resolver'lar](/docs/graphql/resolvers-map) ve [gateway'ler](/docs/websockets/gateways) beklenildiği gibi davranmaz. Benzer şekilde, `MiddlewareConsumer` arayüzünü uygulayarak ortaya çıkan middleware fonksiyonlarını (middleware fonksiyonları) da talep üzerine kaydedemezsiniz.

Örneğin, bir HTTP uygulamasında Fastify sürücüsü kullanarak bir REST API (HTTP uygulaması) oluşturuyorsanız (`@nestjs/platform-fastify` paketini kullanarak), Fastify, uygulama başarılı bir şekilde dinlemeye başladıktan sonra rotaları kaydetme şansı vermez. Bu, modül kontrolcülerinde kaydedilen tüm tembel yüklenen rotaların erişilemez olacağı anlamına gelir, çünkü bunları çalışma zamanında kaydetme olanağı yoktur.

Aynı şekilde, `@nestjs/microservices` paketi içinde sağladığımız bazı taşıma stratejileri (Kafka, gRPC veya RabbitMQ dahil) belirli konulara/kanallara abone olmadan önce belirli konulara/kanallara abone olmamızı gerektirir. Uygulamanız mesaj dinlemeye başladığında, çerçeve yeni konulara abone olamaz.

Son olarak, `@nestjs/graphql` paketi kod tabanlı yaklaşım etkinleştirildiğinde, GraphQL şemasını metadata'ya dayanarak anında oluşturur. Bu, tüm sınıfların önceden yüklenmiş olması gerektiği anlamına gelir. Aksi takdirde, uygun, geçerli bir şema oluşturmak mümkün olmaz.

#### Yaygın kullanım durumları

Çoğu durumda, işçinizin/zamanlayıcınızın/lambda ve serverless fonksiyonunuzun/middleware'ınızın, giriş argümanlarına (route path/date/query parametreleri, vb.) dayalı olarak farklı servisleri (farklı mantıkları) tetiklemesi gereken durumları göreceksiniz. Öte yandan, tembel yükleme modülleri, başlangıç süresinin önemsiz olduğu monolitik uygulamalarda çok anlamlı olmayabilir.