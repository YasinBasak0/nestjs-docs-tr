### Kuyruklar

Kuyruklar, yaygın uygulama ölçeklendirme ve performans zorluklarıyla başa çıkmanıza yardımcı olan güçlü bir tasarım desenidir. Kuyrukların size yardımcı olabileceği bazı örnek sorunlar şunlardır:

- İşleme zirvelerini düzeltme. Örneğin, kullanıcılar herhangi bir zamanda kaynak yoğun görevleri başlatabiliyorsa, bu görevleri senkron bir şekilde gerçekleştirmek yerine kuyruğa ekleyebilirsiniz. Ardından işlemci süreçlerinin görevleri kontrol edilmiş bir şekilde kuyruktan çekmelerini sağlayabilirsiniz. Uygulama büyüdükçe arka uç görev işleme ölçeklendiğinde yeni Kuyruk tüketicilerini kolayca ekleyebilirsiniz.
- Node.js olay döngüsünü bloke edebilecek monolitik görevleri bölebilme. Örneğin, bir kullanıcı isteği, ses transcoding gibi CPU yoğun işi gerektiriyorsa, bu görevi başka süreçlere devredebilir ve kullanıcıya yönelik işlemlerin duyarlı kalmasını sağlayabilirsiniz.
- Çeşitli hizmetler arasında güvenilir bir iletişim kanalı sağlama. Örneğin, bir süreçte veya hizmette görevleri (işleri) kuyruğa ekleyebilir ve bunları başka bir süreçte veya hizmette tüketebilirsiniz. Görev yaşam döngüsündeki tamamlama, hata veya diğer durum değişiklikleri için herhangi bir süreçten veya hizmetten durum olaylarını dinleyerek (onları dinleyerek) bilgilendirilebilirsiniz. Kuyruk üreticileri veya tüketicileri başarısız olduğunda, durumları korunur ve düğmeler yeniden başlatıldığında görev işleme otomatik olarak yeniden başlar.

