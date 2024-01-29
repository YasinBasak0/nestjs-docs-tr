---
sidebar_position: 6
---

### İstisna Filtreleri

Nest, bir uygulama genelinde işlenmeyen tüm istisnaları işleyen yerleşik bir **istisna katmanına** sahiptir. Bir istisna, uygulama kodunuz tarafından işlenmezse, bu katman tarafından yakalanır ve ardından otomatik olarak uygun bir kullanıcı dostu yanıt gönderilir.

<figure>
  <img src="/assets/Filter_1.png" />
</figure>

Out of the box, bu işlem, `HttpException` türündeki istisnaları (ve bunlardan türetilen alt sınıfları) işleyen yerleşik bir **genel istisna filtresi** tarafından gerçekleştirilir. Bir istisna **tanınmadığında** (ne `HttpException` ne de ondan türetilen bir sınıf), yerleşik istisna filtresi aşağıdaki varsayılan JSON yanıtını oluşturur:

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> info **İpucu** Genel istisna filtresi, `http-errors` kütüphanesini kısmen destekler. Temelde, `statusCode` ve `message` özelliklerini içeren atılan herhangi bir istisna, tanınmayan istisnalar için varsayılan `InternalServerErrorException` yerine doğru bir şekilde doldurulur ve bir yanıt olarak gönderilir.

#### Standart istisnalar fırlatma

Nest, `@nestjs/common` paketinden açığa çıkan yerleşik bir `HttpException` sınıfı sunar. Tipik HTTP REST/GraphQL API tabanlı uygulamalar için, belirli hata koşulları oluştuğunda standart HTTP yanıt nesnelerini göndermek en iyisidir.

Örneğin, `CatsController`daki bir `findAll()` yöntemi (bir `GET` rota işleyeni) istisna fırlatıyor gibi düşünelim. Bunun için aşağıdaki gibi elle kodlayacağız:

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

> info **İpucu** Burada `HttpStatus`'u kullandık. Bu, `@nestjs/common` paketinden içe aktarılan yardımcı bir numaradır.

İstemci bu uç noktayı çağırdığında, yanıt şu şekilde görünür:

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

`HttpException`'in yapıcı fonksiyonu iki zorunlu argüman alır ve yanıtı belirler:

- `response` argümanı, JSON yanıt gövdesini tanımlar. Bir `string` veya aşağıda açıklanan gibi bir `object` olabilir.
- `status` argümanı, [HTTP durum kodunu](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) tanımlar.

Varsayılan olarak, JSON yanıt gövdesi iki özelliği içerir:

- `statusCode`: `status` argümanında sağlanan HTTP durum koduna varsayılan olarak ayarlanır
- `message`: `status` temel alınarak HTTP hatası hakkında kısa bir açıklama

Yalnızca JSON yanıt gövdesinin mesaj kısmını geçersiz kılmak için `response` argümanına bir dize sağlayın. Tüm JSON yanıt gövdesini geçersiz kılmak için, `response` argümanına bir nesne iletebilirsiniz. Nest, nesneyi serileştirir ve JSON yanıt gövdesi olarak döndürür.

Üçüncü yapıcı argümanı - `options` - opsiyonel olarak kullanılabilir ve bir [hata nedeni](https://nodejs.org/en/blog/release/v16.9.0/#error-cause) sağlamak için kullanılabilir. Bu `cause` nesnesi, yanıt nesnesine seri hale getirilmez, ancak `HttpException`'in fırlatılmasına neden olan iç hata hakkında değerli bilgiler sağlamak için kullanışlı olabilir.

Aşağıda, tüm yanıt gövdesini geçersiz kılan ve bir hata nedeni sağlayan bir örnek bulunmaktadır:

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  try {
    await this.service.findAll()
  } catch (error) { 
    throw new HttpException({
      status: HttpStatus.FORBIDDEN,
      error: 'This is a custom message',
    }, HttpStatus.FORBIDDEN, {
      cause: error
    });
  }
}
```

Yukarı

daki örneği kullanarak, yanıt şu şekilde görünür:

```json
{
  "status": 403,
  "error": "This is a custom message"
}
```

#### Özel istisnalar

Çoğu durumda, özel istisnalar yazmanıza gerek olmayacak ve bir sonraki bölümde açıklanan yerleşik Nest HTTP istisnasını kullanabilirsiniz. Ancak özel istisnalar oluşturmanız gerekiyorsa, özel istisna sınıflarınızın temel `HttpException` sınıfından türemesi iyi bir uygulama biçimidir. Bu yaklaşımla, Nest özel istisnalarınızı tanıyacak ve hata yanıtlarıyla otomatik olarak ilgilenecektir. Böyle bir özel istisna uygulayalım:

```typescript
@@filename(forbidden.exception)
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

