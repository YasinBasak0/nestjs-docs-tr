### Enjeksiyon Kapsamları

Farklı programlama dilinden gelen kişiler için, Nest'te neredeyse her şeyin gelen istekler arasında paylaşıldığını öğrenmek beklenmedik olabilir. Veritabanı için bir bağlantı havuzumuz, genel durumu olan tekil servislerimiz vb. var. Unutmayın ki Node.js, her isteğin ayrı bir iş parçacığı tarafından işlendiği istek/cevap çoklu iş parçacıklı durum modelini takip etmez. Bu nedenle, uygulamalarımız için tekil örneklerin kullanılması tamamen **güvenli**dir.

Ancak, istek tabanlı yaşam süresi istenen davranış olabilir; örneğin GraphQL uygulamalarında istek bazlı önbellek, istek takibi ve çok kiracılı durumlar. Enjeksiyon kapsamları, istenen sağlayıcı yaşam süresini elde etmek için bir mekanizma sağlar.

#### Sağlayıcı Kapsamı

Bir sağlayıcının şu kapsamlardan birine sahip olabilir:

<table>
  <tr>
    <td><code>DEFAULT</code></td>
    <td>Sağlayıcının tek bir örneği uygulama genelinde paylaşılır. Örnek ömrü doğrudan uygulama yaşam döngüsüne bağlıdır. Uygulama başlatıldıktan sonra, tüm tekil sağlayıcılar örneklendirilmiş olur. Singleton kapsamı varsayılan olarak kullanılır.</td>
  </tr>
  <tr>
    <td><code>REQUEST</code></td>
    <td>Sağlayıcının her gelen <strong>istek</strong> için özel olarak oluşturulan yeni bir örneği vardır. Örnek, istek işleme tamamlandıktan sonra çöp toplama ile temizlenir.</td>
  </tr>
  <tr>
    <td><code>TRANSIENT</code></td>
    <td>Geçici sağlayıcılar tüketiciler arasında paylaşılmaz. Her geçici sağlayıcıyı enjekte eden her tüketici, yeni ve ayrılmış bir örnek alır.</td>
  </tr>
</table>

> bilgi **İpucu** Singleton kapsamının kullanılması, çoğu durum için **önerilir**. Sağlayıcıları tüketiciler ve istekler arasında paylaşmak, bir örneğin önbelleğe alınabileceği ve başlatılması sadece bir kez, uygulama başlatıldığında gerçekleştiği anlamına gelir.

#### Kullanım