Nest, popüler, iyi desteklenen, yüksek performanslı Node.js tabanlı bir Kuyruk sistemi uygulama olan [Bull](https://github.com/OptimalBits/bull) üzerine bir soyutlama/sarjör olarak `@nestjs/bull` paketini sağlar. Bu paket, Bull Kuyruklarını uygulamanıza Nest dostu bir şekilde entegre etmeyi kolaylaştırır.

Bull, iş verilerini saklamak için [Redis](https://redis.io/) kullanır, bu nedenle sisteminizde Redis'in yüklü olması gerekecektir. Redis destekli olduğu için Kuyruk mimariniz tamamen dağıtılmış ve platform bağımsız olabilir. Örneğin, bir (veya birkaç) düğümde Nest'te çalışan bazı Kuyruk [üreticileri](/docs/techniques/queues#producers), [tüketicileri](/docs/techniques/queues#consumers) ve [dinleyicileri](/docs/techniques/queues#event-listeners) olabilir ve diğer üreticiler, tüketiciler ve dinleyiciler başka bir Node.js platformunda başka bir ağ düğümünde çalışabilir.

Bu bölümde, `@nestjs/bull` paketi ele alınmaktadır. Ayrıca, daha fazla arka plan ve özel uygulama ayrıntıları için [Bull belgelerini](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md) okumanızı öneririz.

#### Kurulum

Kullanmaya başlamak için önce gerekli bağımlılıkları yükleriz.

```bash
$ npm install --save @nestjs/bull bull
```

Kurulum işlemi tamamlandığında, `BullModule`'u kök `AppModule` içine alabiliriz.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';

@Module({
  imports: [
    BullModule.forRoot({
      redis

: {
        host: 'localhost',
        port: 6379,
      },
    }),
  ],
})
export class AppModule {}
```

`forRoot()` yöntemi, uygulamada kayıtlı olan tüm kuyruklar tarafından kullanılacak olan bir `bull` paketi yapılandırma nesnesini kaydetmek için kullanılır (aksi belirtilmediği sürece). Bir yapılandırma nesnesi, aşağıdaki özellikleri içeren bir nesne oluşur:

- `limiter: RateLimiter` - Kuyruktaki işlerin işlenme hızını kontrol etmek için seçenekler. Daha fazla bilgi için [RateLimiter](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue) sayfasına bakın. İsteğe bağlıdır.
- `redis: RedisOpts` - Redis bağlantısını yapılandırmak için seçenekler. Daha fazla bilgi için [RedisOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue) sayfasına bakın. İsteğe bağlıdır.
- `prefix: string` - Tüm kuyruk anahtarları için önek. İsteğe bağlıdır.
- `defaultJobOptions: JobOpts` - Yeni işler için varsayılan ayarları kontrol etmek için seçenekler. Daha fazla bilgi için [JobOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd) sayfasına bakın. İsteğe bağlıdır.
- `settings: AdvancedSettings` - Gelişmiş Kuyruk yapılandırma ayarları. Genellikle bunların değiştirilmemesi gerekir. Daha fazla bilgi için [AdvancedSettings](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue) sayfasına bakın. İsteğe bağlıdır.

Tüm seçenekler isteğe bağlıdır ve kuyruk davranışını ayrıntılı bir kontrol sağlar. Bu seçenekler, doğrudan Bull `Queue` kurucusuna iletilir. Bu seçenekler hakkında daha fazla bilgiyi [buradan](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue) okuyabilirsiniz.

Bir kuyruk kaydetmek için `BullModule.registerQueue()` dinamik modülünü içe aktarabiliriz:

```typescript
BullModule.registerQueue({
  name: 'audio',
});
```

> info **İpucu** Birden fazla kuyruk oluşturmak için `registerQueue()` yöntemine birden çok virgülle ayrılmış yapılandırma nesnesi geçirerek bunu gerçekleştirebilirsiniz.

`registerQueue()` yöntemi, kuyrukların örneklenmesi ve/veya kaydedilmesi için kullanılır. Kuyruklar, aynı kimlik bilgileriyle aynı temel Redis veritabanına bağlanan modüller ve süreçler arasında paylaşılır. Her kuyruk, adı özelliğiyle benzersizdir. Bir kuyruk adı, bir enjeksiyon belirteci olarak (kuyruğu denetleyicilere/enjektörlere enjekte etmek için) ve bir dekoratöre bağlı tüketici sınıflarını ve dinleyicileri kuyruklarla ilişkilendirmek için kullanılır.

Ayrıca, belirli bir kuyruk için önceden yapılandırılmış seçenekleri geçersiz kılabilirsiniz:

```typescript
BullModule.registerQueue({
  name: 'audio',
  redis: {
    port: 6380,
  },
});
```

İşler Redis'te saklandığından, belirli bir adlandırılmış kuyruk her örneklendiğinde (örneğin, bir uygulama başlatıldığında/yeniden başlatıldığında), önceki bitmemiş bir oturumdan kalma eski işleri işlemeye çalışır.

Her kuyruk, birçok üretici, tüketici ve dinleyiciye sahip olabilir. Tüketiciler işleri kuyruktan belirli bir sırayla alır: FIFO (varsayılan), LIFO veya önceliklere göre. Kuyruk işleme sırasını kontrol etme hakkında bilgi için [buraya](/docs/techniques/queues#consumers) bakın.

<app-banner-enterprise></app-banner-enterprise>

#### Adlandırılmış Konfigürasyonlar

Eğer kuyruklarınız farklı Redis örneklerine bağlanıyorsa, **adlandırılmış konfigürasyonlar** olarak adlandırılan bir teknik kullanabilirsiniz. Bu özellik, belirli anahtarlar altında birkaç konfigürasyonu kaydetmenize olanak tanır, ardından bu konfigürasyonlara kuyruk seçeneklerinde başvurabilirsiniz.

Örneğin, uygulamanızda kayıtlı birkaç kuyruk tarafından kullanılan varsayılan dışında başka bir Redis örneğiniz olduğunu varsayalım, bu konfigürasyonu aşağıdaki gibi kaydedebilirsiniz:

```typescript
BullModule.forRoot('alternative-config', {
  redis: {
    port: 6381,
  },
});
```

Yukarıdaki örnekte, `'alternative-config'` sadece bir konfigürasyon anahtarıdır (herhangi bir rastgele dize olabilir).

Bu işlemi gerçekleştirdikten sonra, `registerQueue()` seçenekleri nesnesinde bu konfigürasyona işaret edebilirsiniz:

```typescript
BullModule.registerQueue({
  configKey: 'alternative-config',
  name: 'video'
});
```

#### Üreticiler (Producers)

İş üreticileri, kuyruklara iş ekler. Üreticiler genellikle uygulama hizmetleri olup (Nest [sağlayıcıları](/docs/providers)), işleri bir kuyruğa eklemek için önce servise kuyruğu enjekte ederler. Bunu şu şekilde yapabilirsiniz:

```typescript
import { Injectable } from '@nestjs/common';
import { Queue } from 'bull';
import { InjectQueue } from '@nestjs/bull';

@Injectable()
export class AudioService {
  constructor(@InjectQueue('audio') private audioQueue: Queue) {}
}
```

> info **İpucu** `@InjectQueue()` dekoratörü, kuyruğu `registerQueue()` yöntemi çağrısında sağlanan adıyla tanır (örneğin, `'audio'`).

Şimdi, kuyruğun `add()` yöntemini çağırarak kullanıcı tanımlı bir iş nesnesini geçirerek bir iş ekleyin. İşler, genellikle Redis veritabanında nasıl depolandıklarıdır, bu nedenle geçtiğiniz iş nesnesi, semantiklerinizi temsil etmek için kullanılır.

```typescript
const job = await this.audioQueue.add({
  foo: 'bar',
});
```

#### Adlandırılmış İşler

İşlere benzersiz isimler verilebilir. Bu, yalnızca belirli bir isme sahip işleri işleyecek özel <a href="techniques/queues#consumers">tüketiciler</a> oluşturmanıza olanak tanır.

```typescript
const job = await this.audioQueue.add('transcode', {
  foo: 'bar',
});
```

> Uyarı **Uyarı** Adlandırılmış işleri kullanırken, her bir kuyruğa eklenen her benzersiz isim için bir işlemci oluşturmalısınız, aksi takdirde kuyruk, verilen iş için bir işlemci eksik olduğunu bildirecektir. Adlandırılmış işleri tüketme hakkında daha fazla bilgi için [buraya](/docs/techniques/queues#consumers) bakın.

#### İş Seçenekleri

İşlerle ilişkilendirilebilecek ek seçeneklere sahip olabilirsiniz. `Queue.add()` yöntemindeki `job` argümanından sonra bir seçenekler nesnesi geçirin. İş seçenekleri özellikleri şunlardır:

- `priority`: `number` - İsteğe bağlı öncelik değeri. 1 (en yüksek öncelik) ile MAX_INT (en düşük öncelik) arasında değişir. Öncelik kullanmanın performans üzerinde hafif bir etkisi olduğuna dikkat edin, bu nedenle dikkatli kullanın.
- `delay`: `number` - Bu işin işlenmesi için beklenmesi gereken süre (milisaniye cinsinden). Doğru gecikmeler için sunucu ve istemcilerin saatlerinin senkronize olması gerekir.
- `attempts`: `number` - İşin tamamlanana kadar denenecek toplam sayısı.
- `repeat`: `RepeatOpts` - İşi cron belirtilimine göre tekrarla. Daha fazla bilgi için [RepeatOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd) sayfasına bakın.
- `backoff`: `number | BackoffOpts` - İş başarısız olursa otomatik yeniden deneme için geri çekilme ayarı. Daha fazla bilgi için [BackoffOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd) sayfasına bakın.
- `lifo`: `boolean` - true ise işi kuyruğun sağ ucuna (sol yerine) ekler (varsayılan false).
- `timeout`: `number` - İşin bir zaman aşımı hatası ile başarısız olması gereken milisaniye cinsinden süre.
- `jobId`: `number` | `string` - İş ID'sini geçersiz kılmak için - varsayılan olarak, iş ID'si benzersiz bir tamsayıdır, ancak bunu geçersiz kılmak için bu ayarı kullanabilirsiniz. Bu seçeneği kullanıyorsanız, işlem garantisi sizin tarafınızdan sağlanmalıdır. Var olan bir kimliğe sahip bir iş eklemeye çalışırsanız, eklenmez.
- `removeOnComplete`: `boolean | number` - true ise iş başarıyla tamamlandığında işi kuyruktan kaldırır. Bir sayı, tutulacak iş miktarını belirtir. Varsayılan davranış, işi tamamlanan kümede tutmaktır.
- `removeOnFail`: `boolean | number` - true ise iş tüm denemelerden sonra başarısız olursa işi kuyruktan kaldırır. Bir sayı, tutulacak iş miktarını belirtir. Varsayılan davranış, işi başarısız kümede tutmaktır.
- `stackTraceLimit`: `number` - Stack trace'de kaydedilecek satır sayısını sınırlar.

İşleri iş seçenekleriyle özelleştirmenin birkaç örneği aşağıda verilmiştir.

Bir işin başlamasını geciktirmek için `delay` yapılandırma özelliğini kullanın.

```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { delay: 3000 }, // 3 saniye gecikmeli
);
```

Bir işi kuyruğun sağ ucuna eklemek için (işlemi **LIFO** (Son Giren İlk Çıkar) olarak işlemek için) yapılandırma nesnesinin `lifo` özelliğini `true` olarak ayarlayın.

```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { lifo: true },
);
```

Bir işi önceliklendirmek için `priority` özelliğini kullanın.

```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { priority: 2 },
);
```

#### Tüketiciler (Consumers)

Bir tüketici, kuyruğa eklenen işleri işleyen veya kuyruktaki olayları dinleyen veya her ikisini birden tanımlayan bir **sınıf**tır. Bir tüketici sınıfını şu şekilde `@Processor()` dekoratörü kullanarak bildirin:

```typescript
import { Processor } from '@nestjs/bull';

