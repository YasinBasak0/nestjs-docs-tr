### gRPC

[gRPC](https://github.com/grpc/grpc-node), modern, açık kaynaklı, yüksek performanslı bir RPC çerçevesidir ve herhangi bir ortamda çalışabilir. Pluggable load balancing, izleme, sağlık kontrolü ve kimlik doğrulama gibi özelliklere sahip, veri merkezleri arasında hızlı bir şekilde hizmetleri bağlama yeteneğine sahiptir.

gRPC gibi birçok RPC sistemi gibi, gRPC, uzaktan çağrılabilir fonksiyonlar (metodlar) kavramına dayanır. Her metod için parametreleri ve dönüş türlerini tanımlarsınız. Hizmetler, parametreler ve dönüş türleri, Google'ın açık kaynaklı dil bağımsız [protocol buffers](https://protobuf.dev) mekanizması kullanılarak `.proto` dosyalarında tanımlanır.

Nest, gRPC transporter'ını kullanarak istemcileri ve sunucuları dinamik olarak bağlamak için `.proto` dosyalarını kullanır ve uzaktan prosedür çağrılarını uygulamak için kolay bir yol sağlar, veri yapılarını otomatik olarak seri hale getirip seri hale getirir.

#### Kurulum

gRPC tabanlı mikroservisler oluşturmaya başlamak için önce gerekli paketleri kurun:

```bash
$ npm i --save @grpc/grpc-js @grpc/proto-loader
```

#### Genel Bakış

Diğer Nest mikroservis taşıma katmanı uygulamaları gibi, gRPC taşıma mekanizmasını `createMicroservice()` yöntemine iletilen seçenek nesnesinin `transport` özelliği ile seçersiniz. Aşağıdaki örnekte bir kahraman servisi kuracağız. `options` özelliği bu servis hakkında meta bilgi sağlar; özellikleri [aşağıda](#options) açıklanmıştır.

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.GRPC,
  options: {
    package: 'hero',
    protoPath: join(__dirname, 'hero/hero.proto'),
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.GRPC,
  options: {
    package: 'hero',
    protoPath: join(__dirname, 'hero/hero.proto'),
  },
});
```

> info **Hint** `join()` fonksiyonu, `path` paketinden içe aktarılır; `Transport` enum'u `@nestjs/microservices` paketinden içe aktarılır.

`nest-cli.json` dosyasına, `assets` özelliğini ekleriz. Bu özellik, TypeScript olmayan dosyaları dağıtmamıza olanak tanır ve `watchAssets` - tüm TypeScript olmayan varlıkları izlemeyi açmak için kullanılır. Bizim durumumuzda, `.proto` dosyalarının otomatik olarak `dist` klasörüne kopyalanmasını istiyoruz.

```json
{
  "compilerOptions": {
    "assets": ["**/*.proto"],
    "watchAssets": true
  }
}
```

#### Options

<strong>gRPC</strong> taşıma nesnesi seçenek nesnesi aşağıda açıklanan özelliklere sahiptir.

<table>
  <tr>
    <td><code>package</code></td>
    <td>Protobuf paket adı (<code>.proto</code> dosyasındaki <code>package</code> ayarı ile eşleşir).  Gerekli</td>
  </tr>
  <tr>
    <td><code>protoPath</code></td>
    <td>
      <code>.proto</code> dosyasının mutlak (veya kök dizine göre göreceli) yoludur.
      Gerekli
    </td>
  </tr>
  <tr>
    <td><code>url</code></td>
    <td>Bağlantı URL'si. Bağlantıyı kurduğunuz adresi/portu tanımlayan <code>ip adresi/dns adı:port</code> formatındaki bir dizedir (örneğin, <code>'localhost:50051'</code>). İsteğe bağlı. Varsayılan değeri <code>'localhost:5000'</code></td>
  </tr>
  <tr>
    <td><code>protoLoader</code></td>
    <td><code>.proto</code> dosyalarını yüklemek için yardımcı programın NPM paketi adıdır. İsteğe bağlı. Varsayılan değeri <code>'@grpc/proto-loader'</code></td>
  </tr>
  <tr>
    <td><code>loader</code></td>
    <td>
      <code>@grpc/proto-loader</code> seçenekleri. Bu, <code>.proto</code> dosyalarının davranışı üzerinde detaylı kontrol sağlar. İsteğe bağlı. Daha fazla ayrıntı için
      <a
        href="https://github.com/grpc/grpc-node/blob/master/packages/proto-loader/README.md"
        rel="nofollow"
        target="_blank"
        >buraya</a> bakın
    </td>
  </tr>
  <tr>
    <td><code>credentials</code></td>
    <td>
      Sunucu kimlik bilgileri. İsteğe bağlı.
      <a
        href="https://grpc.io/grpc/node/grpc.ServerCredentials.html"
        rel="nofollow"
        target="_blank"
        >Daha fazlası için buraya bakın</a>
    </td>
  </tr>
</table>

### Örnek gRPC Servisi

Şimdi `HeroesService` adlı örnek bir gRPC servisimizi tanımlayalım. Yukarıdaki `options` nesnesinde, `protoPath` özelliği `.proto` tanımlama dosyası `hero.proto`'ya bir yol belirler. `hero.proto` dosyası, <a href="https://developers.google.com/protocol-buffers">protocol buffers</a> kullanılarak yapılandırılmıştır. İşte bu dosyanın içeriği:

```typescript
// hero/hero.proto
syntax = "proto3";

package hero;

service HeroesService {
  rpc FindOne (HeroById) returns (Hero) {}
}

message HeroById {
  int32 id = 1;
}

message Hero {
  int32 id = 1;
  string name = 2;
}
```

`HeroesService` servisimiz bir `FindOne()` metodunu açığa çıkarır. Bu metod, `HeroById` türünde bir giriş argümanını bekler ve bir `Hero` mesajını döndürür (protocol buffers, parametre türlerini ve dönüş türlerini tanımlamak için `message` öğelerini kullanır).

Daha sonra, servisi uygulamamız gerekiyor. Bu tanımı karşılamak için bir işleyiciyi tanımlamak için, aşağıdaki gibi bir denetleyicide `@GrpcMethod()` dekoratörünü kullanırız. Bu dekoratör, bir yöntemi gRPC servis yöntemi olarak ilan etmek için gereken metaveriyi sağlar.

> info **Hint** Önceki mikroservis bölümlerinde tanıtılan `@MessagePattern()` dekoratörü (<a href="microservices/basics#request-response">daha fazlası için</a>) gRPC tabanlı mikroservislerde kullanılmaz. `@GrpcMethod()` dekoratörü, gRPC tabanlı mikroservisler için bu dekoratörün yerini alır.

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService', 'FindOne')
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService', 'FindOne')
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

> info **Hint** `@GrpcMethod()` dekoratörü `@nestjs/microservices` paketinden içe aktarılırken, `Metadata` ve `ServerUnaryCall` paketinden içe aktarılır.

Yukarıda gösterilen dekoratör iki argüman alır. İlk olarak servis adını alır (örneğin, `'HeroesService'`), `hero.proto`'daki `HeroesService` servisi tanımına karşılık gelir. İkinci argüman (dize `'FindOne'`), `hero.proto` dosyasındaki `HeroesService` içindeki `FindOne()` rpc metoduna karşılık gelir.

`findOne()` işleyici yöntemi üç argüman alır: çağrıdan geçen `data`, gRPC talebi metadata'sını depolayan `metadata` ve `call` ile `GrpcCall` nesnesinin özelliklerini almak için. 

`@GrpcMethod()` dekoratörü argümanları her ikisi de isteğe bağlıdır. İkinci argüman olmadan çağrıldığında (örneğin, `'FindOne'`), Nest, otomatik olarak işleyici adını büyük kılavuz harfliye çevirerek `.proto` dosyasındaki rpc yöntemini işleyici ile ilişkilendirecektir (örneğin, `findOne` işleyicisi, `FindOne` rpc çağrısı tanımı ile ilişkilendirilir). Bu, aşağıda gösterildiği gibi bir `@GrpcMethod()` dekoratörü çağrısı ile mümkündür.

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService')
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any

>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService')
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

Ayrıca ilk `@GrpcMethod()` argümanını da atlayabilirsiniz. Bu durumda, Nest, işleyiciyi proto tanımlama dosyalarındaki **class** ismiyle eşleştirecektir. Örneğin, aşağıdaki kodda, `HeroesService` sınıfı, handler yöntemlerini `hero.proto` dosyasındaki `HeroesService` servisi tanımı ile eşleştirmek için isim `'HeroesService'` ile eşleşme yapar.

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

### İstemci

Nest uygulamaları, `.proto` dosyalarında tanımlanan servisleri tüketen gRPC istemcisi olarak hareket edebilir. Uzak servislere, `ClientGrpc` nesnesi üzerinden erişirsiniz. Bir `ClientGrpc` nesnesini elde etmek için birkaç yol vardır.

Tercih edilen teknik, `ClientsModule`'u içe aktarmaktır. `register()` yöntemini kullanarak bir `.proto` dosyasında tanımlanan bir hizmet paketini bir enjeksiyon belirteci ile bağlamak ve hizmeti yapılandırmak için kullanılır. `name` özelliği enjeksiyon belirtecini temsil eder. gRPC servisleri için `transport: Transport.GRPC` kullanın. `options` özelliği, yukarıda [tanımlanan](#options) özelliklere sahip bir nesnedir.

```typescript
imports: [
  ClientsModule.register([
    {
      name: 'HERO_PACKAGE',
      transport: Transport.GRPC,
      options: {
        package: 'hero',
        protoPath: join(__dirname, 'hero/hero.proto'),
      },
    },
  ]),
];
```

> info **Hint** `register()` yöntemi bir dizi nesne alır. Birden çok paketi kaydetmek için bir virgülle ayrılmış kayıt nesneleri listesi sağlayarak yapılır.

Bir kez kaydedildikten sonra yapılandırılmış `ClientGrpc` nesnesini `@Inject()` kullanarak enjekte edebiliriz. Ardından `ClientGrpc` nesnesinin `getService()` yöntemini kullanarak hizmet örneğini alırız, aşağıda gösterildiği gibi.

```typescript
@Injectable()
export class AppService implements OnModuleInit {
  private heroesService: HeroesService;

  constructor(@Inject('HERO_PACKAGE') private client: ClientGrpc) {}

  onModuleInit() {
    this.heroesService = this.client.getService<HeroesService>('HeroesService');
  }

  getHero(): Observable<string> {
    return this.heroesService.findOne({ id: 1 });
  }
}
```

> error **Uyarı** gRPC İstemcisi, alt çizgi `_` içeren alanları içeren alanları içermeyen alanları göndermeyecektir, bu alandaki alanlar mikroservis taşıyıcı yapılandırmasındaki (`options.loader.keepcase` mikroservis taşıyıcı yapılandırmasındaki `true` olması durumunda) `keepCase` seçeneği ayarlanmadığı sürece.

Bu teknik, diğer mikroservis taşıma yöntemlerinde kullanılan teknikle karşılaştırıldığında küçük bir fark vardır. `ClientProxy` sınıfı yerine, `getService()` yöntemini sağlayan `ClientGrpc` sınıfını kullanırız. `getService()` generic yöntemi, bir servis adını argüman olarak alır ve mevcutsa onun örneğini döndürür.

Alternatif olarak, bir `ClientGrpc` nesnesi oluşturmak için `@Client()` dekoratörünü kullanabilirsiniz, aşağıdaki gibi:

```typescript
@Injectable()
export class AppService implements OnModuleInit {
  @Client({
    transport: Transport.GRPC,
    options: {
      package: 'hero',
      protoPath: join(__dirname, 'hero/hero.proto'),
    },
  })
  client: ClientGrpc;

  private heroesService: HeroesService;

  onModuleInit() {
    this.heroesService = this.client.getService<HeroesService>('HeroesService');
  }

  getHero(): Observable<string> {
    return this.heroesService.findOne({ id: 1 });
  }
}
```

Son olarak, daha karmaşık senaryolarda [ClientProxyFactory](/docs/microservices/basics#client) sınıfını kullanarak dinamik olarak yapılandırılmış bir istemci enjekte edebiliriz.

Her iki durumda da `heroesService` adlı referansa sahibiz ve bu referans, `.proto` dosyasında tanımlanan metodlar kümesini açığa çıkarır. Şimdi bu proxy nesnesine eriştiğimizde (yani, `heroesService`), gRPC sistemi otomatik olarak istekleri serileştirir, uzak sistemlere iletir, bir yanıt döndürür ve yanıtı deserialize eder. gRPC, bu ağ iletişimi detaylarından bizi koruduğu için, `heroesService`, yerel bir sağlayıcı gibi görünür ve çalışır.

Not olarak, tüm servis metodları **alt camel case**'de bulunur (dilin doğal sözdizimini takip etmek için). Örneğin, `.proto` dosyamızdaki `HeroesService` tanımı `FindOne()` fonksiyonunu içerse de, `heroesService` örneği `findOne()` metodunu sağlayacaktır.

```typescript
interface HeroesService {
  findOne(data: { id: number }): Observable<any>;
}
```

Bir mesaj işleyicisi ayrıca bir `Observable` döndürebilir, bu durumda sonuç değerleri akıtılacaktır, akış tamamlandığında. 

```typescript
@@filename(heroes.controller)
@Get()
call(): Observable<any> {
  return this.heroesService.findOne({

 id: 1 });
}
@@switch
@Get()
call() {
  return this.heroesService.findOne({ id: 1 });
}
```

gRPC metadata'sını (istekle birlikte) göndermek için ikinci bir argüman geçebilirsiniz:

```typescript
call(): Observable<any> {
  const metadata = new Metadata();
  metadata.add('Set-Cookie', 'yummy_cookie=choco');

  return this.heroesService.findOne({ id: 1 }, metadata);
}
```

> info **Hint** `Metadata` sınıfı, `grpc` paketinden içeri aktarılmıştır. 

Lütfen bu durumun, birkaç adım önce tanımladığımız `HeroesService` arabirimini güncellemeyi gerektireceğini unutmayın.

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/04-grpc) bulunabilir.

#### gRPC Streaming

gRPC, kendi başına uzun süreli canlı bağlantıları destekler, genellikle `streams` olarak bilinir. Akışlar, Sohbet, Gözlemler veya Parça veri transferi gibi durumlar için kullanışlıdır. Daha fazla ayrıntı için [buradaki](https://grpc.io/docs/guides/concepts/) resmi belgelere bakın.

Nest, GRPC akış işleyicilerini iki farklı şekilde destekler:

- RxJS `Subject` + `Observable` işleyici: Cevapları doğrudan bir Denetleyici yöntemi içine yazmak veya `Subject`/`Observable` tüketiciye iletmek için kullanışlı olabilir.
- Saf GRPC çağrısı akış işleyicisi: Node standardı `Duplex` akış işleyici için geri kalanını işleyecek bir yürütücüye geçirilmek için kullanışlı olabilir.

<app-banner-enterprise></app-banner-enterprise>

#### Akış örneği

Yeni bir gRPC servisi olan `HelloService` adında bir örnek servisi tanımlayalım. `hello.proto` dosyası, <a href="https://developers.google.com/protocol-buffers">protocol buffers</a> kullanılarak yapılandırılmıştır. İşte nasıl göründüğü:

```typescript
// hello/hello.proto
syntax = "proto3";

package hello;

service HelloService {
  rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```

> info **Hint** `LotsOfGreetings` yöntemi, döndürülen akışın birden çok değer gönderebileceği için (yukarıdaki örneklerde olduğu gibi) basitçe `@GrpcMethod` dekoratörü ile uygulanabilir.

Bu `.proto` dosyasına dayanarak, `HelloService` arabirimini tanımlayalım:

```typescript
interface HelloService {
  bidiHello(upstream: Observable<HelloRequest>): Observable<HelloResponse>;
  lotsOfGreetings(
    upstream: Observable<HelloRequest>,
  ): Observable<HelloResponse>;
}

interface HelloRequest {
  greeting: string;
}

interface HelloResponse {
  reply: string;
}
```

> info **Hint** Proto arabirimi otomatik olarak [ts-proto](https://github.com/stephenh/ts-proto) paketi tarafından oluşturulabilir, daha fazla bilgi [burada](https://github.com/stephenh/ts-proto/blob/main/NESTJS.markdown)'yı öğrenin.

#### Subject stratejisi

`@GrpcStreamMethod()` dekoratörü, işlev parametresini bir RxJS `Observable` olarak sağlar. Bu sayede çoklu mesajları alabilir ve işleyebiliriz.

```typescript
@GrpcStreamMethod()
bidiHello(messages: Observable<any>, metadata: Metadata, call: ServerDuplexStream<any, any>): Observable<any> {
  const subject = new Subject();

  const onNext = message => {
    console.log(message);
    subject.next({
      reply: 'Hello, world!'
    });
  };
  const onComplete = () => subject.complete();
  messages.subscribe({
    next: onNext,
    complete: onComplete,
  });


  return subject.asObservable();
}
```

> warning **Uyarı** `@GrpcStreamMethod()` dekoratörü ile tam çift yönlü etkileşimi desteklemek için, denetleyici yönteminin bir RxJS `Observable` döndürmesi gerekir.

> info **Hint** `Metadata` ve `ServerUnaryCall` sınıfları/arayüzleri `grpc` paketinden içeri aktarılır.

Servis tanımına göre (`.proto` dosyasında), `BidiHello` yöntemi istekleri akışa sokması gerekiyor. Bir istemciden akışa çoklu asenkron mesaj göndermek için bir RxJS `ReplaySubject` sınıfını kullanıyoruz.

```typescript
const helloService = this.client.getService<HelloService>('HelloService');
const helloRequest$ = new ReplaySubject<HelloRequest>();

helloRequest$.next({ greeting: 'Hello (1)!' });
helloRequest$.next({ greeting: 'Hello (2)!' });
helloRequest$.complete();

return helloService.b

idiHello(helloRequest$);
```

Yukarıdaki örnekte, iki mesajı akışa yazdık (`next()` çağrıları) ve verilerin göndermeyi tamamladığımızı servise bildirdik (`complete()` çağrısı).

#### Çağrı akış işleyici

Yöntem dönüş değeri `stream` olarak tanımlandığında, `@GrpcStreamCall()` dekoratörü işlev parametresini `grpc.ServerDuplexStream` olarak sağlar, bu da `.on('data', callback)`, `.write(message)` veya `.cancel()` gibi standart yöntemleri destekler. Kullanılabilir yöntemler hakkında tam belgeler [burada](https://grpc.github.io/grpc/node/grpc-ClientDuplexStream.html) bulunabilir.

Öte yandan, yöntem dönüş değeri `stream` değilse, `@GrpcStreamCall()` dekoratörü sırasıyla `grpc.ServerReadableStream` (daha fazla bilgi için [buraya](https://grpc.github.io/grpc/node/grpc-ServerReadableStream.html)) ve `callback` olarak iki işlev parametresini sağlar.

`BidiHello`'yu uygulamakla başlayalım, tam çift yönlü etkileşimi desteklemesi gerekiyor.

```typescript
@GrpcStreamCall()
bidiHello(requestStream: any) {
  requestStream.on('data', message => {
    console.log(message);
    requestStream.write({
      reply: 'Hello, world!'
    });
  });
}
```

> info **Hint** Bu dekoratör, belirli bir geri dönüş parametresinin sağlanmasını gerektirmez. Akışın, diğer standart akış türleri gibi işleneceği beklenmektedir.

Yukarıdaki örnekte, `write()` yöntemini kullanarak yanıt akışına nesneleri yazmak için kullandık. `.on()` yöntemine ikinci bir parametre olarak geçirilen geri arama, servisimiz yeni bir veri parçası aldığında her seferinde çağrılır.

Şimdi, `LotsOfGreetings` yöntemini uygulayalım.

```typescript
@GrpcStreamCall()
lotsOfGreetings(requestStream: any, callback: (err: unknown, value: HelloResponse) => void) {
  requestStream.on('data', message => {
    console.log(message);
  });
  requestStream.on('end', () => callback(null, { reply: 'Hello, world!' }));
}
```

Burada, `callback` fonksiyonunu, `requestStream` işleme tamamlandığında yanıtı göndermek için kullandık.

#### gRPC Metadata

Metadata, bir RPC çağrısıyla ilgili bilgileri içeren, anahtar-değer çiftlerinden oluşan bir liste biçimindedir. Anahtarlar genellikle dize ve değerler genellikle dize olsa da ikili veri olabilir. Metadata, gRPC tarafından kendisi için opak olarak kabul edilir - istemcinin çağrıyla ilişkilendirdiği bilgileri sunucuya ve tersi şekilde sağlar. Metadata, genellikle kimlik doğrulama token'ları, istek tanımlayıcılar ve izleme amacıyla etiketler ve bir veri kümesindeki kayıt sayısı gibi veri bilgilerini içerebilir.

`@GrpcMethod()` işleyicisinde metadata'yı okumak için ikinci argümanı (metadata) kullanın; bu, `Metadata` türündedir (`grpc` paketinden içe aktarılmıştır).

Handler'dan metadata göndermek için `ServerUnaryCall#sendMetadata()` yöntemini kullanın (üçüncü handler argümanı).

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const serverMetadata = new Metadata();
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];

    serverMetadata.add('Set-Cookie', 'yummy_cookie=choco');
    call.sendMetadata(serverMetadata);

    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data, metadata, call) {
    const serverMetadata = new Metadata();
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];

    serverMetadata.add('Set-Cookie', 'yummy_cookie=choco');
    call.sendMetadata(serverMetadata);

    return items.find(({ id }) => id === data.id);
  }
}
```

Benzer şekilde, `@GrpcStreamMethod()` işareti ile işaretlenmiş işleyicilerde metadata'yı okumak için ikinci argümanı (metadata) kullanın; bu, `Metadata` türündedir (`grpc` paketinden içe aktarılmıştır).

Handler'dan metadata göndermek için `ServerDuplexStream#sendMetadata()` yöntemini kullanın (üçüncü handler argümanı).

[Çağrı akışı işleyicileri](microservices/grpc#call-stream-handler) içinde metadata'yı okumak için `requestStream` referansında `metadata` etkinliğini dinleyin, aşağıdaki gibi:

```typescript
requestStream.on('metadata', (metadata: Metadata) => {
  const meta = metadata.get('X-Meta');
});
```
