### Helmet

[Helmet](https://github.com/helmetjs/helmet), HTTP başlıklarını uygun bir şekilde ayarlayarak uygulamanızı bazı yaygın web güvenlik açıklarına karşı koruyabilir. Genellikle, Helmet sadece güvenlikle ilgili HTTP başlıklarını ayarlayan küçük ara yazılım fonksiyonlarının bir koleksiyonudur (daha fazla bilgi için [buraya](https://github.com/helmetjs/helmet#how-it-works) bakın).

> info **Hint** `helmet`'i global olarak uygulamadan önce veya kaydetmeden önce diğer `app.use()` çağrılarına veya bu çağrıları yapabilen yapılandırma fonksiyonlarına gelmelidir. Bu, temel platformun (Express veya Fastify) çalışma şekline bağlı olarak, ortaya çıkan sıranın önemli olduğu bir şekilde çalıştığı içindir. Eğer bir rotayı tanımladıktan sonra `helmet` veya `cors` gibi bir ara yazılımı kullanırsanız, bu yazılım yalnızca o rotaya uygulanmaz, yalnızca middleware, rotadan sonraki rotalara uygulanır.

#### Express ile Kullanım (varsayılan)

İlk olarak, gerekli paketi yükleyin.

```bash
$ npm i --save helmet
```

Yükleme tamamlandığında, bunu global bir ara yazılım olarak uygulayın.

```typescript
import helmet from 'helmet';
// başlatma dosyanızın herhangi bir yerinde
app.use(helmet());
```

> warning **Warning** `helmet`, `@apollo/server` (4.x) ve [Apollo Sandbox](https://docs.nestjs.com/graphql/quick-start#apollo-sandbox) ile kullanıldığında, Apollo Sandbox üzerinde [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) ile ilgili bir sorun olabilir. Bu sorunu çözmek için aşağıdaki gibi CSP'yi yapılandırın:
>
> ```typescript
> app.use(helmet({
>   crossOriginEmbedderPolicy: false,
>   contentSecurityPolicy: {
>     directives: {
>       imgSrc: [`'self'`, 'data:', 'apollo-server-landing-page.cdn.apollographql.com'],
>       scriptSrc: [`'self'`, `https: 'unsafe-inline'`],
>       manifestSrc: [`'self'`, 'apollo-server-landing-page.cdn.apollographql.com'],
>       frameSrc: [`'self'`, 'sandbox.embed.apollographql.com'],
>     },
>   },
> }));
> ```

#### Fastify ile Kullanım

Eğer `FastifyAdapter` kullanıyorsanız, [@fastify/helmet](https://github.com/fastify/fastify-helmet) paketini yükleyin:

```bash
$ npm i --save @fastify/helmet
```

[fastify-helmet](https://github.com/fastify/fastify-helmet) middleware olarak değil, bir [Fastify eklentisi](https://www.fastify.io/docs/latest/Reference/Plugins/) olarak kullanılmalıdır, yani `app.register()` kullanılarak:

```typescript
import helmet from '@fastify/helmet'
// başlatma dosyanızın herhangi bir yerinde, başka bir depolama eklentisinden sonra
await app.register(helmet)
```

> warning **Warning** `apollo-server-fastify` ve `@fastify/helmet` kullanırken, GraphQL oyun alanında [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) ile ilgili bir sorun olabilir, bu çakışmayı çözmek için CSP'yi aşağıdaki gibi yapılandırın:
>
> ```typescript
> await app.register(fastifyHelmet, {
>    contentSecurityPolicy: {
>      directives: {
>        defaultSrc: [`'self'`, 'unpkg.com'],
>        styleSrc: [
>          `'self'`,
>          `'unsafe-inline'`,
>          'cdn.jsdelivr.net',
>          'fonts.googleapis.com',
>          'unpkg.com',
>        ],
>        fontSrc: [`'self'`, 'fonts.gstatic.com', 'data:'],
>        imgSrc: [`'self'`, 'data:', 'cdn.jsdelivr.net'],
>        scriptSrc: [
>          `'self'`,
>          `https: 'unsafe-inline'`,
>          `cdn.jsdelivr.net`,
>          `'unsafe-eval'`,
>        ],
>      },
>    },
>  });
>
> // Eğer hiç CSP kullanmayacaksanız, şunu kullanabilirsiniz:
> await app.register(fastifyHelmet, {
>   contentSecurityPolicy: false,
> });
> ```