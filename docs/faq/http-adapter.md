### HTTP Adaptörü

Ara sıra, Nest uygulama bağlamı içinde veya dışından temel HTTP sunucusuna erişmek isteyebilirsiniz.

Her platforma özgü (native) HTTP sunucusu/kütüphanesi (örneğin, Express ve Fastify) örneği bir **adaptörde** (adapter) sarılır. Adaptör, uygulama bağlamından alınabilir ve diğer sağlayıcılara enjekte edilebilen genel olarak kullanılabilir bir sağlayıcı olarak kaydedilir.

#### Uygulama bağlamı dışındaki strateji

Uygulama bağlamının dışından `HttpAdapter` referansına ulaşmak için `getHttpAdapter()` metodunu çağırın.

```typescript
@@filename()
const app = await NestFactory.create(AppModule);
const httpAdapter = app.getHttpAdapter();
```

#### İçerideki strateji

Uygulama bağlamı içinden `HttpAdapterHost` referansına ulaşmak için, onu diğer mevcut sağlayıcılar gibi enjekte ederek (örneğin, constructor enjeksiyonu kullanarak) yapın.

```typescript
@@filename()
export class CatsService {
  constructor(private adapterHost: HttpAdapterHost) {}
}
@@switch
@Dependencies(HttpAdapterHost)
export class CatsService {
  constructor(adapterHost) {
    this.adapterHost = adapterHost;
  }
}
```

> info **İpucu** `HttpAdapterHost`, `@nestjs/core` paketinden içe aktarılır.

`HttpAdapterHost`, gerçek bir `HttpAdapter` değildir. Gerçek `HttpAdapter` örneğini almak için basitçe `httpAdapter` özelliğine erişin.

```typescript
const adapterHost = app.get(HttpAdapterHost);
const httpAdapter = adapterHost.httpAdapter;
```

`httpAdapter`, altındaki çerçeve tarafından kullanılan HTTP adaptörünün gerçek örneğidir. Bu, `ExpressAdapter` veya `FastifyAdapter` (her iki sınıf da `AbstractHttpAdapter`'ı genişletir) sınıflarından biridir.

Adaptör nesnesi, HTTP sunucusu ile etkileşimde bulunmak için birkaç kullanışlı yöntem sunar. Ancak kütüphane örneğine (örneğin, Express örneği) doğrudan erişmek istiyorsanız, `getInstance()` metodunu çağırın.

```typescript
const instance = httpAdapter.getInstance();
```