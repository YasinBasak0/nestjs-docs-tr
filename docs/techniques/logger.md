### Logger

Nest, uygulama başlatma ve diğer birkaç durum (örneğin, yakalanan istisnaları görüntüleme gibi) sırasında kullanılan, metin tabanlı bir yerleşik günlükçü ile birlikte gelir. Bu işlevsellik, `@nestjs/common` paketinde bulunan `Logger` sınıfı aracılığıyla sağlanır. Günlük sisteminin davranışını tamamen kontrol edebilirsiniz, bunlar arasında şunlar bulunur:

- Günlüğü tamamen devre dışı bırakma
- Günlüğün detay seviyesini belirtme (örneğin, hataları, uyarıları, hata ayıklama bilgilerini vb. görüntüleme)
- Varsayılan günlükçüde zaman damgasını geçersiz kılma (örneğin, tarih formatı olarak ISO8601 standardını kullanma)
- Varsayılan günlükçüyü tamamen geçersiz kılma
- Varsayılan günlükçüyü genişleterek özelleştirme
- Uygulamanızı oluşturmayı ve test etmeyi basitleştirmek için bağımlılık enjeksiyonunu kullanma

Ayrıca, kendi uygulama düzeyindeki olaylarınızı ve iletilerinizi günlüklemek için yerleşik günlükçiyi kullanabilir veya özel bir uygulama seviyesi günlükleme uygulamanızı oluşturabilirsiniz.

Daha gelişmiş günlükleme işlevselliği için [Winston](https://github.com/winstonjs/winston) gibi herhangi bir Node.js günlük paketini kullanabilir ve tamamen özel, üretim kalitesinde bir günlükleme sistemini uygulayabilirsiniz.

#### Temel özelleştirme

Günlüğü devre dışı bırakmak için, (isteğe bağlı) Nest uygulama seçenekleri nesnesinin ikinci argümanı olarak geçirilen `{ logger: false }` değerini `NestFactory.create()` yönteminde ayarlayın.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: false,
});
await app.listen(3000);
```

Belirli günlük seviyelerini etkinleştirmek için, `logger` özelliğini görüntülenecek günlük seviyelerini belirten bir dizi dize olarak ayarlayın, aşağıdaki gibi:

```typescript
const app = await NestFactory.create(AppModule, {
  logger: ['error', 'warn'],
});
await app.listen(3000);
```

Dizideki değerler, `'log'`, `'fatal'`, `'error'`, `'warn'`, `'debug'` ve `'verbose'` olmak üzere herhangi bir kombinasyon olabilir.

> info **İpucu** Varsayılan günlükçünün iletilerinde renk kullanımını devre dışı bırakmak için, `NO_COLOR` ortam değişkenini bir boş dizeye ayarlayın.

#### Özel uygulama

Nest'in sistem günlüğü için kullanılacak özel bir günlükçü uygulamasını sağlamak için `logger` özelliğinin değerini `LoggerService` arayüzünü karşılayan bir nesne olarak ayarlayabilirsiniz. Örneğin, Nest'e yerleşik JavaScript `console` nesnesini kullanmasını söyleyebilirsiniz (bu, `LoggerService` arayüzünü uygular), aşağıdaki gibi:

```typescript
const app = await NestFactory.create(AppModule, {
  logger: console,
});
await app.listen(3000);
```

Kendi özel günlükçünüzü uygulamak oldukça basittir. `LoggerService` arayüzünün her bir yöntemini aşağıda gösterildiği gibi uygulamanız yeterlidir.

```typescript
import { LoggerService } from '@nestjs/common';

export class MyLogger implements LoggerService {
  /**
   * 'log' seviyesinde günlük yaz.
   */
  log(message: any, ...optionalParams: any[]) {}

  /**
   * 'fatal' seviyesinde günlük yaz.
   */
  fatal(message: any, ...optionalParams: any[]) {}

  /**
   * 'error' seviyesinde günlük yaz.
   */
 

 error(message: any, ...optionalParams: any[]) {}

  /**
   * 'warn' seviyesinde günlük yaz.
   */
  warn(message: any, ...optionalParams: any[]) {}

  /**
   * 'debug' seviyesinde günlük yaz.
   */
  debug?(message: any, ...optionalParams: any[]) {}

