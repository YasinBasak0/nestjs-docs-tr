---
sidebar_position: 2
---

### Sağlayıcılar (Providers)

Sağlayıcılar, Nest'in temel kavramlarından biridir. Temel Nest sınıflarının birçoğu sağlayıcı olarak ele alınabilir - servisler, depolar, fabrikalar, yardımcılar ve benzerleri. Bir sağlayıcının temel fikri, onun bir bağımlılık olarak **enjekte edilebilmesidir**; bu, nesnelerin birbirleriyle çeşitli ilişkiler kurabilmesi ve bu nesnelerin "bağlanması" işlevinin büyük ölçüde Nest çalışma zamanına devredilebilmesi anlamına gelir.

<figure><img src="/assets/Components_1.png" /></figure>

Önceki bölümde basit bir `CatsController` oluşturduk. Denetleyiciler, HTTP isteklerini ele almalı ve daha karmaşık görevleri **sağlayıcılara** (providers) **devretmelidir**. Sağlayıcılar, bir [modülde](/docs/modules) `providers` olarak tanımlanan basit JavaScript sınıflarıdır.

> info **İpucu** Nest, bağımlılıkları daha nesne yönelimli bir şekilde tasarlamak ve düzenlemek için olanak sağladığı için [SOLID](https://en.wikipedia.org/wiki/SOLID) prensiplerini takip etmenizi şiddetle önerir.

#### Servisler

Önce basit bir `CatsService` oluşturalım. Bu servis, veri depolama ve alımından sorumlu olacak ve `CatsController` tarafından kullanılması için tasarlandığından, bir sağlayıcı olarak tanımlanması için iyi bir adaydır.

```typescript
@@filename(cats.service)
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor() {
    this.cats = [];
  }

  create(cat) {
    this.cats.push(cat);
  }

  findAll() {
    return this.cats;
  }
}
```

> info **İpucu** CLI kullanarak bir servis oluşturmak için, basitçe `$ nest g service cats` komutunu çalıştırın.

`CatsService` sınıfımız, bir özellik ve iki yöntem içeren basit bir sınıftır. Tek yeni özellik, `@Injectable()` dekoratörünü kullanmasıdır. `@Injectable()` dekoratörü, `CatsService` sınıfının Nest [IoC](https://en.wikipedia.org/wiki/Inversion_of_control) konteynırı tarafından yönetilebilen bir sınıf olduğunu belirten bir meta veri ekler. Bu örnekte ayrıca muhtemelen şöyle görünen bir `Cat` arabirimini kullanıyoruz:

```typescript
@@filename(interfaces/cat.interface)
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

Artık kedileri almak için bir servis sınıfımız olduğuna göre, onu `CatsController` içinde kullanalım:

```typescript
@@filename(cats.controller)
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
@@switch
import { Controller, Get, Post, Body, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Post()
  @Bind(Body())
  async create(createCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

`CatsService`, sınıf kurucusu aracılığıyla **enjekte edilir**. `private` sözdizimini dikkate alın. Bu kısayol, `catsService` üyesini aynı konumda hem bildirmemize hem de başlatmamıza izin verir.

#### Bağımlılık Enjeksiyonu

Nest, güçlü ve yaygın olarak bilinen **Bağımlılık Enjeksiyonu** tasarım modeli etrafında inşa edilmiştir. Bu kavram hakkında harika bir makaleyi resmi [Angular](https://angular.dev/guide/di) belgelerinde okumanızı öneririz.

Nest sayesinde, TypeScript yetenekleri sayesinde bağımlılıkları yönetmek son derece kolaydır, çünkü bunlar sadece türle çözülür. Aşağıdaki örnekte, Nest, `CatsController` sınıfının bağımlılıklarını çözmek için `CatsService` sınıfının bir örneğini yaratır ve döndürür (veya normalde bir tekil durumda, daha önce başka bir yerde talep edilmişse mevcut örneği döndürür). Bu bağımlılık, belirttiğiniz kurucu yöntemine (veya belirtilen özelliğe) çözülür ve iletilir:

```typescript
constructor(private catsService: CatsService) {}
```

#### Kapsamlar

Sağlayıcılar genellikle uygulama yaşam döngüsü ("kapsam") ile senkronize bir şekilde çalışır. Uygulama başlatıldığında, her bağımlılık çözülmelidir ve bu nedenle her sağlayıcı örneği oluşturulmalıdır. Benzer şekilde, uygul

ama kapanırken, her sağlayıcı yok edilecektir. Ancak, sağlayıcınızın ömrünü **istek kapsamlı** olarak da yapabilirsiniz. Bu teknikler hakkında daha fazla bilgi için [buraya](/docs/fundamentals/injection-scopes) bakabilirsiniz.

<app-banner-courses></app-banner-courses>

#### Özel Sağlayıcılar

Nest, ilişkileri çözen dahili bir ters kontrol ("IoC") konteynerine sahiptir. Bu özellik, yukarıda açıklanan bağımlılık enjeksiyon özelliğinin temelidir, ancak aslında şimdiye kadar açıkladığımızdan çok daha güçlüdür. Bir sağlayıcıyı tanımlamanın birkaç yolu vardır: düz değerleri, sınıfları ve hem asenkron hem de senkron fabrikaları kullanabilirsiniz. Daha fazla örnek için [buraya](/docs/fundamentals/custom-providers) bakabilirsiniz.

#### İsteğe Bağlı Sağlayıcılar

Bazen çözülmesi gerekli olmayan bağımlılıklarınız olabilir. Örneğin, sınıfınızın bir **yapılandırma nesnesine** bağımlı olabilir, ancak geçilmemişse varsayılan değerler kullanılmalıdır. Bu durumda, yapılandırma sağlayıcısının olmaması bir hataya neden olmaz, bu nedenle bağımlılık isteğe bağlı hale gelir.

Bir sağlayıcının isteğe bağlı olduğunu belirtmek için, kurucu imzasında `@Optional()` dekoratörünü kullanın.

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

Yukarıdaki örnekte, özel bir sağlayıcı kullanıyoruz, bu nedenle `HTTP_OPTIONS` özel **token**'ını içeriyoruz. Önceki örnekler, bir bağımlılığı sınıf içinde bir sınıf ile kurucu aracılığıyla gösteren kurucu tabanlı enjeksiyonu göstermişti. Özel sağlayıcılar ve bunlarla ilişkilendirilmiş tokenlar hakkında daha fazla bilgi için [buraya](/docs/fundamentals/custom-providers) bakın.

#### Özellik Tabanlı Enjeksiyon

Bugüne kadar kullandığımız teknik, kurucu tabanlı enjeksiyon olarak adlandırılır, çünkü sağlayıcılar kurucu yöntemi aracılığıyla enjekte edilir. Bazı çok özel durumlarda, **özellik tabanlı enjeksiyon** yararlı olabilir. Örneğin, üst düzey bir sınıfınızın bir veya birden çok sağlayıcıya bağımlı olup olmadığına bağlı olarak, alt sınıflardan kurucu aracılığıyla `super()` çağırarak onları tüm yol boyunca geçirmek çok zahmetli olabilir. Bu durumdan kaçınmak için özelliğin seviyesinde `@Inject()` dekoratörünü kullanabilirsiniz.

```typescript
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

> warning **Uyarı** Sınıfınız başka bir sınıfı genişletmiyorsa, her zaman **kurucu tabanlı** enjeksiyonu kullanmayı tercih etmelisiniz.

#### Sağlayıcı Kaydı

Artık bir sağlayıcı (`CatsService`) tanımladık ve bu servisin bir tüketeni (`CatsController`) var, bu nedenle hizmetin enjeksiyonunu gerçekleştirebilsin diye hizmeti Nest'e kaydetmemiz gerekiyor. Bunu, modül dosyanızı (`app.module.ts`) düzenleyerek ve `@Module()` dekoratörünün `providers` dizisine hizmeti ekleyerek yaparız.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

Nest artık `CatsController` sınıfının bağımlılıklarını çözebilecektir.

Bu, şu anda dizin yapımızın nasıl görünmesi gerektiği:

<div class="file-tree">
<div class="item">src</div>
<div class="children">
<div class="item">cats</div>
<div class="children">
<div class="item">dto</div>
<div class="children">
<div class="item">create-cat.dto.ts</div>
</div>
<div class="item">interfaces</div>
<div class="children">
<div class="item">cat.interface.ts</div>
</div>
<div class="item">cats.controller.ts</div>
<div class="item">cats.service.ts</div>
</div>
<div class="item">app.module.ts</div>
<div class="item">main.ts</div>
</div>
</div>

### Manuel Örnek

Bugüne kadar, Nest'in bağımlılıkları çözme ayrıntılarının çoğunu nasıl otomatik olarak ele aldığını tartıştık. Belirli durumlarda, yerleşik Bağımlılık Enjeksiyon sistemi dışına çıkmanız ve sağlayıcıları manuel olarak almanız veya örneklemeniz gerekebilir. İki böyle konuyu kısaca aşağıda tartışıyoruz.

Mevcut örnekleri almak veya sağlayıcıları dinamik olarak örneklemek için [Module reference](https://docs.nestjs.com/fundamentals/module-ref) kullanabilirsiniz.

`bootstrap()` fonksiyonu içinde sağlayıcıları almak için (örneğin, denetleyici olmayan bağımsız uygulamalar için veya başlatma sırasında bir yapılandırma servisini kullanmak için) [Bağımsız Uygulamalar](https://docs.nestjs.com/standalone-applications) bölümüne bakın.