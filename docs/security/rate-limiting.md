### Rate Limiting

Uygulamaları brute-force saldırılardan korumanın yaygın bir teknikleri **rate-limiting** (hız sınırlama) kullanmaktır. Başlamak için `@nestjs/throttler` paketini yüklemeniz gerekecek.

```bash
$ npm i --save @nestjs/throttler
```

Yükleme tamamlandıktan sonra `ThrottlerModule`, diğer Nest paketleri gibi `forRoot` veya `forRootAsync` yöntemleriyle yapılandırılabilir.

```typescript
@@filename(app.module)
@Module({
  imports: [
    ThrottlerModule.forRoot([{
      ttl: 60000,
      limit: 10,
    }]),
  ],
})
export class AppModule {}
```

Yukarıdaki örnek, uygulamanızın korunan rotaları için `ttl` (milisaniye cinsinden yaşam süresi) ve `limit` (ttl içindeki maksimum istek sayısı) için genel seçenekleri ayarlar.

Modül içe aktarıldıktan sonra, `ThrottlerGuard`'ı nasıl bağlamak istediğinizi seçebilirsiniz. [guards](https://docs.nestjs.com/guards) bölümünde belirtildiği gibi, `@SkipThrottle()` ve `@Throttle()` dekoratörlerini kullanarak hangi kapsamda kullanmak istediğinize karar verebilirsiniz. Örneğin, bu korumayı global olarak bağlamak istiyorsanız şu şekilde bir sağlayıcı ekleyebilirsiniz:

```typescript
{
  provide: APP_GUARD,
  useClass: ThrottlerGuard
}
```

#### Birden Fazla Throttler Tanımı

Bir saniyede 3'ten fazla çağrı yapma, 10 saniyede 20 çağrı yapma ve bir dakikada 100 çağrı yapma gibi birden fazla throttling tanımı yapmak istediğiniz durumlar olabilir. Bu durumda, adlandırılmış seçeneklerle bir dizi içinde tanımlarınızı yapabilir ve daha sonra bu seçeneklere referanslarla `@SkipThrottle()` ve `@Throttle()` dekoratörlerinde değişiklik yapabilirsiniz.

```typescript
@@filename(app.module)
@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,
        limit: 3,
      },
      {
        name: 'medium',
        ttl: 10000,
        limit: 20
      },
      {
        name: 'long',
        ttl: 60000,
        limit: 100
      }
    ]),
  ],
})
export class AppModule {}
```

#### Özelleştirme

Bir kontrolör veya genel olarak korumayı bağlamak istiyorsanız ancak bir veya daha fazla uç noktanızda rate limiting'i devre dışı bırakmak istiyorsanız, `@SkipThrottle()` dekoratörünü kullanabilirsiniz. Bu dekoratör, bir sınıf veya tek bir rotayı tamamen devre dışı bırakmak için kullanılabilir. `@SkipThrottle()` dekoratörü, istisna yapmak istediğiniz bir veya daha fazla controllerın genelini devre dışı bırakmak istediğiniz durumda kullanılabilir ve eğer birden fazla tanımlamanız varsa bunu throttler setine göre yapılandırabilirsiniz. Eğer bir nesne geçmezseniz, varsayılan olarak `{{ '{' }} default: true {{ '}' }}` kullanılır.

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {}
```

Bu `@SkipThrottle()` dekoratörü, bir rotayı veya bir sınıfı atlamak veya bir sınıfta atlanan bir rotayı atlamamak için kullanılabilir.

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {
  // Bu rota için rate limiting uygulanır.
  @SkipThrottle({ default: false })
  dontSkip() {
    return 'List users work with Rate limiting.';
  }
  // Bu rota rate limiting'i atlar.
  doSkip() {
    return 'List users work without Rate limiting.';
  }
}
```

Ayrıca, `@Throttle()` dekoratörü vardır ki bu, global modülde ayarlanan `limit` ve `ttl`'yi geçersiz kılmak için kullanılabilir. Bu dekoratör, bir sınıf veya bir işlev üzerinde kullanılabilir. Sürüm 5 ve sonrasında, dekoratör, adı throttler seti ile ilgili olan ve limit ve ttl anahtarları ile integer değerlere sahip bir nesne alır, kök modüle iletilen seçeneklere benzer. Eğer orijinal seçeneklerinizde bir isim belirlemediyseniz, `default` adını kullanın. Bu şekilde yapılandırmanız gerekmektedir:

```typescript
// Rate limiting ve süre için varsayılan yapılandırmayı geçersiz kılma.
@Throttle({ default: { limit: 3, ttl: 60000 } })
@Get()
findAll() {
  return "List users works with custom rate limiting.";
}
```

### Proxies

