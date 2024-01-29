### Adaptörler

WebSocket modülü platform bağımsız olduğundan, `WebSocketAdapter` arabirimini kullanarak kendi kütüphanenizi (veya hatta yerel bir uygulamayı) kullanabilirsiniz. Bu arabirim, aşağıdaki tabloda açıklanan birkaç yöntemi uygulamayı zorunlu kılar:

<table>
  <tr>
    <td><code>create</code></td>
    <td>Geçirilen argümanlara dayalı bir soket örneği oluşturur</td>
  </tr>
  <tr>
    <td><code>bindClientConnect</code></td>
    <td>Müşteri bağlantı etkinliğini bağlar</td>
  </tr>
  <tr>
    <td><code>bindClientDisconnect</code></td>
    <td>Müşteri bağlantısı kesme olayını bağlar (isteğe bağlı*)</td>
  </tr>
  <tr>
    <td><code>bindMessageHandlers</code></td>
    <td>Gelen iletiyi ilgili ileti işleyiciye bağlar</td>
  </tr>
  <tr>
    <td><code>close</code></td>
    <td>Bir sunucu örneğini sonlandırır</td>
  </tr>
</table>

#### socket.io'yu Genişletme

[socket.io](https://github.com/socketio/socket.io) paketi, bir `IoAdapter` sınıfında sarılmıştır. Temel işlevselliği artırmak isterseniz ne yapabilirsiniz? Örneğin, teknik gereksinimleriniz, web servisinizin çoklu yük dengelemeli örnekleri arasında olayları yayınlama yeteneğini gerektiriyorsa. Bu durumda, `IoAdapter`'i genişletip yeni bir socket.io sunucu örneği oluşturmakla sorumlu olan tek bir yöntemi geçersiz kılabilirsiniz. Ancak önce gerekli paketi yükleyelim.

> uyarı **Uyarı** socket.io'yu birden çok yük dengelemeli örnekle kullanmak için, müşteri socket.io yapılandırmanızda `transports: ['websocket']`'yi ayarlamanız veya yük dengeleyicinizde çerez tabanlı yönlendirmeyi etkinleştirmeniz gerekmektedir. Redis tek başına yeterli değildir. Daha fazla bilgi için [buraya](https://socket.io/docs/v4/using-multiple-nodes/#enabling-sticky-session) bakın.

```bash
$ npm i --save redis socket.io @socket.io/redis-adapter
```

Paket yüklendikten sonra, bir `RedisIoAdapter` sınıfı oluşturabiliriz.

```typescript
import { IoAdapter } from '@nestjs/platform-socket.io';
import { ServerOptions } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;

  async connectToRedis(): Promise<void> {
    const pubClient = createClient({ url: `redis://localhost:6379` });
    const subClient = pubClient.duplicate();

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
}
```

Ardından, basitçe oluşturulan Redis adaptörüne geçiş yapın.

```typescript
const app = await NestFactory.create(AppModule);
const redisIoAdapter = new RedisIoAdapter(app);
await redisIoAdapter.connectToRedis();

app.useWebSocketAdapter(redisIoAdapter);
```

### Ws Kütüphanesi

Başka bir kullanılabilir adaptör, `WsAdapter` adında bir adaptördür ve sırasıyla çerçeve ile [ws](https://github.com/websockets/ws) kütüphanesini entegre eden ve hızlı ve kapsamlı bir şekilde test edilen bir proxy gibi hareket eder. Bu adaptör, yerel tarayıcı WebSockets'lerine tamamen uyumludur ve socket.io paketinden çok daha hızlıdır. Ne yazık ki, bununla sağlanan işlevsellikler önemli ölçüde daha azdır. Ancak bazı durumlarda bunlara gerçekten ihtiyacınız olmayabilir.

> info **İpucu** `ws` kütüphanesi, (`socket.io` tarafından popülerleştirilen) namespace'leri (iletişim kanalları) desteklemez. Bununla birlikte, bu özelliği bir şekilde taklit etmek için farklı yollar üzerinde birden çok `ws` sunucusunu bağlayabilirsiniz (örnek: `@WebSocketGateway({{ '{' }} path: '/users' {{ '}' }})`).

`ws`'yi kullanabilmek için önce gerekli paketi yüklememiz gerekiyor:

```bash
$ npm i --save @nestjs/platform-ws
```

Paket yüklendikten sonra, bir adaptöre geçebiliriz:

```typescript
const app = await NestFactory.create(AppModule);
app.useWebSocketAdapter(new WsAdapter(app));
```

> info **İpucu** `WsAdapter`, `@nestjs/platform-ws` paketinden içe aktarılır.

#### Gelişmiş (özel adaptör)

Gösterim amaçlı olarak, [ws](https://github.com/websockets/ws) kütüphanesini manuel olarak entegre edeceğiz. Yukarıda belirtildiği gibi, bu kütüphane için adaptör zaten oluşturuldu ve `@nestjs/platform-ws` paketinden `WsAdapter` adlı bir sınıf olarak dışa aktarıldı. İşte basitleştirilmiş bir uygulamanın nasıl görünebileceği:

```typescript
@@filename(ws-adapter)
import * as WebSocket from 'ws';
import { WebSocketAdapter, INestApplicationContext } from '@nestjs/common';
import { MessageMappingProperties } from '@nestjs/websockets';
import { Observable, fromEvent, EMPTY } from 'rxjs';
import { mergeMap, filter } from 'rxjs/operators';

export class WsAdapter implements WebSocketAdapter {
  constructor(private app: INestApplicationContext) {}

  create(port: number, options: any = {}): any {
    return new WebSocket.Server({ port, ...options });
  }

  bindClientConnect(server, callback: Function) {
    server.on('connection', callback);
  }

  bindMessageHandlers(
    client: WebSocket,
    handlers: MessageMappingProperties[],
    process: (data: any) => Observable<any>,
  ) {
    fromEvent(client, 'message')
      .pipe(
        mergeMap(data => this.bindMessageHandler(data, handlers, process)),
        filter(result => result),
      )
      .subscribe(response => client.send(JSON.stringify(response)));
  }

  bindMessageHandler(
    buffer,
    handlers: MessageMappingProperties[],
    process: (data: any) => Observable<any>,
  ): Observable<any> {
    const message = JSON.parse(buffer.data);
    const messageHandler = handlers.find(
      handler => handler.message === message.event,
    );
    if (!messageHandler) {
      return EMPTY;
    }
    return process(messageHandler.callback(message.data));
  }

  close(server) {
    server.close();
  }
}
```

> info **İpucu** [ws](https://github.com/websockets/ws) kütüphanesinden faydalanmak istediğinizde, kendi adaptörünüzü oluşturmak yerine yerleşik `WsAdapter`'i kullanın.

Ardından, `useWebSocketAdapter()` yöntemini kullanarak özel bir adaptör kurabiliriz:

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.useWebSocketAdapter(new WsAdapter(app));
```

#### Örnek

`WsAdapter`'i kullanan çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/16-gateways-ws) bulunabilir.