`ForbiddenException`, temel `HttpException`'ten türediği için, yerleşik istisna işleyiciyle sorunsuz bir şekilde çalışacaktır ve bu nedenle `findAll()` yöntemi içinde kullanabiliriz.

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

#### Yerleşik HTTP istisnaları

Nest, temel `HttpException` sınıfından türeyen bir dizi standart istisna sağlar. Bunlar, `@nestjs/common` paketinden açığa çıkar ve en yaygın HTTP istisnalarını temsil eder:

- `BadRequestException`
- `UnauthorizedException`
- `NotFoundException`
- `ForbiddenException`
- `NotAcceptableException`
- `RequestTimeoutException`
- `ConflictException`
- `GoneException`
- `HttpVersionNotSupportedException`
- `PayloadTooLargeException`
- `UnsupportedMediaTypeException`
- `UnprocessableEntityException`
- `InternalServerErrorException`
- `NotImplementedException`
- `ImATeapotException`
- `MethodNotAllowedException`
- `BadGatewayException`
- `ServiceUnavailableException`
- `GatewayTimeoutException`
- `PreconditionFailedException`

Tüm yerleşik istisnalar, `options` parametresini kullanarak hem bir hata `cause`'sini hem de bir hata açıklamasını sağlayabilir:

```typescript
throw new BadRequestException('Something bad happened', { cause: new Error(), description: 'Some error description' })
```

Yukarıdaki örneği kullanarak, yanıt şu şekilde görünür:

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400
}
```

#### İstisna filtreleri

Temel (yerleşik) istisna filtresi birçok durumu otomatik olarak sizin için işleyebilirken, istisnalar katmanı üzerinde **tam kontrol** isteyebilirsiniz. Örneğin, günlüğe kayıt eklemek veya bazı dinamik faktörlere dayalı olarak farklı bir JSON şeması kullanmak isteyebilirsiniz. **İstisna filtreleri** tam olarak bu amaçla tasarlanmıştır. Bu, kontrol akışının tam olarak kontrolünü ve istemciye gönderilen yanıtın içeriğini kontrol etmenizi sağlar.

Hadi, `HttpException` sınıfının bir örneği olan istisnaları yakalayan ve bunlar için özel yanıt mantığını uygulayan bir istisna filtresi oluşturalım. Bunun için temel platform `Request` ve `Response` nesnelerine erişmemiz gerekecek. `Request` nesnesine erişeceğiz böylece orijinal `url`'yi çıkarabilir ve bu bilgileri günlük bilgilerine dahil edebiliriz. `Response` nesnesini kullanarak gönderilen yanıtı doğrudan kontrol etmek için `response.json()` yöntemini kullanacağız.

```typescript
@@filename(http-exception.filter)
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
@@switch
import { Catch, HttpException } from '@nestjs/common';

