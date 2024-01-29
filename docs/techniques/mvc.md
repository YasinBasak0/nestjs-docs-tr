### Model-View-Controller (MVC) ile Nest

Nest, varsayılan olarak [Express](https://github.com/expressjs/express) kütüphanesini kullanır. Bu nedenle, Express'te MVC (Model-View-Controller) deseni kullanmak için kullanılan her teknik, Nest uygulamalarında da geçerlidir.

İlk olarak, [CLI](https://github.com/nestjs/nest-cli) aracını kullanarak basit bir Nest uygulaması oluşturalım:

```bash
$ npm i -g @nestjs/cli
$ nest new proje
```

MVC uygulaması oluşturmak için HTML görünümlerimizi işlemek için bir [template motoru](https://expressjs.com/en/guide/using-template-engines.html) da gerekir:

```bash
$ npm install --save hbs
```

Biz `hbs` ([Handlebars](https://github.com/pillarjs/hbs#readme)) motorunu kullandık, ancak ihtiyaçlarınıza uygun olanı kullanabilirsiniz. Kurulum işlemi tamamlandığında, Express örneğini aşağıdaki kodu kullanarak yapılandırmamız gerekiyor:

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { NestExpressApplication } from '@nestjs/platform-express';
import { join } from 'path';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(
    AppModule,
  );

  app.useStaticAssets(join(__dirname, '..', 'public'));
  app.setBaseViewsDir(join(__dirname, '..', 'views'));
  app.setViewEngine('hbs');

  await app.listen(3000);
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { join } from 'path';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(
    AppModule,
  );

  app.useStaticAssets(join(__dirname, '..', 'public'));
  app.setBaseViewsDir(join(__dirname, '..', 'views'));
  app.setViewEngine('hbs');

  await app.listen(3000);
}
bootstrap();
```

[Express](https://github.com/expressjs/express)'e, `public` dizininin statik varlıkları depolamak için, `views`'in şablonları içermek için ve HTML çıktısını oluşturmak için `hbs` şablon motorunu kullanacağımızı söyledik.

#### Şablon Renderlama

Şimdi, `views` dizini oluşturup içine `index.hbs` şablonunu oluşturalım. Şablonun içinde, denetleyiciden iletilen `message`'ı yazdıracağız:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>App</title>
  </head>
  <body>
    {{ "{{ message }\}" }}
  </body>
</html>
```

Sonra, `app.controller` dosyasını açın ve `root()` metodunu aşağıdaki kod ile değiştirin:

```typescript
@@filename(app.controller)
import { Get, Controller, Render } from '@nestjs/common';

@Controller()
export class AppController {
  @Get()
  @Render('index')
  root() {
    return { message: 'Merhaba dünya!' };
  }
}
```

Bu kodda, `@Render()` dekoratöründe kullanılacak şablonu belirtiyoruz ve rota işleyici metodunun dönüş değeri, şablon için işlenmek üzere iletilir. Dikkat edilmesi gereken nokta, dönüş değerinin, şablonda oluşturduğumuz `message` yer tutucusuyla eşleşen bir `message` özelliğine sahip bir nesne olmasıdır.

Uygulama çalışırken tarayıcınızı açın ve `http://localhost:3000` adresine gidin. `Merhaba dünya!` mesajını görmelisiniz.

#### Dinamik Şablon Renderlama

Uygulama mantığı dinamik olarak hangi şablonun render edileceğine karar veriyorsa, o zaman `@Render()` dekoratöründe değil, rota işleyicimizde görünüm adını sağlamak için `@Res()` dekoratörünü kullanmalıyız:

> info **İpucu** Nest, `@Res()` dekoratörünü algıladığında, kütüphane özgü `response` nesnesini ekler. Bu nesneyi şablonu dinamik olarak render etmek için kullanabiliriz. `response` nesnesi API'si hakkında daha fazla bilgiyi [buradan](https://expressjs.com/en/api.html) edinebilirsiniz.

```typescript
@@filename(app.controller)
import { Get, Controller, Res, Render } from '@nestjs/common';
import { Response } from 'express';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private appService: AppService) {}

  @Get()
  root(@Res() res: Response) {
    return res.render(
      this.appService.getViewName(),
      { message: 'Merhaba dünya!' },
    );
  }
}
```

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/15-mvc) bulunabilir.

#### Fastify

Bu [bölümde](/docs/techniques/performance) belirtildiği gibi, Nest ile uyumlu herhangi bir HTTP sağlayıcısını kullanabiliriz. Bu tür kütüphanelerden biri de [Fastify](https://github.com/fastify/fastify)'dir. Fastify ile bir MVC uygulaması oluşturmak için aşağıdaki paketleri kurmamız gerekmektedir:

```bash
$ npm i --save @fastify/static @fastify/view handlebars
```

Sonraki adımlar, Express ile kullanılan sürecin neredeyse aynısını kapsar, ancak platforma özgü küçük farklar vardır. Kurulum işlemi tamamlandığında, `main.ts` dosyasını açın ve içeriğini güncelleyin:

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { NestFastifyApplication, FastifyAdapter } from '@nestjs/platform-fastify';
import { AppModule } from './app.module';
import { join } from 'path';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter(),
  );
  app.useStaticAssets({
    root: join(__dirname, '..', 'public'),
    prefix: '/public/',
  });
  app.setViewEngine({
    engine: {
      handlebars: require('handlebars'),
    },
    templates: join(__dirname, '..', 'views'),
  });
  await app.listen(3000);
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { FastifyAdapter } from '@nestjs/platform-fastify';
import { AppModule } from './app.module';
import { join } from 'path';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, new FastifyAdapter());
  app.useStaticAssets({
    root: join(__dirname, '..', 'public'),
    prefix: '/public/',
  });
  app.setViewEngine({
    engine: {
      handlebars: require('handlebars'),
    },
    templates: join(__dirname, '..', 'views'),
  });
  await app.listen(3000);
}
bootstrap();
```

Fastify API'si biraz farklıdır, ancak bu metod çağrıları ile elde edilen sonuç aynı kalır. Fastify ile dikkat edilmesi gereken bir fark, `@Render()` dekoratörüne geçirilen şablon adının bir dosya uzantısı içermesi gerektiğidir.

```typescript
@@filename(app.controller)
import { Get, Controller, Render } from '@nestjs/common';

@Controller()
export class AppController {
  @Get()
  @Render('index.hbs')
  root() {
    return { message: 'Merhaba dünya!' };
  }
}
```

Uygulama çalışırken tarayıcınızı açın ve `http://localhost:3000` adresine gidin. `Merhaba dünya!` mesajını görmelisiniz.

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/17-mvc-fastify) bulunabilir.