  /**
   * 'verbose' seviyesinde günlük yaz.
   */
  verbose?(message: any, ...optionalParams: any[]) {}
}
```

Daha sonra, Nest uygulama seçenekleri nesnesinin `logger` özelliğine `MyLogger` örneğini geçirerek bu özel günlükçüyü sağlayabilirsiniz.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new MyLogger(),
});
await app.listen(3000);
```

Bu teknik, basit olmasına rağmen, `MyLogger` sınıfı için bağımlılık enjeksiyonunu kullanmaz. Bu, özellikle test için zorluklar yaratabilir ve `MyLogger`'ın yeniden kullanılabilirliğini sınırlayabilir. Daha iyi bir çözüm için aşağıdaki <a href="#dependency-injection">Bağımlılık Enjeksiyonu</a> bölümüne bakın.

#### Yerleşik günlükçüyü genişletme

Sıfırdan bir günlükçü yazmak yerine, yerleşik `ConsoleLogger` sınıfını genişleterek ve varsayılan uygulamanın beklediği belirli davranışları geçersiz kılarak ihtiyaçlarınızı karşılayabilirsiniz.

```typescript
import { ConsoleLogger } from '@nestjs/common';

export class MyLogger extends ConsoleLogger {
  error(message: any, stack?: string, context?: string) {
    // özelleştirilmiş mantığınızı buraya ekleyin
    super.error(...arguments);
  }
}
```

Bu tür bir genişletilmiş günlükçeyi aşağıdaki <a href="#using-the-logger-for-application-logging">Uygulama günlüğü için günlükçeyi kullanma</a> bölümünde açıklanan şekilde modül özelliklerinizde kullanabilirsiniz.

Sistem günlüğü için Nest'in bu genişletilmiş günlükçeyi kullanmasını sağlamak için, bir örneğini uygulama seçenekleri nesnesinin `logger` özelliği aracılığıyla geçirerek (yukarıdaki <a href="#custom-logger-implementation">Özel uygulama</a> bölümünde gösterildiği gibi) veya aşağıdaki <a href="#dependency-injection">Bağımlılık Enjeksiyonu</a> bölümünde gösterilen teknikleri kullanarak iletebilirsiniz. Bu şekilde yaparsanız, yukarıdaki örnek kodda gösterildiği gibi özel log yöntem çağrısını (Nest'in beklediği özelliklere güvenebilmesi için) yapmak için `super`'ı çağırdığınızdan emin olmalısınız.

<app-banner-courses></app-banner-courses>

#### Bağımlılık Enjeksiyonu

Daha gelişmiş günlükleme işlevselliği için bağımlılık enjeksiyonundan faydalanmak isteyeceksiniz. Örneğin, `ConfigService`'yi özelleştirmek için logger'ınıza enjekte etmek isteyebilir ve sırasıyla özel logger'ınızı diğer denetleyicilere ve/veya sağlayıcılara enjekte etmek isteyebilirsiniz. Özel logger'ınız için bağımlılık enjeksiyonunu etkinleştirmek için, `LoggerService`'i uygulayan bir sınıf oluşturun ve bu sınıfı bir modülde sağlayıcı olarak kaydedin. Örneğin, aşağıdaki adımları takip edebilirsiniz:

1. Ya yerleşik `ConsoleLogger`'ı genişleten ya da tamamen geçersiz kılan önceki bölümlerde gösterildiği gibi `LoggerService` arabirimini uygulayan bir `MyLogger` sınıfı tanımlayın.
2. Aşağıda gösterildiği gibi bir `LoggerModule` oluşturun ve bu modülden `MyLogger`'ı sağlayın.

```typescript
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```

Bu yapı ile şimdi özel logger'ınızı başka herhangi bir modül tarafından kullanım için sağlıyorsunuzdur. `MyLogger` sınıfınız bir modülün bir parçası olduğu için bağımlılık enjeksiyonunu kullanabilir (örneğin, bir `ConfigService`'yi enjekte etmek için). Bu özel logger'ı Nest için sistem günlüğü için kullanılmak üzere sağlamak için başka bir teknik daha gerekmektedir (örneğin, başlatma ve hata işleme).

