### Yaşam Döngüsü Olayları

Nest uygulaması, her uygulama öğesinin Nest tarafından yönetilen bir yaşam döngüsüne sahiptir. Nest, temel yaşam döngü olaylarına görünürlük sağlayan ve bunlar meydana geldiğinde harekete geçme yeteneği sunan **yaşam döngüsü kancaları (lifecycle hooks)** sağlar.

#### Yaşam Döngüsü Sıralaması

Aşağıdaki diyagram, uygulamanın başlatılmasından nod işlemi sonlandırılana kadar olan temel uygulama yaşam döngü olaylarının sıralamasını gösterir. Genel yaşam döngüsünü üç fazda inceleyebiliriz: **başlatma**, **çalışma** ve **sonlandırma**. Bu yaşam döngüsü kullanılarak modüllerin ve servislerin uygun şekilde başlatılması, aktif bağlantıların yönetilmesi ve uygulamanızın kapatılması için gerekli önlemleri alabilirsiniz.

<figure><img src="/assets/lifecycle-events.png" /></figure>

#### Yaşam Döngüsü Olayları

Yaşam döngüsü olayları, uygulamanın başlatılması ve kapatılması sırasında gerçekleşir. Nest, her bir yaşam döngü olayında (aşağıda belirtilenler arasında) modüllerde, sağlayıcılarda ve denetleyicilerde kayıtlı yaşam döngü kancası yöntemlerini çağırır (**kapatma kancaları** önce açılmalıdır, [aşağıda](https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown) açıklandığı gibi). Yukarıda gösterildiği gibi, Nest, bağlantıları dinlemeye başlamak ve bağlantıları dinlemeyi durdurmak için ilgili altta yatan yöntemleri de çağırır.

Aşağıdaki tabloda, `onModuleDestroy`, `beforeApplicationShutdown` ve `onApplicationShutdown` yalnızca `app.close()`'un açıkça çağrıldığı veya özel bir sistem sinyali alındığında (`SIGTERM` gibi) tetiklenir (aşağıda **Uygulama kapatma** bölümünde açıklandığı gibi).

