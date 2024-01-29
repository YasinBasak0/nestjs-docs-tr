### Sağlık Kontrolleri (Terminus)

**Terminus** entegrasyonu, size **kullanılabilirlik/canlılık** sağlık kontrolleri sunar. Karmaşık backend kurulumları söz konusu olduğunda, sağlık kontrolleri çok önemlidir. Web geliştirme alanında bir sağlık kontrolü genellikle özel bir adresi içerir, örneğin, `https://my-website.com/health/readiness`. Servisiniz veya altyapınızın bir bileşeni (örneğin, Kubernetes), bu adresi sürekli olarak kontrol eder. Bu adrese yapılan bir `GET` isteğinin döndürdüğü HTTP durum koduna bağlı olarak servis, "sağlıklı" olmayan bir yanıt aldığında bir işlem yapacaktır. "Sağlıklı" veya "sağlıksız"ın tanımı sunduğunuz hizmet türüne bağlı olduğu için, **Terminus** entegrasyonu size bir dizi **sağlık göstergesi** ile destek sağlar.

Örneğin, web sunucunuzun verilerini depolamak için MongoDB kullanıyorsa, MongoDB'nin hala çalışıp çalışmadığı hayati bir bilgi olacaktır. Bu durumda, `MongooseHealthIndicator`'ü kullanabilirsiniz. Doğru bir şekilde yapılandırıldığında - daha sonra daha fazla bilgi - sağlık kontrol adresiniz, MongoDB'nin çalışıp çalışmadığına bağlı olarak sağlıklı veya sağlıksız bir HTTP durum kodu döndürecektir.

#### Başlangıç

`@nestjs/terminus` ile başlamak için gereken bağımlılığı kurmamız gerekiyor.

```bash
$ npm install --save @nestjs/terminus
```

#### Sağlık Kontrolü Kurulumu

Bir sağlık kontrolü, **sağlık göstergelerinin** bir özeti olarak görülür. Bir sağlık göstergesi, bir servisin sağlıklı veya sağlıksız durumda olup olmadığını kontrol eder. Tüm atanan sağlık göstergeleri çalışıyorsa, bir sağlık kontrolü pozitiftir. Birçok uygulamanın benzer sağlık göstergelerine ihtiyaç duyacağından, [`@nestjs/terminus`](https://github.com/nestjs/terminus), şu gibi önceden tanımlanmış göstergeler içeren bir dizi gösterge sağlar:

- `HttpHealthIndicator`
- `TypeOrmHealthIndicator`
- `MongooseHealthIndicator`
- `SequelizeHealthIndicator`
- `MikroOrmHealthIndicator`
- `PrismaHealthIndicator`
- `MicroserviceHealthIndicator`
- `GRPCHealthIndicator`
- `MemoryHealthIndicator`
- `DiskHealthIndicator`

İlk sağlık kontrolümüzle başlamak için, `HealthModule`'u oluşturalım ve içine `TerminusModule`'u imports dizisine dahil edelim.

> info **İpucu** [Nest CLI](cli/overview) kullanarak modülü oluşturmak için basitçe `$ nest g module health` komutunu çalıştırın.

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';

@Module({
  imports: [TerminusModule]
})
export class HealthModule {}
```

Sağlık kontrol(leri)miz, [kontrolcü](/docs/controllers) kullanılarak yürütülebilir, ki bu da [Nest CLI](cli/overview) kullanılarak kolayca kurulabilir.

```bash
$ nest g controller health
```

> info **Bilgi** Uygulamanızda kapatma kancalarını etkinleştirmeniz şiddetle önerilir. Terminus entegrasyonu, bu yaşam döngüsü olayını etkinleştirilmişse kullanır. Kapatma kancaları hakkında daha fazla bilgiyi [buradan](fundamentals/lifecycle-events#application-shutdown) okuyabilirsiniz.

#### HTTP Sağlık Kontrolü

`@nestjs/terminus`'u kurduktan, `TerminusModule`'umuzu içe aktardıktan ve yeni bir kontrolcü oluşturduktan sonra sağlık kontrolü oluşturmaya hazırız.

`HTTPHealthIndicator`, `@nestjs/axios` paketini gerektirir, bu yüzden onu yüklü olduğundan emin olun:

```bash
$ npm i --save @nestjs/axios axios
```

Şimdi `HealthController`'ımızı kurabiliriz:

```typescript
@@filename(health.controller)
import { Controller, Get } from '@nestjs/common';
import { HealthCheckService, HttpHealthIndicator, HealthCheck } from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'),
    ]);
  }
}
@@switch
import { Controller, Dependencies, Get } from '@nestjs/common';
import { HealthCheckService, HttpHealthIndicator, HealthCheck } from '@nestjs/terminus';

