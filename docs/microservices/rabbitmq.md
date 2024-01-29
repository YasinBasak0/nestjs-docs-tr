### RabbitMQ

[RabbitMQ](https://www.rabbitmq.com/) açık kaynaklı ve hafif bir mesaj aracıdır ve çoklu iletişim protokollerini destekler. Yüksek ölçekli, yüksek kullanılabilirlik gereksinimlerini karşılamak için dağıtılmış ve federatif yapılandırmalarda kullanılabilir. Ayrıca, dünya genelinde küçük başlangıç ​​şirketlerinden büyük kuruluşlara kadar yaygın bir şekilde kullanılan bir mesaj aracıdır.

#### Kurulum

RabbitMQ tabanlı mikroservisler oluşturmaya başlamak için önce gerekli paketleri yükleyin:

```bash
$ npm i --save amqplib amqp-connection-manager
```

#### Genel Bakış

RabbitMQ taşıyıcısını kullanmak için, aşağıdaki seçenekleri `createMicroservice()` yöntemine ileten bir options nesnesi oluşturun:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.RMQ,
  options: {
    urls: ['amqp://localhost:5672'],
    queue: 'cats_queue',
    queueOptions: {
      durable: false
    },
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.RMQ,
  options: {
    urls: ['amqp://localhost:5672'],
    queue: 'cats_queue',
    queueOptions: {
      durable: false
    },
  },
});
```

> info **İpucu** `Transport` enum'u `@nestjs/microservices` paketinden içe aktarılır.

#### Seçenekler

`options` özelliği seçilen taşıyıcıya özeldir. <strong>RabbitMQ</strong> taşıyıcısı aşağıda açıklanan özellikleri sunar.

<table>
  <tr>
    <td><code>urls</code></td>
    <td>Bağlantı url'leri</td>
  </tr>
  <tr>
    <td><code>queue</code></td>
    <td>Sunucunuzun dinleyeceği kuyruk adı</td>
  </tr>
  <tr>
    <td><code>prefetchCount</code></td>
    <td>Kanal için önişlem sayısını ayarlar</td>
  </tr>
  <tr>
    <td><code>isGlobalPrefetchCount</code></td>
    <td>Kanal başına önişleme etkinleştirir</td>
  </tr>
  <tr>
    <td><code>noAck</code></td>
    <td>Eğer <code>false</code> ise, manuel onay modu etkinleştirilir</td>
  </tr>
  <tr>
    <td><code>queueOptions</code></td>
    <td>Ek kuyruk seçenekleri (daha fazla bilgi için <a href="https://www.squaremobius.net/amqp.node/channel_api.html#channel_assertQueue" rel="nofollow" target="_blank">buraya</a> bakın)</td>
  </tr>
  <tr>
    <td><code>socketOptions</code></td>
    <td>Ek soket seçenekleri (daha fazla bilgi için <a href="https://www.squaremobius.net/amqp.node/channel_api.html#socket-options" rel="nofollow" target="_blank">buraya</a> bakın)</td>
  </tr>
  <tr>
    <td><code>headers</code></td>
    <td>Her mesajla gönderilecek başlıklar</td>
  </tr>
</table>

#### İstemci

Diğer mikroservis taşıyıcıları gibi, RabbitMQ `ClientProxy` örneği oluşturmak için <a href="https://docs.nestjs.com/microservices/basics#client">birkaç seçeneğiniz</a> vardır.

Bir örnek oluşturmanın bir yöntemi `ClientsModule`'u kullanmaktır. `ClientsModule` ile bir istemci örneği oluşturmak için, bunu içe aktarın ve `createMicroservice()` yöntemde gösterilen aynı özelliklere sahip bir options nesnesini geçmek için `register()` yöntemini kullanın. `ClientsModule` hakkında daha fazla bilgiye <a href="https://docs.nestjs.com/microservices/basics#client">buradan</a> ulaşabilirsiniz.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport

: Transport.RMQ,
        options: {
          urls: ['amqp://localhost:5672'],
          queue: 'cats_queue',
          queueOptions: {
            durable: false
          },
        },
      },
    ]),
  ]
  ...
})
```

