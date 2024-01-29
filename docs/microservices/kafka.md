### Kafka

[Kafka](https://kafka.apache.org/), üç temel yeteneği olan açık kaynaklı bir dağıtılmış akış platformudur:

- Kayıtların akışlarına abone olma ve bunlara yayımlama, bir mesaj sırası veya kurumsal mesajlaşma sistemine benzer.
- Kayıtları hata tolere eden dayanıklı bir şekilde depolama.
- Kayıtların meydana geldikleri gibi işlenmesi.

Kafka projesi, gerçek zamanlı veri beslemeleriyle başa çıkma konusunda birleştirilmiş, yüksek kapasiteli, düşük gecikmeli bir platform sağlamayı amaçlamaktadır. Apache Storm ve Spark ile gerçek zamanlı akış veri analizi için çok iyi entegre olur.

#### Kurulum

Kafka tabanlı mikroservisler oluşturmaya başlamak için önce gerekli paketi kurun:

```bash
$ npm i --save kafkajs
```

#### Genel Bakış

Diğer Nest mikroservis taşıma katmanı uygulamaları gibi, Kafka taşıyıcı mekanizmasını `createMicroservice()` yöntemine iletilen seçenekler nesnesinin `transport` özelliği ile seçersiniz, isteğe bağlı bir `options` özelliği ile birlikte, aşağıda gösterildiği gibi:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    }
  }
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    }
  }
});
```

> info **İpucu** `Transport` enum, `@nestjs/microservices` paketinden içe aktarılır.

#### Seçenekler

`options` özelliği seçilen taşıyıcıya özgüdür. <strong>Kafka</strong> taşıyıcı, aşağıda açıklanan özellikleri ortaya çıkar.

<table>
  <tr>
    <td><code>client</code></td>
    <td>İstemci yapılandırma seçenekleri (daha fazla bilgi için
      <a
        href="https://kafka.js.org/docs/configuration"
        rel="nofollow"
        target="blank"
        >buraya</a>bakın)</td>
  </tr>
  <tr>
    <td>
      <code>consumer</code>
    </td>
    <td>
    Tüketici yapılandırma seçenekleri (daha fazla bilgi için
      <a
        href="https://kafka.js.org/docs/consuming#a-name-options-a-options"
        rel="nofollow"
        target="blank"
        >buraya</a>bakın)
    </td>
  </tr>
  <tr>
    <td><code>run</code></td>
    <td>Çalıştırma yapılandırma seçenekleri (daha fazla bilgi için
      <a
        href="https://kafka.js.org/docs/consuming"
        rel="nofollow"
        target="blank"
        >buraya</a>bakın)</td>
  </tr>
  <tr>
    <td><code>subscribe</code></td>
    <td>Abonelik yapılandırma seçenekleri (daha fazla bilgi için
      <a
        href="https://kafka.js.org/docs/consuming#frombeginning"
        rel="nofollow"
        target="blank"
        >buraya</a>bakın)</td>
  </tr>
  <tr>
    <td><code>producer</code></td>
    <td>Üretici yapılandırma seçenekleri (daha fazla bilgi için
      <a
        href="https://kafka.js.org/docs/producing#options"
        rel="nofollow"
        target="blank"
        >buraya</a>bakın)</td>
  </tr>
  <tr>
    <td><code>send</code></td>
    <td>Gönderme yapılandırma seçenekleri (daha fazla bilgi için
      <a
        href="https://kafka.js.org/docs/producing#options"
        rel="nofollow"
        target="blank"
        >buraya</a>bakın)</td>
  </tr>
  <tr>
    <td><code>producerOnlyMode</code></td>
    <td>Tüketici grubu kaydını atlamak ve yalnızca bir üretici olarak işlev göstermek için özellik bayrağı (<code>boolean</code>)</td>
  </tr>
  <tr>
    <td><code>postfixId</code></td>
    <td>clientId değerinin sonekini değiştirme (<code>string</code>)</td>
  </tr>
</table>

#### İstemci

Kafka'da, diğer mikroservis taşıyıcılarına kıyasla küçük bir fark vardır. `ClientProxy` sınıfı yerine `ClientKafka` sınıfını kullanıyoruz.

Diğer mikroservis taşıyıcılarıyla benzer şekilde, bir `ClientKafka` örneği oluşturmanın [birkaç seçeneğiniz](https://docs.nestjs.com/microservices/basics#client) vardır.

`ClientsModule` kullanarak bir örnek oluşturmanın bir yolu vardır. `ClientsModule` ile bir istemci örneği oluşturmak için, onu içe aktarın ve `createMicroservice()` yönteminde gösterilen aynı özelliklere sahip bir seçenek nesnesi ile birlikte kullanmak için `register()` yöntemini kullanın, ayrıca bir enjeksiyon belirteci olarak kullanılacak bir `name` özelliği de bulunmalıdır. `ClientsModule` hakkında daha fazla bilgiye [buradan](https://docs.nestjs.com/microservices/basics#client) ulaşabilirsiniz.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'HERO_SERVICE',
        transport: Transport.KAFKA,
        options: {
          client: {
            clientId: 'hero',
            brokers: ['localhost:9092'],
          },
          consumer: {
            groupId: 'hero-consumer'
          }
        }
      },
    ]),
  ]
  ...
})
```