@Controller('health')
@Dependencies(HealthCheckService, HttpHealthIndicator)
export class HealthController {
  constructor(
    private health,
    private http,
  ) { }

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'),
    ])
  }
}
```

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}
@@switch
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}
```

Sağlık kontrolümüz şimdi `https://docs.nestjs.com` adresine bir _GET_-istek gönderecek. Eğer bu adresten sağlıklı bir yanıt alırsak, rotamız olan `http://localhost:3000/health`, aşağıdaki nesneyi 200 durum koduyla döndürecektir.

```json
{
  "status": "ok",
  "info": {
    "nestjs-docs": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "nestjs-docs": {
      "status": "up"
    }
  }
}
```

Bu yanıt nesnesinin arayüzü, `@nestjs/terminus` paketinden `HealthCheckResult` arayüzü ile erişilebilir.

|           |                                                                                                                                                                                             |                                      |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| `status`  | Herhangi bir sağlık göstergesi başarısız olursa durum `'error'` olacaktır. Eğer NestJS uygulaması kapanıyorsa ancak hala HTTP istekleri alınıyorsa, sağlık kontrolü `'shutting_down'` durumunu alacaktır. | `'error' \| 'ok' \| 'shutting_down'` |
| `info`    | Sağlıklı olan yani durumu `'up'` olan her sağlık göstergesinin bilgisini içeren nesne.                                                                              | `object`                             |
| `error`   | Sağlıksız yani durumu `'down'` olan her sağlık göstergesinin bilgisini içeren nesne.                                                                          | `object`                             |
| `details` | Her sağlık göstergesinin tüm bilgilerini içeren nesne                                                                                                                                  | `object`                             |

##### Belirli HTTP yanıt kodlarını kontrol etme

Belirli durumları kontrol etmek ve yanıtı doğrulamak isteyebileceğiniz durumlar olabilir. Örneğin, `https://my-external-service.com` adresinin `204` yanıt kodu döndürdüğünü varsayalım. `HttpHealthIndicator.responseCheck` kullanarak belirli bir yanıt kodu için kontrol edebilir ve tüm diğer kodları sağlıklı olarak kabul edebilirsiniz.

`204` dışındaki herhangi bir başka yanıt kodu dönerse, aşağıdaki örnek sağlıklı olmaz. Üçüncü parametre, size yanıtın sağlıklı (`true`) veya sağlıksız (`false`) olarak kabul edilip edilmediğini belirleyen bir işlev (eşzamanlı veya asenkron) sağlamanızı gerektirir.

```typescript
@@filename(health.controller)
// `HealthController` sınıfı içinde

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () =>
      this.http.responseCheck(
        'my-external-service',
        'https://my-external-service.com',
        (res) => res.status === 204,
      ),
  ]);
}
```

#### TypeOrm sağlık göstergesi

Terminus, sağlık kontrolünüze veritabanı kontrolleri eklemenize olanak tanır. Bu sağlık göstergesiyle başlamak için, [Veritabanı bölümünü](/docs/techniques/sql) kontrol etmeli ve uygulamanız içindeki veritabanı bağlantınızın kurulduğundan emin olmalısınız.

