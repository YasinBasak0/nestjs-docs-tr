### Özel Taşıyıcılar

Nest, bir dizi **taşıyıcıyı** (transporters) kutudan çıkar, ayrıca geliştiricilere yeni özel taşıma stratejileri oluşturmalarına izin veren bir API sağlar. Taşıyıcılar, takılabilir bir iletişim katmanı ve çok basit bir uygulama düzeyi ileti protokolü kullanarak bileşenleri bir ağ üzerinde bağlamanıza olanak tanır (tam [makaleyi](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) okuyun).

> info **Hint** Nest ile bir mikroservis oluşturmak, `@nestjs/microservices` paketini kullanmanız gerektiği anlamına gelmez. Örneğin, harici servislerle iletişim kurmak istiyorsanız (farklı dillerde yazılmış diğer mikroservisler gibi), `@nestjs/microservice` kütüphanesinin sağladığı tüm özelliklere ihtiyacınız olmayabilir.
> Aslında, dekoratörleri (`@EventPattern` veya `@MessagePattern`) kullanmanıza gerek olmadıkça (ki bunlar aboneleri açıkça tanımlamanıza olanak tanır), bir [Bağımsız Uygulama](/docs/application-context) çalıştırmak ve bağlantıyı elle sürdürmek / kanallara abone olmak çoğu durum için yeterli olacaktır ve daha fazla esneklik sağlayacaktır.

