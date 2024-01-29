### Genel Bakış

Nest, geleneksel (bazen monolitik olarak adlandırılan) uygulama mimarilerine ek olarak, doğal olarak mikroservis tabanlı bir geliştirme tarzını destekler. Bu belgelemenin başka yerlerinde tartışılan bağımlılık enjeksiyonu, dekoratörler, istisna filtreleri, borular, korumalar ve interceptor'lar gibi kavramların çoğu, mikroservislere de aynı şekilde uygulanır. Mümkün olduğunca, Nest, aynı bileşenlerin HTTP tabanlı platformlarda, WebSockets'te ve Mikroservislerde çalışmasını sağlayacak şekilde uygulama detaylarını soyutlar. Bu bölüm, Nest'e özgü olan mikroservislerle ilgili yönleri ele almaktadır.

Nest'te bir mikroservis, temel olarak HTTP'den farklı bir **taşıma** katmanı kullanan bir uygulamadır.

<figure><img src="/assets/Microservices_1.png" /></figure>

Nest, **taşıyıcılar** adı verilen yerleşik taşıma katmanı uygulamalarını destekler. Bu taşıyıcılar, farklı mikroservis örnekleri arasında iletişim kurma sorumluluğundadır. Çoğu taşıyıcı, genellikle hem **istek-cevap** hem de **olay-tabanlı** mesaj stillerini doğal olarak destekler. Nest, her bir taşıyıcının uygulama kodunu etkilemeden kolayca bir taşıma katmanından diğerine geçmenizi sağlar. Örneğin, belirli bir taşıma katmanının özel güvenilirlik veya performans özelliklerini kullanmak için bir taşıma katmanından diğerine geçebilirsiniz.

#### Kurulum

Mikroservisler oluşturmak için ilk olarak gerekli paketi yükleyin:

```bash
$ npm i --save @nestjs/microservices
```

#### Başlangıç

