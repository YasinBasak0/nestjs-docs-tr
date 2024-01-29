### Görev Zamanlaması

Görev zamanlaması, belirli bir tarihte/saatte, tekrarlayan aralıklarla veya belirli bir aralıktan sonra bir kere olmak üzere herhangi bir kodu (metot/fonksiyon) zamanlamayı sağlar. Linux dünyasında, bu genellikle OS seviyesinde [cron](https://en.wikipedia.org/wiki/Cron) gibi paketlerle ele alınır. Node.js uygulamaları için, cron benzeri işlevselliği taklit eden birkaç paket bulunmaktadır. Nest, popüler Node.js [cron](https://github.com/kelektiv/node-cron) paketiyle entegre olan `@nestjs/schedule` paketini sağlar. Bu paketi bu bölümde ele alacağız.

#### Kurulum

Kullanmaya başlamak için önce gerekli bağımlılıkları yükleriz.

```bash
$ npm install --save @nestjs/schedule
```

Görev zamanlamasını etkinleştirmek için, `ScheduleModule`'u ana `AppModule`'a içe aktarır ve aşağıdaki gibi `forRoot()` statik yöntemini çalıştırırız:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot()
  ],
})
export class AppModule {}
```

`.forRoot()` çağrısı, zamanlayıcıyı başlatır ve uygulamanızdaki var olan herhangi bir deklaratif <a href="techniques/task-scheduling#declarative-cron-jobs">cron işi</a>, <a href="techniques/task-scheduling#declarative-timeouts">zaman aşımı</a> ve <a href="techniques/task-scheduling#declarative-intervals">aralıklar</a> 'ı kaydeder. Kayıt işlemi, `onApplicationBootstrap` yaşam döngüsü kancası gerçekleştiğinde, tüm modüllerin yüklendiğinden ve herhangi bir zamanlanmış işin bildirildiğinden emin olur.

#### Deklaratif cron işleri

Cron işlemi, herhangi bir fonksiyonu (metodun çağrısını) otomatik olarak çalıştırmak için bir zamanlamadır. Cron işleri şunlarda çalışabilir:

- Belirli bir tarih/saatte bir kere.
- Tekrarlanan aralıklarla; tekrarlayan işler, belirli bir aralık içinde belirli bir anında çalışabilir (örneğin, her saat başında, her hafta bir kez, her 5 dakikada bir)

Bir cron işi, yürütilecek kodu içeren metodu tanımlayan `@Cron()` dekoratörü ile şu şekilde ilan edilir:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron('45 * * * * *')
  handleCron() {
    this.logger.debug('Current second is 45');
  }
}
```

Bu örnekte, `handleCron()` metodu mevcut saniye `45` olduğunda her çağrıldığında çalışacaktır. Başka bir deyişle, metot her dakika başında, 45. saniye işaretinde çalışacaktır.