Özel bir taşıyıcı ile, herhangi bir iletişim sistemi/protokolünü entegre edebilir (Google Cloud Pub/Sub, Amazon Kinesis ve diğerleri de dahil olmak üzere) veya mevcut olanı genişletebilir ve üzerine ek özellikler ekleyebilirsiniz (örneğin, MQTT için [QoS](https://github.com/mqttjs/MQTT.js/blob/master/README.md#qos)).

> info **Hint** Nest mikroservislerinin nasıl çalıştığını ve mevcut taşıyıcıların yeteneklerini nasıl genişletebileceğinizi daha iyi anlamak için [NestJS Microservices in Action](https://dev.to/johnbiundo/series/4724) ve [Advanced NestJS Microservices](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l) makale serisini okumanızı öneririz.

#### Strateji Oluşturma

İlk olarak, özel taşıyıcıyı temsil eden bir sınıf tanımlayalım.

```typescript
import { CustomTransportStrategy, Server } from '@nestjs/microservices';

class GoogleCloudPubSubServer
  extends Server
  implements CustomTransportStrategy {
  /**
   * "app.listen()" çalıştırıldığında bu yöntem tetiklenir.
   */
  listen(callback: () => void) {
    callback();
  }

  /**
   * Uygulama kapatıldığında bu yöntem tetiklenir.
   */
  close() {}
}
```

> warning **Uyarı** Lütfen bu bölümde tam özellikli bir Google Cloud Pub/Sub sunucusunu uygulamayacağımızı unutmayın; çünkü bu, taşıyıcı özel teknik detaylarına dalmayı gerektirir.

Yukarıdaki örnekte, `GoogleCloudPubSubServer` sınıfını bildirdik ve `CustomTransportStrategy` arabirimince zorunlu kılınan `listen()` ve `close()` yöntemlerini sağladık.
Ayrıca, sınıfımız `@nestjs/microservices` paketinden içe aktarılan `Server` sınıfını genişletir, bu da Nest çalışma zamanının ileti işleyicilerini kaydetmek için kullanılan bazı yararlı yöntemleri sağlar. Varolan bir taşıma stratejisinin yeteneklerini genişletmek istiyorsanız, ilgili sunucu sınıfını genişletebilirsiniz, örneğin, `ServerRedis`.
Geleneksel olarak, sınıfımıza `"Server"` sonekini ekledik, çünkü bu sınıf, iletilere/etkinliklere abone olma (ve gerekirse onlara yanıt verme) sorumluluğuna sahip olacaktır.

Bu yerine, özel bir stratejiyi şu şekilde kullanabiliriz:

```typescript
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    strategy: new GoogleCloudPubSubServer(),
  },
);
```

Temelde, `transport` ve `options` özelliklerine sahip normal bir taşıma seçenekleri nesnesini iletmek yerine, yalnızca `strategy` adlı tek bir özelliği iletiyoruz ve değeri özel taşıyıcı sınıfının bir örneğidir.

`GoogleCloudPubSubServer` sınıfımıza geri dönersek, gerçek bir uygulamada, `listen()` yöntemi içinde mesaj işlemcisine/harici servise bir bağlantı kuruyor ve belirli kanallara abone oluyor olacaktık (ve daha sonra abonelikleri kaldırıyor ve bağlantıyı kapatıyoruz), ancak bunun Nest mikroservislerinin birbirleriyle nasıl iletişim kurduğu hakkında iyi bir anlayışa ihtiyaç duyduğundan, bu [makale serisini](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l) okumanızı öneririz. Bu bölümde, `Server` sınıfının sağladığı yeteneklere odaklanacak ve bunları özel stratejiler oluşturmak için nasıl kullanabileceğinizi göstereceğiz.

Örneğin, uygulamamızda aşağıdaki mesaj işlemcisinin tanımlandığını varsayalım:

```typescript
@MessagePattern('echo')
echo(@Payload() data: object) {
  return data;
}
``

`

Bu mesaj işlemcisi, Nest çalışma zamanı tarafından otomatik olarak kaydedilecektir. `Server` sınıfını kullanarak kaydedilen mesaj desenlerini görebilir ve ayrıca bunlara atanan gerçek yöntemlere erişebilir ve bunları çalıştırabilirsiniz.
Bunu denemek için, `callback` fonksiyonu çağrılmadan önce `listen()` yöntemi içine bir `console.log` ekleyelim:

```typescript
listen(callback: () => void) {
  console.log(this.messageHandlers);
  callback();
}
```

Uygulamanız yeniden başlatıldığında, terminalinizde şu günlüğü göreceksiniz:

```typescript
Map { 'echo' => [AsyncFunction] { isEventHandler: false } }
```

> info **Hint** `@EventPattern` dekoratörünü kullansaydık, çıktıyı aynı görecektiniz, ancak `isEventHandler` özelliği `true` olarak ayarlanmış olacaktı.

Görüldüğü gibi, `messageHandlers` özelliği, tüm mesaj (ve olay) işlemcilerinin, desenlerin anahtarlar olarak kullanıldığı bir `Map` koleksiyonudur.
Artık bir anahtar (örneğin, `"echo"`) kullanarak mesaj işlemcisine bir referans alabilirsiniz:

```typescript
async listen(callback: () => void) {
  const echoHandler = this.messageHandlers.get('echo');
  console.log(await echoHandler('Hello world!'));
  callback();
}
```

`echoHandler`'ı bir argüman olarak (bu durumda "Hello world!") geçirerek bir yöntem işlemcisini yürüttüğümüzde, konsolda bunu görmeliyiz:

```json
Hello world!
```

Bu, yöntem işlemcimizin uygun şekilde yürütüldüğü anlamına gelir.

[Interceptor'lar](/docs/interceptors) ile bir [CustomTransportStrategy](https://docs.nestjs.com/microservices/custom-transport) kullanırken, işlemciler yöntemlere RxJS akışlarına sarılır. Bu, mantık altında devam etmek için onlara abone olmanız gerektiği anlamına gelir (örneğin, bir interceptor çalıştırıldıktan sonra denetleyici mantığına devam etmek için).

Bu, aşağıdaki gibi bir örnekle görülebilir:

```typescript
async listen(callback: () => void) {
  const echoHandler = this.messageHandlers.get('echo');
  const streamOrResult = await echoHandler('Hello World');
  if (isObservable(streamOrResult)) {
    streamOrResult.subscribe();
  }
  callback();
}
```

### İstemci Proxy

İlk bölümde belirttiğimiz gibi, mikroservisler oluşturmak için `@nestjs/microservices` paketini kullanmanıza gerek yoktur, ancak bunu yapmaya karar verirseniz ve özel bir strateji entegre etmeniz gerekiyorsa, bir "istemci" sınıfı sağlamanız da gerekecektir.

> info **Hint** Yine de tüm `@nestjs/microservices` özelliklerine uyumlu tam özellikli bir istemci sınıfı uygulamak, çerçeve tarafından kullanılan iletişim tekniklerinin iyi bir anlayışını gerektirir. Daha fazla bilgi için bu [makaleye](https://dev.to/nestjs/part-4-basic-client-component-16f9) göz atın.

Bir harici servisle iletişim kurmak/ileti göndermek ve yayınlamak (veya olaylar) için ya kütüphane özel bir SDK paketi kullanabilirsiniz ya da `ClientProxy`'yi genişleten bir özel istemci sınıfı uygulayabilirsiniz, aşağıdaki gibi:

```typescript
import { ClientProxy, ReadPacket, WritePacket } from '@nestjs/microservices';

class GoogleCloudPubSubClient extends ClientProxy {
  async connect(): Promise<any> {}
  async close() {}
  async dispatchEvent(packet: ReadPacket<any>): Promise<any> {}
  publish(
    packet: ReadPacket<any>,
    callback: (packet: WritePacket<any>) => void,
  ): Function {}
}
```

> warning **Uyarı** Lütfen, bu bölümde tam özellikli bir Google Cloud Pub/Sub istemci uygulamayacağımızı unutun; çünkü bu, taşıyıcı özel teknik detaylarına dalmayı gerektirir.

Görüldüğü gibi, `ClientProxy` sınıfı, bağlantıyı kurmak ve kapatmak, mesajları (`publish`) ve olayları (`dispatchEvent`) yayınlamak için birkaç yöntem sağlamamızı gerektirir.
Unutmayın, eğer bir istek-cevap iletişim tarzı desteğine ihtiyacınız yoksa, `publish()` yöntemini boş bırakabilirsiniz. Benzer şekilde, olay tabanlı iletişimi desteklemeniz gerekmiyorsa, `dispatchEvent()` yöntemini atlayabilirsiniz.

Bu yöntemlerin ne zaman ve neden yürütüldüğünü gözlemlemek için, aşağıdaki gibi birkaç `console.log` çağrısı ekleyelim:

```typescript
class GoogleCloudPubSubClient extends ClientProxy {
  async connect(): Promise<any> {
    console.log('connect');
  }

  async close() {
    console.log('close');
  }

  async dispatchEvent(packet: ReadPacket<any>): Promise<any> {
    return console.log('event to dispatch: ', packet);
  }

  publish(
    packet: ReadPacket<any>,
    callback: (packet: WritePacket<any>) => void,
  ): Function {
    console.log('message:', packet);

    // Gerçek bir uygulamada, "callback" fonksiyonu,
    // responder tarafından gönderilen yanıtla yürütülmelidir.
    // Burada, (5 saniye gecikme ile) orijinal olarak ilettiğimiz
    // "data" ile aynı şeyi geçirerek yanıtın geldiğini basitçe simüle edeceğiz.
    setTimeout(() => callback({ response: packet.data }), 5000);

    return () => console.log('teardown');
  }
}
```

Bu durumda, `GoogleCloudPubSubClient` sınıfının bir örneğini oluşturalım ve `send()` yöntemini çalıştıralım (daha önceki bölümlerde görmüş olabileceğiniz), dönen observable akışına abone olalım.

```typescript
const googlePubSubClient = new GoogleCloudPubSubClient();
googlePubSubClient
  .send('pattern', 'Hello world!')
  .subscribe((response) => console.log(response));
```

Şimdi, terminalinizde şu çıktıyı görmelisiniz:

```typescript
connect
message: { pattern: 'pattern', data: 'Hello world!' }
Hello world! // <-- 5 saniye sonra
```

"teardown" yöntemimizin (ki `publish()` yöntemimiz geri döndüğü) uygun şekilde yürütülüp yürütülmediğini test etmek için, akışımıza uygulanan bir `timeout` operatörü ekleyelim ve bu operatörü 2 saniye olarak ayarlayalım, böylece `setTimeout` fonksiyonumuz `callback` fonksiyonunu çağırmadan önce daha erken bir hata fırlatsın.

```typescript
const googlePubSubClient = new GoogleCloudPubSubClient();
googlePubSubClient
  .send('pattern', 'Hello world!')
  .pipe(timeout(2000))
  .subscribe(
    (response) => console.log(response),
    (error) => console.error(error.message),
  );
``

`

> info **Hint** `timeout` operatörü, `rxjs/operators` paketinden içe aktarılır.

`timeout` operatörü uygulandığında, terminal çıktınız şu şekilde görünmelidir:

```typescript
connect
message: { pattern: 'pattern', data: 'Hello world!' }
teardown // <-- yıkım
Timeout has occurred
```

Bir olayı iletmek (bir ileti göndermek yerine), `emit()` yöntemini kullanın:

```typescript
googlePubSubClient.emit('event', 'Hello world!');
```

Ve bunu konsolda görmelisiniz:

```typescript
connect
event to dispatch:  { pattern: 'event', data: 'Hello world!' }
```

#### Mesaj Serileştirme

İstemci tarafındaki yanıtların serileştirilmesi etrafında özel mantık eklemeniz gerekiyorsa, `ClientProxy` sınıfını veya onun alt sınıflarından birini genişleten özel bir sınıf kullanabilirsiniz. Başarılı istekleri değiştirmek için `serializeResponse` yöntemini geçersiz kılabilir ve bu istemciden geçen hataları değiştirmek için `serializeError` yöntemini geçersiz kılabilirsiniz. Bu özel sınıfı kullanmak için, sınıfı kendisi üzerine `ClientsModule.register()` yöntemine `customClass` özelliğini kullanarak iletebilirsiniz. Aşağıda, her hatayı bir `RpcException` içine seri hale getiren özel bir `ClientProxy` örneği bulunmaktadır.

```typescript
@@filename(error-handling.proxy)
import { ClientTcp, RpcException } from '@nestjs/microservices';

class ErrorHandlingProxy extends ClientTCP {
  serializeError(err: Error) {
    return new RpcException(err);
  }
}
```

ve ardından `ClientsModule` içinde şu şekilde kullanabilirsiniz:

```typescript
@@filename(app.module)
@Module({
  imports: [
    ClientsModule.register({
      name: 'CustomProxy',
      customClass: ErrorHandlingProxy,
    }),
  ]
})
export class AppModule
```

> info **Hint** Bu, `customClass`'a geçirilen sınıfın kendisi olup, sınıfın örneği değildir. Nest, arka planda sizin için örnek oluşturacak ve yeni `ClientProxy`'ye geçirilen herhangi bir seçeneği yeni `ClientProxy`'ye geçirecektir.