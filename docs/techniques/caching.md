### Önbellekleme

Önbellekleme, uygulamanızın performansını artırmaya yardımcı olan harika ve basit bir **tekniktir**. Geçici bir veri deposu olarak hareket ederek yüksek performanslı veri erişimi sağlar.

#### Kurulum

İlk olarak gerekli paketleri kurun:

```bash
$ npm install @nestjs/cache-manager cache-manager
```

> warning **Uyarı** `cache-manager` sürüm 4, `TTL (Time-To-Live)` için saniyeleri kullanır. Şu anki `cache-manager` sürümü (v5), bunun yerine milisaniyeleri kullanmaya geçti. NestJS, değeri dönüştürmez ve sadece sağladığınız ttl'yi kütüphaneye iletir. Başka bir deyişle:
> * `cache-manager` v4 kullanılıyorsa, ttl'yi saniye cinsinden sağlayın
> * `cache-manager` v5 kullanılıyorsa, ttl'yi milisaniye cinsinden sağlayın
> * Belgeleme saniyeleri referans almaktadır, çünkü NestJS, `cache-manager`'ün sürüm 4'ünü hedefleyerek piyasaya sürülmüştür.

#### Bellek içi önbellek

Nest, çeşitli önbellek depolama sağlayıcıları için birleşik bir API sunar. Dahili olanı, bellek içi bir veri deposudur. Ancak, Redis gibi daha kapsamlı bir çözüme kolayca geçiş yapabilirsiniz.

Önbelleği etkinleştirmek için, `CacheModule`'ü içeri aktarın ve `register()` yöntemini çağırın.

```typescript
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { AppController } from './app.controller';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
})
export class AppModule {}
```

#### Önbellek deposuyla etkileşim

Önbellek yöneticisi örneğiyle etkileşimde bulunmak için, sınıfınıza `CACHE_MANAGER` belirteci kullanarak enjekte edin, aşağıdaki gibi:

```typescript
constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}
```

> info **İpucu** `Cache` sınıfı, `cache-manager` paketinden içe aktarılırken, `CACHE_MANAGER` belirteci `@nestjs/cache-manager` paketindendir.

`Cache` örneği üzerindeki `get` yöntemi ( `cache-manager` paketinden) öğeleri önbellekten almak için kullanılır. Öğe önbellekte mevcut değilse, `null` döner.

```typescript
const value = await this.cacheManager.get('key');
```

Bir öğeyi önbelleğe eklemek için, `set` yöntemini kullanın:

```typescript
await this.cacheManager.set('key', 'value');
```

Önbelleğin varsayılan süresi 5 saniyedir.

Bu belirli anahtar için elle bir TTL (saniye cinsinden son kullanma süresi) belirleyebilirsiniz, aşağıdaki gibi:

```typescript
await this.cacheManager.set('key', 'value', 1000);
```

Önbelleğin sona ermesini devre dışı bırakmak için, `ttl` yapılandırma özelliğini `0` olarak ayarlayın:

```typescript
await this.cacheManager.set('key', 'value', 0);
```

Bir öğeyi önbellekten kaldırmak için, `del` yöntemini kullanın:

```typescript
await this.cacheManager.del('key');
```

Tüm önbelleği temizlemek için, `reset` yöntemini kullanın:

```typescript
await this.cacheManager.reset();
```

#### Yanıtları Otomatik Önbellekleme

> warning **Uyarı** [GraphQL](/docs/graphql/quick-start) uygulamalarında, interceptor'lar her alan çözücü için ayrı ayrı çalıştırılır. Bu nedenle, yanıtları önbelleğe alan `CacheModule` (ki bu, yanıtları önbelleğe almak için interceptor'ları kullanır) doğru bir şekilde çalışmaz.

Yanıtları otomatik olarak önbellekleme özelliğini etkinleştirmek için, `CacheInterceptor`'u önbelleği nereye bağlamak istiyorsanız bağlayın.

```typescript
@Controller()
@UseInterceptors(CacheInterceptor)
export class AppController {
  @Get()
  findAll(): string[] {
    return [];
  }
}
```

> warning **Uyarı** Sadece `GET` uç noktaları önbelleğe alınır. Ayrıca, HTTP sunucusu rotaları (`@Res()`) tarafından yerel yanıt nesnesini enjekte eden HTTP sunucusu rotaları, Önbellek İnterceptor'ını kullanamaz. Daha fazla ayrıntı için
> <a href="https://docs.nestjs.com/interceptors#response-mapping">yanıt eşleme</a>'ye bakın.