Diğer istemci oluşturma seçenekleri (`ClientProxyFactory` veya `@Client()`) de kullanılabilir. Bunlar hakkında bilgi almak için [buraya](https://docs.nestjs.com/microservices/basics#client) göz atabilirsiniz.

`@Client()` dekoratörünü şu şekilde kullanabilirsiniz:

```typescript
@Client({
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero',
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer'
    }
  }
})
client: ClientKafka;
```

#### Mesaj deseni

Kafka mikroservis mesaj deseni, istek ve cevap kanalları için iki konuyu kullanır. `ClientKafka#send()` yöntemi, bir [geri dönüş adresi](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ReturnAddress.html) ile istek mesajına bir [ilişki kimliği](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html), cevap konusu ve cevap bölümü atanarak mesaj gönderir. Bu, `ClientKafka` örneğinin cevap konusuna abone olması ve bir mesaj göndermeden önce en az bir bölüme atanması gerektiği anlamına gelir.

Daha sonra, çalışan her Nest uygulaması için en az bir cevap konusu bölümünüz olmalıdır. Örneğin, 4 Nest uygulaması çalıştırıyorsanız ancak cevap konusunda yalnızca 3 bölüm varsa, 1 Nest uygulaması bir mesaj göndermeye çalıştığında hata alacaktır.

Yeni `ClientKafka` örnekleri başlatıldığında, tüketici grubuna katılır ve ilgili konulara abone olur. Bu süreç, tüketici grubuna ayrılan konu bölümlerini tetikler.

Normalde, konu bölümleri, yuvarlak-robin bölüleyiciyi kullanarak atanır; bu bölüleyici, uygulama başlatıldığında rastgele ayarlanan tüketici adlarına göre sıralanan bir tüketici koleksiyonuna konu bölümlerini atar. Ancak yeni bir tüketici grubuna katıldığında, yeni tüketici, tüketici koleksiyonu içinde herhangi bir konumda olabilir. Bu, yeni tüketici konumlandırıldıktan sonra önce var olan tüketiciye pozisyonlandırıldığında önceden var olan tüketicilere farklı bölümler atanabileceği bir durum yaratır. Sonuç olarak, farklı bölümlere atanmış olan tüketiciler, yeniden dengeleme yapıldığında önceki dengeleme öncesinde gönderilen isteklerin cevap mesajlarını kaybeder.

`ClientKafka` tüketicilerinin cevap mesajlarını kaybetmesini önlemek için, Nest'e özgü yerleşik özel bir

 bölüleyici kullanılır. Bu özel bölüleyici, uygulama başlatıldığında ayarlanan yüksek çözünürlüklü zaman damgalarına göre sıralanan bir tüketici koleksiyonuna konu bölümlerini atar (`process.hrtime()` kullanılarak). 

#### Mesaj cevap aboneliği

> warning **Not** Bu bölüm, [istek-cevap](https://docs.nestjs.com/microservices/basics#request-response) mesaj stiline ( `@MessagePattern` dekoratörü ve `ClientKafka#send` yöntemi ile) geçerlidir. Cevap konusuna abone olmak, [olay temelli](https://docs.nestjs.com/microservices/basics#event-based) iletişim için (`@EventPattern` dekoratörü ve `ClientKafka#emit` yöntemi) gerekli değildir.

`ClientKafka` sınıfı, `subscribeToResponseOf()` yöntemini sağlar. `subscribeToResponseOf()` yöntemi, bir isteğin konu adını argüman olarak alır ve türetilmiş cevap konu adını bir cevap konuları koleksiyonuna ekler. Bu yöntem, mesaj deseni uygulandığında gereklidir.

```typescript
@@filename(heroes.controller)
onModuleInit() {
  this.client.subscribeToResponseOf('hero.kill.dragon');
}
```

Eğer `ClientKafka` örneği asenkron olarak oluşturuluyorsa, `subscribeToResponseOf()` yöntemi `connect()` yöntemini çağırmadan önce çağrılmalıdır.

```typescript
@@filename(heroes.controller)
async onModuleInit() {
  this.client.subscribeToResponseOf('hero.kill.dragon');
  await this.client.connect();
}
```

#### Gelen

Nest, Kafka mesajlarını `key`, `value` ve `headers` özelliklerini içeren bir nesne olarak alır. Bu özelliklerin değerleri `Buffer` türündedir. Nest, bu değerleri dönüştürerek kullanır. Eğer string "obje benzeri" ise, Nest bu string'i `JSON` olarak ayrıştırmaya çalışır. `value`, ilişkilendirilmiş olan işleyiciye iletilir.

#### Giden

Nest, olayları yayınlarken veya mesajlar gönderirken çıkış Kafka mesajlarını bir seri işlem sürecinden geçirdikten sonra gönderir. Bu işlem, `ClientKafka` `emit()` ve `send()` yöntemlerine iletilen argümanlarda veya bir `@MessagePattern` yönteminden dönen değerlerde meydana gelir. Bu seri işlem, string veya buffer olmayan nesneleri `JSON.stringify()` veya `toString()` prototip yöntemi kullanarak "stringe" dönüştürür.

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const dragonId = message.dragonId;
    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];
    return items;
  }
}
```

> info **Hint** `@Payload()` `@nestjs/microservices` paketinden içe aktarılmıştır.

Çıkış mesajları ayrıca `key` ve `value` özelliklerine sahip bir nesne ile anahtarlandırılabilir. Mesajları anahtarlamak, [eş bölümele gereksinimi](https://docs.confluent.io/current/ksql/docs/developer-guide/partition-data.html#co-partitioning-requirements) karşılamak için önemlidir.

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const realm = 'Nest';
    const heroId = message.heroId;
    const dragonId = message.dragonId;

    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];

    return {
      headers: {
        realm
      },
      key: heroId,
      value: items
    }
  }
}
```