Enjeksiyon kapsamını belirtmek için `@Injectable()` dekoratör seçenekler nesnesine `scope` özelliğini geçirerek belirtin:

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {}
```

Benzer şekilde, [özel sağlayıcılar](/docs/fundamentals/custom-providers) için bir sağlayıcı kaydında `scope` özelliğini uzun biçimde ayarlayın:

```typescript
{
  provide: 'CACHE_MANAGER',
  useClass: CacheManager,
  scope: Scope.TRANSIENT,
}
```

> bilgi **İpucu** `@nestjs/common` içinden `Scope` enum'unu içe aktarın.

Singleton kapsamı varsayılan olarak kullanılır ve belirtilmesi gerekmez. Sağlayıcıyı tekil kapsamlı olarak belirtmek istiyorsanız, `scope` özelliği için `Scope.DEFAULT` değerini kullanın.

> uyarı **Dikkat** Websocket Gateway'leri istek-tabanlı sağlayıcılar kullanmamalıdır çünkü tekil öğeler gibi davranmalıdırlar. Her gateway, gerçek bir soketi kapsar ve birden fazla kez örneklenemez. Bu kısıtlama, bazı diğer sağlayıcılara da uygulanır, örneğin [_Passport stratejileri_](../security/authentication#request-scoped-strategies) veya _Cron kontrolleri_. 

#### Kontrolcü Kapsamı

Kontrolcüler de kapsama sahip olabilir ve bu, o kontrolcüde bildirilen tüm istek yöntemi işleyicilerine uygulanır. Sağlayıcı kapsamı gibi, bir kontrolcünün kapsamı ömrünü belirtir. İstek-tabanlı bir kontrolcü için, her gelen istek için yeni bir örnek oluşturulur ve istek işleme tamamlandığında çöp toplanır.

Kontrolcü kapsamını `ControllerOptions` nesnesinin `scope` özelliği ile belirtin:

```typescript
@Controller({
  path: 'cats',
  scope: Scope.REQUEST,
})
export class CatsController {}
```

### Kapsam Hiyerarşisi

`REQUEST` kapsamı enjeksiyon zinciri boyunca yukarı doğru hareket eder. Bir kontrolcü, istek tabanlı bir sağlayıcıya bağlıysa, kendisi de istek tabanlı olacaktır.

Aşağıdaki bağımlılık grafiğini düşünün: `CatsController <- CatsService <- CatsRepository`. Eğer `CatsService` istek tabanlı ise (ve diğerleri varsayılan tekiller ise), `CatsController` enjekte edilen servise bağlı olduğundan dolayı istek tabanlı olacaktır. Bağımlı olmayan `CatsRepository`, tekil kapsamda kalacaktır.

Geçici kapsamlı bağımlılıklar bu kalıbı takip etmez. Eğer tekil kapsamlı bir `DogsService` geçici bir `LoggerService` sağlayıcısını enjekte ediyorsa, ona taze bir örnek alacaktır. Ancak, `DogsService` tekil kapsamlı kalacaktır, bu nedenle onu herhangi bir yerde enjekte etmek yeni bir `DogsService` örneği çözümlemez. Bu istenen bir davranışsa, `DogsService` de açıkça `TRANSIENT` olarak işaretlenmelidir.

<app-banner-courses></app-banner-courses>

#### İstek Sağlayıcısı

Bir HTTP sunucusu tabanlı uygulamada (örneğin, `@nestjs/platform-express` veya `@nestjs/platform-fastify` kullanılarak), istek tabanlı sağlayıcıları kullanırken orijinal istek nesnesine bir referansa erişmek isteyebilirsiniz. Bunu, `REQUEST` nesnesini enjekte ederek yapabilirsiniz.

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(REQUEST) private request: Request) {}
}
```

Platform/protokol farklılıkları nedeniyle, Microservice veya GraphQL uygulamaları için gelen isteği biraz farklı şekilde alırsınız. [GraphQL](/docs/graphql/quick-start) uygulamalarında, `REQUEST` yerine `CONTEXT`'i enjekte edersiniz:

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT } from '@nestjs/graphql';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private context) {}
}
```

Daha sonra `context` değerinizi (GraphQLModule içinde) özelliği olarak `request` içermesi şeklinde yapılandırırsınız.

#### Inquirer Sağlayıcısı

Bir sağlayıcının oluşturulduğu sınıfı almak istiyorsanız, örneğin günlükleme veya ölçümler sağlayıcılarında, `INQUIRER` simgesini enjekte edebilirsiniz.

```typescript
import { Inject, Injectable, Scope } from '@nestjs/common';
import { INQUIRER } from '@nestjs/core';

@Injectable({ scope: Scope.TRANSIENT })
export class HelloService {
  constructor(@Inject(INQUIRER) private parentClass: object) {}

  sayHello(message: string) {
    console.log(`${this.parentClass?.constructor?.name}: ${message}`);
  }
}
```

Ardından, aşağıdaki gibi kullanabilirsiniz:

```typescript
import { Injectable } from '@nestjs/common';
import { HelloService } from './hello.service';

@Injectable()
export class AppService {
  constructor(private helloService: HelloService) {}

  getRoot(): string {
    this.helloService.sayHello('Benim adım getRoot');

    return 'Merhaba dünya!';
  }
}
```

Yukarıdaki örnekte `AppService#getRoot` çağrıldığında, konsola `"AppService: Benim adım getRoot"` yazdırılacaktır.

#### Performans