> info **Hint** Arka planda `TypeOrmHealthIndicator`, genellikle veritabanının hala çalışıp çalışmadığını doğrulamak için kullanılan `SELECT 1` SQL komutunu basitçe yürütür. Oracle veritabanı kullanıyorsanız `SELECT 1 FROM DUAL` kullanır.

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, TypeOrmHealthIndicator)
export class HealthController {
  constructor(
    private health,
    private db,
  ) { }

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ])
  }
}
```

Eğer veritabanınıza erişilebiliyorsa, artık `http://localhost:3000` adresine bir `GET` isteği yaparak aşağıdaki JSON sonucunu görmelisiniz:

```json
{
  "status": "ok",
  "info": {
    "database": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "database": {
      "status": "up"
    }
  }
}
```

Eğer uygulamanız [birden fazla veritabanı kullanıyorsa](techniques/database#multiple-databases), her bağlantıyı `HealthController`'ınıza enjekte etmeniz gerekir. Sonra bağlantı referansını sadece `TypeOrmHealthIndicator`'a iletebilirsiniz.

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    @InjectConnection('albumsConnection')
    private albumsConnection: Connection,
    @InjectConnection()
    private defaultConnection: Connection,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('albums-database', { connection: this.albumsConnection }),
      () => this.db.pingCheck('database', { connection: this.defaultConnection }),
    ]);
  }
}
```

#### Disk sağlık göstergesi

`DiskHealthIndicator` ile kullanılan depolama alanının ne kadar kullanıldığını kontrol edebiliriz. Başlamak için `DiskHealthIndicator`'ü `HealthController`'ınıza enjekte ettiğinizden emin olun. Aşağıdaki örnek, `/` (veya Windows'ta `C:\\`) yolundaki depolama kullanımını kontrol eder. Eğer bu, toplam depolama alanının yüzde 50'sinden fazla olursa, sağlıksız bir Sağlık Kontrolü ile yanıt verecektir.

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly disk: DiskHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.disk.checkStorage('storage', { path: '/', thresholdPercent: 0.5 }),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, DiskHealthIndicator)
export class HealthController {
  constructor(health, disk) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.disk.checkStorage('storage', { path: '/', thresholdPercent: 0.5 }),
    ])
  }
}
```

`DiskHealthIndicator.checkStorage` fonksiyonu ile ayrıca belirli bir alan miktarını kontrol etme olasılığınız vardır. Aşağıdaki örnek, `/my-app/` yolunun 250GB'dan fazla olması durumunda sağlıksız olacaktır.

```typescript
@@filename(health.controller)
// `HealthController` sınıfı içinde

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.disk.checkStorage('storage', {  path: '/', threshold: 250 * 1024 * 1024 * 1024, })
  ]);
}
```

#### Bellek sağlık göstergesi

İşlemizin belirli bir bellek sınırını aşmadığından emin olmak için `MemoryHealthIndicator` kullanılabilir.
Aşağıdaki örnek, işleminizin heap'ini kontrol etmek için kullanılabilir.

> info **Hint** Heap, dinamik olarak ayrılan belleğin (yani malloc aracılığıyla ayrılan bellek) bulunduğu belleğin bir bölümüdür. Heap'ten ayrılan bellek, aşağıdakilerden biri gerçekleşene kadar ayrılmış kalacaktır:
> - Bellek `free` edildiğinde
> - Program sonlandığında

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private memory: MemoryHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.memory.checkHeap('memory_heap', 150 * 1024 * 1024),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, MemoryHealthIndicator)
export class HealthController {
  constructor(health, memory) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.memory.checkHeap('memory_heap', 150 * 1024 * 1024),
    ])
  }
}
```

Ayrıca `MemoryHealthIndicator.checkRSS` ile işleminizin bellek RSS'ini de doğrulamak mümkündür. Bu örnek,
işleminizin 150MB'dan fazla ayrılmış belleğe sahip olduğu durumda sağlıksız bir yanıt kodu döndürecektir.

> info **Hint** RSS, Kalıcı Set Boyutu'nu temsil eder ve bu işleme ayrılan belleği ve RAM'de bulunan belleği göstermek için kullanılır.
> Bu, bellek swap edilen belleği içermez. Paylaşılan kütüphanelerden gelen sayfaların bellekte olup olmamasına bağlı olarak
> paylaşılan kütüphane belleğini içerir. Tüm yığın ve heap belleği de içerir.

```typescript
@@filename(health.controller)
// `HealthController` sınıfı içinde

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.memory.checkRSS('memory_rss', 150 * 1024 * 1024),
  ]);
}
```


#### Özel sağlık göstergesi

Bazı durumlarda, `@nestjs/terminus` tarafından sağlanan önceden tanımlanmış sağlık göstergeleri tüm sağlık kontrolü gereksinimlerinizi kapsamıyorsa, ihtiyaçlarınıza göre özel bir sağlık göstergesi kurabilirsiniz.

Özel göstergeyi temsil edecek bir servis oluşturarak başlayalım. Bir göstergenin nasıl yapılandırıldığını temel bir anlayış kazanmak için örnek bir `DogHealthIndicator` oluşturacağız. Bu servisin durumu, her `Dog` nesnesinin tipinin `'goodboy'` olması durumunda `'up'` olmalıdır. Bu koşul sağlanmazsa bir hata fırlatmalıdır.

```typescript
@@filename(dog.health)
import { Injectable } from '@nestjs/common';
import { HealthIndicator, HealthIndicatorResult, HealthCheckError } from '@nestjs/terminus';