@Processor('audio')
export class AudioConsumer {}
```

> info **İpucu** Tüketicilerin `providers` olarak kaydedilmesi gerekir, böylece `@nestjs/bull` paketi onları alabilir.

Dekoratörün dize argümanı (örneğin, `'audio'`) sınıf yöntemleriyle ilişkilendirilecek kuyruğun adıdır.

Bir tüketici sınıfı içinde, iş yönlendiricilerini, dekoratörle işleme yöntemlerini kullanarak bildirin.

```typescript
import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';

@Processor('audio')
export class AudioConsumer {
  @Process()
  async transcode(job: Job<unknown>) {
    let progress = 0;
    for (let i = 0; i < 100; i++) {
      await doSomething(job.data);


      progress += 1;
      await job.progress(progress);
    }
    return {};
  }
}
```

Dekore edilmiş yöntem (örneğin, `transcode()`) işçi boşta olduğunda ve kuyruktaki işleri işlemek için bir iş varsa çağrılır. Bu yönlendirici yöntemi, tek argüman olarak `job` nesnesini alır. Yönlendirici yöntemi tarafından döndürülen değer, daha sonra tamamlanan olayı dinleyici içinde erişilebilir ve kullanılabilir.

`Job` nesneleri, durumlarıyla etkileşimde bulunmanıza izin veren çeşitli yöntemlere sahiptir. Örneğin, yukarıdaki kod, işin ilerlemesini güncellemek için `progress()` yöntemini kullanır. Tüm `Job` nesnesi API referansı için [buraya](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#job) bakın.

Bir iş yönlendirici yönteminin **yalnızca** belirli bir türdeki işleri işleyeceğini (belirli bir `name` ile işleri) belirtmek için bu `name`'yi `@Process()` dekoratörüne şu şekilde geçirerek gösterilebilir. Bir tüketici sınıfında her bir iş türü (`name`) için birçok `@Process()` yönlendiricisi olabilir. Adlandırılmış işleri kullanıyorsanız, her bir isme karşılık gelen bir işleyici olduğundan emin olun.

```typescript
@Process('transcode')
async transcode(job: Job<unknown>) { ... }
```

> uyarı **Uyarı** Aynı kuyruk için birden çok tüketici tanımlarsanız, `@Process({{ '{' }} concurrency: 1 {{ '}' }})` içindeki `concurrency` seçeneği etkili olmayacaktır. Minimum `concurrency`, tanımlanan tüketici sayısıyla eşleşecektir. Bu, `@Process()` yönlendiricileri adlandırılmış işleri işlemek için farklı bir `name` kullanıyorsa da geçerlidir.

#### İstek Kapsamlı Tüketiciler

Bir tüketici, istek kapsamlı olarak işaretlendiğinde (enjeksiyon kapsamları hakkında daha fazla bilgi için [buraya](/docs/fundamentals/injection-scopes#provider-scope) bakın), sınıfın her bir iş için özel olarak oluşturulacak bir örneği olacaktır. Örnek, iş tamamlandıktan sonra bellekten temizlenecektir.

```typescript
@Processor({
  name: 'audio',
  scope: Scope.REQUEST,
})
```

Çünkü istek kapsamlı tüketici sınıfları dinamik olarak oluşturulur ve yalnızca bir iş için kapsanır, bir `JOB_REF`'yi standart bir yaklaşımı kullanarak yapıcı aracılığıyla enjekte edebilirsiniz.

```typescript
constructor(@Inject(JOB_REF) jobRef: Job) {
  console.log(jobRef);
}
```

> info **İpucu** `JOB_REF` simgesi `@nestjs/bull` paketinden içe aktarılır.

#### Olay Dinleyicileri

Bull, kuyruk ve/veya iş durumu değişiklikleri meydana geldiğinde kullanışlı olaylar kümesi oluşturur. Nest, bu olaylara abone olmak için bir dizi standart olaya izin veren dekoratör seti sağlar. Bu dekoratörler `@nestjs/bull` paketinden dışa aktarılır.

Olay dinleyicileri, bir [tüketici](/docs/techniques/queues#consumers) sınıfı içinde (yani, `@Processor()` dekoratörü ile süslenmiş bir sınıf içinde) bildirilmelidir. Bir olayı dinlemek için aşağıdaki tabloda bir dekoratör kullanarak olay için bir işleyici bildirin. Örneğin, `audio` kuyruğunda bir iş aktif duruma girdiğinde yayımlanan olayı dinlemek için aşağıdaki yapıyı kullanın:

```typescript
import { Processor, Process, OnQueueActive } from '@nestjs/bull';
import { Job } from 'bull';