Çünkü uygulama başlatma (`NestFactory.create()`) işlemi herhangi bir modül bağlamında gerçekleşmez, bu nedenle normal Bağımlılık Enjeksiyonu başlatma aşamasına katılmaz. Bu nedenle, en azından bir uygulama modülünün `LoggerModule`'u içe aktarmasını sağlayarak Nest'in `MyLogger` sınıfının tekil bir örneğini oluşturmasını sağlamalıyız.

Daha sonra, Nest'e `MyLogger`'ın aynı tekil örneğini kullanması için aşağıdaki şekilde talimat verebiliriz:

```typescript
const app = await NestFactory.create(AppModule, {
  bufferLogs: true,
});
app.useLogger(app.get(MyLogger));
await app.listen(3000);
```

> Bilgi **Not:** Yukarıdaki örnekte `bufferLogs`'u `true` olarak ayarladık, bu sayede özel logger (`MyLogger` bu durumda) eklenene kadar tüm günlüklerin biriktirileceğinden ve uygulama başlatma sürecinin tamamlanıp veya başarısız olduğundan emin olabiliriz. Başlatma süreci başarısız olursa, Nest, herhangi rapor edilmiş hata mesajlarını bastırmak için orijinal `ConsoleLogger`'a geri düşecektir. Ayrıca, `autoFlushLogs`'u `false` olarak ayarlayabilirsiniz (varsayılan `true`) ve günlükleri manuel olarak boşaltmak için (`Logger#flush()` yöntemini kullanarak).

Burada, `NestApplication` örneğindeki `get()` yöntemini kullanarak `MyLogger` nesnesinin tekil örneğini almak için kullanıyoruz. Bu teknik, aslında Nest tarafından kullanılmak üzere bir logger örneğini "enjekte" etmenin bir yoludur. `app.get()` çağrısı, `MyLogger`'ın tekil örneğini alır ve yukarıda açıklandığı gibi bu örneğin önce başka bir modülde enjekte edilmiş olmasına bağlıdır.

Ayrıca, bu `MyLogger` sağlayıcısını özellik sınıflarınıza enjekte edebilir ve böylece hem Nest sistem günlüğü hem de uygulama günlüğü için tutarlı günlükleme davranışını sağlamış olursunuz. Daha fazla bilgi için aşağıdaki bağlantılara bakabilirsiniz: <a href="techniques/logger#using-the-logger-for-application-logging">Uygulama Günlüğü İçin Logger Kullanma</a> ve <a href="techniques/logger#injecting-a-custom-logger">Özel Logger Enjekte Etme</a>.

#### Uygulama Günlüğü İçin Logger Kullanma

Yukarıdaki tekniklerin birkaçını birleştirerek, hem Nest sistem günlüğü hem de kendi uygulama olay/mesaj günlüğümüzde tutarlı davranış ve biçimlendirme sağlayabiliriz.

Her bir servisimizde `@nestjs/common` modülünden `Logger` sınıfını örneklemek iyi bir uygulama olacaktır. Servis adımızı `Logger` kurucu metodundaki `context` argümanı olarak sağlayabiliriz, örneğin:

```typescript
import { Logger, Injectable } from '@nestjs/common';

@Injectable()
class MyService {
  private readonly logger = new Logger(MyService.name);

  doSomething() {
    this.logger.log('Bir şeyler yapılıyor...');
  }
}
```

Varsayılan logger uygulamasında, `context` köşeli parantez içinde yazdırılır, örneğin aşağıdaki örnekteki gibi `NestFactory`:

```bash
[Nest] 19096   - 12/08/2019, 7:12:59 AM   [NestFactory] Starting Nest application...
```

Eğer `app.useLogger()` ile özel bir logger sağlarsak, bu Nest içinde

 gerçekten kullanılacaktır. Bu, kodumuzun uygulama bağımsız olmasını sağlar, ancak `app.useLogger()`'ı çağırarak varsayılan logger'ı özel logger ile kolayca değiştirebiliriz.

Bu şekilde, yukarıdaki bölümdeki adımları takip eder ve `app.useLogger(app.get(MyLogger))`'ı çağırırsak, `MyService`'den gelen `this.logger.log()` çağrıları, `MyLogger` örneğinin `log` metodunu çağırır hale gelir.