Bir mikroservis örneği oluşturmak için, `NestFactory` sınıfının `createMicroservice()` yöntemini kullanabilirsiniz:

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
    },
  );
  await app.listen();
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.TCP,
  });
  await app.listen();
}
bootstrap();
```

> info **Öneri** Mikroservisler, varsayılan olarak **TCP** taşıma katmanını kullanır.

`createMicroservice()` yönteminin ikinci argümanı bir `options` nesnesidir. Bu nesne iki üyeden oluşabilir:

<table>
  <tr>
    <td><code>transport</code></td>
    <td>Taşıyıcıyı belirtir (örneğin, <code>Transport.NATS</code> kullanabilirsiniz)</td>
  </tr>
  <tr>
    <td><code>options</code></td>
    <td>Taşıyıcı davranışını belirleyen taşıyıcıya özgü bir seçenekler nesnesi</td>
  </tr>
</table>
<p>
  <code>options</code> nesnesi, seçilen taşıyıcıya özgüdür. <strong>TCP</strong> taşıyıcısı aşağıda açıklanan özellikleri ortaya çıkarır.
  Diğer taşıyıcılar için (örneğin, Redis, MQTT, vb.), kullanılabilir seçeneklerin açıklaması için ilgili bölüme bakın.
</p>
<table>
  <tr>
    <td><code>host</code></td>
    <td>Bağlantı ana bilgisayarı</td>
  </tr>
  <tr>
    <td><code>port</code></td>
    <td>Bağlantı portu</td>
  </tr>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>Mesajı tekrar deneme sayısı (varsayılan: <code>0</code>)</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>Mesaj tekrar deneme girişimleri arasındaki gecikme (ms) (varsayılan: <code>0</code>)</td>
  </tr>
</table>

#### Desenler

Mikroservisler, hem mesajları hem de olayları **desenler** ile tanır. Bir desen, örneğin, bir kelime nesnesi veya bir dizedir. Desenler otomatik olarak seri hale getirilir ve bir mesajın veri kısmı ile birlikte ağ üzerinden gönderilir. Bu şekilde, mesaj gönderenler ve tüketiciler, hangi isteklerin hangi işleyiciler tarafından tüketildiğini koordine edebilir.

#### İstek-cevap

İstek-cevap mesaj stili, çeşitli harici servisler arasında mesajlar **değiştirmeniz gerektiğinde** kullanışlıdır. Bu paradigm ile, servisin mesajı gerçekten aldığından emin olabilirsiniz (manuel olarak bir mesaj ACK protokolü uygulamaya gerek kalmadan). Ancak, istek-cevap paradigmı her zaman en iyi seçenek olmayabilir. Örneğin, [Kafka](https://docs.confluent.io/3.0.0/streams/) veya [NATS streaming](https://github.com/nats-io/node-nats-streaming) gibi günlük tabanlı kalıcılık kullanan akış taşıyıcıları, daha çok olay tabanlı bir mesajlaşma paradigmı ile uyumlu olan farklı bir sorun kümesini çözmek için optimize edilmiştir (daha fazla ayrıntı için aşağıya bakınız [olay tabanlı mesajlaşma](https://docs.nestjs.com/microservices/basics#event-based)).

İstek-cevap mesaj türünü etkinleştirmek için, Nest iki mantıksal kanal oluşturur - bir tanesi veriyi aktarmakla görevliyken, diğeri gelen yanıtları bekler. Bazı temel taşıyıcılarda, [NATS](https://nats.io/) gibi, bu çift kanal desteği otomatik olarak sağlanır. Diğerlerinde, Nest bu durumu manuel olarak ayrı kanallar oluşturarak telafi eder. Bu durumda bir maliyet olabilir, bu nedenle istek-cevap mesaj stiline ihtiyaç duymuyorsanız, olay tabanlı yöntemi kullanmayı düşünmelisiniz.

İstek-cevap paradigmına dayalı bir mesaj işleyici oluşturmak için `@MessagePattern()` dekoratörünü kullanın, bu dekoratör `@nestjs/microservices` paketinden içe aktarılır. Bu dekoratör, uygulamanızın giriş noktaları olan [controller](https://docs.nestjs.com/controllers) sınıfları içinde kullanılmalıdır. Onları sağlayıcılarda kullanmak, Nest çalışma zamanı tarafından sadece görmezden gelindiği için herhangi bir etkisi olmayacaktır.

```typescript
@@filename(math.controller)
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data: number[]): number {
    return (data || []).reduce((a, b) => a + b);
  }
}
@@switch
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data) {
    return (data || []).reduce((a, b) => a + b);
  }
}
```

Yukarıdaki kodda, `accumulate()` **mesaj işleyici**, `{{ '{' }} cmd: 'sum' {{ '}' }}` mesaj desenini karşılayan mesajları dinler. Mesaj işleyicisi, istemciden gelen `data` adlı tek bir argüman alır. Bu durumda, veri bir sayı dizisidir ve bu sayılar biriktirilmelidir.

#### Asenkron cevaplar

Mesaj işleyicileri hem senkron olarak hem de **asenkron olarak** yanıt verebilir. Bu nedenle, `async` yöntemleri desteklenir.

```typescript
@@filename()
@MessagePattern({ cmd: 'sum' })
async accumulate(data: number[]): Promise<number> {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@MessagePattern({ cmd: 'sum' })
async accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```

Bir mesaj işleyicisi ayrıca bir `Observable` döndürebilir, bu durumda sonuç değerleri akış tamamlandığında yayımlanacaktır.

```typescript
@@filename()
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): Observable<number> {
  return from([1, 2, 3]);
}
@@switch
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): Observable<number> {
  return from([1, 2, 3]);
}
```

Yukarıdaki örnekte, mesaj işleyicisi **3 kez** yanıt verecektir (dizideki her öğe ile).

#### Olay tabanlı

İstek-cevap yöntemi, servisler arasında mesaj değişiminde bulunmak için ideal olsa da, mesaj stiliniz olay tabanlı olduğunda - yalnızca bir yanıt beklemeksizin **olayları** yayınlamak istediğinizde, daha az uygun olabilir. Bu durumda, iki kanalın sürdürülmesi için istek-cevap tarafından gereken iş yükünü istemezsiniz.

Örneğin, bu sistem bölümünde belirli bir koşulun meydana geldiğini başka bir servise bildirmek istiyorsanız, bu, olay tabanlı mesaj stili için ideal kullanım durumudur.

Bir olay işleyicisi oluşturmak için, `@EventPattern()` dekoratörünü kullanırız, bu dekoratör `@nestjs/microservices` paketinden içe aktarılır.

```typescript
@@filename()
@EventPattern('user_created')
async handleUserCreated(data: Record<string, unknown>) {
  // iş mantığı
}
@@switch
@EventPattern('user_created')
async handleUserCreated(data) {
  // iş mantığı
}
```

> info **Hint** **Tek** olay deseni için birden fazla olay işleyicisini kaydedebilir ve hepsi otomatik olarak paralel olarak tetiklenir.

`handleUserCreated()` **olay işleyicisi**, `'user_created'` olayını dinler. Olay işleyicisi, istemciden geçen `data` adlı tek bir argüman alır (bu durumda, ağ üzerinden gönderilmiş olan bir olay yüküdür).

<app-banner-enterprise></app-banner-enterprise>

#### Dekoratörler

Daha karmaşık senaryolarda, gelen isteğe dair daha fazla bilgiye erişmek isteyebilirsiniz. Örneğin, NATS ile joker abonelikleri kullanılıyorsa, üreticinin mesajı hangi konuya gönderdiğini elde etmek isteyebilirsiniz. Benzer şekilde, Kafka'da mesaj başlıklarına erişmek isteyebilirsiniz. Bunu başarmak için yerleşik dekoratörleri şu şekilde kullanabilirsiniz:

```typescript
@@filename()
@MessagePattern('time.us.*')
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`); // örn. "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('time.us.*')
getDate(data, context) {
  console.log(`Subject: ${context.getSubject()}`); // örn. "time.us.east"
  return new Date().toLocaleTimeString(...);
}
```

