---
sidebar_position: 7
---

### İlk Adımlar

Bu makale serisinde, Nest'in **temel prensiplerini** öğreneceksiniz. Nest uygulamalarının temel yapı taşlarıyla tanışmak için, birçok temel düzeyde alanı kapsayan temel bir CRUD uygulaması oluşturacağız.

#### Dil

[TypeScript'e](https://www.typescriptlang.org/) aşığız, ancak her şeyden önce - [Node.js'i](https://nodejs.org/en/) seviyoruz. Bu yüzden Nest, TypeScript ve **saf JavaScript** ile uyumludur. Nest, en son dil özelliklerinden faydalanır, bu nedenle bunu saf JavaScript ile kullanmak için bir [Babel](https://babeljs.io/) derleyicisine ihtiyaç duyarız.

Genellikle örneklerimizde TypeScript'i kullanacağız, ancak kod örneklerini her zaman (her örnek için sağ üst köşedeki dil düğmesine tıklayarak) saf JavaScript sözdizimine geçirebilirsiniz.

#### Önkoşullar

Lütfen işletim sisteminizde [Node.js](https://nodejs.org) (sürüm >= 16) kurulu olduğundan emin olun.

#### Kurulum

Yeni bir proje oluşturmak, [Nest CLI](/docs/cli/overview) ile oldukça basittir. [npm](https://www.npmjs.com/) kuruluysa, aşağıdaki komutları OS terminalinizde kullanarak yeni bir Nest projesi oluşturabilirsiniz:

```bash
$ npm i -g @nestjs/cli
$ nest new proje-adı
```

> info **İpucu** Yeni bir proje oluşturmak için TypeScript'in [daha sıkı](https://www.typescriptlang.org/tsconfig#strict) özellik setini kullanmak için `nest new` komutuna `--strict` bayrağını geçirin.

`proje-adı` dizini oluşturulacak, node modülleri ve birkaç diğer şablon dosyası yüklenecek ve `src/` dizini oluşturulacak ve birkaç temel dosya ile doldurulacaktır.

<div class="file-tree">
  <div class="item">src</div>
  <div class="children">
    <div class="item">app.controller.spec.ts</div>
    <div class="item">app.controller.ts</div>
    <div class="item">app.module.ts</div>
    <div class="item">app.service.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

İşte bu temel dosyaların kısa bir genel bakışı:

|                          |                                                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `app.controller.ts`      | Tek bir rota içeren temel bir denetleyici.                                                                             |
| `app.controller.spec.ts` | Denetleyici için birim testler.                                                                                  |
| `app.module.ts`          | Uygulamanın kök modülü.                                                                                 |
| `app.service.ts`         | Tek bir yöntem içeren temel bir servis.                                                                               |
| `main.ts`                | Uygulamanın giriş dosyası, bir Nest uygulama örneği oluşturmak için çekirdek `NestFactory` işlevini kullanır. |

`main.ts` dosyası, uygulamamızı **başlatmak** için kullanılacak asenkron bir işlev içerir:

```typescript
@@filename(main)

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

Nest uygulama örneği oluşturmak için çekirdek `NestFactory` sınıfını kullanıyoruz. `NestFactory`, bir uygulama örneği oluşturmayı sağlayan birkaç statik yöntem sunar. `create()` yöntemi, bir uygulama nesnesini döndürür ve bu nesne `INestApplication` arabirimini karşılar. Bu nesne, gelen HTTP isteklerini bekleyen uygulama nesnesi sağlayan bir dizi yöntem sunar. Yukarıdaki `main.ts` örneğinde, sadece HTTP dinleyicimizi başlatıyoruz ve bu, uygulamanın gelen HTTP isteklerini beklemesine olanak tanır.

Unutulmamalıdır ki, Nest CLI ile çatılandırılmış bir proje, her modülü kendi özel dizininde tutma konvansiyonunu takip etmeye teşvik eden başlangıçtaki bir proje yapısı oluşturur.

> info **İpucu** Varsayılan olarak, uygulama oluşturulurken herhangi bir hata oluşursa uygulamanız, kod `1` ile çıkacaktır. Bir hata fırlatmasını sağlamak istiyorsanız, bu seçeneği devre dışı bırakın `abortOnError` (örneğin, `NestFactory.create(AppModule, {{ '{' }} abortOnError: false {{ '}' }})`).

<app-banner-courses></app-banner-courses>

#### Platform

Nest, platformdan bağımsız bir çerçeve olmayı amaçlamaktadır. Platform bağımsızlığı, geliştiricilerin çeşitli uygulama türlerinde avantaj sağlayabileceği yeniden kullanılabilir mantıksal parçalar oluşturmasını mümkün kılar. Teknik olarak, Nest, bir adaptör oluşturulduğunda herhangi bir Node HTTP çerç

evesiyle çalışabilir. Out-of-the-box olarak desteklenen iki HTTP platformu vardır: [express](https://expressjs.com/) ve [fastify](https://www.fastify.io). İhtiyacınıza en uygun olanı seçebilirsiniz.

|                    |                                                                                                                                                                                                                                                                                                                                    |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `platform-express` | [Express](https://expressjs.com/), node için iyi bilinen minimalist bir web çerçevesidir. Bu, topluluk tarafından uygulanan birçok kaynakla test edilmiş, üretime hazır bir kütüphanedir. `@nestjs/platform-express` paketi varsayılan olarak kullanılır. Birçok kullanıcı, Express ile iyi hizmet alır ve hiçbir şey yapmadan bu özelliği etkinleştirmelerine gerek yoktur. |
| `platform-fastify` | [Fastify](https://www.fastify.io/), maksimum verimlilik ve hız sağlama odaklı yüksek performanslı ve düşük başlık çerçevesidir. Nasıl kullanılacağını [buradan](/docs/techniques/performance) okuyun.                                                                                                                                  |

Kullanılan platform her ne olursa olsun, kendi uygulama arabirimini açığa çıkarır. Bunlar sırasıyla `NestExpressApplication` ve `NestFastifyApplication` olarak görülür.

Aşağıdaki örnekte olduğu gibi `NestFactory.create()` yöntemine bir tür geçirildiğinde, `app` nesnesi yalnızca o özel platform için kullanılabilir yöntemlere sahip olacaktır. Ancak, temel platform API'sine gerçekten erişmek istemiyorsanız bir tür **belirtmenize gerek yoktur**.

```typescript
const app = await NestFactory.create<NestExpressApplication>(AppModule);
```

#### Uygulamayı Çalıştırma

Kurulum işlemi tamamlandığında, gelen HTTP isteklerini dinlemek üzere uygulamayı başlatmak için OS komut istemine aşağıdaki komutu çalıştırabilirsiniz:

```bash
$ npm run start
```

> info **İpucu** Geliştirme sürecini hızlandırmak için (x20 kat daha hızlı derlemeler), `npm run start` betiğine `-b swc` bayrağını geçirerek [SWC derleyici](/docs/recipes/swc) kullanabilirsiniz, örneğin `npm run start -- -b swc`.

Bu komut, HTTP sunucusunun `src/main.ts` dosyasında tanımlanan portta dinlemeye başlanan uygulamayı başlatır. Uygulama çalıştığında, tarayıcınızı açın ve `http://localhost:3000/` adresine gidin. "Hello World!" mesajını görmelisiniz.

Dosyalarınızı izlemek için aşağıdaki komutu kullanarak uygulamayı başlatabilirsiniz:

```bash
$ npm run start:dev
```

Bu komut, dosyalarınızı izleyecek, otomatik olarak derleyecek ve sunucuyu yeniden yükleyecektir.

#### Lintleme ve Biçimlendirme

[CLI](/docs/cli/overview), ölçeklenebilir bir geliştirme iş akışı oluşturmak için en iyi çabayı sağlar. Bu nedenle, bir Nest projesi hem bir kod **linter** hem de **biçimleyici** içerir (sırasıyla [eslint](https://eslint.org/) ve [prettier](https://prettier.io/)).

> info **İpucu** Biçimleyicilerin linterlardan ne farklı olduğunu mu bilmiyorsunuz? Farkı [buradan](https://prettier.io/docs/en/comparison.html) öğrenin.

Maksimum kararlılık ve genişletilebilirlik sağlamak için temel [`eslint`](https://www.npmjs.com/package/eslint) ve [`prettier`](https://www.npmjs.com/package/prettier) cli paketlerini kullanıyoruz. Bu kurulum, tasarım açısından resmi uzantılarla düzgün bir IDE entegrasyonuna izin verir.

IDE ile entegrasyon, uygun olduğu tasarım uzantılarıyla sağlanır.

Başlıksız ortamlarda bir IDE'nin ilgili olmadığı durumlarda (Sürekli Entegrasyon, Git kancaları vb.), bir Nest projesi hazır `npm` komut dosyaları ile gelir.

```bash
# Eslint ile lint ve otomatik düzeltme yapın
$ npm run lint

# Prettier ile biçimlendirme yapın
$ npm run format
```