İstek-tabanlı sağlayıcıların kullanılması, uygulama performansını etkileyecektir. Nest, mümkün olan kadar çok meta veriyi önbelleğe alsa da, hala her istekte sınıfınızın bir örneğini oluşturmak zorunda kalacaktır. Bu nedenle, ortalama yanıt sürenizi ve genel ölçüm sonuçlarınızı yavaşlatacaktır. Sağlayıcının kesinlikle istek-tabanlı olması gerekmiyorsa, varsayılan tekil kapsamın kullanılması şiddetle önerilir.

> bilgi **İpucu** Her şey oldukça ürkütücü görünse de, istek-tabanlı sağlayıcıları kullanarak tasarlanmış bir uygulama, gecikme açısından ~5%'ten fazla yavaşlamamalıdır.

#### Dayanıklı (Durable) Sağlayıcılar

Yukarıdaki bölümde bahsedildiği gibi, istek-tabanlı sağlayıcılar artan gecikmeye yol açabilir çünkü en azından bir istek-tabanlı sağlayıcıya sahip olmak (kontrolcü örneğine enjekte edilmiş veya daha derine - sağlayıcılarının birine enjekte edilmiş) kontrolcüyü de istek tabanlı yapar. Bu, her bir bireysel isteğin ardından (ve ardından çöp toplandıktan sonra) kontrolcünün tekrar oluşturulması (öğrenilmesi) gerektiği anlamına gelir. Şimdi, bu aynı zamanda, örneğin eşzamanlı olarak 30.000 istek için, kontrolcünün (ve onun istek tabanlı sağlayıcılarının) 30.000 geçici örneği olacaktır.

Çoğu sağlayıcının bağımlı olduğu ortak bir sağlayıcıya sahip olmak (bir veritabanı bağlantısı veya bir günlük hizmeti gibi düşünün), tüm bu sağlayıcıları otomatik olarak istek-tabanlı sağlayıcılar haline getirir. Bu, özellikle bir merkezi istek-tabanlı "veri kaynağı" sağlayıcısına sahip çok kiracılı uygulamalarda bir zorluk oluşturabilir. Bu sağlayıcı, istek nesnesinden başlıkları/anahtarı alır ve bu değerlere dayanarak ilgili veritabanı bağlantısını/şemasını (kiracıya özgü) alır.

Örneğin, uygulamanızı sırasıyla kullanan 10 farklı müşteriniz olsun. Her müşterinin **kendi özel veri kaynağı** bulunmakta ve müşteri A'nın müşteri B'nin veritabanına asla erişememesini sağlamak istiyorsunuz. Bu hedefe ulaşmanın bir yolu, istek nesnesine dayanarak "veri kaynağı" sağlayıcısını istek-tabanlı olarak belirtmektir - bu, uygulamanızı birkaç dakika içinde çok kiracılı bir uygulamaya dönüştürebilir. Ancak, bu yaklaşımın önemli bir dezavantajı, uygulamanızın büyük bir kısmının muhtemelen "veri kaynağı" sağlayıcısına bağlı olması nedeniyle, bunların otomatik olarak "istek-tabanlı" hale gelmesidir ve bu nedenle uygulama performansını etkileyebilirsiniz.