Ayrıca, bu formatta iletilen mesajlar, `headers` özelliğinde ayarlanan özel başlıkları da içerebilir. Başlık özelliği değerleri `string` veya `Buffer` türünde olmalıdır.

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const realm = 'Nest';
    const heroId = message.heroId;
    const dragonId = message.dragonId;

    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];

    return {
      headers: {
        kafka_nestRealm: realm
      },
      key: heroId,
      value: items
    }
  }
}
```

#### Olay Temelli

İstek-cevap yöntemi, hizmetler arasında mesaj alışverişi için ideal olsa da, mesaj stilinizin olay tabanlı olduğu durumlarda (ki bu durumda Kafka için idealdir) - yalnızca **bir yanıt beklemeksizin** olayları yayınlamak istediğinizde - bu yöntem daha az uygun olabilir. Bu durumda, iki konuyu sürdürmek için istek-cevap için gereken aşırı başa çıkma maliyetini istemezsiniz.

Bu konuda daha fazla bilgi edinmek için bu iki bölüme göz atın: [Genel Bakış: Olay Tabanlı](/docs/microservices/basics#event-based) ve [Genel Bakış: Olayları Yayınlama](/docs/microservices/basics#publishing-events).

#### Bağlam

Daha sofistike senaryolarda, gelen isteğe dair daha fazla bilgiye erişmek isteyebilirsiniz. Kafka taşıyıcısını kullanırken, `KafkaContext` nesnesine erişebilirsiniz.

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  console.log(`Konu: ${context.getTopic()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('hero.kill.dragon')
killDragon(message, context) {
  console.log(`Konu: ${context.getTopic()}`);
}
```

> info **Hint** `@Payload()`, `@Ctx()` ve `KafkaContext` `@nestjs/microservices` paketinden içe aktarılmıştır.

Kafka `IncomingMessage` nesnesine erişmek için `KafkaContext` nesnesinin `getMessage()` yöntemini kullanın:

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  const originalMessage = context.getMessage();
  const partition = context.getPartition();
  const { headers, timestamp } = originalMessage;
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('hero.kill.dragon')
killDragon(message, context) {
  const originalMessage = context.getMessage();
  const partition = context.getPartition();
  const { headers, timestamp } = originalMessage;
}
```

Burada `IncomingMessage`, aşağıdaki arabirimi karşılar:

```typescript
interface IncomingMessage {
  topic: string;
  partition: number;
  timestamp: string;
  size: number;
  attributes: number;
  offset: string;
  key: any;
  value: any;
  headers: Record<string, any>;
}
```

Eğer işleyiciniz her bir alınan mesaj için yavaş işleme süresi içeriyorsa, `heartbeat` geri çağrısını kullanmayı düşünmelisiniz. `KafkaContext`'in `getHeartbeat()` yöntemini kullanarak `heartbeat` fonksiyonunu alabilirsiniz:

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
async killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  const heartbeat = context.getHeartbeat();

  // Yavaş işleme yap
  await doWorkPart1();

  // sessionTimeout'u aşmamak için heartbeat gönder
  await heartbeat();

  // Biraz daha yavaş işleme yap
  await doWorkPart2();
}
```

#### İsimlendirme Kuralları

Kafka mikroservis bileşenleri, çakışmaları önlemek için `client.clientId` ve `consumer.groupId` seçeneklerine kendi rol açıklamalarını ekler. Varsayılan olarak, `ClientKafka` bileşenleri bu seçeneklere `-client` ekler ve `ServerKafka` bileşenleri her iki seçeneğe de `-server` ekler. Aşağıda verilen değerlerin bu şekilde nasıl dönüştürüldüğüne dikkat edin (yorumlarda gösterildiği gibi).

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero', // hero-server
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer' // hero-consumer-server
    },
  }
});
```