`@Cron()` dekoratörü, tüm standart [cron desenlerini](http://crontab.org/) destekler:

- Yıldız işareti (örneğin, `*`)
- Aralıklar (örneğin, `1-3,5`)
- Adımlar (örneğin, `*/2`)

Yukarıdaki örnekte, dekoratöre `45 * * * * *` değerini geçtik. Aşağıdaki anahtar, cron desen dizgisindeki her pozisyonun nasıl yorumlandığını gösterir:

<pre class="language-javascript"><code class="language-javascript">
* * * * * *
| | | | | |
| | | | | haftanın günü
| | | | ay
| | | ayın günü
| | saat
| dakikalar
saniyeler (isteğe bağlı)
</code></pre>

Bazı örnek cron desenleri şunlardır:

<table>
  <tbody>
    <tr>
      <td><code>* * * * * *</code></td>
      <td>her saniye</td>
    </tr>
    <tr>
      <td><code>45 * * * * *</code></td>
      <td>her dakika, 45. saniye</td>
    </tr>
    <tr>
      <td><code>0 10 * * * *</code></td>
      <td>her saat, 10. dakikanın başında</td>
    </tr>
    <tr>
      <td><code>0 */30 9-17 * * *</code></td>
      <td>9 ila 17 saat arasında her 30 dakikada bir</td>
    </tr>
   <tr>
      <td><code>0 30 11 * * 1-5</code></td>
      <td>Pazartesi'den Cuma'ya 11:30'da</td>
    </tr>
  </tbody>
</table>

`@nestjs/schedule` paketi, yaygın olarak kullanılan cron desenleriyle uyumlu bir enum sağlar. Bu enumu aşağıdaki gibi kullanabilirsiniz:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron(CronExpression.EVERY_30_SECONDS)
  handleCron() {
    this.logger.debug('Her 30 saniyede bir çağrıldı');
  }
}
```

Bu örnekte, `handleCron()` metodu her `30` saniyede bir çağrılacaktır.

Alternatif olarak, `@Cron()` dekoratörüne JavaScript `Date` nesnesi sağlayabilirsiniz. Bu, işin belirtilen tarihte tam olarak bir kez çalışmasına neden olur.

> info **İpucu** İşleri, mevcut tarihe göre zamanlanmış olarak planlamak için JavaScript tarih aritmetiğini kullanın. Örneğin, uygulama başladıktan 10 saniye sonra bir işi planlamak için `@Cron(new Date(Date.now() + 10 * 1000))` kullanın.

Ayrıca, `@Cron()` dekoratörüne ikinci parametre olarak geçici bir seçenek nesnesi sağlayabilirsiniz.

<table>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>
        Bir cron işlemine daha sonra erişmek ve kontrol etmek için kullanışlıdır.
      </td>
    </tr>
    <tr>
      <td><code>timeZone</code></td>
      <td>
        Yürütme için zaman dilimini belirtin. Bu, zamanın gerçek zamanına göre dilimi değiştirir. Zaman dilimi geçersizse bir hata fırlatılır. Tüm kullanılabilir saat dilimlerini <a href="http://momentjs.com/timezone/">Moment Timezone</a> web sitesinde kontrol edebilirsiniz.
      </td>
    </tr>
    <tr>
      <td><code>utcOffset</code></td>
      <td>
        Bu, zaman diliminizin yerine geçmek için bir ofset belirtmenize olanak tanır.
      </td>
    </tr>
    <tr>
      <td><code>disabled</code></td>
      <td>
       Bu, işin hiç çalıştırılıp çalıştırılmayacağını belirtir.
      </td>
    </tr>
  </tbody>
</table>

```typescript
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class NotificationService {
  @Cron('* * 0 * * *', {


    name: 'notifications',
    timeZone: 'Europe/Paris',
  })
  triggerNotifications() {}
}
```

Bir cron işlemine daha sonra erişmek ve kontrol etmek veya <a href="/techniques/task-scheduling#dynamic-schedule-module-api">Dinamik API</a> ile çalışma zamanında bir cron işlemi oluşturmak (cron deseni çalışma zamanında tanımlandığında) için API üzerinden bir cron işlemine erişmek istiyorsanız, dekoratörün ikinci argümanı olarak opsiyonel bir seçenek nesnesinde `name` özelliğini belirtmelisiniz.

#### Deklaratif aralıklar

Bir metodun (tekrar eden) belirli bir aralıkta çalışması gerektiğini bildirmek için metot tanımını `@Interval()` dekoratörü ile öncelemelisiniz. Aşağıda gösterildiği gibi dekoratöre aralık değerini, milisaniye cinsinden bir sayı olarak geçirin:

```typescript
@Interval(10000)
handleInterval() {
  this.logger.debug('Her 10 saniyede bir çağrıldı');
}
```

> info **İpucu** Bu mekanizma, JavaScript `setInterval()` fonksiyonunu arka planda kullanır. Ayrıca, tekrarlayan işleri planlamak için bir cron işlemi de kullanabilirsiniz.

Deklaratif aralığınızı bildiren sınıf dışında deklare edilen bir aralığı <a href="/techniques/task-scheduling#dynamic-schedule-module-api">Dinamik API</a> ile kontrol etmek istiyorsanız, aşağıdaki yapıyı kullanarak aralığı bir isimle ilişkilendirebilirsiniz:

```typescript
@Interval('notifications', 2500)
handleInterval() {}
```

<a href="techniques/task-scheduling#dynamic-intervals">Dinamik API</a>, aralıkların özelliklerinin çalışma zamanında tanımlandığı **oluşturulması** ve aynı zamanda bunları **listeleyip silmesini** sağlar.

<app-banner-enterprise></app-banner-enterprise>

#### Deklaratif zaman aşımı

Bir metodun (bir kere) belirli bir zaman aşımında çalışması gerektiğini bildirmek için metot tanımını `@Timeout()` dekoratörü ile öncelemelisiniz. Aşağıda gösterildiği gibi, uygulama başlangıcından itibaren geçen süreyi (milisaniye cinsinden) dekoratöre geçirin:

```typescript
@Timeout(5000)
handleTimeout() {
  this.logger.debug('5 saniye sonra bir kez çağrıldı');
}
```

> info **İpucu** Bu mekanizma, JavaScript `setTimeout()` fonksiyonunu arka planda kullanır.

Deklaratif zaman aşımınızı bildiren sınıf dışında deklare edilen bir zaman aşımını <a href="/techniques/task-scheduling#dynamic-schedule-module-api">Dinamik API</a> ile kontrol etmek istiyorsanız, aşağıdaki yapıyı kullanarak zaman aşımını bir isimle ilişkilendirebilirsiniz:

```typescript
@Timeout('notifications', 2500)
handleTimeout() {}
```

<a href="techniques/task-scheduling#dynamic-timeouts">Dinamik API</a>, zaman aşımının özelliklerinin çalışma zamanında tanımlandığı **oluşturulması** ve aynı zamanda bunları **listeleyip silmesini** sağlar.

#### Dinamik zamanlama modülü API'si

`@nestjs/schedule` modülü, deklaratif <a href="techniques/task-scheduling#declarative-cron-jobs">cron işleri</a>, <a href="techniques/task-scheduling#declarative-timeouts">zaman aşımı</a> ve <a href="techniques/task-scheduling#declarative-intervals">aralıklar</a> yönetmeyi sağlayan bir dinamik API sunar. API ayrıca özellikleri çalışma zamanında tanımlanan **dinamik** cron işleri, zaman aşımı ve aralıklar oluşturmayı ve yönetmeyi mümkün kılar.

#### Dinamik cron işleri

Herhangi bir yerden adıyla bir `CronJob` örneğine başvurmak için `SchedulerRegistry` API'sini kullanın. İlk olarak, standart kurucu enjeksiyonunu kullanarak `SchedulerRegistry`'yi enjekte edin:

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

> info **İpucu** `SchedulerRegistry`'yi `@nestjs/schedule` paketinden içe aktarın.

Ardından, bir sınıfta şu şekilde kullanın. Aşağıdaki deklarasyonla bir cron işi oluşturulduğunu varsayalım:

```typescript
@Cron('* * 8 * * *', {
  name: 'notifications',
})
triggerNotifications() {}
```

Bu işe aşağıdaki gibi erişin:

```typescript
const job = this.schedulerRegistry.getCronJob('notifications');

