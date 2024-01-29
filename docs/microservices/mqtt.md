### MQTT

[MQTT](https://mqtt.org/) (Message Queuing Telemetry Transport), açık kaynaklı, hafif bir mesajlaşma protokolüdür ve düşük gecikme süreleri için optimize edilmiştir. Bu protokol, cihazları bağlamak için ölçeklenebilir ve maliyet-etkin bir **yayın/abone** modelini sağlar. MQTT üzerine inşa edilmiş bir iletişim sistemi, yayın sunucusu, bir aracı (broker) ve bir veya daha fazla istemci içerir. Bu protokol, sınırlı kaynaklara sahip cihazlar ve düşük bant genişliği, yüksek gecikme süreli veya güvenilmez ağlar için tasarlanmıştır.

#### Kurulum

MQTT tabanlı mikro servisler oluşturmaya başlamak için önce gerekli paketi kurun:

```bash
$ npm i --save mqtt
```

#### Genel Bakış

MQTT taşıyıcısını kullanmak için, `createMicroservice()` yöntemine şu seçenekleri içeren bir nesne geçirin:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
  },
});
```

> info **İpucu** `Transport` numarası `@nestjs/microservices` paketinden içe aktarılır.

#### Seçenekler

`options` nesnesi seçilen taşıyıcıya özgüdür. <strong>MQTT</strong> taşıyıcısı [burada](https://github.com/mqttjs/MQTT.js/#mqttclientstreambuilder-options) açıklanan özelliklere sahiptir.

#### İstemci

Diğer mikroservis taşıyıcıları gibi, MQTT `ClientProxy` örneği oluşturmak için <a href="https://docs.nestjs.com/microservices/basics#client">birkaç seçeneğiniz</a> bulunmaktadır.

Bir örnek oluşturmanın bir yöntemi, `ClientsModule`'u kullanmaktır. `ClientsModule` ile bir istemci örneği oluşturmak için, içe aktarın ve `createMicroservice()` yönteminde gösterilen aynı özelliklere sahip bir nesneyi iletmek için `register()` yöntemini kullanın. `ClientsModule` hakkında daha fazla bilgiye <a href="https://docs.nestjs.com/microservices/basics#client">buradan</a> ulaşabilirsiniz.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.MQTT,
        options: {
          url: 'mqtt://localhost:1883',
        }
      },
    ]),
  ]
  ...
})
```

Diğer bir seçenek ise istemciyi oluşturmak için `ClientProxyFactory` veya `@Client()`'ı kullanmaktır. Bu konuda daha fazla bilgiye <a href="https://docs.nestjs.com/microservices/basics#client">buradan</a> ulaşabilirsiniz.

#### Bağlam

Daha karmaşık senaryolarda, gelen isteğe dair daha fazla bilgiye erişmek isteyebilirsiniz. MQTT taşıyıcısını kullanırken, `MqttContext` nesnesine erişebilirsiniz.

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
  console.log(`Konu: ${context.getTopic()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Konu: ${context.getTopic()}`);
}
```

> info **İpucu** `@Payload()`, `@Ctx()` ve `MqttContext`'in içe aktarıldığını unutmayın. Bunlar `@nestjs/microservices` paketinden gelir.

Mqtt [paketini](https://github.com/mqttjs/mqtt-packet) orijinal olarak almak için `MqttContext` nesnesinin `getPacket()` yöntemini kullanın, aşağıdaki gibi:

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
 

 console.log(context.getPacket());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getPacket());
}
```

#### Jokalar

Bir abonelik açık bir konuya veya jokaları içerebilir. İki joker bulunmaktadır, `+` ve `#`. `+` tek seviyeli bir jokerken, `#` birçok konu seviyesini kapsayan çok seviyeli bir jokerdir.

```typescript
@@filename()
@MessagePattern('sensors/+/temperature/+')
getTemperature(@Ctx() context: MqttContext) {
  console.log(`Konu: ${context.getTopic()}`);
}
@@switch
@Bind(Ctx())
@MessagePattern('sensors/+/temperature/+')
getTemperature(context) {
  console.log(`Konu: ${context.getTopic()}`);
}
```

#### Kayıt Oluşturucular

Mesaj seçeneklerini yapılandırmak için (QoS seviyesini ayarlamak, Retain veya DUP bayraklarını belirlemek veya yük eklemek gibi), `MqttRecordBuilder` sınıfını kullanabilirsiniz. Örneğin, `QoS`'u `2` olarak ayarlamak için `setQoS` yöntemini kullanın:

```typescript
const userProperties = { 'x-version': '1.0.0' };
const record = new MqttRecordBuilder(':cat:')
  .setProperties({ userProperties })
  .setQoS(1)
  .build();
client.send('replace-emoji', record).subscribe(...);
```

> info **İpucu** `MqttRecordBuilder` sınıfı, `@nestjs/microservices` paketinden içe aktarılır.

Ayrıca bu seçenekleri sunucu tarafında da okuyabilirsiniz, `MqttContext`'e erişerek.

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: MqttContext): string {
  const { properties: { userProperties } } = context.getPacket();
  return userProperties['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const { properties: { userProperties } } = context.getPacket();
  return userProperties['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

Bazı durumlarda birden çok isteğe yönelik kullanıcı özelliklerini yapılandırmak isteyebilirsiniz, bu seçenekleri `ClientProxyFactory`'ye iletebilirsiniz.

```typescript
import { Module } from '@nestjs/common';
import { ClientProxyFactory, Transport } from '@nestjs/microservices';

@Module({
  providers: [
    {
      provide: 'API_v1',
      useFactory: () =>
        ClientProxyFactory.create({
          transport: Transport.MQTT,
          options: {
            url: 'mqtt://localhost:1833',
            userProperties: { 'x-version': '1.0.0' },
          },
        }),
    },
  ],
})
export class ApiModule {}
```