### Pipe'lar

[Normal pipe'lar](/docs/pipes) ile web soket pipe'ları arasında temel bir fark yoktur. Tek fark, `HttpException` fırlatmak yerine `WsException` kullanmanız gerektiğidir. Ayrıca, tüm pipe'lar yalnızca `data` parametresine uygulanacaktır (çünkü `client` örneğini doğrulamak veya dönüştürmek gereksizdir).

> info **Hint** The `WsException` class is exposed from `@nestjs/websockets` package.

#### Pipe'ları Bağlama

Aşağıdaki örnek, yöntem kapsamlı elle oluşturulmuş bir pipe kullanmaktadır. HTTP tabanlı uygulamalarda olduğu gibi, aynı zamanda ağ geçidi kapsamlı pipe'ları da kullanabilirsiniz (yani, ağ geçidi sınıfını `@UsePipes()` dekoratörü ile önekleyebilirsiniz).

```typescript
@@filename()
@UsePipes(new ValidationPipe())
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UsePipes(new ValidationPipe())
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```