Bu çoğu durum için uygun olmalıdır. Ancak daha fazla özelleştirme ihtiyacınız varsa (örneğin, özel yöntemler eklemek ve çağırmak gibi), bir sonraki bölüme geçebilirsiniz.

#### Özel Bir Logger Enjekte Etme

İlk olarak, aşağıdaki gibi yerleşik logger'ı genişletin. `ConsoleLogger` sınıfına yapılandırma metadata olarak `scope` seçeneğini sağlarız, [geçici](/docs/fundamentals/injection-scopes) bir kapsam belirtiriz, böylece her özellik modülünde benzersiz bir `MyLogger` örneğine sahip olacağımızı sağlarız. Bu örnekte, `ConsoleLogger` yöntemlerini (`log()`, `warn()`, vb.) genişletmiyoruz, ancak isterseniz genişletebilirsiniz.

```typescript
import { Injectable, Scope, ConsoleLogger } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class MyLogger extends ConsoleLogger {
  customLog() {
    this.log('Lütfen kediye yemek verin!');
  }
}
```

Daha sonra, şu şekilde bir yapıya sahip bir `LoggerModule` oluşturun:

```typescript
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```

Sonra, `LoggerModule`'u özellik modülünüze içe aktarın. Varsayılan `Logger`'ı genişlettiğimiz için `setContext` yöntemini kullanma kolaylığına sahibiz. Bu nedenle, bağlam bilincine sahip özel logger'ı kullanmaya başlayabiliriz, örneğin:

```typescript
import { Injectable } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  constructor(private myLogger: MyLogger) {
    // Geçici kapsam nedeniyle, CatsService'in kendi benzersiz MyLogger örneğine sahiptir,
    // bu nedenle burada bağlam ayarlamak diğer hizmetlerdeki diğer örnekleri etkilemeyecektir
    this.myLogger.setContext('CatsService');
  }

  findAll(): Cat[] {
    // Tüm varsayılan yöntemleri çağırabilirsiniz
    this.myLogger.warn('Kedileri döndürmeye hazırlanılıyor!');
    // Ve özel yöntemlerinizi çağırabilirsiniz
    this.myLogger.customLog();
    return this.cats;
  }
}
```

Son olarak, `main.ts` dosyanızda aşağıda gösterildiği gibi Nest'e özel logger örneğini kullanmasını söyleyin. Elbette bu örnekte, logger davranışını gerçekten özelleştirmedik (örneğin, `log()`, `warn()`, vb. yöntemleri genişleterek), bu nedenle bu adım aslında gerekli değildir. Ancak bu adım, bu yöntemlere özel mantık eklediyseniz ve Nest'in aynı uygulamayı kullanmasını istiyorsanız gereklidir.

```typescript
const app = await NestFactory.create(AppModule, {
  bufferLogs: true,
});
app.useLogger(new MyLogger());
await app.listen(3000);
```

> Bilgi **İpucu:** Bunun yerine `bufferLogs`'u `true` olarak ayarlamak yerine, geçici olarak `logger: false` talimatı ile logger'ı devre dışı bırakabilirsiniz. Unutmayın ki `NestFactory.create`'e `logger: false` sağlarsanız, `useLogger`'ı çağırmadıkça hiçbir şey kaydedilmez, bu nedenle bazı önemli başlatma hatalarını kaçırabilirsiniz. Başlangıçtaki mesajlarınızın bir kısmının varsayılan logger ile kaydedilmesini önemsemiyorsanız, `logger: false` seçeneğini sadece atlayabilirsiniz.

#### Harici Logger Kullanma

Üretim uygulamaları genellikle gelişmiş filtreleme, biçimlendirme ve merkezi günlükleme gibi belirli günlük tutma gereksinimlerine sahiptir. Nest'in yerleşik logger'ı, Nest sistemi davranışını izleme amacıyla kullanılır ve geliştirmede özellik modüllerinizde temel biçimlendirilmiş metin günlükleme için de kullanışlı olabilir, ancak üretim uygulamaları genellikle [Winston](https://github.com/winstonjs/winston) gibi özel günlük modüllerinden yararlanır. Standart bir Node.js uygulaması gibi, Nest'te bu tür modüllerin tam avantajını alabilirsiniz.

