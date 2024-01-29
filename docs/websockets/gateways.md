### Gateways

Bu belgede başka yerlerde tartışılan çoğu kavram, bağımlılık enjeksiyonu, dekoratörler, istisna filtreleri, borular, korumalar ve interceptor'lar gibi, aynı şekilde gateway'ler için de geçerlidir. Mümkünse, Nest uygulama ayrıntılarını soyutlar, böylece aynı bileşenler HTTP tabanlı platformlarda, WebSockets'te ve Microservices'te çalışabilir. Bu bölüm, WebSockets'e özgü olan Nest'in yönlerini kapsar.

Nest'te, bir ağ geçidi basitçe `@WebSocketGateway()` dekoratörü ile işaretlenmiş bir sınıftır. Teknik olarak, gateway'ler platforma bağımsızdır ve bir adaptör oluşturulduğunda herhangi bir WebSockets kütüphanesiyle uyumludur. İki WS platformu da kutudan çıkarılmıştır: [socket.io](https://github.com/socketio/socket.io) ve [ws](https://github.com/websockets/ws). İhtiyacınıza en uygun olanı seçebilirsiniz. Ayrıca, kendi adaptörünüzü [bu rehberi](/docs/websockets/adapter) takip ederek oluşturabilirsiniz.

<figure><img src="/assets/Gateways_1.png" /></figure>

> info **İpucu** Gateway'ler [sağlayıcılar](/docs/providers) olarak ele alınabilir; bu, bağımlılıkları sınıf kurucusu aracılığıyla enjekte edebilecekleri anlamına gelir. Ayrıca, gateway'ler başka sınıflar (sağlayıcılar ve denetleyiciler) tarafından da enjekte edilebilir.

#### Kurulum

WebSockets tabanlı uygulamalar oluşturmaya başlamak için önce gerekli paketi yükleyin:

```bash
@@filename()
$ npm i --save @nestjs/websockets @nestjs/platform-socket.io
@@switch
$ npm i --save @nestjs/websockets @nestjs/platform-socket.io
```

#### Genel Bakış

Genel olarak, her ağ geçidi, uygulamanız bir web uygulaması değilse veya portu manuel olarak değiştirmediyseniz, **HTTP sunucusu** ile aynı portu dinler. Bu varsayılan davranışı, `@WebSocketGateway(80)` dekoratörüne bir argüman geçirerek değiştirebilirsiniz, burada `80` seçilen bir port numarasıdır. Aynı zamanda ağ geçidi tarafından kullanılan [namespace](https://socket.io/docs/v4/namespaces/) 'yi aşağıdaki yapısı kullanılarak ayarlayabilirsiniz:

```typescript
@WebSocketGateway(80, { namespace: 'events' })
```

> warning **Uyarı** Gateway'ler, varolan bir modülün sağlayıcılar dizisinde başvurulana kadar başlatılmazlar.

İkinci bir argüman olarak `@WebSocketGateway()` dekoratörüne soket oluşturucuya herhangi bir desteklenen [seçenek](https://socket.io/docs/v4/server-options/) 'i geçirebilirsiniz, aşağıda gösterildiği gibi:

```typescript
@WebSocketGateway(81, { transports: ['websocket'] })
```

Artık ağ geçidi dinleniyor, ancak henüz gelen mesajlara abone olmadık. `events` mesajlarına abone olacak ve kullanıcıya aynı veri ile yanıt verecek bir işleyici oluşturalım.

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(@MessageBody() data: string): string {
  return data;
}
@@switch
@Bind(MessageBody())
@SubscribeMessage('events')
handleEvent(data) {
  return data;
}
```

> info **İpucu** `@SubscribeMessage()` ve `@MessageBody()` dekoratörleri, `@nestjs/websockets` paketinden içe aktarılır.

Gateway oluşturulduğunda, onu modülümüze kaydedebiliriz.

```typescript
import { Module } from '@nestjs/common';
import { EventsGateway } from './events.gateway';

@@filename(events.module)
@Module({
  providers: [EventsGateway]
})
export class EventsModule {}
```

Ayrıca, gelen mesaj gövdesinden çıkarmak için dekoratöre bir özellik anahtarı geçirebilirsiniz:

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(@MessageBody('id') id: number): number {
  // id === messageBody.id
  return id;
}
@@switch
@Bind(MessageBody('id'))
@SubscribeMessage('events')
handleEvent(id) {
  // id === messageBody.id
  return id;
}
```

Eğer dekoratörleri kullanmak istemiyorsanız, aşağıdaki kod da fonksiyonel olarak eşdeğerdir:

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(client: Socket, data: string): string {
  return data;
}
@@switch
@SubscribeMessage('events')
handleEvent(client, data) {
  return data;
}
```

Yukarıdaki örnekte, `handleEvent()` fonksiyonu iki argüman alır. İlk argüman platforma özgü bir [soket örneği](https://socket.io/docs/v4/server-api/#socket) iken, ikinci argüman istemciden alınan veridir. Bu yaklaşım önerilmez, çünkü her birim testte `socket` örneğini taklit etmeyi gerektirir.

`events` mesajı alındığında, işleyici, ağ üzerinden gönderilen aynı veri ile bir onay gönderir. Ayrıca, kütüphane özgü bir yaklaşım kullanarak mesajları iletmek mümkündür, örneğin `client.emit()` yöntemini kullanarak. Bağlı bir soket örneğine erişmek için `@ConnectedSocket()` dekoratörünü kullanın.

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(
  @MessageBody() data: string,
  @ConnectedSocket

() client: Socket,
): string {
  return data;
}
@@switch
@Bind(MessageBody(), ConnectedSocket())
@SubscribeMessage('events')
handleEvent(data, client) {
  return data;
}
```

> info **İpucu** `@ConnectedSocket()` dekoratörü, `@nestjs/websockets` paketinden içe aktarılır.

Ancak bu durumda interceptor'lardan yararlanamazsınız. Kullanıcıya yanıt vermek istemiyorsanız, `return` ifadesini atlayabilirsiniz (veya açıkça bir "yanlış" değeri, örneğin `undefined`, döndürebilirsiniz).

Şimdi bir istemci aşağıdaki gibi bir mesaj gönderdiğinde:

```typescript
socket.emit('events', { name: 'Nest' });
```

`handleEvent()` yöntemi çalıştırılacaktır. Yukarıdaki işleyiciden gönderilen iç mesajları dinlemek için, istemcinin karşılık gelen onay dinleyicisini eklemesi gerekir:

```typescript
socket.emit('events', { name: 'Nest' }, (data) => console.log(data));
```

#### Multiple responses

Onay sadece bir kez gönderilir. Dahası, bu, yerel WebSockets uygulaması tarafından desteklenmez. Bu sınırlamayı çözmek için, iki özellikten oluşan bir nesneyi döndüren bir nesne döndürebilirsiniz. `event`, gönderilen olayın adı ve `data`, istemciye iletilmesi gereken veri olan iki özelliğe sahiptir.

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(@MessageBody() data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@Bind(MessageBody())
@SubscribeMessage('events')
handleEvent(data) {
  const event = 'events';
  return { event, data };
}
```

> info **İpucu** `WsResponse` arabirimini, `@nestjs/websockets` paketinden içe aktarılır.

> warning **Uyarı** `data` alanınız, `ClassSerializerInterceptor`'a güveniyorsa `WsResponse`'ı uygulayan bir sınıf örneği döndürmelisiniz, çünkü bu, düz JavaScript nesne yanıtlarını yok sayar.

Gelen yanıtları dinlemek için istemcinin başka bir olay dinleyici uygulaması gerekmektedir.

```typescript
socket.on('events', (data) => console.log(data));
```

#### Asenkron yanıtlar

Mesaj işleyicileri ya senkron bir şekilde ya da **asenkron olarak** yanıt verebilir. Bu nedenle, `async` yöntemler desteklenir. Bir mesaj işleyicisi ayrıca sonuç değerleri tamamlanana kadar bir `Observable` dönebilir.

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
onEvent(@MessageBody() data: unknown): Observable<WsResponse<number>> {
  const event = 'events';
  const response = [1, 2, 3];

  return from(response).pipe(
    map(data => ({ event, data })),
  );
}
@@switch
@Bind(MessageBody())
@SubscribeMessage('events')
onEvent(data) {
  const event = 'events';
  const response = [1, 2, 3];

  return from(response).pipe(
    map(data => ({ event, data })),
  );
}
```

Yukarıdaki örnekte, mesaj işleyicisi **3 kez** yanıt verecektir (diziden her öğeyle birlikte).

#### Yaşam döngüsü kancaları

Üç kullanışlı yaşam döngüsü kancası mevcuttur. Hepsi karşılık gelen arayüze sahiptir ve aşağıdaki tabloda açıklanmıştır:

<table>
  <tr>
    <td>
      <code>OnGatewayInit</code>
    </td>
    <td>
      <code>afterInit()</code> yöntemini uygulamayı zorlar. Kütüphane özgü sunucu örneğini bir argüman olarak alır (ve
      gerekirse geriye yayılır).
    </td>
  </tr>
  <tr>
    <td>
      <code>OnGatewayConnection</code>
    </td>
    <td>
      <code>handleConnection()</code> yöntemini uygulamayı zorlar. Kütüphane özgü istemci soketi örneğini bir
      argüman olarak alır.
    </td>
  </tr>
  <tr>
    <td>
      <code>OnGatewayDisconnect</code>
    </td>
    <td>
      <code>handleDisconnect()</code> yöntemini uygulamayı zorlar. Kütüphane özgü istemci soketi örneğini bir
      argüman olarak alır.
    </td>
  </tr>
</table>

> info **İpucu** Her yaşam döngüsü arabirimi, `@nestjs/websockets` paketinden içe aktarılır.

#### Sunucu

Bazen, doğrudan, **platforma özgü** sunucu örneğine erişim sağlamak isteyebilirsiniz. Bu nesnenin referansı, `afterInit()` yöntemine (`OnGatewayInit` arabirimi) bir argüman olarak iletilir. Başka bir seçenek de `@WebSocketServer()` dekoratörünü kullanmaktır.

```typescript
@WebSocketServer()
server: Server;
```

> warning **Uyarı** `@WebSocketServer()` dekoratörü, `@nestjs/websockets` paketinden içe aktarılır.

Nest, sunucu örneğini kullanıma hazır olduğunda bu özelliğe otomatik olarak atar.

<app-banner-enterprise></app-banner-enterprise>

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/02-gateways) bulunmaktadır.