export interface Dog {
  name: string;
  type: string;
}

@Injectable()
export class DogHealthIndicator extends HealthIndicator {
  private dogs: Dog[] = [
    { name: 'Fido', type: 'goodboy' },
    { name: 'Rex, type: 'badboy' },
  ];

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    const badboys = this.dogs.filter(dog => dog.type === 'badboy');
    const isHealthy = badboys.length === 0;
    const result = this.getStatus(key, isHealthy, { badboys: badboys.length });

    if (isHealthy) {
      return result;
    }
    throw new HealthCheckError('Dogcheck failed', result);
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { HealthCheckError } from '@nestjs/terminus';

@Injectable()
export class DogHealthIndicator extends HealthIndicator {
  dogs = [
    { name: 'Fido', type: 'goodboy' },
    { name: 'Rex, type: 'badboy' },
  ];

  async isHealthy(key) {
    const badboys = this.dogs.filter(dog => dog.type === 'badboy');
    const isHealthy = badboys.length === 0;
    const result = this.getStatus(key, isHealthy, { badboys: badboys.length });

    if (isHealthy) {
      return result;
    }
    throw new HealthCheckError('Dogcheck failed', result);
  }
}
```

Bir sonraki yapmamız gereken şey, sağlık göstergesini bir sağlayıcı olarak kaydetmektir.

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { DogHealthIndicator } from './dog.health';

@Module({
  controllers: [HealthController],
  imports: [TerminusModule],
  providers: [DogHealthIndicator]
})
export class HealthModule { }
```

> info **Hint** Gerçek bir uygulamada, `DogHealthIndicator`'u, örneğin `DogModule` olarak adlandırabileceğiniz ayrı bir modülde sağlanmalıdır ve ardından `HealthModule` tarafından içe aktarılmalıdır.

Şimdi, artık sağlık kontrolü endpoint'ine eklenen sağlık göstergesini gereken son adıma geçiyoruz. Bunun için `HealthController`'ımıza geri dönüp onu `check` fonksiyonumuza ekleyelim.

```typescript
@@filename(health.controller)
import { HealthCheckService, HealthCheck } from '@nestjs/terminus';
import { Injectable, Dependencies, Get } from '@nestjs/common';
import { DogHealthIndicator } from './dog.health';

@Injectable()
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private dogHealthIndicator: DogHealthIndicator
  ) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.dogHealthIndicator.isHealthy

('dog'),
    ])
  }
}
@@switch
import { HealthCheckService, HealthCheck } from '@nestjs/terminus';
import { Injectable, Get } from '@nestjs/common';
import { DogHealthIndicator } from './dog.health';

@Injectable()
@Dependencies(HealthCheckService, DogHealthIndicator)
export class HealthController {
  constructor(
    private health,
    private dogHealthIndicator
  ) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.dogHealthIndicator.isHealthy('dog'),
    ])
  }
}
```

#### Günlükleme

Terminus yalnızca hata mesajlarını kaydeder, örneğin bir Sağlık Kontrolü başarısız olduğunda. `TerminusModule.forRoot()` yöntemi ile hataların nasıl kaydedileceği üzerinde daha fazla kontrol sağlayabilir,
ayrıca günlüğü tamamen kendiniz kontrol altına alabilirsiniz.

Bu bölümde, özel bir günlükçü olan `TerminusLogger`'ı nasıl oluşturacağınızı size anlatacağız. Bu günlükçü yerleşik günlükçüyü genişletir.
Bu nedenle, hangi bölümü üzerine yazmak istediğinizi seçebilirsiniz.

> info **Bilgi** NestJS'te özel günlükçüler hakkında daha fazla bilgi almak istiyorsanız, [buradan daha fazlasını okuyun](/docs/techniques/logger#injecting-a-custom-logger).

```typescript
@@filename(terminus-logger.service)
import { Injectable, Scope, ConsoleLogger } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class TerminusLogger extends ConsoleLogger {
  error(message: any, stack?: string, context?: string): void;
  error(message: any, ...optionalParams: any[]): void;
  error(
    message: unknown,
    stack?: unknown,
    context?: unknown,
    ...rest: unknown[]
  ): void {
    // Burada hata mesajlarının nasıl kaydedileceğini üzerine yazın
  }
}
```

Özel günlükçünüzü oluşturduktan sonra, yapmanız gereken tek şey, onu `TerminusModule.forRoot()` içine geçirmektir.

```typescript
@@filename(health.module)
@Module({
imports: [
  TerminusModule.forRoot({
    logger: TerminusLogger,
  }),
],
})
export class HealthModule {}
```

Terminus'tan gelen tüm günlük mesajlarını, hata mesajları dahil olmak üzere tamamen bastırmak için, Terminus'ı şu şekilde yapılandırın.

```typescript
@@filename(health.module)
@Module({
imports: [
  TerminusModule.forRoot({
    logger: false,
  }),
],
})
export class HealthModule {}
```



Terminus, Sağlık Kontrolü hatalarının günlüklerinizde nasıl görüntüleneceğini yapılandırmanıza olanak tanır.

| Hata Günlük Stili          | Açıklama                                                                                                                        | Örnek                                                               |
|:------------------|:--------------------------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------------------|
| `json`  (varsayılan) | Bir hatada sağlık kontrolü sonucunun özetini JSON nesnesi olarak yazdırır                                                     | <figure><img src="/assets/Terminus_Error_Log_Json.png" /></figure>   |
| `pretty`          | Bir hatada sağlık kontrolü sonucunun, biçimlendirilmiş kutular içinde ve başarılı/yanlış sonuçları vurgulayan şekilde özetini yazdırır | <figure><img src="/assets/Terminus_Error_Log_Pretty.png" /></figure> |

Günlük stili değiştirmek için `errorLogStyle` yapılandırma seçeneğini aşağıdaki örnek gibi kullanabilirsiniz.

```typescript
@@filename(health.module)
@Module({
  imports: [
    TerminusModule.forRoot({
      errorLogStyle: 'pretty',
    }),
  ]
})
export class HealthModule {}
```

#### Zarif kapanış zaman aşımı

Uygulamanızın kapatma işlemini ertelemesi gerekiyorsa, Terminus bunu sizin için yönetebilir.
Bu ayarı kullanmak, özellikle Kubernetes gibi bir orkestratörle çalışırken faydalı olabilir.
Hazır olma kontrol aralığından biraz daha uzun bir gecikme belirleyerek, konteynerleri kapatırken sıfır kesintiye ulaşabilirsiniz.

```typescript
@@filename(health.module)
@Module({
  imports: [
    TerminusModule.forRoot({
      gracefulShutdownTimeoutMs: 1000,
    }),
  ]
})
export class HealthModule {}
```

#### Daha fazla örnek

Daha fazla çalışan örnek [burada](https://github.com/nestjs/terminus/tree/master/sample) bulunmaktadır.