> info **Hint** `@Payload()`, `@Ctx()` ve `NatsContext` `@nestjs/microservices` paketinden içe aktarılır.

> info **Hint** Ayrıca, `@Payload()` dekoratörüne gelen yük nesnesinden belirli bir özelliği çıkarmak için dekoratöre bir özellik anahtarı iletebilirsiniz, örneğin, `@Payload('id')`.

#### İstemci

Bir Nest uygulaması, `ClientProxy` sınıfını kullanarak bir Nest mikroservisi ile mesaj alışverişi yapabilir veya olaylar yayınlayabilir. Bu sınıf, `send()` (istek-cevap iletişimi için) ve `emit()` (olay tabanlı iletişim için) gibi bir dizi yöntem tanımlar ve uzak bir mikroservisle iletişim kurmanıza olanak tanır. Bu sınıfın bir örneğini aşağıdaki şekillerden biriyle elde edebilirsiniz.

Bir teknik, `ClientsModule`'u içe aktarmaktır, bu modül statik `register()` yöntemini açığa çıkarır. Bu yöntem, mikroservis taşıyıcılarını temsil eden nesnelerin bir dizisini içeren bir bağımsızlık tanımlayıcısı olarak kullanılabilen bir `name` özelliği, isteğe bağlı bir `transport` özelliği (varsayılan `Transport.TCP`'dir) ve isteğe bağlı bir taşıyıcıya özgü `options` özelliği olan her bir nesne alır.

`name` özelliği, bir enjeksiyon belirteci olarak hizmetin ihtiyaç duyulduğu yerlere bir `ClientProxy` örneğini enjekte etmek için kullanılabilir. `name` özelliğinin değeri, bir enjeksiyon belirteci olarak, burada açıklanan [burada

](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens) betimlendiği gibi herhangi bir dize veya JavaScript sembolü olabilir.