Eğer uygulamanız bir proxy sunucunun arkasında çalışıyorsa, belirli HTTP adaptör seçeneklerini kontrol edin ([express](http://expressjs.com/en/guide/behind-proxies.html) ve [fastify](https://www.fastify.io/docs/latest/Reference/Server/#trustproxy)) ve `trust proxy` seçeneğini etkinleştirin. Bu, `X-Forwarded-For` başlığından orijinal IP adresini almanıza izin verecek ve `req.ip` yerine başlıktan değeri çekmek için `getTracker()` metodunu geçersiz kılmanıza olanak tanıyacaktır. Aşağıdaki örnek, hem express hem de fastify ile çalışır:

```typescript
// throttler-behind-proxy.guard.ts
import { ThrottlerGuard } from '@nestjs/throttler';
import { Injectable } from '@nestjs/common';

@Injectable()
export class ThrottlerBehindProxyGuard extends ThrottlerGuard {
  protected async getTracker(req: Record<string, any>): Promise<string> {
    return req.ips.length ? req.ips[0] : req.ip; // IP çıkarma işlemini kendi ihtiyaçlarınıza uygun hale getirin
  }
}

// app.controller.ts
import { ThrottlerBehindProxyGuard } from './throttler-behind-proxy.guard';

@UseGuards(ThrottlerBehindProxyGuard)
```

> info **Hint** express için `req` İstek nesnesinin API'sini [buradan](https://expressjs.com/en/api.html#req.ips) ve fastify için [buradan](https://www.fastify.io/docs/latest/Reference/Request/) bulabilirsiniz.

#### Websockets

Bu modül websockets ile çalışabilir, ancak bazı sınıf uzantılarına ihtiyaç duyar. `ThrottlerGuard`'ı genişleterek ve `handleRequest` metodunu geçersiz kılabilirsiniz:

```typescript
@Injectable()
export class WsThrottlerGuard extends ThrottlerGuard {
  async handleRequest(context: ExecutionContext, limit: number, ttl: number, throttler: ThrottlerOptions): Promise<boolean> {
    const client = context.switchToWs().getClient();
    const ip = client._socket.remoteAddress;
    const key = this.generateKey(context, ip, throttler.name);
    const { totalHits } = await this.storageService.increment(key, ttl);

    if (totalHits > limit) {
      throw new ThrottlerException();
    }

    return true;
  }
}
```

> info **Hint** Eğer ws kullanıyorsanız, `_socket` yerine `conn` ile değiştirmeniz gerekmektedir.

WebSockets ile çalışırken akılda bulundurmanız gereken birkaç şey var:

- Guard `APP_GUARD` veya `app.useGlobalGuards()` ile kaydedilemez
- Bir limit ulaşıldığında, Nest `exception` etkinliği yayınlar, bu nedenle bu için bir dinleyici olduğundan emin olun

> info **Hint** Eğer `@nestjs/platform-ws` paketini kullanıyorsanız, `client._socket.remoteAddress`'ı kullanabilirsiniz.

### GraphQL

`ThrottlerGuard` ayrıca GraphQL istekleriyle çalışmak için kullanılabilir. Yine, guard genişletilebilir, ancak bu sefer `getRequestResponse` metodu geçersiz kılınacaktır:

```typescript
@Injectable()
export class GqlThrottlerGuard extends ThrottlerGuard {
  getRequestResponse(context: ExecutionContext) {
    const gqlCtx = GqlExecutionContext.create(context);
    const ctx = gqlCtx.getContext();
    return { req: ctx.req, res: ctx.res };
  }
}
```

#### Konfigürasyon

`ThrottlerModule`'ün seçenekler dizisine geçirilen nesne için aşağıdaki seçenekler geçerlidir:

<table>
  <tr>
    <td><code>name</code></td>
    <td>hangi throttler set'inin kullanıldığını içsel olarak takip etmek için isim. Geçilmezse varsayılan olarak `default`</td>
  </tr>
  <tr>
    <td><code>ttl</code></td>
    <td>her isteğin depolamada ne kadar süreyle kalacağının milisaniye cinsinden sayısı</td>
  </tr>
  <tr>
    <td><code>limit</code></td>
    <td>TTL sınırları içindeki maksimum istek sayısı</td>
  </tr>
  <tr>
    <td><code>ignoreUserAgents</code></td>
    <td>isteğin throttling'e tabi tutulmasında göz ardı edilecek kullanıcı aracı regular expressions dizisi</td>
  </tr>
  <tr>
    <td><code>skipIf</code></td>
    <td>Throttler mantığını kısaltmak için kullanılan, ExecutionContext'ı içine alan ve bir boolean döndüren bir işlev. `@SkipThrottler()` gibi, ancak isteğe dayalı</td>
  </tr>
</table>

Eğer depolama ayarları yapmanız gerekiyorsa veya yukarıdaki seçeneklerin bir kısmını daha genel bir anlamda, her throttler seti için uygulamak istiyorsanız, yukarıdaki tabloyu kullanarak bu seçenekleri `throttlers` anahtarını kullanarak geçebilirsiniz.

<table>
  <tr>
    <td><code>storage</code></td>
    <td>throttling'in izlenmesi gereken özel bir depolama servisi. <a href="/security/rate-limiting#storages">Buraya bakın.</a></td>
  </tr>
  <tr>
    <td><code>ignoreUserAgents</code></td>
    <td>isteğin throttling'e tabi tutulmasında göz ardı edilecek kullanıcı aracı regular expressions dizisi</td>
  </tr>
  <tr>
    <td><code>skipIf</code></td>
    <td>Throttler mantığını kısaltmak için kullanılan, ExecutionContext'ı içine alan ve bir boolean döndüren bir işlev. `@SkipThrottler()` gibi, ancak isteğe dayalı</td>
  </tr>
  <tr>
    <td><code>throttlers</code></td>
    <td>yukarıdaki tabloyu kullanarak tanımlanan bir throttler setleri dizisi</td>
  </tr>
</table>

#### Asenkron Konfigürasyon

Konfigürasyonunuzu senkron olarak almak yerine asenkron olarak almak isteyebilirsiniz. `forRootAsync()` yöntemini kullanabilirsiniz, bu yöntem bağımlılık enjeksiyonuna ve `async` yöntemlere izin verir.

Bir yaklaşım, bir fabrika işlevi kullanmaktır:

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => [
        {
          ttl: config.get('THROTTLE_TTL'),
          limit: config.get('THROTTLE_LIMIT'),
        },
      ],
    }),
  ],
})
export class AppModule {}
```

Ayrıca `useClass` sözdizimini de kullanabilirsiniz:

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      useClass: ThrottlerConfigService,
    }),
  ],
})
export class AppModule {}
```

