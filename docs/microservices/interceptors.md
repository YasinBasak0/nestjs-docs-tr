### Interceptörler

[Normal interceptörler](/docs/interceptors) ile mikroservisler interceptörleri arasında bir fark yoktur. Aşağıdaki örnek, manuel olarak örneğin alınmış bir yöntem kapsamlı interceptor kullanır. HTTP tabanlı uygulamalarda olduğu gibi, aynı zamanda denetleyici kapsamlı interceptorları da kullanabilirsiniz (yani, denetleyici sınıfını `@UseInterceptors()` dekoratörü ile öne ekleyebilirsiniz).

```typescript
@@filename()
@UseInterceptors(new TransformInterceptor())
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseInterceptors(new TransformInterceptor())
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```
