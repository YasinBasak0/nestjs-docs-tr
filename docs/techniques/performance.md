### Performans (Fastify)

Nest varsayılan olarak [Express](https://expressjs.com/) çerçevesini kullanır. Daha önce belirtildiği gibi, Nest ayrıca [Fastify](https://github.com/fastify/fastify) gibi diğer kütüphanelerle uyumluluk sağlar. Nest, bir çerçeve adaptörü uygulayarak bu çerçeve bağımsızlığını elde eder ve bu adaptörün temel işlevi middleware'leri ve işleyicileri uygun kütüphane özgü uygulamalara yönlendirmektir.

> info **İpucu** Unutmayın ki bir çerçeve adaptörünün uygulanabilmesi için hedef kütüphanenin Express'teki gibi bir istek/yanıt boru işleme işlemesi gerekmektedir.

[Fastify](https://github.com/fastify/fastify), Express ile benzer bir şekilde tasarım sorunlarını çözdüğü için Nest için iyi bir alternatif çerçeve sunar. Ancak, fastify, Express'e göre neredeyse iki kat daha iyi benchmark sonuçları elde ederek çok daha **hızlı** bir performans sağlar. Peki, Nest neden Express'i varsayılan HTTP sağlayıcısı olarak kullanır? Sebep, Express'in yaygın bir şekilde kullanılması, iyi bilinmesi ve Nest kullanıcılarına kutudan çıkan bir dizi uyumlu middleware'in bulunmasıdır.

Ancak, Nest çerçeve bağımsızlığı sağladığından, kolayca aralarında geçiş yapabilirsiniz. Fastify, performansa çok yüksek değer veriyorsanız daha iyi bir seçenek olabilir. Fastify'i kullanmak için, bu bölümde gösterildiği gibi yerleşik `FastifyAdapter`'ı seçmeniz yeterlidir.

#### Kurulum

İlk olarak, gerekli paketi kurmamız gerekmektedir:

```bash
$ npm i --save @nestjs/platform-fastify
```

#### Adaptör

Fastify platformu kurulduğunda, `FastifyAdapter`'ı kullanabiliriz.

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter()
  );
  await app.listen(3000);
}
bootstrap();
```

Fastify, varsayılan olarak yalnızca `localhost 127.0.0.1` arabiriminde dinler ([daha fazla bilgi](https://www.fastify.io/docs/latest/Guides/Getting-Started/#your-first-server)). Diğer ana bilgisayarlar üzerinden bağlantıları kabul etmek istiyorsanız, `listen()` çağrısında `'0.0.0.0'`'ı belirtmelisiniz:

```typescript
async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter(),
  );
  await app.listen(3000, '0.0.0.0');
}
```

#### Platforma özgü paketler

`FastifyAdapter`'ı kullandığınızda, Nest Fastify'i **HTTP sağlayıcı** olarak kullanır. Bu, Express'e dayanan her bir tarife işlemi artık çalışmayabilir anlamına gelir. Bunun yerine, Fastify'e özgü paketleri kullanmalısınız.

#### Yönlendirme yanıtı

Fastify, yönlendirme yanıtlarını Express'ten biraz farklı işler. Fastify ile uygun bir yönlendirme yapmak için, durum kodunu ve URL'yi şu şekilde döndürmelisiniz:

```typescript
@Get()
index(@Res() res) {
  res.status(302).redirect('/login');
}
```

#### Fastify seçenekleri

Fastify yapıcısına seçenekleri `FastifyAdapter` yapıcısı aracılığıyla iletebilirsiniz. Örneğin:

```typescript
new FastifyAdapter({ logger: true });
```

#### Middleware

Middleware fonksiyonları, Fastify'nin sarmallarını değil, doğrudan `req` ve `res` nesnelerini alır. Bu, altında kullanılan `middie` paketinin çalışma şeklidir ve `fastify`'in bu [sayfasında](https://www.fastify.io/docs/latest/Reference/Middleware/) daha fazla bilgi bulabilirsiniz.

```typescript
@@filename(logger.middleware)
import { Injectable, NestMiddleware } from '@nestjs/common';
import

 { FastifyRequest, FastifyReply } from 'fastify';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: FastifyRequest['raw'], res: FastifyReply['raw'], next: () => void) {
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

#### Rota Yapılandırması

`@RouteConfig()` dekoratörü ile Fastify'in [rota yapılandırma](https://fastify.dev/docs/latest/Reference/Routes/#config) özelliğini kullanabilirsiniz.

```typescript
@RouteConfig({ output: 'merhaba dünya' })
@Get()
index(@Req() req) {
  return req.routeConfig.output;
}
```

#### Rota Kısıtlamaları

V10.3.0 sürümünden itibaren, `@nestjs/platform-fastify` `@RouteConstraints` dekoratörü ile Fastify'in [rota kısıtlamaları](https://fastify.dev/docs/latest/Reference/Routes/#constraints) özelliğini destekler.

```typescript
@RouteConstraints({ version: '1.2.x' })
newFeature() {
  return 'Bu sadece sürüm >= 1.2.x için çalışır';
}
```

> info **İpucu** `@RouteConfig()` ve `@RouteConstraints`, `@nestjs/platform-fastify` tarafından içeri aktarılır.

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/10-fastify) bulunabilir.