### Asenkron Sağlayıcılar

Bazı durumlarda, uygulama başlatması bir veya daha fazla **asenkron görevin** tamamlanmasını beklemelidir. Örneğin, veritabanı ile bağlantı kurulana kadar istekleri kabul etmek istemeyebilirsiniz. Bu, asenkron sağlayıcılar kullanılarak mümkündür.

Bu için kullanılan sözdizimi, `useFactory` sözdizimi ile `async/await` kullanmaktır. Fabrika bir `Promise` döndürür ve fabrika fonksiyonu asenkron görevleri `await` edebilir. Nest, bu tür bir sağlayıcının bağımlılığı olan (enjekte eden) herhangi bir sınıfın örneğini oluşturmadan önce vaadin çözülmesini bekler.

```typescript
{
  provide: 'ASYNC_CONNECTION',
  useFactory: async () => {
    const connection = await createConnection(options);
    return connection;
  },
}
```

> info **İpucu** Özel sağlayıcı sözdizimi hakkında daha fazla bilgi için [buraya](/docs/fundamentals/custom-providers) bakın.

#### Enjeksiyon

Asenkron sağlayıcılar, diğer bileşenlere diğer sağlayıcılar gibi tokenlarıyla enjekte edilir. Yukarıdaki örnekte, `@Inject('ASYNC_CONNECTION')` yapısını kullanırdınız.

#### Örnek

[TypeORM tarifi](/docs/recipes/sql-typeorm), asenkron bir sağlayıcı örneğine daha kapsamlı bir örnek sunmaktadır.