job.stop();
console.log(job.lastDate());
```

`getCronJob()` metodu adlandırılmış cron işini döndürür. Dönen `CronJob` nesnesi şu yöntemlere sahiptir:

- `stop()` - çalıştırılması planlanan bir işi durdurur.
- `start()` - durdurulmuş bir işi yeniden başlatır.
- `setTime(time: CronTime)` - bir işi durdurur, yeni bir zaman ayarlar ve ardından başlatır.
- `lastDate()` - işin en son ne zaman çalıştığının dize temsili.
- `nextDates(count: number)` - önümüzdeki iş yürütme tarihlerini temsil eden `moment` nesneleri dizisini döndürür.

> info **İpucu** `moment` nesnelerinde `toDate()` kullanarak onları insan okunabilir biçimde rendere edebilirsiniz.

Yeni bir cron işi **dynamically** oluşturmak için `SchedulerRegistry#addCronJob` metodunu kullanın, aşağıdaki gibi:

```typescript
addCronJob(name: string, seconds: string) {
  const job = new CronJob(`${seconds} * * * * *`, () => {
    this.logger.warn(`job ${name}'in çalışması için ${seconds} saniye!`);
  });

  this.schedulerRegistry.addCronJob(name, job);
  job.start();

  this.logger.warn(
    `${name} adlı iş her dakikada bir ${seconds} saniyesinde eklendi!`,
  );
}
```

Bu kodda, `CronJob`'u `cron` paketinden alır ve kullanırken `CronJob`'un oluşturulması bir cron deseni (tıpkı `@Cron()` <a href="techniques/task-scheduling#declarative-cron-jobs">dekoratörü</a> gibi) ve tetiklendiğinde çalışacak bir geri çağrı fonksiyonu olmak üzere iki argüman alır. `SchedulerRegistry#addCronJob` metodu iki argüman alır: bir `CronJob` için bir ad ve `CronJob` nesnesi.

> warning **Uyarı** Erişmeden önce `SchedulerRegistry`'yi enjekte etmeyi unutmayın. `CronJob`'u `cron` paketinden içe aktarın.

Adı verilen bir cron işi **silin** `SchedulerRegistry#deleteCronJob` metodu kullanılarak şu şekilde:

```typescript
deleteCron(name: string) {
  this.schedulerRegistry.deleteCronJob(name);
  this.logger.warn(`job ${name} silindi!`);
}
```