`options` özelliği, daha önce gördüğümüz gibi `createMicroservice()` yöntemine ait özelliklere sahip bir nesnedir.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      { name: 'MATH_SERVICE', transport: Transport.TCP },
    ]),
  ]
  ...
})
```

Modül içe aktarıldığında, yukarıda gösterilen `'MATH_SERVICE'` taşıyıcı seçenekleri ile yapılandırılmış bir `ClientProxy` örneğini enjekte edebiliriz, bunun için `@Inject()` dekoratörünü kullanırız.

```typescript
constructor(
  @Inject('MATH_SERVICE') private client: ClientProxy,
) {}
```

> info **Hint** `ClientsModule` ve `ClientProxy` sınıfları `@nestjs/microservices` paketinden içe aktarılır.

Bazen taşıyıcı yapılandırmasını istemci uygulamamıza sabit bir şekilde yerleştirmek yerine başka bir servisten almak isteyebiliriz (örneğin bir `ConfigService`). Bunu yapmak için, `ClientProxyFactory` sınıfını kullanarak [özel bir sağlayıcı](/docs/fundamentals/custom-providers) kaydedebiliriz. Bu sınıfın bir statik `create()` yöntemi vardır, bu yöntem bir taşıyıcı seçenekleri nesnesini kabul eder ve özelleştirilmiş bir `ClientProxy` örneği döndürür.

```typescript
@Module({
  providers: [
    {
      provide: 'MATH_SERVICE',
      useFactory: (configService: ConfigService) => {
        const mathSvcOptions = configService.getMathSvcOptions();
        return ClientProxyFactory.create(mathSvcOptions);
      },
      inject: [ConfigService],
    }
  ]
  ...
})
```

> info **Hint** `ClientProxyFactory` sınıfı `@nestjs/microservices` paketinden içe aktarılır.

Başka bir seçenek de `@Client()` özellik dekoratörünü kullanmaktır.

```typescript
@Client({ transport: Transport.TCP })
client: ClientProxy;
```

> info **Hint** `@Client()` dekoratörü `@nestjs/microservices` paketinden içe aktarılır.

`@Client()` dekoratörünü kullanmak, test etmesi ve bir istemci örneğini paylaşmak daha zor olduğu için tercih edilen teknik değildir.

`ClientProxy` **tembel**dir. Hemen bir bağlantı başlatmaz. Bunun yerine, ilk mikroservis çağrısından önce kurulacak ve ardından her bir sonraki çağrıda yeniden kullanılacaktır. Ancak, bir bağlantının kurulmasını beklemek istiyorsanız, `OnApplicationBootstrap` yaşam döngü kancası içinde `ClientProxy` nesnesinin `connect()` yöntemini manuel olarak başlatabilirsiniz.

```typescript
@@filename()
async onApplicationBootstrap() {
  await this.client.connect();
}
```

Bağlantı oluşturulamazsa, `connect()` yöntemi karşılık gelen hata nesnesi ile reddedilecektir.

#### Mesaj Gönderme

`ClientProxy`, bir `send()` yöntemini açığa çıkarır. Bu yöntem, mikroservisi çağırmak için kullanılır ve yanıtıyla bir `Observable` döner. Bu nedenle, yayılan değerlere kolayca abone olabiliriz.

```typescript
@@filename()
accumulate(): Observable<number> {
  const pattern = { cmd: 'sum' };
  const payload = [1, 2, 3];
  return this.client.send<number>(pattern, payload);
}
@@switch
accumulate() {
  const pattern = { cmd: 'sum' };
  const payload = [1, 2, 3];
  return this.client.send(pattern, payload);
}
```

`send()` yöntemi iki argüman alır, `pattern` ve `payload`. `pattern`, `@MessagePattern()` dekoratöründe tanımlanan bir desene uymalıdır. `payload`, uzak mikroservise iletmek istediğimiz bir iletişimdir. Bu yöntem, **soğuk `Observable`** döndürür, bu da mesajın gönderilmeden önce ona açıkça abone olmanız gerektiği anlamına gelir.

#### Olayları Yayınlama

Bir olay göndermek için, `ClientProxy` nesnesinin `emit()` yöntemini kullanın. Bu yöntem, bir olayı mesaj aracına yayınlar.

```typescript
@@filename()
async publish() {
  this.client.emit<number>('user_created', new UserCreatedEvent());
}
@@switch
async publish() {
  this.client.emit('user_created', new UserCreatedEvent());
}
```

`emit()` yöntemi iki argüman alır, `pattern` ve `payload`. `pattern`, `@EventPattern()` dekoratöründe tanımlanan bir desene uymalıdır. `payload`, uzak mikroservise iletmek istediğimiz bir olay yüküdür. Bu yöntem, `send()` tarafından döndürülen soğuk `Observable`'dan farklı olarak **sıcak `Observable`** döndürür, bu da olaya açıkça abone olup olmamanıza bakılmaksızın proxy'nin olayı hemen iletmeye çalışacağı anlamına gelir.

<app-banner-devtools></app-banner-devtools>

#### Kapsamlar

Farklı programlama dilinden gelen kişiler için, Nest'te neredeyse her şeyin gelen istekler arasında paylaşıldığını öğrenmek şaşırtıcı olabilir. Veritabanı için bir bağlantı havuzumuz, küresel duruma sahip tekil servisler vb. vardır. Unutmayın ki Node.js, her isteğin ayrı bir iş parçacığı tarafından işlendiği istek/yanıt çoklu iş parçacıklı durum modelini takip etmez. Bu nedenle, uygulamalarımız için tekil örnekler kullanmak tamamen **güvenlidir**.

Ancak, GraphQL uygulamalarında istek tabanlı yaşam döngüsü, istek izleme veya çok kiracılığa özel durumlar gibi durumlar için istek tabanlı yaşam döngüsü istenen davranış olabilir. Kapsamları kontrol etmeyi öğrenmek için [buraya](/docs/fundamentals/injection-scopes) bakın.

İstek tabanlı yaşam döngüsüne sahip işleyiciler ve sağlayıcılar, `@Inject()` dekoratörünü `CONTEXT` belirteci ile birleştirerek `RequestContext`'i enjekte edebilir:

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT, RequestContext } from '@nestjs/microservices';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private ctx: RequestContext) {}
}
```

