---
sidebar_position: 11
---

### Middleware

Middleware, bir rotaya yönlendirici işleyici önce **çağrılan** bir işlevdir. Middleware işlevleri, [request](https://expressjs.com/en/4x/api.html#req) ve [response](https://expressjs.com/en/4x/api.html#res) nesnelerine ve uygulamanın istek–yanıt döngüsündeki `next()` middleware işlevine erişim sağlar. **next** middleware işlevi genellikle `next` adındaki bir değişkenle gösterilir.

<figure><img src="/assets/Middlewares_1.png" /></figure>

Nest middleware'leri varsayılan olarak [express](https://expressjs.com/en/guide/using-middleware.html) middleware'leri ile eşdeğerdir. Aşağıdaki açıklama, resmi express belgelerinden middleware'lerin yeteneklerini açıklar:

<blockquote class="external">
  Middleware işlevleri şu görevleri gerçekleştirebilir:
  <ul>
    <li>herhangi bir kodu yürütebilir.</li>
    <li>isteği ve yanıt nesnelerini değiştirebilir.</li>
    <li>isteğin-yanıt döngüsünü sonlandırabilir.</li>
    <li>yığındaki bir sonraki middleware işlevini çağırabilir.</li>
    <li>geçerli middleware işlevi isteğin-yanıt döngüsünü sonlandırmazsa, kontrolü bir sonraki middleware işlevine geçirmek için <code>next()</code>'i çağırmalıdır. Aksi takdirde istek asılı kalır.</li>
  </ul>
</blockquote>

Özel Nest middleware'ini ya bir işlevde ya da `@Injectable()` dekoratörü ile bir sınıfta uygulayabilirsiniz. Sınıf, `NestMiddleware` arabirimini uygulamalıdır, ancak işlevin herhangi özel gereksinimi yoktur. Basit bir middleware özelliği uygulamaya sınıf yöntemi kullanarak başlayalım.

>  warning **Uyarı** `Express` ve `fastify`, middleware'leri farklı bir şekilde ele alır ve farklı yöntem imzaları sunar, daha fazla bilgi için [buraya](/docs/techniques/performance#middleware) bakın.


```typescript
@@filename(logger.middleware)
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class LoggerMiddleware {
  use(req, res, next) {
    console.log('Request...');
    next();
  }
}
```

#### Bağımlılık Enjeksiyonu

Nest middleware, Bağımlılık Enjeksiyonunu tamamen destekler. Sağlayıcılar ve denetleyicilerle olduğu gibi, aynı modül içinde mevcut olan **bağımlılıkları enjekte edebilirler**. Bu genellikle `constructor` aracılığıyla yapılır.

#### Middleware Uygulama

`@Module()` dekoratörü için middleware'ler için bir yer yoktur. Bunun yerine, modül sınıfının `configure()` yöntemini kullanarak onları kurarız. Middleware içeren modüllerin `NestModule` arabirimini uygulaması gerekir. `LoggerMiddleware`'yi `AppModule` düzeyinde kurmaya başlayalım.

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
@@switch
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
```

Yukarıdaki örnekte, daha önce `CatsController` içinde tanımlanan `/cats` rotası işleyicileri için `LoggerMiddleware`'yi kurduk. Middleware'yi belirli bir istek yöntemi için daha da sınırlayabiliriz, middleware'yi yapılandırırken `forRoutes()` yöntemine bir nesne geçirerek rotayı ve istek yöntemini içeren bir nesneyi iletebiliriz. Aşağıdaki örnekte, istenen istek yöntemini referans almak için `RequestMethod` enumunu içe aktardığımıza dikkat edin.

```typescript
@@filename(app.module)
import { Module, NestModule, RequestMethod, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
@@switch
import { Module, RequestMethod } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

> info **İpucu** `configure()` yöntemini `async/await` kullanarak asenkron hale getirebilirsiniz (örneğin, `configure()` yöntemi içindeki bir asenkron işlemin tamamlanmasını `await` edebilirsiniz).

> warning **Uyarı** `express` adaptörünü kullanırken, NestJS uygulaması varsayılan olarak `body-parser` paketinden `json` ve `urlencoded`'ı kaydeder. Bu,

 middleware'yi `MiddlewareConsumer` aracılığıyla özelleştirmek istiyorsanız, uygulamayı `NestFactory.create()` ile oluştururken `bodyParser` bayrağını `false` olarak ayarlamanız gerektiği anlamına gelir.

 #### Rota jokerleri

Desen tabanlı rotalar da desteklenir. Örneğin, yıldız işareti (**joker**) kullanılabilir ve herhangi bir karakter kombinasyonunu eşleştirir:

```typescript
forRoutes({ path: 'ab*cd', method: RequestMethod.ALL });
```

`'ab*cd'` rota yolu, `abcd`, `ab_cd`, `abecd` ve benzeri tüm kombinasyonları eşleştirir. Karakterler `?`, `+`, `*` ve `()` bir rota yolu içinde kullanılabilir ve bunlar düzenli ifadelerdeki karşılıklarının alt kümesidir. Kısa çizgi (`-`) ve nokta (`.`), dize tabanlı yollar tarafından kelime anlamında yorumlanır.

> warning **Uyarı** `fastify` paketi, artık joker yıldızları `*` desteklemeyen `path-to-regexp` paketinin en son sürümünü kullanır. Bunun yerine, parametreleri (örneğin, `(.*)`, `:splat*`) kullanmalısınız.

#### Middleware consumer

`MiddlewareConsumer`, yardımcı bir sınıftır. Middleware'leri yönetmek için bir dizi yerleşik yöntem sağlar. Hepsi [akıcı stilde](https://en.wikipedia.org/wiki/Fluent_interface) basitçe **zincirlenebilir**. `forRoutes()` yöntemi bir dize, birden çok dize, bir `RouteInfo` nesnesi, bir denetleyici sınıfı ve hatta birden çok denetleyici sınıfı alabilir. Çoğu durumda, muhtemelen virgülle ayrılmış bir liste olarak geçeceğiniz **denetleyiciler** listesini geçireceksinizdir. Aşağıda tek bir denetleyici ile bir örnek bulunmaktadır:

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
@@switch
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
```

> info **İpucu** `apply()` yöntemi ya tek bir middleware'yi alır ya da <a href="/middleware#multiple-middleware">çoklu middleware'leri</a> belirtmek için birden fazla argüman alabilir.

#### Rotaları hariç tutma

Bazen belirli rotaları middleware uygulamaktan **hariç tutmak** istiyoruz. Belirli rotaları hariç tutmak için `exclude()` yöntemini kullanabiliriz. Bu yöntem, hariç tutulacak rotaları tanımlayan tek bir dize, birden çok dize veya `RouteInfo` nesnesini alabilir, aşağıda gösterildiği gibi:

```typescript
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/(.*)',
  )
  .forRoutes(CatsController);
```

> info **İpucu** `exclude()` yöntemi, [path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters) paketini kullanarak joker parametreleri destekler.

Yukarıdaki örnekle, `LoggerMiddleware` `CatsController` içinde tanımlanan tüm rotalara bağlanacak, ancak `exclude()` yöntemine geçirilen üç rota hariç.

#### Fonksiyonel middleware

Kullandığımız `LoggerMiddleware` sınıfı oldukça basit. Üyeleri, ek yöntemleri veya bağımlılıkları yok. Neden bunu bir sınıf yerine basit bir işlev olarak tanımlamayalım? Aslında yapabiliriz. Bu tür middleware'e **fonksiyonel middleware** denir. Farkı göstermek için logger middleware'ini sınıf tabanlı değil de fonksiyonel bir middleware'e dönüştürelim:

```typescript
@@filename(logger.middleware)
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request...`);
  next();
};
@@switch
export function logger(req, res, next) {
  console.log(`Request...`);
  next();
};
```

Ve bunu `AppModule` içinde kullanalım:

```typescript
@@filename(app.module)
consumer
  .apply(logger)
  .forRoutes(CatsController);
```

> info **İpucu** Middleware'inizde herhangi bir bağımlılığa ihtiyaç duymuyorsa, daha basit olan **fonksiyonel middleware** alternatifini kullanmayı düşünün.

#### Birden çok middleware

Yukarıda belirtildiği gibi, sıralı olarak çalıştırılan birden çok middleware bağlamak için, `apply()` yöntemi içine virgülle ayrılmış bir liste sağlamak yeterlidir:

```typescript
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```

#### Global middleware

Eğer her kaydedilmiş rotaya bir seferde middleware bağlamak istiyorsak, `INestApplication` örneği tarafından sağlanan `use()` yöntemini kullanabiliriz:

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(3000);
```

> info **İpucu** Bir global middleware içinde DI konteynerine erişmek mümkün değildir. `app.use()` kullanırken [fonksiyonel middleware](middleware#functional-middleware) kullanabilirsiniz. Alternatif olarak, bir sınıf middleware'ini kullanabilir ve `AppModule` (veya başka bir modül) içinde `.forRoutes('*')` ile tüketebilirsiniz.
