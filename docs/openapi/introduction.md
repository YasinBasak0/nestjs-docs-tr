### Giriş

[OpenAPI](https://swagger.io/specification/) spesifikasyonu, dil bağımsız bir tanım formatıdır ve RESTful API'leri tanımlamak için kullanılır. Nest, bu spesifikasyonu dekoratörleri kullanarak oluşturmayı sağlayan özel bir [modül](https://github.com/nestjs/swagger) sağlar.

#### Kurulum

Kullanmaya başlamak için önce gerekli bağımlılığı kuruyoruz.

```bash
$ npm install --save @nestjs/swagger
```

#### Başlatma

Kurulum işlemi tamamlandığında, `main.ts` dosyasını açın ve `SwaggerModule` sınıfını kullanarak Swagger'ı başlatın:

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Kediler örneği')
    .setDescription('Kediler API açıklaması')
    .setVersion('1.0')
    .addTag('kedi')
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);

  await app.listen(3000);
}
bootstrap();
```

> info **Hint** `document` (`SwaggerModule#createDocument()` methodu tarafından döndürülen) OpenAPI Belgesi'ne uygun bir seri hale getirilebilir nesnedir. Onu HTTP üzerinde barındırmak yerine JSON/YAML dosyası olarak da kaydedebilir ve farklı yollarla tüketebilirsiniz.

`DocumentBuilder`, OpenAPI Spesifikasyonu'na uygun bir temel belge yapısını oluşturmaya yardımcı olur. Başlık, açıklama, sürüm vb. gibi özellikleri ayarlamaya izin veren birkaç yöntem sağlar. Tüm HTTP yollarını tanımlayan tam bir belge oluşturmak için `SwaggerModule` sınıfının `createDocument()` yöntemini kullanırız. Bu yöntem, bir uygulama örneği ve bir Swagger seçenekleri nesnesi alır. Alternatif olarak, üçüncü bir argüman sağlayabiliriz, bu da `SwaggerDocumentOptions` türünde olmalıdır. Bu konuda daha fazla bilgiyi [Belge seçenekleri bölümünde](/docs/openapi/introduction#document-options) bulabilirsiniz.

Belge oluşturduktan sonra `setup()` yöntemini çağırabiliriz. Bu yöntem şunları kabul eder:

1. Swagger UI'nin monte edileceği yol
2. Bir uygulama örneği
3. Yukarıda oluşturulan belge nesnesi
4. İsteğe bağlı yapılandırma parametresi (daha fazlasını [burada](/docs/openapi/introduction#document-options) okuyun)

Şimdi HTTP sunucusunu başlatmak için aşağıdaki komutu çalıştırabilirsiniz:

```bash
$ npm run start
```

Uygulama çalışırken tarayıcınızı açın ve `http://localhost:3000/api` adresine gidin. Swagger UI'yi görmelisiniz.

<figure><img src="/assets/swagger1.png" /></figure>

Görüleceği gibi, `SwaggerModule` otomatik olarak tüm uç noktalarınızı yansıtır.

> info **Hint** Swagger JSON dosyasını oluşturup indirmek için `http://localhost:3000/api-json` adresine gidin (Swagger belgelerinizin `http://localhost:3000/api` adresinde bulunduğunu varsayalım).

> warning **Uyarı** `fastify` ve `helmet` kullanırken [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) ile ilgili bir sorun olabilir, bu çakışmayı çözmek için CSP'yi aşağıdaki gibi yapılandırın:
>
> ```typescript
> app.register(helmet, {
>   contentSecurityPolicy: {
>     directives: {
>       defaultSrc: [`'self'`],
>       styleSrc: [`'self'`, `'unsafe-inline'`],
>       imgSrc: [`'self'`, 'data:', 'validator.swagger.io'],
>       scriptSrc: [`'self'`, `https: 'unsafe-inline'`],
>     },
>   },
> });
>
> // Eğer hiç CSP kullanmayacaksanız, bunu kullanabilirsiniz:
> app.register(helmet, {
>   contentSecurityPolicy: false,
> });
> ```

#### Belge seçenekleri

Bir belge oluştururken, kütüphanenin davranışını ince ayarlamak için bazı ek seçenekler sağlamak mümkündür. Bu seçeneklerin `SwaggerDocumentOptions` türünde olması gerekmektedir ve şunlar olabilir:

```TypeScript
export interface SwaggerDocumentOptions {
  /**
   * Spesifikasyona dahil edilecek modül listesi
   */
  include?: Function[];

  /**
   * İncelenip spesifikasyona dahil edilmesi gereken ek modeller
   */


  extraModels?: Function[];

  /**
   * Eğer `true` ise, Swagger global öneğini `setGlobalPrefix()` yöntemi aracılığıyla ayarlamaz
   */
  ignoreGlobalPrefix?: boolean;

  /**
   * Eğer `true` ise, Swagger aynı zamanda `include` modülleri tarafından içe aktarılan modüllerden de rotaları yükleyecektir
   */
  deepScanRoutes?: boolean;

  /**
   * `operationId`'yi oluşturmak için kullanılacak özel operationIdFactory
   * `controllerKey` ve `methodKey` üzerinden
   * @default () => controllerKey_methodKey
   */
  operationIdFactory?: (controllerKey: string, methodKey: string) => string;
}
```

Örneğin, kütüphanenin `createUser` yerine `UserController_createUser` gibi işlem adları oluşturmasını sağlamak istiyorsanız, aşağıdaki gibi ayarlayabilirsiniz:

```TypeScript
const options: SwaggerDocumentOptions =  {
  operationIdFactory: (
    controllerKey: string,
    methodKey: string
  ) => methodKey
};
const document = SwaggerModule.createDocument(app, config, options);
```

#### Kurulum seçenekleri

Swagger UI'yi yapılandırmak için, `SwaggerModule#setup` yönteminin dördüncü argümanı olarak `ExpressSwaggerCustomOptions` (eğer express kullanıyorsanız) arayüzünü karşılayan bir seçenek nesnesi geçirerek yapılandırabilirsiniz.

```TypeScript
export interface ExpressSwaggerCustomOptions {
  explorer?: boolean;
  swaggerOptions?: Record<string, any>;
  customCss?: string;
  customCssUrl?: string;
  customJs?: string;
  customfavIcon?: string;
  customSwaggerUiPath?: string;
  swaggerUrl?: string;
  customSiteTitle?: string;
  validatorUrl?: string;
  url?: string;
  urls?: Record<'url' | 'name', string>[];
  patchDocumentOnRequest?: <TRequest = any, TResponse = any> (req: TRequest, res: TResponse, document: OpenAPIObject) => OpenAPIObject;
}
```

#### Örnek

Çalışan bir örnek [burada bulunabilir](https://github.com/nestjs/nest/tree/master/sample/11-swagger).