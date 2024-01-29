### NATS

[NATS](https://nats.io), bulut tabanlı uygulamalar, IoT iletişimi ve mikroservis mimarileri için basit, güvenli ve yüksek performanslı bir açık kaynak mesajlaşma sistemidir. NATS sunucusu Go programlama diliyle yazılmıştır, ancak sunucu ile etkileşimde bulunmak için istemci kitaplıkları onlarca önemli programlama dili için mevcuttur. NATS, hem **En Fazla Bir Kez** hem de **En Az Bir Kez** teslimatı destekler. Büyük sunuculardan ve bulut örneklerinden, kenar ağ geçitlerinden ve hatta Nesnelerin İnterneti (IoT) cihazlarından çalışabilir.

#### Kurulum

NATS tabanlı mikroservisler oluşturmaya başlamak için önce gerekli paketi kurun:

```bash
$ npm i --save nats
```

#### Genel Bakış

NATS taşıyıcısını kullanmak için, `createMicroservice()` yöntemine şu seçenekleri içeren bir nesne geçirin:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
  },
});
```

> info **İpucu** `Transport` numarası `@nestjs/microservices` paketinden içe aktarılır.

#### Seçenekler

`options` nesnesi seçilen taşıyıcıya özgüdür. <strong>NATS</strong> taşıyıcısı [burada](https://github.com/nats-io/node-nats#connection-options) açıklanan özelliklere sahiptir.
Ayrıca, `queue` özelliği sunucunuzun abone olmasını istediğiniz kuyruğun adını belirtmenize olanak tanır (bu ayarı yok saymak için `undefined` bırakın). NATS kuyruk grupları hakkında daha fazla bilgi için <a href="https://docs.nestjs.com/microservices/nats#queue-groups">aşağıya</a> bakın.

#### İstemci

Diğer mikroservis taşıyıcıları gibi, NATS `ClientProxy` örneği oluşturmak için <a href="https://docs.nestjs.com/microservices/basics#client">birkaç seçeneğiniz</a> bulunmaktadır.

Bir örnek oluşturmanın bir yöntemi, `ClientsModule`'u kullanmaktır. `ClientsModule` ile bir istemci örneği oluşturmak için, içe aktarın ve `createMicroservice()` yönteminde gösterilen aynı özelliklere sahip bir nesneyi iletmek için `register()` yöntemini kullanın. `ClientsModule` hakkında daha fazla bilgiye <a href="https://docs.nestjs.com/microservices/basics#client">buradan</a> ulaşabilirsiniz.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.NATS,
        options: {
          servers: ['nats://localhost:4222'],
        }
      },
    ]),
  ]
  ...
})
```

Diğer bir seçenek ise istemciyi oluşturmak için `ClientProxyFactory` veya `@Client()`'ı kullanmaktır. Bu konuda daha fazla bilgiye <a href="https://docs.nestjs.com/microservices/basics#client">buradan</a> ulaşabilirsiniz.

#### İstek-cevap

**İstek-cevap** mesaj stili için ([daha fazla bilgi](https://docs.nestjs.com/microservices/basics#request-response)), NATS taşıyıcısı NATS'ın yerleşik [Request-Reply](https://docs.nats.io/nats-concepts/reqreply) mekanizmasını kullanmaz. Bunun yerine, bir "istek" belirli bir konuda benzersiz bir cevap konu adı kullanılarak `publish()` yöntemiyle yayınlanır, ve cevap verenler bu konuyu dinler ve cevapları cevap konusuna gönderir. Cevap konuları, talep sahibine dinamik olarak yönlendirilir, tarafların konumlarına bakılmaksızın.

#### Olay-tabanlı

**Olay-tabanlı** mesaj stili için ([daha fazla bilgi](https://docs.nestjs.com/microservices/basics#event-based)), NATS

 taşıyıcısı NATS'ın yerleşik [Yayın-Abone](https://docs.nats.io/nats-concepts/pubsub) mekanizmasını kullanır. Bir yayıncı bir iletiyi bir konuda gönderir ve o konuyu dinleyen herhangi bir etkin abone iletiyi alır. Aboneler, biraz bir düzenleme gibi çalışan joker konularda da ilgi kaydeder. Bu birden çoğa deseni bazen fan-out olarak adlandırılır.

#### Kuyruk grupları

NATS, [dağıtılmış kuyruklar](https://docs.nats.io/nats-concepts/queue) adlı yerleşik bir yük dengeleme özelliği sağlar. Bir kuyruk aboneliği oluşturmak için `queue` özelliğini şu şekilde kullanın:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
    queue: 'cats_queue',
  },
});
```

#### Bağlam

Daha karmaşık senaryolarda, gelen istekle ilgili daha fazla bilgiye erişmek isteyebilirsiniz. NATS taşıyıcısını kullanırken, `NatsContext` nesnesine erişebilirsiniz.

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Konu: ${context.getSubject()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Konu: ${context.getSubject()}`);
}
```

> info **İpucu** `@Payload()`, `@Ctx()` ve `NatsContext` paketinden içe aktarılır.

#### Jokerler

Bir abonelik, açık bir konuya veya jokerler içerebilir.

```typescript
@@filename()
@MessagePattern('time.us.*')
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Konu: ${context.getSubject()}`); // örneğin "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('time.us.*')
getDate(data, context) {
  console.log(`Konu: ${context.getSubject()}`); // örneğin "time.us.east"
  return new Date().toLocaleTimeString(...);
}
```

#### Kayıt Oluşturucular

İleti seçeneklerini yapılandırmak için `NatsRecordBuilder` sınıfını kullanabilirsiniz (not: bu, olay tabanlı akışlar için de yapılabilir). Örneğin, `x-version` başlığını eklemek için `setHeaders` yöntemini kullanın:

```typescript
import * as nats from 'nats';

// kodunuzun bir yerinde
const headers = nats.headers();
headers.set('x-version', '1.0.0');

const record = new NatsRecordBuilder(':cat:').setHeaders(headers).build();
this.client.send('replace-emoji', record).subscribe(...);
```

> info **İpucu** `NatsRecordBuilder` sınıfı, `@nestjs/microservices` paketinden içe aktarılır.

Ve bu başlıkları sunucu tarafında da `NatsContext`'e erişerek okuyabilirsiniz, aşağıdaki gibi:

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: NatsContext): string {
  const headers = context.getHeaders();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const headers = context.getHeaders();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

Bazı durumlarda birden çok isteğe yönelik başlıkları yapılandırmak isteyebilirsiniz, bunları `ClientProxyFactory`'ye seçenekler olarak iletebilirsiniz:

```typescript
import { Module } from '@nestjs/common';
import { ClientProxyFactory, Transport } from '@nestjs/microservices';

@Module({
  providers: [
    {
      provide: 'API_v1',
      useFactory: () =>
        ClientProxyFactory.create({
          transport: Transport.NATS,
          options: {
            servers: ['nats://localhost:4222'],
            headers: { 'x-version': '1.0.0' },
          },
        }),
    },
  ],
})
export class ApiModule {}
```