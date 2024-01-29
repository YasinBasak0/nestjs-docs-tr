### Guards

Mikroservis koruyucuları ile [standart HTTP uygulama koruyucuları](/docs/guards) arasında temel bir fark yoktur.
Tek fark, `HttpException` fırlatmak yerine `RpcException`'ı kullanmanız gerektiğidir.

> info **İpucu** `RpcException` sınıfı, `@nestjs/microservices` paketinden açığa çıkar.

#### Koruyucuları Bağlama

Aşağıdaki örnek, yöntem kapsamlı bir koruyucu kullanır. HTTP tabanlı uygulamalarda olduğu gibi, aynı zamanda denetleyici kapsamlı koruyucuları da kullanabilirsiniz (yani, denetleyici sınıfını `@UseGuards()` dekoratörü ile öne ekleyebilirsiniz).

```typescript
@@filename()
@UseGuards(AuthGuard)
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseGuards(AuthGuard)
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```
