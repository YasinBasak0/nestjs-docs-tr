### Oturum (Session)

**HTTP oturumları**, kullanıcı hakkındaki bilgileri birden çok istekte saklama yöntemi sağlar ve bu özellik [MVC](/docs/techniques/mvc) uygulamaları için özellikle faydalıdır.

#### Express ile Kullanım (varsayılan)

İlk olarak, [gerekli paketi](https://github.com/expressjs/session) (ve TypeScript kullanıcıları için türlerini) yükleyin:

```shell
$ npm i express-session
$ npm i -D @types/express-session
```

Kurulum tamamlandığında, `express-session` ara yazılımını genel ara yazılım olarak uygulayın (örneğin, `main.ts` dosyanızda).

```typescript
import * as session from 'express-session';
// başlatma dosyanızın bir yerinde
app.use(
  session({
    secret: 'my-secret',
    resave: false,
    saveUninitialized: false,
  }),
);
```

> warning **Uyarı** Varsayılan sunucu taraflı oturum depolama bilerek bir üretim ortamı için tasarlanmamıştır. Çoğu durumda bellek sızıntısına neden olur, tek bir sürecin ötesine ölçeklenmez ve genellikle hata ayıklama ve geliştirme için kullanılır. Daha fazla bilgi için [resmi depoya](https://github.com/expressjs/session) bakın.

`secret`, oturum kimliği çerezini imzalamak için kullanılır. Bu, tek bir sır için bir dize veya birden çok sır için bir dizi olabilir. Birden çok sır sağlanıyorsa, oturum kimliği çerezini imzalamak için yalnızca ilk öğe kullanılırken, tüm öğeler, isteklerde imzanın doğrulanması için dikkate alınır. Kendisi bir insan tarafından kolayca ayrıştırılamayacak ve en iyi rastgele karakter seti olan bir sır olmalıdır.

`resave` seçeneğini etkinleştirmek, oturumun istek sırasında değiştirilmemiş olsa bile oturumu oturum deposuna geri kaydetmeye zorlar. Varsayılan değer `true`'dır, ancak kullanımı artık önerilmiyor, çünkü varsayılan gelecekte değişecek.

Aynı şekilde, `saveUninitialized` seçeneğini etkinleştirmek, "başlatılmamış" bir oturumun depoya kaydedilmesine zorlar. Bir oturum yeni ama değiştirilmemiş olduğunda oturum başlatılmamıştır. `false` seçeneğini seçmek, giriş oturumları uygulamak, sunucu depolama kullanımını azaltmak veya bir çerez ayarlamadan önce izin gerektiren yasalara uymak için uygundur. `false` seçeneği aynı zamanda bir oturum olmadan birden çok paralel isteğin yapıldığı yarış koşullarına da yardımcı olacaktır ([kaynak](https://github.com/expressjs/session#saveuninitialized)).

`session` ara yazılımına başka birçok seçenek iletebilirsiniz, bunlar hakkında daha fazla bilgiyi [API belgelerinde](https://github.com/expressjs/session#options) bulabilirsiniz.

> info **İpucu** Lütfen `secure: true` seçeneğinin önerildiğini unutmayın. Ancak, güvenli çerezler için HTTPS gerekir. Eğer güvenli olarak ayarlanmışsa ve sitenize HTTP üzerinden erişiyorsanız, çerez ayarlanmaz. Node.js'nizi bir proxy arkasına koyduysanız ve `secure: true` kullanıyorsanız, express'te `"trust proxy"` ayarını yapmalısınız.

Bununla birlikte, şimdi rotada işleyiciler içinden oturum değerlerini ayarlayabilir ve okuyabilirsiniz, örneğin:

```typescript
@Get()
findAll(@Req() request: Request) {
  request.session.visits = request.session.visits ? request.session.visits + 1 : 1;
}
```

> info **İpucu** `@Req()` dekoratörü `@nestjs/common` tarafından içe aktarılır, `Request` ise `express` paketinden gelir.

Alternatif olarak, bir oturum nesnesini isteğinden çıkarmak için `@Session()` dekoratörünü kullanabilirsiniz, örneğin:

```typescript
@Get()
findAll(@Session() session: Record<string, any>) {
  session.visits = session.visits ? session.visits + 1 : 1;
}
```

> info **İpucu** `@Session()` dekoratörü `@nestjs/common` paketinden içe aktarılır.

#### Fastify ile Kullanım

İlk olarak, gerekli paketi yükleyin:

```shell
$ npm i @fastify/secure-session
```

Kurulum tamamlandığında, `fastify-secure-session` eklentisini kaydedin:

```typescript
import secureSession from '@fastify/secure-session';

// başlatma dosyanızın bir yerinde
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
await app.register(secureSession, {
  secret: 'averylogphrasebiggerthanthirtytwochars',
  salt: 'mq9hDxBVDbspDR6n',
});
```

> info **İpucu** Ayrıca bir anahtar üretebilir ([talimatları görmek için](https://github.com/fastify/fastify-secure-session)) veya [anahtar rotasyonunu](https://github.com/fastify/fastify-secure-session#using-keys-with-key-rotation) kullanabilirsiniz.

Kullanılabilir seçenekler hakkında daha fazla bilgiyi [resmi depoda](https://github.com/fastify/fastify-secure-session) bulabilirsiniz.

Bununla birlikte, şimdi rotada işleyiciler içinden oturum değerlerini ayarlay

abilir ve okuyabilirsiniz, örneğin:

```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  const visits = request.session.get('visits');
  request.session.set('visits', visits ? visits + 1 : 1);
}
```

Alternatif olarak, bir oturum nesnesini isteğinden çıkarmak için `@Session()` dekoratörünü kullanabilirsiniz, örneğin:

```typescript
@Get()
findAll(@Session() session: secureSession.Session) {
  const visits = session.get('visits');
  session.set('visits', visits ? visits + 1 : 1);
}
```

> info **İpucu** `@Session()` dekoratörü `@nestjs/common` tarafından içe aktarılır, `secureSession.Session` paketi `@fastify/secure-session`'dan gelir (ithalat ifadesi: `import * as secureSession from '@fastify/secure-session'`).