@Catch(HttpException)
export class HttpExceptionFilter {
  catch(exception, host) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

> info **Hint** Tüm istisna filtreleri, genel `ExceptionFilter<T>` arabirimini uygulamalıdır. Bu, `catch(exception: T, host: ArgumentsHost)` yöntemini belirtilen imza ile sağlamanızı gerektirir. `T`, istisna türünü belirtir.

> warning **Warning** Eğer `@nestjs/platform-fastify` kullanıyorsanız, `response.json()` yerine `response.send()` kullanabilirsiniz. Doğru türleri `fastify`'dan içe aktarmayı unutmayın.

#### Arguments host

Haydi, `catch()` yönteminin parametrelerine bir göz atalım. `exception` parametresi şu anda işlenen istisna nesnesidir. `host` parametresi bir `ArgumentsHost` nesnesidir. `ArgumentsHost`, [yürütme bağlamı bölümünde](/docs/fundamentals/execution-context)\* daha ayrıntılı bir şekilde inceleyeceğimiz güçlü bir yardımcı nesnedir. Bu kod örneğinde, bu nesneleri almak için `ArgumentsHost` üzerinde bazı yardımcı yöntemleri kullandık. `ArgumentsHost` hakkında daha fazla bilgi için [buraya](/docs/fundamentals/execution-context) bakın.

\*Bu seviyedeki soyutlama seviyesinin nedeni, `ArgumentsHost`'un tüm bağlamlarda (şu anda çalıştığımız HTTP sunucusu bağlamı gibi, ancak aynı zamanda Mikroservisler ve WebSockets gibi) çalışmasıdır. Yürütme bağlamı bölümünde, `ArgumentsHost` ve yardımcı işlevlerinin gücü ile **herhangi bir** yürütme bağlamı için uygun <a href="https://docs.nestjs.com/fundamentals/execution-context#host-methods">altındaki argümanlara</a> nasıl erişebileceğimizi göreceğiz. Bu, tüm bağlamlarda çalışan genel istisna filtreleri yazmamıza olanak tanır.

<app-banner-courses></app-banner-courses>

#### Filtreleri bağlama

Yeni `HttpExceptionFilter`'ımızı `CatsController`'ın `create()` yöntemine bağlayalım.

```typescript
@@filename(cats.controller)
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
@@switch
@Post()
@UseFilters(new HttpExceptionFilter())
@Bind(Body())
async create(createCatDto) {
  throw a ForbiddenException();
}
```

> info **Hint** `@UseFilters()` dekoratörü `@nestjs/common` paketinden içe aktarılır.

Burada `@UseFilters()` dekoratörünü kullandık. `@Catch()` dekoratörü gibi, tek bir filtre örneği veya virgülle ayrılmış bir filtre örneği listesi alabilir. Burada `HttpExceptionFilter` örneğini yerinde oluşturduk. İsteğe bağlı olarak, bağımlılık enjeksiyonunu **etkinleştiren** çerçeveye sınıfı (bir örneğin yerine) geçirebilir ve çerçeveye **bağımlılık enjeksiyonunu** bırakabilirsiniz.

```typescript
@@filename(cats.controller)
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
@@switch
@Post()
@UseFilters(HttpExceptionFilter)
@Bind(Body())
async create(createCatDto) {
  throw new ForbiddenException();
}
```

> info **Hint** Mümkünse sınıfları örnekler yerine kullanarak filtreleri uygulamayı tercih edin. Bu, Nest'in aynı sınıfın örneklerini modülünüz boyunca kolayca yeniden kullanabilmesini sağlar ve **bellek kullanımını** azaltır.

Yukarıdaki örnekte, `HttpExceptionFilter` yalnızca tek `create()` rota işleyicisine uygulandı, bu da onu yöntem kapsamlı yaptı. İstisna filtreleri farklı düzeylerde kapsamlandırılabilir: yöntem kapsamlı bir denetleyici/çözücü/geçişte, denetleyici kapsamlı veya global kapsamlı.  
Örneğin, bir filtreyi denetleyici kapsamlı olarak ayarlamak için şunları yapardınız:

```typescript
@@filename(cats.controller)
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

Bu yapı, `CatsController` içinde tanımlanan her rota işleyicisi için `HttpExceptionFilter`'i ayarlar.

Bir global kapsamlı filtre oluşturmak için şunları yapardınız:

```typescript
@@filename(main)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```

> warning **Warning** `useGlobalFilters()` yöntemi filtreleri geçitler veya hibrit uygulamalar için filtreleri kurmaz.

Global kapsamlı filtreler, tüm uygulama genelinde, her denetleyici ve her rota işleyici için kullanılır. Bağımlılık enjeksiyonu açısından, dış modül dışından (yukarıdaki örnekte olduğu gibi) kaydedilen global filtreler, bu modülün bağlamı dışında yapılır çünkü bu modül dışında yapılır. Bu sorunu çözmek için **doğrudan herhangi bir modülden** global kapsamlı bir filtre kaydedebilirsiniz:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

> info **Hint** Bu yaklaşımı kullanarak filtre için bağımlılık enjeksiyonu yaparken, bu yapının nerede kullanılacağına dikkat edin. Bu yapının kullanılacağı modülü seçin (yukarıdaki örnekte `HttpExceptionFilter`). Ayrıca, `useClass`, özel sağlayıcı kayıtlarıyla başa çıkmanın tek yolu değildir. Daha fazla bilgi için [buraya](/docs/fundamentals/custom-providers) bakın.

Bu teknikle bu yöntemle istenen kadar çok filtre ekleyebilirsiniz; sadece her birini sağlayıcılara ekleyin.

#### Her şeyi yakalama

İşlenmemiş **her** istisnayı yakalamak için (istisna türünden bağımsız olarak) `@Catch()` dekoratörünün parametre listesini boş bırakın, örneğin `@Catch()`.

Aşağıdaki örnekte, yanıtı iletmek için [HTTP adaptörünü](./faq/http-adapter) kullandığından ve platforma özgü nesneleri (`Request` ve `Response`) doğrudan kullanmadığından, kodun platformdan bağımsız olduğu bir kodumuz var:

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // Belirli durumlarda `httpAdapter`'ın constructor yönteminde
    // kullanılamayabileceğinden, burada çözmemiz gerekir.
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

> warning **Uyarı** Her şeyi yakalayan bir istisna filtresini, belirli bir türle bağlı bir filtreyi birleştirirken, "Her şeyi yakala" filtresinin bağlı türü doğru bir şekilde işlemesine izin vermek için ilk olarak tanımlanmalıdır.

#### Kalıtım

Genellikle, uygulama gereksinimlerinizi karşılamak üzere özel oluşturulmuş tamamen özelleştirilmiş istisna filtreleri oluşturacaksınız. Ancak, belirli faktörlere dayalı olarak davranışı geçersiz kılma amacınız olduğunda, yerleşik varsayılan **genel istisna filtresini** genişletmek isteyebileceğiniz durumlar olabilir.

İstisna işleme görevini temel filtreye devretmek için `BaseExceptionFilter`'ı genişletmeniz ve miras alınan `catch()` yöntemini çağırmanız gerekir.

```typescript
@@filename(all-exceptions.filter)
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception, host) {
    super.catch(exception, host);
  }
}
```

> warning **Uyarı** Yöntem kapsamlı ve Denetleyici kapsamlı filtrelerin `BaseExceptionFilter`'ı genişletirken `new` ile örneklendirilmemesi gerekir. Bunun yerine, çerçeve tarafından bunların otomatik olarak örneklenmesine izin verin.

Yukarıdaki uygulama, yaklaşımı gösteren bir kabuktan ibarettir. Genişletilmiş istisna filtresinin uygulamanızın özelleştirilmiş **iş** mantığını içermelidir (örneğin, çeşitli koşulları işleme alma).

Global filtreler **BaseExceptionFilter**'ı genişletebilir. Bu, iki farklı yöntemle yapılabilir.

İlk yöntem, özel global filtreyi örneklendirirken `HttpAdapter` referansını enjekte etmektir:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(3000);
}
bootstrap();
```

İkinci yöntem, `APP_FILTER` belirtecini kullanmaktır <a href="exception-filters#binding-filters">burada gösterildiği gibi</a>.
