### Çerezler (Cookies)

**HTTP çerezi (cookie)**, kullanıcının tarayıcısı tarafından depolanan küçük bir veri parçasıdır. Çerezler, web sitelerinin durum bilgisini hatırlaması için güvenilir bir mekanizma oluşturmak amacıyla tasarlanmıştır. Kullanıcı web sitesini tekrar ziyaret ettiğinde, çerez otomatik olarak istekle birlikte gönderilir.

#### Express ile Kullanım (varsayılan)

İlk olarak, [gerekli paketi](https://github.com/expressjs/cookie-parser) ve TypeScript kullanıcıları için türlerini yükleyin:

```shell
$ npm i cookie-parser
$ npm i -D @types/cookie-parser
```

Kurulum tamamlandıktan sonra, `cookie-parser` ara yazılımını genel ara yazılım olarak uygulayın (örneğin, `main.ts` dosyanızda).

```typescript
import * as cookieParser from 'cookie-parser';
// başlatma dosyanızda
app.use(cookieParser());
```

`cookieParser` ara yazılımına birkaç seçenek geçirebilirsiniz:

- `secret`: Çerezleri imzalamak için kullanılan bir dizedir veya dizedir. Bu isteğe bağlıdır ve belirtilmezse imzalı çerezleri ayrıştırmaz. Dize sağlanırsa, bu, gizli olarak kullanılır. Dizi sağlanırsa, her sırada bir çözme girişimi yapılır.
- `options`: Bu, ikinci seçenek olarak `cookie.parse`'a iletilen bir nesnedir. Daha fazla bilgi için [cookie](https://www.npmjs.org/package/cookie) adresine bakın.

Ara yazılım, isteğin `Cookie` başlığını ayrıştırır ve çerez verilerini `req.cookies` özelliği olarak ve eğer bir sır verildiyse `req.signedCookies` özelliği olarak ortaya çıkarır. Bu özellikler, çerez adı değer çiftleri olarak çerez adını çerez değeriyle eşleştirir.

Bir sır verildiğinde, bu modül, imzalanmış çerez değerlerini çözer ve doğrular ve bu çiftleri `req.cookies`'den `req.signedCookies`'e taşır. İmzalı çerez, değeri `s:` ile başlayan bir çerezdir. İmza doğrulamasında başarısız olan imzalı çerezlerin değeri, bozulmuş değerin yerine `false` olacaktır.

Bu ayarla, şimdi rotada çerezleri şu şekilde okuyabilirsiniz:

```typescript
@Get()
findAll(@Req() request: Request) {
  console.log(request.cookies); // veya "request.cookies['cookieKey']"
  // veya console.log(request.signedCookies);
}
```

> info **Hint** `@Req()` dekoratörü `@nestjs/common` paketinden alınırken, `Request` `express` paketinden alınır.

Bir çereziyi çıkan bir yanıtla iliştirmek için, `Response#cookie()` yöntemini kullanın:

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: Response) {
  response.cookie('key', 'value')
}
```

> warning **Uyarı** Yanıt işleme mantığını çerçeveye bırakmak istiyorsanız, yukarıda gösterildiği gibi `passthrough` seçeneğini `true` olarak ayarlamayı unutmayın. Daha fazla bilgi için [buraya](/docs/controllers#library-specific-approach) bakın.

> info **Hint** `@Res()` dekoratörü `@nestjs/common` paketinden alınırken, `Response` `express` paketinden alınır.

#### Fastify ile Kullanım

İlk olarak, gerekli paketi yükleyin:

```shell
$ npm i @fastify/cookie
```

Kurulum tamamlandıktan sonra, `@fastify/cookie` eklentisini kaydedin:

```typescript
import fastifyCookie from '@fastify/cookie';

// başlatma dosyanızda
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
await app.register(fastifyCookie, {
  secret: 'my-secret', // çerezlerin imzası için
});
```

Bu ayarla, şimdi rotada çerezleri şu şekilde okuyabilirsiniz:

```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  console.log(request.cookies); // veya "request.cookies['cookieKey']"
}
```

> info **Hint** `@Req()` dekoratörü `@nestjs/common` paketinden alınırken, `FastifyRequest` `fastify` paketinden alınır.

Bir çereziyi çıkan bir yanıtla iliştirmek için, `FastifyReply#setCookie()` yöntemini kullanın:

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: FastifyReply) {
  response.setCookie('key', 'value')
}
```

`FastifyReply#setCookie()` yöntemi hakkında daha fazla bilgi için bu [sayfaya](https://github.com/fastify/fastify-cookie#sending) bakın.

> warning **Uyarı** Yanıt işleme mantığını çerçeveye bırakmak istiyorsanız, yukarıda gösterildiği gibi `passthrough` seçeneğini `true` olarak ayarlamayı unutmayın. Daha fazla bilgi için [buraya](/docs/controllers#library-specific-approach) bakın.

> info **Hint** `@Res()` dekoratörü `@nestjs/common` paketinden alınırken, `FastifyReply` `fastify` paketinden alınır.

#### Özel Bir Dekoratör Oluşturma (çoklu platform)

Gelen çerezlere erişmek için pratik, açıklayıcı bir yol sağlamak için bir [özel dekoratör](/docs/custom-decorators) oluşturabiliriz.

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const Cookies = createParamDecorator((data: string, ctx: ExecutionContext) => {
  const

 request = ctx.switchToHttp().getRequest();
  return data ? request.cookies?.[data] : request.cookies;
});
```

`@Cookies()` dekoratörü, `req.cookies` nesnesinden tüm çerezleri veya adlandırılmış bir çerezi çıkartacak ve dekore edilmiş parametreyi bu değerle dolduracaktır.

Bu ayarla, dekoratörü bir rota işleyicisinin imzasında şu şekilde kullanabiliriz:

```typescript
@Get()
findAll(@Cookies('name') name: string) {}
```
