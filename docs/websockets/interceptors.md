### Interceptor'lar

[Normal interceptor'lar](/docs/interceptors) ile web soket interceptor'ları arasında bir fark yoktur. Aşağıdaki örnek, yöntem kapsamlı bir interceptor kullanmaktadır. HTTP tabanlı uygulamalarda olduğu gibi, aynı zamanda ağ geçidi kapsamlı interceptor'ları da kullanabilirsiniz (yani, ağ geçidi sınıfını `@UseInterceptors()` dekoratörü ile önekleyebilirsiniz).

```typescript
@@filename()
@UseInterceptors(new TransformInterceptor())
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UseInterceptors(new TransformInterceptor())
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```
