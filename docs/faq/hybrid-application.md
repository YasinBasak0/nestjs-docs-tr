### Hibrit Uygulama

Hibrit bir uygulama, iki veya daha fazla farklı kaynaktan gelen istekleri dinleyen bir uygulamadır. Bu, bir HTTP sunucusunu bir mikroservis dinleyici ile veya hatta sadece birden fazla farklı mikroservis dinleyicisi ile birleştirebilir. Varsayılan `createMicroservice` yöntemi birden çok sunum için izin vermez, bu nedenle bu durumda her bir mikroservis manuel olarak oluşturulup başlatılmalıdır. Bunun için `connectMicroservice()` yöntemi aracılığıyla `INestApplication` örneği, `INestMicroservice` örnekleri ile bağlanabilir.

```typescript
const app = await NestFactory.create(AppModule);
const microservice = app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.TCP,
});

await app.startAllMicroservices();
await app.listen(3001);
```

> info **İpucu** `app.listen(port)` yöntemi belirtilen adres üzerinde bir HTTP sunucusunu başlatır. Uygulamanız HTTP isteklerini işlemiyorsa, bunun yerine `app.init()` yöntemini kullanmalısınız.

Birden çok mikroservis örneğini bağlamak için, her mikroservis için `connectMicroservice()` çağrısını yapın:

```typescript
const app = await NestFactory.create(AppModule);
// mikroservis #1
const microserviceTcp = app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.TCP,
  options: {
    port: 3001,
  },
});
// mikroservis #2
const microserviceRedis = app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});

await app.startAllMicroservices();
await app.listen(3001);
```

Birden çok mikroservisi olan bir hibrit uygulamada `@MessagePattern()`'i sadece bir taşıma stratejisine (örneğin, MQTT) bağlamak için, ikinci argüman olarak `Transport` tipinde bir enum olan ve tüm yerleşik taşıma stratejilerini tanımlayan bir argümanı geçirebiliriz.

```typescript
@@filename()
@MessagePattern('time.us.*', Transport.NATS)
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`); // örn. "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@MessagePattern({ cmd: 'time.us' }, Transport.TCP)
getTCPDate(@Payload() data: number[]) {
  return new Date().toLocaleTimeString(...);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('time.us.*', Transport.NATS)
getDate(data, context) {
  console.log(`Subject: ${context.getSubject()}`); // örn. "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@Bind(Payload(), Ctx())
@MessagePattern({ cmd: 'time.us' }, Transport.TCP)
getTCPDate(data, context) {
  return new Date().toLocaleTimeString(...);
}
```

> info **İpucu** `@Payload()`, `@Ctx()`, `Transport` ve `NatsContext` `@nestjs/microservices` tarafından içe aktarılmıştır.

#### Yapılandırma Paylaşma

Varsayılan olarak, bir hibrit uygulama, ana (HTTP tabanlı) uygulama için yapılandırılmış genel boruları, interceptor'ları, bekçileri ve filtreleri devralmaz. Bu yapılandırma özelliklerini ana uygulamadan devralmak için, `connectMicroservice()` çağrısının ikinci argümanına (isteğe bağlı bir seçenekler nesnesi) `inheritAppConfig` özelliğini ayarlayın, aşağıdaki gibi:

```typescript
const microservice = app.connectMicroservice<MicroserviceOptions>(
  {
    transport: Transport.TCP,
  },
  { inheritAppConfig: true },
);
```