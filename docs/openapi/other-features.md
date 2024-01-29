### Diğer Özellikler

Bu sayfa, yararlı bulabileceğiniz diğer tüm mevcut özellikleri listeleyerek sunmaktadır.

#### Global Prefix

`setGlobalPrefix()` ile belirlenen rotalar için global bir ön eki yok saymak için `ignoreGlobalPrefix`'i kullanın:

```typescript
const document = SwaggerModule.createDocument(app, options, {
  ignoreGlobalPrefix: true,
});
```

#### Global Parametreler

`DocumentBuilder` kullanarak tüm rotalara parametre tanımları ekleyebilirsiniz:

```typescript
const options = new DocumentBuilder().addGlobalParameters({
  name: 'tenantId',
  in: 'header',
});
```

#### Birden Çok Tanımlama

`SwaggerModule`, birden çok belgeyi desteklemek için bir yol sağlar. Başka bir deyişle, farklı UI'lar üzerinde farklı belgeleri, farklı uç noktalarda sunabilirsiniz.

Birden çok tanıma destek sağlamak için uygulamanız modüler bir yaklaşımla yazılmış olmalıdır. `createDocument()` yöntemi, bir 3. argüman olan `extraOptions`'ı alır, bu da `include` adlı bir özelliğe sahip bir nesnedir. `include` özelliği, modülleri içeren bir dizi değerini alır.

Aşağıda birden çok tanıma destek nasıl sağlanır örneklenmiştir:

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { CatsModule } from './cats/cats.module';
import { DogsModule } from './dogs/dogs.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  /**
   * createDocument(application, configurationOptions, extraOptions);
   *
   * createDocument method takes an optional 3rd argument "extraOptions"
   * which is an object with "include" property where you can pass an Array
   * of Modules that you want to include in that Swagger Specification
   * E.g: CatsModule and DogsModule will have two separate Swagger Specifications which
   * will be exposed on two different SwaggerUI with two different endpoints.
   */

  const options = new DocumentBuilder()
    .setTitle('Cats example')
    .setDescription('The cats API description')
    .setVersion('1.0')
    .addTag('cats')
    .build();

  const catDocument = SwaggerModule.createDocument(app, options, {
    include: [CatsModule],
  });
  SwaggerModule.setup('api/cats', app, catDocument);

  const secondOptions = new DocumentBuilder()
    .setTitle('Dogs example')
    .setDescription('The dogs API description')
    .setVersion('1.0')
    .addTag('dogs')
    .build();

  const dogDocument = SwaggerModule.createDocument(app, secondOptions, {
    include: [DogsModule],
  });
  SwaggerModule.setup('api/dogs', app, dogDocument);

  await app.listen(3000);
}
bootstrap();
```

Şimdi sunucunuzu aşağıdaki komutla başlatabilirsiniz:

```bash
$ npm run start
```

`http://localhost:3000/api/cats` adresine giderek kediler için Swagger UI'yi görebilirsiniz:

<figure><img src="/assets/swagger-cats.png" /></figure>

Sırasıyla, `http://localhost:3000/api/dogs` adresi, köpekler için Swagger UI'yi sunacaktır:

<figure><img src="/assets/swagger-dogs.png" /></figure>