Bu, `ThrottlerConfigService`'nin `ThrottlerOptionsFactory` arabirimini uyguladığı sürece mümkündür.

#### Depolama

Dahili depolama, isteklerin global seçenekler tarafından belirlenen TTL süresini geçene kadar izlenen bir bellek önbelleğidir. `ThrottlerModule`'ün `storage` seçeneğine kendi depolama seçeneğinizi ekleyebilirsiniz, yeter ki sınıf `ThrottlerStorage` arabirimini uygulasın.

Dağıtılmış sunucular için [Redis](https://github.com/kkoomen/nestjs-throttler-storage-redis) için topluluk depolama sağlayıcısını kullanabilir ve tek bir doğruluk kaynağına sahip olabilirsiniz.

> info **Not** `ThrottlerStorage` `@nestjs/throttler` tarafından içe aktarılabilir.

#### Zaman Yardımcıları

Zamanları daha okunabilir hale getirmek için birkaç yardımcı yöntem bulunmaktadır. Bunlar, doğrudan tanımlama yerine bunları kullanmayı tercih ediyorsanız kullanılabilir. `@nestjs/throttler` beş farklı yardımcıyı, `seconds`, `minutes`, `hours`, `days` ve `weeks`, dışa aktarır. Bunları kullanmak için sadece `seconds(5)` veya diğer yardımcılardan birini çağırın ve doğru milisaniye sayısı dönecektir.

#### Göç Rehberi

Çoğu kişi için seçeneklerinizi bir dizi içine almak yeterli olacaktır.

Özel bir depolama kullanıyorsanız, `ttl` ve `limit` değerlerinizi bir dizi içine almalı ve bunu seçenek nesnesinin `throttlers` özelliğine atamalısınız.

Herhangi bir `@ThrottleSkip()` artık bir nesne almalıdır, bu nesnenin `string: boolean` özellikleri olmalıdır. Dizeler, throttler'ların adlarıdır. Bir isminiz yoksa, altyapıda bunun altında kullanılacak olan `'default'` dizesini geçirin.

Herhangi bir `@Throttle()` dekoratörü artık adları throttler bağlamlarına (tekrar, ad yoksa `'default'`) ilişkin olan ve `limit` ve `ttl` anahtarlarına sahip nesnelerin değerlerini içeren bir nesne almalıdır.

> Uyarı **Önemli** `ttl` şimdi **milisaniye** cinsindendir. TTL'nizi okunabilirlik için saniye cinsinden tutmak istiyorsanız, bu paketten `seconds` yardımcısını kullanın. Bu, ttl'yi sadece milisaniye cinsine çarpar. 

Daha fazla bilgi için [Değişiklik Günlüğüne](https://github.com/nestjs/throttler/blob/master/CHANGELOG.md#500) bakın.