Gerekli belirliğin miktarını azaltmak için, `CacheInterceptor`'u tüm uç noktalara global olarak bağlayabilirsiniz:

```typescript
import { Module } from '@nestjs/common';
import { CacheModule, CacheInterceptor } from '@nestjs/cache-manager';
import { AppController } from './app.controller';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: CacheInterceptor,
    },
  ],
})
export class AppModule {}
```

#### Önbelleği Özelleştirme

Tüm önbelleklenen verilerin kendi son kullanma süresi ([TTL](https://en.wikipedia.org/wiki/Time_to_live)) vardır. Varsayılan değerleri özelleştirmek için, `register()` yöntemine seçenekleri içeren bir nesne iletilmelidir.

```typescript
CacheModule.register({
  ttl: 5, // saniye
  max: 10, // önbellekteki maksimum öğe sayısı
});
```

#### Modülü global olarak kullanma

`CacheModule`'ü diğer modüllerde kullanmak istediğinizde, onu içeri aktarmanız gerekir (herhangi bir Nest modülüyle olduğu gibi). Alternatif olarak, seçenek nesnesinin `isGlobal` özelliğini `true` olarak ayarlayarak [global bir modül](https://docs.nestjs.com/modules#global-modules) olarak bildirebilirsiniz. Bu durumda, `CacheModule`'ü kök modülde (örneğin, `AppModule`'de) yüklendikten sonra diğer modüllerde içe aktarmaya ihtiyaç duymazsınız.

```typescript
CacheModule.register({
  isGlobal: true,
});
```

#### Global önbellek geçersiz kılma

Global önbellek etkin olduğunda, önbellek girişleri, rota yoluna dayalı olarak otomatik olarak oluşturulan bir `CacheKey` altında depolanır. Belirli denetleyici yöntemleri için önbellek ayarlarını (`@CacheKey()` ve `@CacheTTL()`) geçersiz kılabilir ve böylece belirli denetleyici yöntemleri için özelleştirilmiş önbellek stratejilerine izin verebilirsiniz. Bu, [farklı önbellek depolarını](https://docs.nestjs.com/techniques/caching#different-stores) kullanırken özellikle ilgili olabilir.

```typescript
@Controller()
export class AppController {
  @CacheKey('custom_key')
  @CacheTTL(20)
  findAll(): string[] {
    return [];
  }
}
```

> info **İpucu** `@CacheKey()` ve `@CacheTTL()` dekoratörleri, `@nestjs/cache-manager` paketinden içe aktarılır.

`@CacheKey()` dekoratörü, bir karşılık `@CacheTTL()` dekoratörü ile birlikte veya olmadan kullanılabilir ve tam tersi. Yalnızca `@CacheKey()`'i veya yalnızca `@CacheTTL()`'yi geçersiz kılma seçebilirsiniz. Bir dekoratör ile geçersiz kılınmayan ayarlar, global olarak kaydedilen varsayılan değerleri kullanacaktır (bkz. [Önbelleği Özelleştirme](https://docs.nestjs.com/techniques/caching#customize-caching)).

#### WebSockets ve Microservices

`CacheInterceptor`'ı WebSocket abonelerine ve Microservice'in desenlerine (kullanılan taşıma yönteminden bağımsız olarak) uygulayabilirsiniz.

```typescript
@@filename()
@CacheKey('events')
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
@@switch
@CacheKey('events')
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client, data) {
  return [];
}
```

Ancak, önbelleklenmiş verileri depolamak ve almak için kullanılacak bir anahtar belirtmek için ek `@CacheKey()` dekoratörü gereklidir. Ayrıca, **her şeyi önbelleğe almamalısınız**. İşlemler, yalnızca verileri sorgulamak yerine iş işlemleri gerçekleştiren eylemler önbelleğe alınmamalıdır.

Ek olarak, önbellek sona erme süresini (TTL) belirli bir anahtar kullanarak aşağıdaki gibi geçersiz kılabilirsiniz. Bu, global varsayılan TTL değerini geçersiz kılacaktır.

```typescript
@@filename()
@CacheTTL(10)
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
@@switch
@CacheTTL(10)
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client, data) {
  return [];
}
```

> info **İpucu** `@CacheTTL()` dekoratörü, bir karşılık `@CacheKey()` dekoratörü ile birlikte veya olmadan kullanılabilir.

#### İzlemeyi Ayarlama

Nest, varsayılan olarak, önbellek kayıtlarınızı uç noktalarınızla ilişkilendirmek için istek URL'sini (HTTP uygulamasında) veya önbellek anahtarını (web soketleri ve mikroservis uygulamalarında, `@CacheKey()` dekoratörü aracılığıyla ayarlanır) kullanır. Ancak bazen farklı faktörlere dayalı izleme kurmak isteyebilirsiniz, örneğin, HTTP başlıklarını (`profile` uç noktalarını doğru bir şekilde tanımlamak için `Authorization` gibi) kullanarak.

Bunu başarmak için, `CacheInterceptor` sınıfını alt sınıf olarak oluşturun ve `trackBy()` yöntemini geçersiz kılın.

```typescript
@Injectable()
class HttpCacheInterceptor extends CacheInterceptor {
  trackBy(context: ExecutionContext): string | undefined {
    return 'key';
  }
}
```

#### Farklı Depolar

Bu servis, [cache-manager](https://github.com/node-cache-manager/node-cache-manager) paketinin altında çalışır. `cache-manager` paketi bir dizi kullanışlı depoyu destekler, örneğin, [Redis deposu](https://github.com/dabroek/node-cache-manager-redis-store). Desteklenen tüm depoların tam listesi [burada](https://github.com/node-cache-manager/node-cache-manager#store-engines) bulunabilir. Redis deposunu kurmak için, ilgili seçeneklerle birlikte paketi `register()` yöntemine iletmek yeterlidir.

```typescript
import type { RedisClientOptions } from 'redis';
import * as redisStore from 'cache-manager-redis-store';
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { AppController } from './app.controller';

@Module({
  imports: [
    CacheModule.register<RedisClientOptions>({
      store: redisStore,

      // Mağaza özgün yapılandırma:
      host: 'localhost',
      port: 6379,
    }),
  ],
  controllers: [AppController],
})
export class AppModule {}
```

> uyarı**Uyarı** `cache-manager-redis-store`, redis v4'ü desteklemez. `ClientOpts` arabiriminin var olması ve doğru çalışabilmesi için
> en son `redis` 3.x.x ana sürümünü yüklemeniz gerekir. Bu yükseltmenin ilerlemesini takip etmek için bu [soruna](https://github.com/dabroek/node-cache-manager-redis-store/issues/40) göz atın.

#### Async Yapılandırma

Modül seçeneklerini derleme zamanında statik olarak iletmek yerine, bunları asenkron olarak iletmek isteyebilirsiniz. Bu durumda, `registerAsync()` yöntemini kullanın, bu yöntem asenkron yapılandırmayla başa çıkmanın birkaç yolunu sağlar.

Bir yaklaşım, bir fabrika fonksiyonu kullanmaktır:

```typescript
CacheModule.registerAsync({
  useFactory: () => ({
    ttl: 5,
  }),
});
```

Fabrikamız, diğer tüm asenkron modül fabrikaları gibi davranır (asenkron olabilir ve bağımlılıkları `inject` aracılığıyla enjekte edebilir).

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    ttl: configService.get('CACHE_TTL'),
  }),
  inject: [ConfigService],
});
```

Alternatif olarak, `useClass` yöntemini kullanabilirsiniz:

```typescript
CacheModule.registerAsync({
  useClass: CacheConfigService,
});
```

Yukarıdaki yapı, `CacheModule` içinde `CacheConfigService`'yi örnekleyecektir ve seçenekler nesnesini almak için onu kullanacaktır. `CacheConfigService`, yapılandırma seçeneklerini sağlamak için `CacheOptionsFactory` arabirimini uygulamak zorundadır:

```typescript
@Injectable()
class CacheConfigService implements CacheOptionsFactory {
  createCacheOptions(): CacheModuleOptions {
    return {
      ttl: 5,
    };
  }
}
```

Mevcut bir yapılandırma sağlayıcısını kullanmak istiyorsanız, `useExisting` sözdizimini kullanın:

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

Bu, yalnızca bir kritik farkla çalışır - `CacheModule`, `ConfigService`'i kendi örneklemelerini oluşturmak yerine yeniden kullanmak için içeri aktarılan modülleri arayacaktır.

> info **İpucu** `CacheModule#register` ve `CacheModule#registerAsync` ve `CacheOptionsFactory`'nin seçmeli bir generic (tip argümanı) vardır, bu da depo özgü yapılandırma seçeneklerini daraltmak için, tip güvenliğini sağlar.

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/20-cache) bulunabilir.