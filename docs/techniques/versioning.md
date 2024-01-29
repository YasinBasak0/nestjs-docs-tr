### Sürümleme

> info **İpucu** Bu bölüm, yalnızca HTTP tabanlı uygulamalar için geçerlidir.

Sürümleme, aynı uygulama içinde farklı sürümlerde çalışan denetleyicilere veya bireysel rotalara sahip olmanıza olanak tanır. Uygulamalar çok sık değişir ve genellikle hala önceki uygulama sürümünü desteklemeniz gereken kırıcı değişiklikler yapmanız gerektiği durumlarla karşılaşılabilir.

Desteklenen 4 sürümleme türü bulunmaktadır:

<table>
  <tr>
    <td><a href='techniques/versioning#uri-versioning-type'><code>URI Sürümleme</code></a></td>
    <td>Sürüm, isteğin URI'si içinde iletilir (varsayılan)</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#header-versioning-type'><code>Başlık Sürümleme</code></a></td>
    <td>Özel bir istek başlığı, sürümü belirler</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#media-type-versioning-type'><code>Medya Türü Sürümleme</code></a></td>
    <td>İsteğin <code>Accept</code> başlığı, sürümü belirtir</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#custom-versioning-type'><code>Özel Sürümleme</code></a></td>
    <td>İsteğin herhangi bir yönü, sürüm(leri) belirtmek için kullanılabilir. Belirtilen sürüm(leri) çıkarmak için özel bir işlev sağlanır.</td>
  </tr>
</table>

#### URI Sürümleme Türü

URI Sürümleme, isteğin URI'si içinde iletilen sürümü kullanır, örneğin `https://example.com/v1/route` ve `https://example.com/v2/route`.

> uyarı **Dikkat** URI Sürümleme ile sürüm, (varsa) <a href="faq/global-prefix">global yol öneki</a> sonrasında ve herhangi bir denetleyici veya rota yolu öncesi URI'ye otomatik olarak eklenir.

Uygulamanız için URI Sürümleme'yi etkinleştirmek için şunları yapın:

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
// veya "app.enableVersioning()"
app.enableVersioning({
  type: VersioningType.URI,
});
await app.listen(3000);
```

> uyarı **Dikkat** URI'deki sürüm varsayılan olarak `v` ile otomatik olarak öne eklenir, ancak isteğe bağlı önek değerinizi belirlemek için `prefix` anahtarını istediğiniz önek olarak veya devre dışı bırakmak için `false` olarak ayarlayabilirsiniz.

> info **İpucu** `VersioningType` numarasını `type` özelliği için kullanabilirsiniz ve bu öğe `@nestjs/common` paketinden içeri aktarılır.

#### Başlık Sürümleme Türü

Başlık Sürümleme, sürümü belirtmek için özel, kullanıcı tarafından belirlenmiş bir istek başlığını kullanır, başlığın değeri isteğin kullanılacak sürümünü içerir.

Başlık Sürümleme için örnek HTTP İstekleri:

Başlık Sürümleme'yi uygulamanız için etkinleştirmek için şunları yapın:

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.HEADER,
  header: 'Custom-Header',
});
await app.listen(3000);
```

`header` özelliği, isteğin sürümünü içerecek başlığın adı olmalıdır.

> info **İpucu** `VersioningType` numarasını `type` özelliği için kullanabilirsiniz ve bu öğe `@nestjs/common` paketinden içeri aktarılır.

#### Medya Türü Sürümleme Türü

Medya Türü Sürümleme, isteğin sürümünü belirtmek için isteğin `Accept` başlığını kullanır.

`Accept` başlığı içinde sürüm, medya türünden noktalı virgül `;` ile ayrılır. Ardından, sürümü temsil eden bir anahtar-değer çiftini içeren bir dizi bulunmalıdır, örneğin `Accept: application/json;v=2`. Anahtar, sürüm belirlenirken anahtar ve ayırıcıyı içermesi açısından daha çok bir ön ek gibi davranır.