Tüm cron işlerini **listeleyin** `SchedulerRegistry#getCronJobs` metodu kullanılarak şu şekilde:

```typescript
getCrons() {
  const jobs = this.schedulerRegistry.getCronJobs();
  jobs.forEach((value, key, map) => {
    let next;
    try {
      next = value.nextDates().toDate();
    } catch (e) {
      next = 'hata: bir sonraki çalışma tarihi geçmişte!';
    }
    this.logger.log(`iş: ${key} -> bir sonraki: ${next}`);
  });
}
```

`getCronJobs()` metodu bir `map` döndürür. Bu kodda, `map` üzerinde dolaşır ve her `CronJob`'un `nextDates()` metoduna erişmeye çalışırız. `CronJob` API'sinde, bir iş zaten çalıştıysa ve gelecekteki çalışma tarihleri yoksa, bir istisna fırlatır.

#### Dinamik aralıklar

Bir aralığa `SchedulerRegistry#getInterval` yöntemiyle başvurun. Yukarıdaki gibi, standart kurucu enjeksiyonunu kullanarak `SchedulerRegistry`'yi enjekte edin:

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

Ve aşağıdaki gibi kullanın:

```typescript
const interval = this.schedulerRegistry.getInterval('notifications');
clearInterval(interval);
```

`SchedulerRegistry#addInterval` yöntemi kullanılarak yeni bir aralık **dinamik** olarak oluşturun, aşağıdaki gibi:

```typescript
addInterval(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Aralık ${name}, zaman (${milliseconds})'te çalışıyor!`);
  };

  const interval = setInterval(callback, milliseconds);
  this.schedulerRegistry.addInterval(name, interval);
}
```

Bu kodda, bir standart JavaScript aralığı oluşturuyoruz, ardından `SchedulerRegistry#addInterval` yöntemine geçiriyoruz.
Bu yöntem iki argüman alır: bir aralık için bir ad ve aralık kendisi.

Adı verilen bir aralığı `SchedulerRegistry#deleteInterval` yöntemi kullanarak **silin**, aşağıdaki gibi:

```typescript
deleteInterval(name: string) {
  this.schedulerRegistry.deleteInterval(name);
  this.logger.warn(`Aralık ${name} silindi!`);
}
```

Tüm aralıkları `SchedulerRegistry#getIntervals` yöntemi kullanılarak **listeleyin**:

```typescript
getIntervals() {
  const intervals = this.schedulerRegistry.getIntervals();
  intervals.forEach(key => this.logger.log(`Aralık: ${key}`));
}
```

#### Dinamik zaman aşımı

Zaman aşımına `SchedulerRegistry#getTimeout` yöntemiyle başvurun. Yukarıdaki gibi, standart kurucu enjeksiyonunu kullanarak `SchedulerRegistry`'yi enjekte edin:

```typescript
constructor(private readonly schedulerRegistry: SchedulerRegistry) {}
```

Ve aşağıdaki gibi kullanın:

```typescript
const timeout = this.schedulerRegistry.getTimeout('notifications');
clearTimeout(timeout);
```

Yeni bir zaman aşımını `SchedulerRegistry#addTimeout` yöntemi kullanılarak **dinamik** olarak oluşturun, aşağıdaki gibi:

```typescript
addTimeout(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Zaman Aşımı ${name}, (${milliseconds})'dan sonra çalışıyor!`);
  };

  const timeout = setTimeout(callback, milliseconds);
  this.schedulerRegistry.addTimeout(name, timeout);
}
```

Bu kodda, bir standart JavaScript zaman aşımı oluşturuyoruz, ardından `SchedulerRegistry#addTimeout` yöntemine geçiriyoruz.
Bu yöntem iki argüman alır: bir zaman aşımı için bir ad ve zaman aşımı kendisi.

Adı verilen bir zaman aşımını `SchedulerRegistry#deleteTimeout` yöntemi kullanarak **silin**, aşağıdaki gibi:

```typescript
deleteTimeout(name: string) {
  this.schedulerRegistry.deleteTimeout(name);
  this.logger.warn(`Zaman Aşımı ${name} silindi!`);
}
```

Tüm zaman aşımını `SchedulerRegistry#getTimeouts` yöntemi kullanılarak **listeleyin**:

```typescript
getTimeouts() {
  const timeouts = this.schedulerRegistry.getTimeouts();
  timeouts.forEach(key => this.logger.log(`Zaman Aşımı: ${key}`));
}
```

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/27-scheduling) bulunmaktadır.