### Pipes

[Normal borular](/docs/pipes) ve mikroservis boruları arasında temel bir fark yoktur. Tek fark, `HttpException` fırlatmak yerine `RpcException`'ı kullanmanız gerektiğidir.

> info **İpucu** `RpcException` sınıfı, `@nestjs/microservices` paketinden içe aktarılır.

#### Boruları Bağlama

Aşağıdaki örnek, manuel olarak örneğin bir yönteme bağlı boru kullanır. HTTP tabanlı uygulamalarda olduğu gibi, aynı zamanda denetleyiciye bağlı boruları da kullanabilirsiniz (yani, denetleyici sınıfını `@UsePipes()` dekoratörü ile önekleyebilirsiniz).

```typescript
@@filename()
@UsePipes(new ValidationPipe())
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UsePipes(new ValidationPipe())
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```