@Processor('audio')
export class AudioConsumer {

  @OnQueueActive()
  onActive(job: Job) {
    console.log(
      `Processing job ${job.id} of type ${job.name} with data ${job.data}...`,
    );
  }
  ...
```

Bull, dağıtık (çoklu düğme) bir ortamda çalıştığından, olay yerelliği kavramını tanımlar. Bu kavram, olayların tamamen bir işlem içinde veya farklı işlemlerden paylaşılan kuyruklarda tetiklenebileceğini tanır. **Yerel** bir olay, bir eylemin veya durum değişikliğinin yerel işlemdeki bir kuyruktan tetiklendiğinde oluşan bir olaydır. Başka bir deyişle, olay üreticileriniz ve tüketicileriniz tek bir işlem için yerel olduğunda, kuyruklarda meydana gelen tüm olaylar yereldir.

Bir kuyruk birden fazla işlem arasında paylaşıldığında, **global** olaylar olasılığıyla karşılaşırız. Başka bir işlem tarafından tetiklenen bir olay bildirimini almak için bir dinleyicinin global bir olaya kaydolması gerekir.

Olay işleyicileri, karşılık gelen olay yayımlandığında her zaman çağrılır. İşleyici, aşağıdaki tabloda gösterilen imza ile çağrılır ve olayla ilgili bilgilere erişim sağlar. Aşağıda yerel ve global olay işleyicisi imzaları arasındaki önemli bir farkı tartışıyoruz.

<table>
  <tr>
    <th>Yerel olay dinleyicileri</th>
    <th>Global olay dinleyicileri</th>
    <th>İşleyici yöntem imzası / Tetiklendiğinde</th>
  </tr>
  <tr>
    <td><code>@OnQueueError()</code></td><td><code>@OnGlobalQueueError()</code></td><td><code>handler(error: Error)</code> - Bir hata meydana geldi. <code>error</code>, tetikleyen hatayı içerir.</td>
  </tr>
  <tr>
    <td><code>@OnQueueWaiting()</code></td><td><code>@OnGlobalQueueWaiting()</code></td><td><code>handler(jobId: number | string)</code> - Bir iş, bir işçi boşta olduğunda hemen işlenmeye hazır durumda bekliyor. <code>jobId</code>, bu duruma giren işin kimliğini içerir.</td>
  </tr>
  <tr>
    <td><code>@OnQueueActive()</code></td><td><code>@OnGlobalQueueActive()</code></td><td><code>handler(job: Job)</code> - İş <code>job</code> başladı.</td>
  </tr>
  <tr>
    <td><code>@OnQueueStalled()</code></td><td><code>@OnGlobalQueueStalled()</code></td><td><code>handler(job: Job)</code> - İş <code>job</code>, durmuş olarak işaretlendi. Bu, çöken veya olay döngüsünü durduran iş işçilerini hata ayıklamak için kullanışlıdır.</td>
  </tr>
  <tr>
    <td><code>@OnQueueProgress()</code></td><td><code>@OnGlobalQueueProgress()</code></td><td><code>handler(job: Job, progress: number)</code> - İş <code>job</code>'nin ilerlemesi, değer <code>progress</code> olarak güncellendi.</td>
  </tr>
  <tr>
    <td><code>@OnQueueCompleted()</code></td><td><code>@OnGlobalQueueCompleted()</code></td><td><code>handler(job: Job, result: any)</code> İş <code>job</code>, bir sonuç <code>result</code> ile başarıyla tamamlandı.</td>
  </tr>
  <tr>
    <td><code>@OnQueueFailed()</code></td><td><code>@OnGlobalQueueFailed()</code></td><td><code>handler(job: Job, err: Error)</code> İş <code>job</code>, bir hata nedeniyle başarısız oldu <code>err</code>.</td>
  </tr>
  <tr>
    <td><code>@OnQueuePaused()</code></td><td><code>@OnGlobalQueuePaused()</code></td><td><code>handler()</code> Kuyruk duraklatıldı.</td>
  </tr>
  <tr>
    <td><code>@OnQueueResumed()</code></td><td><code>@OnGlobalQueueResumed()</code></td><td><code>handler(job: Job)</code> Kuyruk devam etti.</td>
  </tr>
  <tr>
    <td><code>@OnQueueCleaned()</code></td><td><code>@OnGlobalQueueCleaned()</code></td><td><code>handler(jobs: Job[], type: string)</code> Eski işler kuyruktan temizlendi. <code>jobs</code>, temizlenen işlerin bir dizisidir ve <code>type</code>, temizlenen iş türüdür.</td>
  </tr>
  <tr>
    <td><code>@OnQueueDrained()</code></td><td><code>@OnGlobalQueueDrained()</code></td><td><code>handler()</code> Kuyruk, tüm bekleyen işleri işlediğinde (henüz işlenmemiş bazı gecikmiş işler olabilir) tetiklenir.</td>
  </tr>
  <tr>
    <td><code>@OnQueueRemoved()</code></td><td><code>@OnGlobalQueueRemoved()</code></td><td><code>handler(job: Job)</code> İş <code>job</code> başarıyla kaldırıldı.</td>
  </tr>
</table>

Global olayları dinlerken, yöntem imzaları yerel karşılıklarından biraz farklı olabilir. Özellikle, yerel sürümde `job` nesnelerini alan herhangi bir yöntem imzası, global sürümde `jobId` (`number`) alır. Bu durumda gerçek `job` nesnesine başvurmak için `Queue#getJob` yöntemini kullanın. Bu çağrı beklenmelidir ve bu nedenle işleyici `async` olarak bildirilmelidir. Örneğin:

```typescript
@OnGlobalQueueCompleted()
async onGlobalCompleted(jobId: number, result: any) {
  const job = await this.immediateQueue.getJob(jobId);
  console.log('(Global) on completed: job ', job.id, ' -> result: ', result);
}
```

> info **İpucu** `Queue` nesnesine erişmek için (bir `getJob()` çağrısı yapmak için) bunu enjekte etmelisiniz. Ayrıca, Queue'yu enjekte ettiğiniz modülde kaydedilmiş olmalıdır.

Belirli olay dinleyici dekoratörlerinin yanı sıra, `@OnQueueEvent()` dekoratörünü `BullQueueEvents` veya `BullQueueGlobalEvents` enumları ile birlikte kullanabilirsiniz. Olaylar hakkında daha fazla bilgi için [buraya](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#events) bakın.

#### Kuyruk Yönetimi

Kuyrukların, duraklatma, devam ettirme, çeşitli durumlardaki iş sayısını almak gibi yönetim işlemlerini gerçekleştirmenizi sağlayan bir API'si vardır. Tam kuyruk API'sini [burada](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue) bulabilirsiniz. Bu yöntemleri, aşağıda duraklatma/devam etme örnekleri ile gösterildiği gibi doğrudan `Queue` nesnesi üzerinde çağırın.

`pause()` methodu çağrısı ile bir kuyruğu duraklatın. Duraklatılan bir kuyruk, devam eden işleri tamamlanana kadar yeni işleri işlemeyecektir.

```typescript
await audioQueue.pause();
```

Duraklatılmış bir kuyruğu devam ettirmek için şu şekilde `resume()` methodunu kullanın:

```typescript
await audioQueue.resume();
```

#### Ayrı İşlemler

İş işleyicileri ayrı bir (çatallanmış) işlemde çalıştırılabilir ([kaynak](https://github.com/OptimalBits/bull#separate-processes)). Bu, birkaç avantaja sahiptir:

- İşlem çökse bile işçiyi etkilemez çünkü işlem izole edilmiştir.
- Kuyruğu etkilemeden engelleyici kod çalıştırabilirsiniz (işler durmaz).
- Çok çekirdekli CPU'ların çok daha iyi kullanımı.
- Redis'e daha az bağlantı.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { join } from 'path';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'audio',
      processors: [join(__dirname, 'processor.js')],
    }),
  ],
})
export class AppModule {}
```

Lütfen işlevinizin çatallanmış bir işlemde yürütüldüğü göz önüne alındığında Bağımlılık Enjeksiyonu (ve IoC konteyneri) kullanılamayacağını unutun. Bu, işlemci işlevinizin ihtiyaç duyduğu tüm harici bağımlılıkların örneklerini içermesi (veya oluşturması) gerektiği anlamına gelir.

```typescript
@@filename(processor)
import { Job, DoneCallback } from 'bull';