Peki, daha iyi bir çözümümüz olsaydı ne olurdu? Yalnızca 10 müşterimiz olduğunu düşünürsek, her bir müşteri için ayrı bir [DI alt ağaç](/docs/fundamentals/module-ref#resolving-scoped-providers) oluşturamaz mıydık (her alt ağacı her istekte yeniden oluşturmak yerine)? Sağlayıcılarınızın gerçekten her ardışık isteğe özgü olan herhangi bir özelliğe dayanmıyorsa (örneğin, istek UUID'si), ancak bunları gruplamak (sınıflandırmak) için kullanabileceğimiz bazı belirli özellikler varsa, her gelen istekte _DI alt ağacını yeniden oluşturmak_ için hiçbir neden yoktur.

Ve işte tam da bu durumda **dayanıklı sağlayıcılar** devreye girer.

Sağlayıcıları dayanıklı olarak işaretlemeye başlamadan önce, Nest'e "ortak istek özellikleri" nelerdir ve istekleri gruplamak - bunları karşılayan DI alt ağaçlarını ilişkilendirmek için mantık sağlayan bir **strateji** kaydetmeliyiz.

```typescript
import {
  HostComponentInfo,
  ContextId,
  ContextIdFactory,
  ContextIdStrategy,
} from '@nestjs/core';
import { Request } from 'express';

const tenants = new Map<string, ContextId>();

export class AggregateByTenantContextIdStrategy implements ContextIdStrategy {
  attach(contextId: ContextId, request: Request) {
    const tenantId = request.headers['x-tenant-id'] as string;
    let tenantSubTreeId: ContextId;

    if (tenants.has(tenantId)) {
      tenantSubTreeId = tenants.get(tenantId);
    } else {
      tenantSubTreeId = ContextIdFactory.create();
      tenants.set(tenantId, tenantSubTreeId);
    }

    // Eğer ağaç dayanıklı değilse, orijinal "contextId" nesnesini döndür
    return (info: HostComponentInfo) =>
      info.isTreeDurable ? tenantSubTreeId : contextId;
  }
}
```

> bilgi **İpucu** İstek kapsamı gibi, dayanıklılık enjeksiyon zinciri boyunca yukarı doğru hareket eder. Bu, A'nın B'ye bağlı olduğu bir durumda A'nın da (A sağlayıcısı için açıkça `false` olarak ayarlanmadıkça) dayanıklı hale geleceği anlamına gelir.

> uyarı **Uyarı** Bu strateji, çok sayıda kiracı ile çalışan uygulamalar için ideal değildir.

`attach` yönteminden dönen değer, Nest'e hangi bağlam kimliğinin belirli bir ana bilgisayar için kullanılması gerektiğini söyler. Bu durumda, host bileşeninin (örneğin, istek-tabanlı kontrolcü) dayanıklı olarak işaretlenmişse, `tenantSubTreeId`'nin, otomatik olarak oluşturulan orijinal `contextId` nesnesi yerine kullanılması gerektiğini belirttik. Yukarıdaki örnekte, dayanıklı bir ağaç için (paylaşılan `REQUEST`/`CONTEXT` sağlayıcısı alt ağ

acının üstü - ana) hiçbir yük (payload) kaydedilmez.

Dayanıklı bir ağaç için yük (payload) kaydetmek istiyorsanız, aşağıdaki yapısı kullanılır:

```typescript
// `AggregateByTenantContextIdStrategy#attach` yönteminin dönüşü:
return {
  resolve: (info: HostComponentInfo) =>
    info.isTreeDurable ? tenantSubTreeId : contextId,
  payload: { tenantId },
}
```

Şimdi, `@Inject(REQUEST)` (veya GraphQL uygulamaları için `@Inject(CONTEXT)`) kullanarak isteği enjekte ettiğinizde, `payload` nesnesi enjekte edilecek (bu durumda tek bir özellik içeren `tenantId`).

Peki, bu strateji yerine getirildiğinde, onu kodunuzun herhangi bir yerine kaydedebilirsiniz (zaten genel olarak uygulandığından), bu nedenle örneğin `main.ts` dosyasına yerleştirebilirsiniz:

```typescript
ContextIdFactory.apply(new AggregateByTenantContextIdStrategy());
```

> bilgi **İpucu** `ContextIdFactory` sınıfı, `@nestjs/core` paketinden içe aktarılmıştır.

Kayıt, herhangi bir isteğin uygulamanıza ulaşmadan önce gerçekleştiği sürece, her şey istendiği gibi çalışacaktır.

Son olarak, düzenli bir sağlayıcıyı dayanıklı bir sağlayıcıya dönüştürmek için sadece `durable` bayrağını `true` olarak ayarlamanız ve kapsamını `Scope.REQUEST` olarak değiştirmeniz yeterlidir (REQUEST kapsamı zaten enjeksiyon zincirinde ise gerekli değildir):

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST, durable: true })
export class CatsService {}
```

Benzer şekilde, [özel sağlayıcılar](/docs/fundamentals/custom-providers) için bir sağlayıcı kaydında `durable` özelliğini uzun biçimde ayarlayın:

```typescript
{
  provide: 'foobar',
  useFactory: () => { ... },
  scope: Scope.REQUEST,
  durable: true,
}
```