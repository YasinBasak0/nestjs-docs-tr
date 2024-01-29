#### Korumalar

Web soket korumaları ile [normal HTTP uygulama korumaları](/docs/guards) arasında temel bir fark yoktur. Tek fark, `HttpException` yerine `WsException` kullanmanız gerektiğidir.

> info **İpucu** `WsException` sınıfı, `@nestjs/websockets` paketinden içe aktarılır.

#### Korumaları bağlama

Aşağıdaki örnek, yöntem kapsamlı bir koruma kullanmaktadır. HTTP tabanlı uygulamalarda olduğu gibi, aynı zamanda ağ geçidi kapsamlı korumaları da kullanabilirsiniz (yani, ağ geçidi sınıfını `@UseGuards()` dekoratörü ile önekleyebilirsiniz).

```typescript
@@filename()
@UseGuards(AuthGuard)
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UseGuards(AuthGuard)
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```