export default function (job: Job, cb: DoneCallback) {
  console.log(`[${process.pid}] ${JSON.stringify(job.data)}`);
  cb(null, 'It works');
}
```

#### Asenkron Yapılandırma

Belirli `bull` seçeneklerini statik olarak değil de asenkron olarak iletmek isteyebilirsiniz. Bu durumda, asenkron yapılandırma ile başa çıkmak için `forRootAsync()` yöntemini kullanın. Benzer şekilde, kuyruk seçeneklerini asenkron olarak iletmek istiyorsanız, `registerQueueAsync()` yöntemini kullanın.

Bu konuda kullanılabilecek bir yöntem, bir fabrika işlevi kullanmaktır:

```typescript
BullModule.forRootAsync({
  useFactory: () => ({
    redis: {
      host: 'localhost',
      port: 6379,
    },
  }),
});
```

Fabrikamız, diğer [asenkron sağlayıcılar](https://docs.nestjs.com/fundamentals/async-providers) gibi davranır (örneğin, `async` olabilir ve `inject` aracılığıyla bağımlılıkları enjekte edebilir).

```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    redis: {
      host: configService.get('QUEUE_HOST'),
      port: configService.get('QUEUE_PORT'),
    },
  }),
  inject: [ConfigService],
});
```

Alternatif olarak, `useClass` sözdizimini kullanabilirsiniz:

```typescript
BullModule.forRootAsync({
  useClass: BullConfigService,
});
```

Yapılan bu inşa, `BullModule` içinde `BullConfigService`'yi örneklendirecek ve `createSharedConfiguration()`'ı çağırarak bir seçenek nesnesi sağlamak için kullanacaktır. Bu, `BullConfigService`'nin `SharedBullConfigurationFactory` arabirimini uygulaması gerektiği anlamına gelir, aşağıda gösterildiği gibi:

```typescript
@Injectable()
class BullConfigService implements SharedBullConfigurationFactory {
  createSharedConfiguration(): BullModuleOptions {
    return {
      redis: {
        host: 'localhost',
        port: 6379,
      },
    };
  }
}
```

`BullConfigService`'nin `BullModule` içinde oluşturulmasını önlemek ve başka bir modülden içe aktarılan bir sağlayıcıyı kullanmak için `useExisting` sözdizimini kullanabilirsiniz.

```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

Bu yapı, `useClass` ile aynı şekilde çalışır, ancak kritik bir farkla - `BullModule`, yeni bir tane oluşturmak yerine mevcut bir `ConfigService`'yi yeniden kullanmak için içe aktarılan modülleri kontrol eder. 

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/26-queues) bulunmaktadır.