Ve istemci için:

```typescript
@@filename(heroes.controller)
@Client({
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero', // hero-client
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer' // hero-consumer-client
    }
  }
})
client: ClientKafka;
```

> info **İpucu** Kafka istemci ve tüketici isimlendirme kuralları, `ClientKafka` ve `KafkaServer`'ı kendi özel sağlayıcınızda genişleterek ve yapıcıyı geçersiz kılarak özelleştirilebilir.

Kafka mikroservis mesaj deseni, istek ve yanıt kanalları için iki konu kullanır, bu nedenle yanıt deseni, istek konusundan türetilmelidir. Varsayılan olarak, yanıt konusunun adı, istek konusunun adına `.reply` eklendiği bileşik bir isimdir.

```typescript
@@filename(heroes.controller)
onModuleInit() {
  this.client.subscribeToResponseOf('hero.get'); // hero.get.reply
}
```

> info **İpucu** Kafka yanıt konusu adlandırma kuralları, `ClientKafka`'yı kendi özel sağlayıcınızda genişleterek ve `getResponsePatternName` yöntemini geçersiz kılarak özelleştirilebilir.

#### Tekrarlanabilir istisnalar

Diğer taşıyıcılar gibi, ele alınmayan tüm istisnalar otomatik olarak bir `RpcException` içine sarılır ve "kullanıcı dostu" bir formata dönüştürülür. Ancak, istisna yönetim mekanizmasını atlamak ve istisnaların `kafkajs` sürücüsü tarafından tüketilmesine izin vermek isteyebileceğiniz durumlar vardır. Bir mesaj işlenirken bir istisna fırlatmak, `kafkajs`'e bunu **yeniden denemesini** (tekrar teslim etmesini) bildirir; bu da, mesaj (veya olay) işleyicisi tetiklendiği halde ofsetin Kafka'ya taahhüt edilmeyeceği anlamına gelir.

> uyarı **Uyarı** Olay işleyicileri için (olay tabanlı iletişim için), varsayılan olarak tüm ele alınmayan istisnalar **tekrarlanabilir istisnalar** olarak kabul edilir.

Bunun için `KafkaRetriableException` adlı özel bir sınıfı kullanabilirsiniz, şu şekilde:

```typescript
throw new KafkaRetriableException('...');
```

> info **İpucu** `KafkaRetriableException` sınıfı, `@nestjs/microservices` paketinden içe aktarılır.

#### Ofsetleri Taahhüt Etme

Kafka ile çalışırken ofsetleri taahhüt etmek önemlidir. Varsayılan olarak, mesajlar belirli bir süre sonra otomatik olarak taahhüt edilecektir. Daha fazla bilgi için [KafkaJS belgelerini](https://kafka.js.org/docs/consuming#autocommit) ziyaret edebilirsiniz. `ClientKafka`, ofsetleri [yerel KafkaJS uygulaması gibi](https://kafka.js.org/docs/consuming#manual-committing) manuel olarak taahhüt etme olanağı sunar.

```typescript
@@filename()
@EventPattern('user.created')
async handleUserCreated(@Payload() data: IncomingMessage, @Ctx() context: KafkaContext) {
  // iş mantığı
  
  const { offset } = context.getMessage();
  const partition = context.getPartition();
  const topic = context.getTopic();
  await this.client.commitOffsets([{ topic, partition, offset }])
}
@@switch
@Bind(Payload(), Ctx())
@EventPattern('user.created')
async handleUserCreated(data, context) {
  // iş mantığı

  const { offset } = context.getMessage();
  const partition = context.getPartition();
  const topic = context.getTopic();
  await this.client.commitOffsets([{ topic, partition, offset }])
}
```

Mesajların otomatik taahhüt edilmesini devre dışı bırakmak için `run` yapılandırmasında `autoCommit: false` olarak ayarlayın, aşağıdaki gibi:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    },
    run: {
      autoCommit: false
    }
  }
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    },
    run: {
      autoCommit: false
    }
  }
});
```
