---
sidebar_position: 13
---

### Modüller

Bir modül, `@Module()` dekoratörü ile işaretlenmiş bir sınıftır. `@Module()` dekoratörü, uygulama yapısını düzenlemek için **Nest** tarafından kullanılan meta bilgileri sağlar.

<figure>
  <img src="/assets/Modules_1.png" alt="" />
</figure>

Her uygulamanın en az bir modülü vardır, yani bir **kök modül**. Kök modül, Nest'in modül ve sağlayıcı ilişkilerini ve bağımlılıklarını çözmek için kullandığı **uygulama grafiği**'ni oluşturmak için kullandığı başlangıç noktasıdır. Çok küçük uygulamalar teorik olarak sadece kök modülü içerebilir, ancak bu tipik bir durum değildir. Modüllerin bileşenlerinizi düzenlemenin etkili bir yolu olarak **güçlü bir şekilde** önerildiğini vurgulamak istiyoruz. Bu nedenle, çoğu uygulama için sonuçta oluşan mimari, her biri bir dizi yakından ilgili **yetenekleri** kapsayan birden çok modülü kullanacaktır.

`@Module()` dekoratörü, modülü tanımlayan özelliklere sahip bir nesne alır:

|               |                                                                                                                                                                                                          |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `providers`   | Nest enjektörü tarafından örneklenen ve en azından bu modül üzerinde paylaşılabilecek olan sağlayıcılar                                                                                          |
| `controllers` | bu modülde tanımlanmış ve örneklenmesi gereken denetleyicilerin kümesi                                                                                                                              |
| `imports`     | bu modülde gerekli olan sağlayıcıları ihraç eden içe aktarılan modüllerin listesi                                                                                                                 |
| `exports`     | bu modül tarafından sağlanan ve bu modülü içe aktaran diğer modüllerde kullanılabilir olması gereken `providers`'ın alt kümesi. Hem sağlayıcı kendisi hem de yalnızca token'ı (`provide` değeri) kullanabilirsiniz |

Modül, sağlayıcıları varsayılan olarak **kapsar**. Bu, mevcut modülün doğrudan bir parçası veya içe aktarılan modüllerden ihraç edilmeyen sağlayıcıları enjekte etmenin imkansız olduğu anlamına gelir. Bu nedenle, bir modülden ihraç edilen sağlayıcılara bir modülün genel arabirimi veya API'si olarak veya API olarak bakabilirsiniz.

#### Özellik modülleri

