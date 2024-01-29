### Raw body

Raw isteği gövdesine erişmenin en yaygın kullanım durumlarından biri webhook imza doğrulamalarını gerçekleştirmektir. Genellikle webhook imza doğrulamalarını gerçekleştirmek için HMAC özeti hesaplamak için ayrıştırılmamış istek gövdesine ihtiyaç duyulur.

> warning **Uyarı** Bu özellik, yerleşik global gövde ayrıştırıcı ara yazılım etkin olduğunda kullanılabilir, yani uygulama oluştururken `bodyParser: false` parametresini geçmemelisiniz.

#### Express ile Kullanım

İlk olarak, Nest Express uygulamanızı oluştururken seçeneği etkinleştirin:

```typescript
import { NestFactory } from '@nestjs/core';
import type { NestExpressApplication } from '@nestjs/platform-express';
import { AppModule } from './app.module';

// "bootstrap" fonksiyonunda
const app = await NestFactory.create<NestExpressApplication>(AppModule, {
  rawBody: true,
});
await app.listen(3000);
```

Bir denetleyicide raw istek gövdesine erişmek için, `RawBodyRequest` adında bir kolaylık arayüzü sağlanır ve isteği üzerinde bir `rawBody` alanını açığa çıkarır: `RawBodyRequest` arayüzü tipini kullanın:

```typescript
import { Controller, Post, RawBodyRequest, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
class CatsController {
  @Post()
  create(@Req() req: RawBodyRequest<Request>) {
    const raw = req.rawBody; // bir `Buffer` döndürür.
  }
}
```

#### Farklı bir ayrıştırıcıyı kaydetme

Varsayılan olarak, yalnızca `json` ve `urlencoded` ayrıştırıcıları kaydedilir. Farklı bir ayrıştırıcı kaydetmek istiyorsanız, bunu açıkça yapmanız gerekecektir.

Örneğin, bir `text` ayrıştırıcı kaydetmek için aşağıdaki kodu kullanabilirsiniz:

```typescript
app.useBodyParser('text');
```

> warning **Uyarı** `.useBodyParser` yönteminin bulunabilmesi için `NestFactory.create` çağrısına doğru uygulama türünü sağladığınızdan emin olun. Express uygulamaları için doğru tip `NestExpressApplication`'dır. Aksi takdirde `.useBodyParser` yöntemi bulunamaz.

#### Ayrıştırıcı boyutu sınırlaması

Uygulamanızın varsayılan `100kb`'dan daha büyük bir gövdeyi ayrıştırması gerekiyorsa, aşağıdakini kullanın:

```typescript
app.useBodyParser('json', { limit: '10mb' });
```

`.useBodyParser` yöntemi, uygulama seçeneklerine iletilen `rawBody` seçeneğini dikkate alacaktır.

#### Fastify ile Kullanım

İlk olarak, Nest Fastify uygulamanızı oluştururken seçeneği etkinleştirin:

```typescript
import { NestFactory } from '@nestjs/core';
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

// "bootstrap" fonksiyonunda
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
  {
    rawBody: true,
  },
);
await app.listen(3000);
```

Bir denetleyicide raw istek gövdesine erişmek için, `RawBodyRequest` adında bir kolaylık arayüzü sağlanır ve isteği üzerinde bir `rawBody` alanını açığa çıkarır: `RawBodyRequest` arayüzü tipini kullanın:

```typescript
import { Controller, Post, RawBodyRequest, Req } from '@nestjs/common';
import { FastifyRequest } from 'fastify';

@Controller('cats')
class CatsController {
  @Post()
  create(@Req() req: RawBodyRequest<FastifyRequest>) {
    const raw = req.rawBody; // bir `Buffer` döndürür.
  }
}
```

#### Farklı bir ayrıştırıcıyı kaydetme

Varsayılan olarak, yalnızca `application/json` ve `application/x-www-form-urlencoded` ayrıştırıcıları kaydedilir. Farklı bir ayrıştırıcı kaydetmek istiyorsanız, bunu açıkça yapmanız gerekecektir.

Örneğin, bir `text/plain` ayrıştırıcı kaydetmek için aşağıdaki kodu kullanabilirsiniz:

```typescript
app.useBodyParser('text/plain');
```

> warning **Uyarı** `.useBodyParser` yönteminin bulunabilmesi için `NestFactory.create` çağrısına doğru uygulama türünü sağladığınızdan emin olun. Fastify uygulamaları için doğru tip `NestFastifyApplication`'dır. Aksi takdirde `.useBodyParser` yöntemi bulunamaz.

#### Ayrıştırıcı boyutu sınırlaması

Uygulamanızın varsayılan 1MiB'ından daha büyük bir gövdeyi ayrıştırması gerekiyorsa, aşağıdakini kullanın:

```typescript
const bodyLimit = 10_485_760; // 10MiB
app.useBodyParser('application/json', { bodyLimit });
```

`.useBodyParser` yöntemi, uygulama seçeneklerine iletilen `rawBody` seçeneğini dikkate alacaktır.