Uygulamanız için **Medya Türü Sürümleme**'yi etkinleştirmek için şunları yapın:

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key: 'v=',
});
await app.listen(3000);
```

`key` özelliği, sürümü içeren anahtar-değer çiftinin anahtar ve ayırıcısını içermelidir. Örneğin, `Accept: application/json;v=2` için `key` özelliği `v=` olarak ayarlanmalıdır.

> info **İpucu** `VersioningType` numarasını `type` özelliği için kullanabilirsiniz ve bu öğe `@nestjs/common` paketinden içeri aktarılır.

#### Özel Sürümleme Türü

Özel Sürümleme, sürümü (veya sürümleri) belirtmek için isteğin herhangi bir yönünü kullanır. Gelen istek, bir dize veya dize dizisi döndüren bir `extractor` işlevi kullanılarak analiz edilir.

İsteyen tarafından birden çok sürüm sağlanıyorsa, extractor işlevi bir dize veya dize dizisi döndürebilir. Sürümler, en yüksek sürümden en düşük sürüme doğru sıralanmış bir dize dizisi olarak döndürülür. Sürümler, en yüksekten en düşüğe doğru sırayla eşleşen rotalara eşlenir.

Eğer `extractor` tarafından döndürülen bir boş dize veya dize dizisi durumunda hiçbir rota eşlenmez ve bir 404 döndürülür.

Örneğin, bir gelen istek, `1`, `2` ve `3` sürümlerini desteklediğini belirtiyorsa, `extractor` **MUTLAKA** `[3, 2, 1]`'i döndürmelidir. Bu, en yüksek olası rota sürümünün ilk seçildiğini sağlar.

Eğer `[3, 2, 1]` sürümleri çıkartılırsa, ancak sürüm `2` ve `1` için rotalar mevcutsa, `2` sürümüne uyan rota seçilir (sürüm `3` otomatik olarak yok sayılır).

> uyarı **Dikkat** Express adaptörü ile en yüksek eşleşen sürümün seçilmesi, `extractor` tarafından döndürülen diziden kaynaklanan tasarım kısıtlamaları nedeniyle **güvenilir bir şekilde çalışmaz**. Express için tek bir sürüm (ya bir dize ya da 1 elemanlı bir dize dizisi) sorunsuz çalışır. Fastify, en yüksek eşleşen sürüm seçimi ve tek sürüm seçimi konusunda doğru bir şekilde her ikisini de destekler.

Uygulamanız için **Özel Sürümleme**'yi etkinleştirmek için bir `extractor` işlevi oluşturun ve uygulamanıza iletişim kurun:

```typescript
@@filename(main)
// Örnek bir extractor, özel bir başlıktan sürümleri çıkarır ve sıralı bir dizi haline getirir.
// Bu örnek Fastify kullanır, ancak Express istekleri benzer bir şekilde işlenebilir.
const extractor = (request: FastifyRequest): string | string[] =>
  [request.headers['custom-versioning-field'] ?? '']
     .flatMap(v => v.split(','))
     .filter(v => !!v)
     .sort()
     .reverse()