| Yaşam Döngüsü Kancası Metodu   | Kancanın çağrıldığı yaşam döngüsü olayı                                                                                                                                           |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onModuleInit()`                | Ana modülün bağımlılıkları çözüldükten sonra çağrılır.                                                                                                                            |
| `onApplicationBootstrap()`      | Tüm modüller başlatıldıktan sonra, bağlantıları dinlemeye başlamadan önce çağrılır.                                                                                                |
| `onModuleDestroy()`\*           | Bir sonlandırma sinyali alındıktan sonra çağrılır (örneğin, `SIGTERM`).                                                                                                           |
| `beforeApplicationShutdown()`\* | Tüm `onModuleDestroy()` işleyicileri tamamlandıktan sonra çağrılır (Promiseler çözüldü veya reddedildi);<br />tamamlandıktan sonra (Promiseler çözüldü veya reddedildi), mevcut tüm bağlantılar kapatılır (`app.close()` çağrıldı). |
| `onApplicationShutdown()`\*     | Bağlantılar kapatıldıktan sonra (`app.close()` çözüldüğünde) çağrılır.                                                                                                            |

\* Bu olaylar için, `app.close()` açıkça çağırmıyorsanız, bunların sistem sinyalleriyle çalışması için **kendiniz etkinleştirmeniz gerekmektedir** (`SIGTERM` gibi). Aşağıdaki **Uygulama kapatma** bölümüne bakın.

> warning **Uyarı** Yukarıda listelenen yaşam döngü kancaları, **isteğe bağlı olmayan** sınıflar için tetiklenmez. İsteğe bağlı olmayan sınıflar, uygulama yaşam döngü

süne bağlı değildir ve ömürleri tahmin edilemez. Bunlar yalnızca her bir istek için oluşturulur ve yanıt gönderildikten sonra otomatik olarak bellekten atılır.

> info **Hint** `onModuleInit()` ve `onApplicationBootstrap()`'un yürütme sırası, önceki kancanın beklemesine bağlı olarak doğrudan modül içe aktarmalarının sırasına bağlıdır.

#### Kullanım

Her yaşam döngüsü kancası bir arayüzle temsil edilir. Arayüzler teknik olarak isteğe bağlıdır çünkü TypeScript derlemesinden sonra mevcut değildir. Bununla birlikte, güçlü yazım ve düzenleyici araçlardan yararlanmak için bunları kullanmak iyi bir uygulama yöntemidir. Bir yaşam döngüsü kancası kaydetmek için uygun arayüzü uygulayın. Örneğin, belirli bir sınıfta (Örneğin, Controller, Sağlayıcı veya Modül) modül başlatma sırasında çağrılacak bir yöntemi kaydetmek için `OnModuleInit` arayüzünü uygulayın ve bir `onModuleInit()` yöntemi sağlayın, aşağıdaki gibi:

```typescript
@@filename()
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class UsersService implements OnModuleInit {
  onModuleInit() {
    console.log(`The module has been initialized.`);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  onModuleInit() {
    console.log(`The module has been initialized.`);
  }
}
```

#### Asenkron Başlatma

`OnModuleInit` ve `OnApplicationBootstrap` kancaları, uygulama başlatma sürecini erteleme olanağı sağlar (`Promise` döndürün veya yöntemi `async` olarak işaretleyin ve yöntem gövdesinde asenkron bir yöntemin tamamlanmasını `await` ile bekleyin).

```typescript
@@filename()
async onModuleInit(): Promise<void> {
  await this.fetch();
}
@@switch
async onModuleInit() {
  await this.fetch();
}
```

#### Uygulama Kapatma

`onModuleDestroy()`, `beforeApplicationShutdown()` ve `onApplicationShutdown()` kancaları, sonlandırma aşamasında çağrılır (`app.close()` açıkça çağrıldığında veya `SIGTERM` gibi sistem sinyalleri alındığında). Bu özellik genellikle [Kubernetes](https://kubernetes.io/) tarafından konteyner ömür döngülerini yönetmek için, [Heroku](https://www.heroku.com/) için dynolar veya benzeri hizmetler tarafından kullanılır.

Kapatma kancaları dinleyicileri sistem kaynakları tüketir, bu nedenle varsayılan olarak devre dışı bırakılmıştır. Kapatma kancalarını kullanmak için, **dinleyicileri etkinleştirmeniz gerekir** ve bunu `enableShutdownHooks()` çağrarak yapmalısınız:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Kapatma kancalarını dinlemeye başlar
  app.enableShutdownHooks();

  await app.listen(3000);
}
bootstrap();
```

> warning **Uyarı** Doğuştan gelen platform kısıtlamaları nedeniyle, NestJS'nin Windows'ta uygulama kapatma kancalarına sınırlı destek sağlamaktadır. `SIGINT`'in çalışmasını bekleyebilirsiniz, ayrıca `SIGBREAK` ve bir ölçüde `SIGHUP` - [daha fazlasını okuyun](https://nodejs.org/api/process.html#process_signal_events). Bununla birlikte, `SIGTERM`, Windows'ta asla çalışmayacaktır çünkü görev yöneticisinde bir işlemi sonlandırmak koşulsuzdur, "yani bir uygulamanın bunu algılaması veya önlemesi için bir yol yoktur". İşte libuv'den [ilgili belge](https://docs.libuv.org/en/v1.x/signal.html) ve Windows'ta `SIGINT`, `SIGBREAK` ve diğerlerinin nasıl ele alındığını daha iyi anlamak için Node.js belgeleri [Process Signal Events](https://nodejs.org/api/process.html#process_signal_events).

> info **Info** `enableShutdownHooks`, dinleyicileri başlatarak bellek tüketir. Birden fazla Nest uygulamasını tek bir Node işleminde çalıştırıyorsanız (örneğin, Jest ile paralel testler çalıştırırken), Node, aşırı dinleyici işlemi hakkında şikayette bulunabilir. Bu nedenle, `enableShutdownHooks` varsayılan olarak etkin değildir. Bir Node işleminde birden çok örnek çalıştırırken bu durumu göz önünde bulundurun.