### Redis

[Redis](https://redis.io/), yayın/abone mesajlaşma paradigmasını uygular ve Redis'in [Pub/Sub](https://redis.io/topics/pubsub) özelliğini kullanır. Yayınlanan mesajlar, mesajı nihai olarak alacak aboneleri (varsa) bilmeksizin kanallara kategorize edilir. Her mikroservis, herhangi sayıda kanala abone olabilir. Ayrıca, aynı anda birden fazla kanala abone olunabilir. Kanallar aracılığıyla değiştirilen mesajlar **fire-and-forget** (gönder ve unut) özelliğine sahiptir, yani bir mesaj yayınlanır ve ilgilenen abone yoksa mesaj silinir ve kurtarılamaz. Bu nedenle, mesajların veya olayların en az bir hizmet tarafından işlenmesine dair bir garanti yoktur. Bir mesaj, birden çok abone tarafından abone olunabilir (ve alınabilir).

<figure><img src="/assets/Redis_1.png" /></figure>

#### Kurulum

Redis tabanlı mikroservisler oluşturmaya başlamak için önce gerekli paketi yükleyin:

```bash
$ npm i --save ioredis
```

#### Genel Bakış

Redis taşıyıcısını kullanmak için, aşağıdaki seçenekleri içeren bir nesneyi `createMicroservice()` yöntemine geçirin:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});
```

> info **İpucu** `Transport` enum'ı, `@nestjs/microservices` paketinden içe aktarılır.

#### Seçenekler

`options` özelliği, seçilen taşıyıcıya özeldir. <strong>Redis</strong> taşıyıcısı aşağıda açıklanan özellikleri sunar.

<table>
  <tr>
    <td><code>host</code></td>
    <td>Bağlantı URL'si</td>
  </tr>
  <tr>
    <td><code>port</code></td>
    <td>Bağlantı portu</td>
  </tr>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>Mesajı yeniden deneme sayısı (varsayılan: <code>0</code>)</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>Mesaj yeniden deneme arasındaki gecikme (ms) (varsayılan: <code>0</code>)</td>
  </tr>
   <tr>
    <td><code>wildcards</code></td>
    <td>Redis wilcard aboneliklerini etkinleştirir, taşıyıcının altında <code>psubscribe</code>/<code>pmessage</code> kullanmasını sağlar (varsayılan: <code>false</code>)</td>
  </tr>
</table>

Bu taşıyıcı tarafından desteklenen resmi [ioredis](https://redis.github.io/ioredis/index.html#RedisOptions) istemcisinin tüm özellikleri de desteklenir.

#### İstemci

Diğer mikroservis taşıyıcıları gibi, Redis `ClientProxy` örneği oluşturmak için <a href="https://docs.nestjs.com/microservices/basics#client">birkaç seçeneğiniz</a> vardır.

Bir örnek oluşturmanın bir yolu `ClientsModule`'u kullanmaktır. `ClientsModule` ile bir istemci örneği oluşturmak için, onu içe aktarın ve `createMicroservice()` yöntemine yukarıda gösterilenle aynı özelliklere sahip bir seçenek nesnesini iletmek için `register()` yöntemini kullanın, ayrıca bir enjeksiyon belirteci olarak kullanılmak üzere bir `name` özelliği de ekleyin. `ClientsModule` hakkında daha fazla bilgiyi <a href="https://docs.nestjs.com/microservices/basics#client">buradan</a> edinebilirsiniz.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.REDIS,
        options: {
          host: 'localhost',
          port: 6379,
        }
      },
    ]),
  ]
  ...
})
```

İstemci oluşturmak için başka seçenekler de kullanılabilir (ya `ClientProxyFactory` ya da `@Client()`). Bunlar hakkında daha fazla bilgiyi <a href="https://docs.nestjs.com/microservices/basics#client">buradan</a> edinebilirsiniz.

#### Bağlam

Daha karmaşık senaryolarda, gelen istekle ilgili daha fazla bilgiye erişmek isteyebilirsiniz. Redis taşıyıcısını kullanırken, `RedisContext` nesnesine erişebilirsiniz.

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RedisContext) {
  console.log(`Channel: ${context.getChannel()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Channel: ${context.getChannel()}`);
}
```

> info **İpucu** `@Payload()`, `@Ctx

()` ve `RedisContext` `@nestjs/microservices` paketinden içe aktarılır.