`CatsController` ve `CatsService` aynı uygulama alanına aittir. Yakından ilişkili olduklarından, bunları bir özellik modülüne taşımak mantıklıdır. Bir özellik modülü, sadece belirli bir özellikle ilgili kodu düzenler, kodu düzenli tutar ve açık sınırlar kurar. Bu, karmaşıklığı yönetmemize ve [SOLID](https://en.wikipedia.org/wiki/SOLID) prensipleriyle geliştirmemize yardımcı olur, özellikle uygulamanın boyutu ve/veya takım büyüdükçe.

Bunu göstermek için, `CatsModule`'u oluşturacağız.

```typescript
@@filename(cats/cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

> info **İpucu** CLI kullanarak modül oluşturmak için sadece `$ nest g module cats` komutunu çalıştırın.

Yukarıda, `CatsModule`'u `cats.module.ts` dosyasında tanımladık ve bu modülle ilgili her şeyi `cats` dizinine taşıdık. Yapmamız gereken son şey, bu modülü kök modüle ( `app.module.ts` dosyasında tanımlanan `AppModule` ) içe aktarmaktır.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

İşte şu anda dizin yapımız nasıl gör

ünüyor:

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
      <div class="item">cats.module.ts</div>
      <div class="item">cats.service.ts</div>
    </div>
    <div class="item">app.module.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

#### Paylaşılan modüller

Nest'te modüller varsayılan olarak **tekil** olarak kabul edilir ve bu nedenle aynı sağlayıcının aynı örneğini birden çok modül arasında kolayca paylaşabilirsiniz.

<figure><img src="/assets/Shared_Module_1.png" /></figure>

Her modül otomatik olarak bir **paylaşılan modül**tür. Bir kere oluşturulduktan sonra başka herhangi bir modül tarafından tekrar kullanılabilir. `CatsService` sağlayıcısının bir örneğini birkaç başka modül arasında paylaşmak istediğimizi hayal edelim. Bunu yapabilmek için, önce sağlayıcıyı modülün `exports` dizisine ekleyerek `CatsService` sağlayıcısını **ihraç etmemiz** gerekir, aşağıda gösterildiği gibi:

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

Şimdi `CatsModule`'u içe aktaran herhangi bir modül, `CatsService`'e erişebilir ve bunu içe aktaran diğer tüm modüllerle aynı örneği paylaşacaktır.

<app-banner-devtools></app-banner-devtools>

#### Modül yeniden ihraç etme

Yukarıda görüldüğü gibi, modüller içindeki sağlayıcıları ihraç edebilir. Ayrıca, içe aktardıkları modülleri **ihraç edebilirler**. Aşağıdaki örnekte, `CommonModule` hem `CoreModule` içine **ithal edilir** hem de **ihraç edilir**, bu da onu bu modülü içe aktaran diğer modüller için kullanılabilir hale getirir.

```typescript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

#### Bağımlılık enjeksiyonu

Bir modül sınıfı sağlayıcıları **enjekte edebilir** (örneğin, yapılandırma amaçları için):

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
@@switch
import { Module, Dependencies } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
@Dependencies(CatsService)
export class CatsModule {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

Ancak, modül sınıfları kendileri sağlayıcı olarak **enjekte edilemez** çünkü [döngüsel bağımlılık](/docs/fundamentals/circular-dependency) sorunlarına yol açabilir.

#### Global modüller

Eğer her yerde aynı seti modülü içe aktarmak zorunda kalıyorsanız, bu sıkıcı olabilir. Farklı olarak, [Angular](https://angular.dev) `providers` global kapsamda kaydedilir. Bir kez tanımlandıktan sonra her yerde kullanılabilirler. Ancak, Nest sağlayıcıları modül kapsamı içinde kapsar. Bir modülün sağlayıcılarını, kapsayıcı modülü önce içe aktarmadan başka bir yerde kullanamazsınız.

Her yerde kullanılabilir olması gereken sağlayıcı kümesini (örneğin, yardımcılar, veritabanı bağlantıları vb.) sağlamak istediğinizde, modülü `@Global()` dekoratörü ile global hale getirebilirsiniz.

```typescript
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

`@Global()` dekoratörü modülü global kapsamlı hale getirir. Global modüller genellikle yalnızca kök veya çekirdek modül tarafından **yalnızca bir kez** kaydedilmelidir. Yukarıdaki örnekte, `CatsService` sağlayıcısı her yerde bulunabilir olacak ve servisi enjekte etmek isteyen modüller, `CatsModule`'ü içe aktarmalarına gerek kalmayacak.

> info **İpucu** Her şeyi global yapmak iyi bir tasarım kararı değildir. Global modüller, gereken işlemlerdeki gereksiz tekrarı azaltmak için mevcuttur. `imports` dizisi genellikle modülün API'sini tüketiciye kullanılabilir yapmanın tercih edilen yoludur.

#### Dinamik modüller

Nest modül sistemi, **dinamik modüller** olarak adlandırılan güçlü bir özelliği içerir. Bu özellik, dinamik olarak sağlayıcıları kaydedip yapılandırabilen özelleştirilebilir modüller oluşturmanıza olanak tanır. Dinamik modüller, [burada](/docs/fundamentals/dynamic-modules) detaylı bir şekilde ele alınmıştır. Bu bölümde, modüllere girişi tamamlamak için kısaca bir inceleme yapacağız.

Aşağıda, bir `DatabaseModule` için dinamik bir modül tanımının bir örneği bulunmaktadır:

```typescript
@@filename()
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
@@switch
import { Module } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options) {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

> info **İpucu** `forRoot()` yöntemi, dinamik bir modülü senkron veya asenkron olarak (örneğin, bir `Promise` aracılığıyla) döndürebilir.

Bu modül, varsayılan olarak `Connection` sağlayıcısını tanımlar ( `@Module()` dekoratör metadata'sında), ancak ayrıca `forRoot()` yöntemine iletilen `entities` ve `options` nesnelerine bağlı olarak örneğin depolar gibi bir sağlayıcı koleksiyonunu ortaya çıkarır. Dinamik modül tarafından döndürülen özellikler, `@Module()` dekoratöründe tanımlanan temel modül metadata'sini **genişletir** (geçersiz kılmaz). İşte hem statik olarak tanımlanmış `Connection` sağlayıcısı **hem de** dinamik olarak oluşturulan depo sağlayıcıları modülden ihraç edildiği şekilde.

Eğer bir dinamik modülü global kapsamda kaydetmek istiyorsanız, `global` özelliğini `true` olarak ayarlayın.

```typescript
{
  global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

> warning **Uyarı** Yukarıda belirtildiği gibi, her şeyi global yapmak **iyi bir tasarım kararı değildir**.

`DatabaseModule` şu şekilde içe aktarılıp yapılandırılabilir:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

Eğer sırasıyla bir dinamik modülü tekrar ihraç etmek istiyorsanız, `forRoot()` yöntemi çağrısını ihraç dizisinde ihmal edebilirsiniz:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```

[Dinamik modüller](/docs/fundamentals/dynamic-modules) bölümü bu konuyu daha ayrıntılı bir şekilde ele almaktadır ve bir [çalışan örnek](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules) içermektedir.

> info **İpucu** `ConfigurableModuleBuilder`'ı kullanarak son derece özelleştirilebilir dinamik modüller nasıl oluşturacağınızı [bu bölümde](/docs/fundamentals/dynamic-modules#configurable-module-builder) öğrenin.