const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.CUSTOM,
  extractor,
});
await app.listen(3000);
```

#### Kullanım

Versiyonlama, denetleyicileri, bireysel rotaları sürümlemenize olanak tanır ve ayrıca belirli kaynakların sürümlendirmeyi devre dışı bırakmasını sağlayan bir yol sunar. Sürümleme Türü'nüze bağlı olarak uygulamanın kullanımı aynıdır.

> uyarı **Dikkat** Eğer uygulama için sürümleme etkinleştirilmişse ancak denetleyici veya rota sürümü belirtmezse, o denetleyici/rota için yapılan tüm isteklere `404` yanıt durumu döndürülür. Benzer şekilde, sürüm içeren ancak karşılık gelen bir denetleyici veya rota olmayan bir istek alınırsa, aynı şekilde `404` yanıt durumu döndürülür.

#### Denetleyici Sürümleri

Bir denetleyiciye bir sürüm uygulanabilir ve bu sürüm denetleyici içindeki tüm rotalar için sürümü belirler.

Bir denetleyiciye bir sürüm eklemek için aşağıdaki adımları takip edin:

```typescript
@@filename(cats.controller)
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1';
  }
}
@@switch
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll() {
    return 'This action returns all cats for version 1';
  }
}
```

#### Rota Sürümleri

Bir sürüm, bir bireysel rota üzerine uygulanabilir. Bu sürüm, rota üzerindeki Denetleyici Sürümü gibi rotayı etkileyen diğer sürümleri geçersiz kılacaktır.

Bir bireysel rota üzerine bir sürüm eklemek için aşağıdaki adımları takip edin:

```typescript
@@filename(cats.controller)
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1(): string {
    return 'This action returns all cats for version 1';
  }

  @Version('2')
  @Get('cats')
  findAllV2(): string {
    return 'This action returns all cats for version 2';
  }
}
@@switch
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1() {
    return 'This action returns all cats for version 1';
  }

  @Version('2')
  @Get('cats')
  findAllV2() {
    return 'This action returns all cats for version 2';
  }
}
```

#### Birden fazla sürüm

Bir denetleyiciye veya rotaya birden fazla sürüm uygulanabilir. Birden fazla sürüm kullanmak için, sürümü bir dizi olarak ayarlamanız gerekir.

Birden fazla sürüm eklemek için aşağıdaki adımları takip edin:

```typescript
@@filename(cats.controller)
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1 or 2';
  }
}
@@switch
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll() {
    return 'This action returns all cats for version 1 or 2';
  }
}
```

#### Sürüm "Nötr"

Bazı denetleyiciler veya rotalar, sürümden bağımsız olarak aynı işlevselliğe sahip olabilir. Bunu karşılamak için sürüm, `VERSION_NEUTRAL` sembolüne ayarlanabilir.

Gelen bir istek, istekte gönderilen sürüme bağlı olarak değil, ayrıca istek hiçbir sürüm içermiyorsa, `VERSION_NEUTRAL` denetleyici veya rotaya eşlenir.

> uyarı **Dikkat** URI Sürümleme için, `VERSION_NEUTRAL` bir kaynakta sürüm URI'de bulunmaz.

Sürüm nötr bir denetleyici veya rota eklemek için aşağıdaki adımları takip edin:

```typescript
@@filename(cats.controller)
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats regardless of version';
  }
}
@@switch
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll() {
    return 'This action returns all cats regardless of version';
  }
}
```

#### Global varsayılan sürüm

Eğer her denetleyiciye veya bireysel rotaya bir sürüm belirtmek istemiyorsanız veya belirtilmemiş her denetleyici/rota için belirli bir sürümü varsayılan sürüm olarak ayarlamak istiyorsanız, `defaultVersion`'ı aşağıdaki gibi ayarlayabilirsiniz:

```typescript
@@filename(main)
app.enableVersioning({
  // ...
  defaultVersion: '1'
  // veya
  defaultVersion: ['1', '2']
  // veya
  defaultVersion: VERSION_NEUTRAL
});
```

#### Ara yazılım sürümleme

[Middleware'ler](https://docs.nestjs.com/middleware), ayrı bir rota için middleware'i yapılandırmak için sürümle ilgili meta verileri de kullanabilir. Bunun için `MiddlewareConsumer.forRoutes()` yöntemi için bir parametre olarak sürüm numarasını sağlayın:

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET, version: '2' });
  }
}
```

Yukarıdaki kodla, `LoggerMiddleware` yalnızca `/cats` uç noktasının '2' sürümüne uygulanacaktır.

> bilgi **Dikkat** Middleware'ler, bu bölümde açıklanan `URI`, `Header`, `Media Type` veya `Custom` sürümleme türleri ile çalışır.