Diğer bir seçenek, bir istemci oluşturmak için (`ClientProxyFactory` veya `@Client()`) kullanılabilir. Bunlar hakkında daha fazla bilgiye <a href="https://docs.nestjs.com/microservices/basics#client">buradan</a> ulaşabilirsiniz.

#### Bağlam

Daha karmaşık senaryolarda, gelen istekle ilgili daha fazla bilgiye erişmek isteyebilirsiniz. RabbitMQ taşıyıcısını kullanırken, `RmqContext` nesnesine erişebilirsiniz.

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(`Pattern: ${context.getPattern()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Pattern: ${context.getPattern()}`);
}
```

> info **İpucu** `@Payload()`, `@Ctx()` ve `RmqContext` `@nestjs/microservices` paketinden içe aktarılır.

RabbitMQ [mesajlarını](https://www.rabbitmq.com/channels.html) (`properties`, `fields` ve `content` ile birlikte) elde etmek için, `RmqContext` nesnesinin `getMessage()` yöntemini kullanın:

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getMessage());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getMessage());
}
```

RabbitMQ [kanalına](https://www.rabbitmq.com/channels.html) bir referans almak için, `RmqContext` nesnesinin `getChannelRef` yöntemini kullanın:

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getChannelRef());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getChannelRef());
}
```

#### Mesaj Onayı

Bir mesajın asla kaybolmamasını sağlamak için, RabbitMQ [mesaj onaylarını](https://www.rabbitmq.com/confirms.html) destekler. Bir onay, bir tüketicinin belirli bir mesajın alındığını, işlendiğini ve RabbitMQ'nun onu silebileceği bilgisini RabbitMQ'ya bildirmek için tüketici tarafından geri gönderilir. Bir tüketici ölürse (kanalı kapatılır, bağlantı kapatılır veya TCP bağlantısı kaybolursa) ve bir onay göndermezse, RabbitMQ, bir mesajın tamamen işlenmediğini anlar ve bunu yeniden sıraya alır.

Manuel onay modunu etkinleştirmek için, `noAck` özelliğini `false` olarak ayarlayın:

```typescript
options: {
  urls: ['amqp://localhost:5672'],
  queue: 'cats_queue',
  noAck: false,
  queueOptions: {
    durable: false
  },
},
```

Manuel tüketici onayları etkinleştirildiğinde, bir görevle bittiğimizi bildirmek için işçiden uygun bir onay göndermemiz gerekir.

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  const channel = context.getChannelRef();
  const originalMsg = context.getMessage();

  channel.ack(originalMsg);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  const channel = context.getChannelRef();
  const originalMsg = context.getMessage();

  channel.ack(originalMsg);
}
```

#### Record Oluşturucular

Mesaj seçeneklerini yapılandırmak için, `RmqRecordBuilder` sınıfını kullanabilirsiniz (not: bu, etkinlik tabanlı akışlar için de yapılabilir). Örneğin, `headers` ve `priority` özelliklerini ayarlamak için `setOptions` yöntemini kullanın:

```typescript
const message = ':cat:';
const record = new RmqRecordBuilder(message)
  .setOptions({
    headers: {
      ['x-version']: '1.0.0',
    },
    priority: 3,
  })
  .build();

this.client.send('replace-emoji', record).subscribe(...);
```

> info **İpucu** `RmqRecordBuilder` sınıfı `@nestjs/microservices` paketinden içe aktarılır.

Ve bu değerlere sunucu tarafından da erişebilirsiniz, `RmqContext`'e erişerek:

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: RmqContext): string {
  const { properties: { headers } } = context.getMessage();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const { properties: { headers } } = context.getMessage();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```