Bu, `RequestContext` nesnesine erişim sağlar ve iki özelliğe sahiptir:

```typescript
export interface RequestContext<T = any> {
  pattern: string | Record<string, any>;
  data: T;
}
```

`data` özelliği, mesaj üretici tarafından gönderilen mesaj yüküdür. `pattern` özelliği, gelen mesajı işlemek için uygun bir işleyiciyi belirlemek için kullanılan desptir.

#### Zaman Aşımı İşleme

Dağıtık sistemlerde, bazen mikroservisler kapalı veya kullanılamaz olabilir. Sonsuz sürenin önüne geçmek için Zaman Aşımı kullanabilirsiniz. Zaman aşımı, diğer servislerle iletişim kurarken oldukça kullanışlı bir desenidir. Mikroservis çağrılarına zaman aşımı uygulamak için [RxJS](https://rxjs.dev) `timeout` operatörünü kullanabilirsiniz. Mikroservis belirli bir süre içinde yanıt vermezse, bir istisna fırlatılır ve bu istisna uygun şekilde yakalanabilir ve işlenebilir.

Bu sorunu çözmek için [`rxjs`](https://github.com/ReactiveX/rxjs) paketini kullanmalısınız. Pipe içinde `timeout` operatörünü kullanın:

```typescript
@@filename()
this.client
      .send<TResult, TInput>(pattern, data)
      .pipe(timeout(5000));
@@switch
this.client
      .send(pattern, data)
      .pipe(timeout(5000));
```

> info **Hint** `timeout` operatörü, `rxjs/operators` paketinden içe aktarılır.

5 saniye sonra, mikroservis yanıt vermiyorsa bir hata fırlatılacaktır.