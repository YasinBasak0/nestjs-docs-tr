### Server-Sent Events (Sunucu Tarafından Gönderilen Olaylar)

Server-Sent Events (SSE), bir istemcinin HTTP bağlantısı aracılığıyla sunucudan otomatik güncellemeler almasını sağlayan bir sunucu itme teknolojisidir. Her bildirim, yeni satırla biten bir metin bloğu olarak gönderilir (daha fazla bilgiye [buradan](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) ulaşabilirsiniz).

#### Kullanım

Server-Sent Events'ı bir rotada etkinleştirmek için (bir **denetleyici sınıfı** içinde kaydedilen bir rota), yöntem işleyicisine `@Sse()` dekoratörü ile işaretleme yapılır.

```typescript
@Sse('sse')
sse(): Observable<MessageEvent> {
  return interval(1000).pipe(map((_) => ({ data: { hello: 'world' } })));
}
```

> info **İpucu** `@Sse()` dekoratörü ve `MessageEvent` arabirimi `@nestjs/common` tarafından içe aktarılır, `Observable`, `interval` ve `map` ise `rxjs` paketinden içe aktarılır.

> warning **Uyarı** Server-Sent Events rotaları bir `Observable` akışı döndürmelidir.

Yukarıdaki örnekte, gerçek zamanlı güncellemeleri iletmemizi sağlayan bir `sse` adında bir rota tanımladık. Bu olayları dinlemek için [EventSource API](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) kullanılabilir.

`sse` yöntemi, birden çok `MessageEvent` nesnesi yayınlayan bir `Observable` döndürür (bu örnekte her saniyede yeni bir `MessageEvent` yayınlar). `MessageEvent` nesnesi, özellikle aşağıdaki arabirimi karşılamalıdır:

```typescript
export interface MessageEvent {
  data: string | object;
  id?: string;
  type?: string;
  retry?: number;
}
```

Bu yapı ile artık istemci tarafındaki uygulamamızda `EventSource` sınıfının bir örneğini oluşturabiliriz. Bu sınıf, `@Sse()` dekoratörüne yukarıda geçirdiğimiz uç noktayı bir yapılandırıcı argümanı olarak alır.

`EventSource` örneği, bir HTTP sunucusuna kalıcı bir bağlantı açar ve olayları `text/event-stream` biçiminde gönderir. Bağlantı, `EventSource.close()` çağrılana kadar açık kalır.

Bağlantı açıldığında, sunucudan gelen giriş mesajları olaylar şeklinde kodunuza teslim edilir. Gelen mesajda bir olay alanı varsa, tetiklenen olay, olay alanı değeriyle aynıdır. Eğer olay alanı yoksa, o zaman genel bir `message` olayı tetiklenir ([kaynak](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)).

```javascript
const eventSource = new EventSource('/sse');
eventSource.onmessage = ({ data }) => {
  console.log('Yeni mesaj', JSON.parse(data));
};
```

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/28-sse) bulunmaktadır.