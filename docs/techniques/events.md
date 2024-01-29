### Olaylar (Events)

[Event Emitter](https://www.npmjs.com/package/@nestjs/event-emitter) paketi (`@nestjs/event-emitter`), uygulamanızdaki çeşitli olaylara abone olmanıza ve bu olayları dinlemenize olanak tanıyan basit bir gözlemci uygulaması sağlar. Olaylar, uygulamanızın çeşitli yönlerini bağımsız hale getirmenin harika bir yoludur, çünkü tek bir olay, birbirine bağlı olmayan birden fazla dinleyiciye sahip olabilir.

`EventEmitterModule` dahili olarak [eventemitter2](https://github.com/EventEmitter2/EventEmitter2) paketini kullanır.

#### Başlangıç

İlk olarak gerekli paketi yükleyin:

```shell
$ npm i --save @nestjs/event-emitter
```

Kurulum tamamlandıktan sonra, `EventEmitterModule`'u kök `AppModule`'a içe aktarın ve aşağıda gösterildiği gibi `forRoot()` statik yöntemini çalıştırın:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    EventEmitterModule.forRoot()
  ],
})
export class AppModule {}
```

`.forRoot()` çağrısı, olay yayınlayıcısını başlatır ve uygulamanızdaki mevcut bildirimsel olay dinleyicilerini kaydeder. Kayıt, `onApplicationBootstrap` yaşam döngü kancası meydana geldiğinde gerçekleşir, böylece tüm modüller yüklenmiş ve bildirim yapılmış olur.

Altta yatan `EventEmitter` örneğini yapılandırmak için, yapılandırma nesnesini `.forRoot()` yöntemine iletebilirsiniz, aşağıdaki gibi:

```typescript
EventEmitterModule.forRoot({
  // wildcards kullanmak için bunu `true` olarak ayarlayın
  wildcard: false,
  // isim alanlarını ayırmak için kullanılan ayraç
  delimiter: '.',
  // newListener olayını yayınlamak istiyorsanız bunu `true` olarak ayarlayın
  newListener: false,
  // removeListener olayını yayınlamak istiyorsanız bunu `true` olarak ayarlayın
  removeListener: false,
  // bir olaya atanabilecek maksimum dinleyici sayısı
  maxListeners: 10,
  // bellek sızıntısı mesajında olay adını gösterin
  verboseMemoryLeak: false,
  // bir hata olayı yayınladığında uncaughtException atmayı devre dışı bırakın
  ignoreErrors: false,
});
```

#### Olayları Tetikleme

Bir olayı tetiklemek (yani, başlatmak) için önce `EventEmitter2`'yi standart bir enjeksiyon kullanarak içe aktarın:

```typescript
constructor(private eventEmitter: EventEmitter2) {}
```

> info **Hint** `EventEmitter2`'yi `@nestjs/event-emitter` paketinden içe aktarın.

Ardından, bir sınıfta şu şekilde kullanın:

```typescript
this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({
    orderId: 1,
    payload: {},
  }),
);
```

#### Olayları Dinleme

Bir olay dinleyiciyi bildirmek için, bir yöntemi `@OnEvent()` dekoratörü ile önceleme yöntemi tanımını içeren bir yöntemin önünde kullanın:

```typescript
@OnEvent('order.created')
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // "OrderCreatedEvent" olayını işle ve işle
}
```

> warning **Uyarı** Olay aboneleri talep kapsamında olamaz.

İlk argüman, basit bir olay yayıncısı için bir `string` veya `symbol` olabilir ve bir joker yayıncısı durumunda bir `string | symbol | Array<string | symbol>` olabilir.  

İkinci argüman (isteğe bağlı), aşağıdaki gibidir:

```typescript
export type OnEventOptions = OnOptions & {
  /**
   * "true" ise, verilen dinleyiciyi (eklem

ek yerine) diziye atanmış dinleyicilerin başına ekler.
   *
   * @see https://github.com/EventEmitter2/EventEmitter2#emitterprependlistenerevent-listener-options
   *
   * @default false
   */
  prependListener?: boolean;

  /**
   * "true" ise, onEvent geri çağrısı olayı işlerken bir hata oluşmaz. Aksi takdirde, "false" ise bir hata fırlatır.
   * 
   * @default true
   */
  suppressErrors?: boolean;
};
```

> info **Hint** `eventemitter2` tarafından sağlanan `OnOptions` nesnesi hakkında daha fazla bilgi için [buraya](https://github.com/EventEmitter2/EventEmitter2#emitteronevent-listener-options-objectboolean) bakın.

```typescript
@OnEvent('order.created', { async: true })
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // "OrderCreatedEvent" olayını işle ve işle
}
```

Ad alanları/joker karakterleri kullanmak için `EventEmitterModule#forRoot()` yöntemine `wildcard` seçeneğini geçirin. Ad alanları/joker karakterleri etkinleştirildiğinde, olaylar bir ayraç ile ayrılmış olan dize (`foo.bar`) veya bir dizi (`['foo', 'bar']`) olabilir. Ayraç da yapılandırılabilir bir konfigürasyon özelliği (`delimiter`) olarak mevcuttur. Ayraç özelliği etkin olduğunda, bir joker kullanarak olaylara abone olabilirsiniz:

```typescript
@OnEvent('order.*')
handleOrderEvents(payload: OrderCreatedEvent | OrderRemovedEvent | OrderUpdatedEvent) {
  // bir olayı işle ve işle
}
```

Bu tür bir joker yalnızca bir bloğa uygulanır. `order.created` ve `order.shipped` gibi olaylarla eşleşecektir ancak `order.delayed.out_of_stock` gibi olaylara eşleşmez.
Bu tür olayları dinlemek için, `multilevel wildcard` modelini (yani, `**`) kullanın, `EventEmitter2` [belgelerinde](https://github.com/EventEmitter2/EventEmitter2#multi-level-wildcards) açıklandığı gibi.

Bu modelle, örneğin tüm olayları yakalayan bir olay dinleyici oluşturabilirsiniz.

```typescript
@OnEvent('**')
handleEverything(payload: any) {
  // bir olayı işle ve işle
}
```

> info **Hint** `EventEmitter2` sınıfı, `waitFor` ve `onAny` gibi olaylarla etkileşimde bulunmak için birkaç kullanışlı yöntem sağlar. Bunlar hakkında daha fazla bilgi için [buraya](https://github.com/EventEmitter2/EventEmitter2) bakabilirsiniz.

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/30-event-emitter) bulunmaktadır.