### Dinamik Modüller

[Modüller bölümü](/docs/modules), Nest modüllerinin temellerini kapsar ve [dinamik modüller](https://docs.nestjs.com/modules#dynamic-modules) hakkında kısa bir giriş içerir. Bu bölüm, dinamik modüller konusunu daha ayrıntılı bir şekilde ele alır. Tamamlandığında, bunların ne oldukları ve nasıl ve ne zaman kullanılacakları konusunda iyi bir kavrayışa sahip olmanız gerekmektedir.

#### Giriş

Belgelemenin **Genel Bakış** bölümündeki çoğu uygulama kodu örneği, düzenli veya statik modüllerin kullanımını içerir. Modüller, bir uygulamanın genel bir parçası olarak bir araya gelen [sağlayıcılar](/docs/providers) ve [denetleyiciler](/docs/controllers) gibi bileşen gruplarını tanımlar. Bu, bu bileşenler için bir yürütme bağlamı veya kapsam sağlar. Örneğin, bir modülde tanımlanan sağlayıcılar, onları dışa aktarmadan modülün diğer üyeleri tarafından görülebilir. Bir sağlayıcının modül dışında görünmesi gerektiğinde, önce ana modülünden dışa aktarılır ve ardından tüketen modülü içine alır.

Tanıdık bir örnek üzerinden geçelim.

İlk olarak, `UsersService`'i sağlamak ve dışa aktarmak için bir `UsersModule` tanımlayacağız. `UsersModule`, `UsersService` için **ana** modüldür.

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

Ardından, `UsersModule`'i içe alarak `UsersModule`'in dışa aktarılan sağlayıcılarını `AuthModule` içinde kullanılabilir hale getiren bir `AuthModule` tanımlayacağız:

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

Bu yapılar, örneğin, `AuthModule` içinde barındırılan `AuthService` içinde `UsersService`'i enjekte etmemize izin verir:

```typescript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  /*
    this.usersService'i kullanarak uygulama
  */
}
```

Bunu **statik** modül bağlama olarak adlandıracağız. Modüllerin birbirine nasıl bağlanacağıyla ilgili tüm bilgiler zaten ana ve tüketen modüllerde beyan edilmiştir. Bu süreçte neler olduğunu açalım. Nest, `UsersService`'i `AuthModule` içinde kullanılabilir hale getirir:

1. `UsersModule`'i anında başlatarak, `UsersModule`'in kendisinin tükettiği diğer modülleri transitif olarak içe alır ve bağımlılıkları transitif olarak çözer (bkz. [Özel sağlayıcılar](https://docs.nestjs.com/fundamentals/custom-providers)).
2. `AuthModule`'i anında başlatarak, `UsersModule`'in dışa aktarılan sağlayıcılarını `AuthModule`'in bileşenlerine kullanılabilir hale getirir (tıpkı `AuthModule` içinde beyan edilmiş gibi).
3. `AuthService` içindeki `UsersService` örneğini enjekte eder.

#### Dinamik modül kullanımı

Statik modül bağlamasıyla, tüketen modülün ana modülden sağlayıcıların nasıl yapılandırıldığını **etkileme** fırsatı yoktur. Neden önemli? Bir modülün farklı kullanım durumlarında farklı şekillerde davranması gereken genel bir amaç modülünün olduğu durumu düşünün. Bu, birçok sistemde "eklenti" kavramına benzer, genel bir özellik bir tüketici tarafından kullanılmadan önce biraz yapılandırmaya ihtiyaç duyan duruma benzer.

Nest için iyi bir örnek, bir **yapılandırma modülü**'dür. Birçok uygulama, yapılandırma ayrıntılarını bir yapılandırma modülü kullanarak dışa aktarmayı faydalı bulur. Bu, uygulama ayarlarını farklı dağıtımlarda dinamik olarak değiştirmeyi kolaylaştırır: örneğin, geliştiriciler için bir geliştirme veritabanı, aşama/test ortamı için bir aşama veritabanı vb. Yapılandırma parametrelerinin yönetimini yapılandırma modülüne delega ederek, uygulama kaynak kodu yapılandırma parametrelerinden bağımsız kalır.

Zorluk, yapılandırma modülünün kendisinin genel (bir "eklenti" gibi) olduğu ve onu kullanan modül tarafından özelleştirilmesi gerektiğidir. İşte burada _dinamik modüller_ devreye girer. Dinamik modül özelliklerini kullanarak, yapılandırma modülümüzü tüketen modülün onu içe aktardığında nasıl özelleştirileceğini kontrol etmesine izin verecek şekilde **dinamik** yapabiliriz

.

Diğer bir deyişle, dinamik modüller, bir modülü bir başka modüle içe aktarmak ve bu modülün içe alındığında özelliklerini ve davranışını özelleştirmek için bir API sağlar, şimdiye kadar gördüğümüz statik bağlamaların aksine.

<app-banner-devtools></app-banner-devtools>

#### Config Modülü Örneği

Bu bölümde, [konfigürasyon bölümünden](https://docs.nestjs.com/techniques/configuration#service) alınan örnek kodun temel versiyonunu kullanacağız. Bu bölümün sonunda tamamlanan sürümü, çalışan bir [örnek olarak burada](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules) bulabilirsiniz.

İhtiyacımız, `ConfigModule`'un özelleştirmek için bir `options` nesnesini kabul etmesidir. Desteklemek istediğimiz özellik şu şekildedir: Temel örnek, `.env` dosyasının konumunu proje kök klasörüne sert bir şekilde kodlamıştır. Bu konumu özelleştirilebilir hale getirmek istiyoruz, böylece `.env` dosyalarını istediğiniz bir klasörde yönetebilirsiniz. Örneğin, çeşitli `.env` dosyalarını proje kök klasörü altında `config` adlı bir klasörde (yani, `src` klasörünün kardeşi) saklamak istiyorsunuz. `ConfigModule`'u farklı projelerde kullanırken farklı klasörleri seçebilmek istiyorsunuz.

Dinamik modüller, içine parametreler geçirerek içe aktarılan modülün davranışını değiştirme yeteneği sağlar. Bu nasıl çalıştığını görelim. Bu, tüketen modülün perspektifinden son hedefin nasıl görünebileceğinden başlamak ve ardından geriye doğru çalışmak açısından yararlı olacaktır. İlk olarak, _statik olarak_ `ConfigModule`'u içe aktaran örneğin (yani, içe aktarılan modülün davranışını etkileme yeteneğine sahip olmayan bir yaklaşım) `@Module()` dekoratöründeki `imports` dizisine dikkatlice bakın:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Şimdi, bir yapılandırma nesnesini içe aktardığımız _dinamik bir modül_ içe aktarımını düşünelim. Bu iki örnek arasındaki `imports` dizisindeki farka dikkat edin:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Yukarıdaki dinamik örnekte neler olduğuna bir bakalım. Hareket eden parçalar nelerdir?

1. `ConfigModule` normal bir sınıftır, bu nedenle onun `register()` adında bir **statik metoduna** sahip olmalıdır. Bu metodun statik olduğunu biliyoruz, çünkü onu sınıfın bir **örneği** üzerinde değil, sınıfın kendisi üzerinde çağırıyoruz. Not: Bu metodun adı, herhangi bir keyfi isme sahip olabilir, ancak geleneksel olarak `forRoot()` veya `register()` olarak adlandırmalıyız.
2. `register()` metodu tarafımızca tanımlanmıştır, bu nedenle istediğimiz giriş argümanlarını kabul edebiliriz. Bu durumda, tipik bir durum olan uygun özelliklere sahip basit bir `options` nesnesini kabul edeceğiz.
3. `register()` metodunun bir tür `module` benzeri bir şeyi döndürmesi gerektiğini çıkarabiliriz, çünkü döndürdüğü değer aşina olduğumuz `imports` listesinde görünüyor, ki bu listenin içinde modül listesini içerir.

Aslında `register()` metodumuzun döndüreceği şey bir `DynamicModule` olacaktır. Dinamik bir modül, tam olarak aynı özelliklere sahip statik bir modül gibi çalışan, aynı zamanda `module` adında ek bir özelliğe sahip bir nesnedir. Aşağıdaki örnekte, statik bir modül deklarasyonunun örnek bir tanıtımını hızlıca gözden geçirirken modül seçeneklerine dikkatlice bakın:

```typescript
@Module({
  imports: [DogsModule],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
```

Dinamik modüller, tam olarak aynı arayüze sahip bir nesneyi döndürmek zorundadır, artı adı `module` olan bir özellik ekstra olarak bulunmalıdır. `module` özelliği, modülün adı olarak işlev görür ve örneğin aşağıdaki örnekte gösterildiği gibi modülün sınıf adıyla aynı olmalıdır.

> info **İpucu** Dinamik bir modül için, modül seçenek nesnesinin tüm özelliklerinin opsiyonel olduğunu, **ancak** `module` haricinde, belirtmeye gerek olmadığını söyleyebiliriz.

Peki, statik `register()` metodu hakkında ne? Şimdi, işinin, `DynamicModule` arayüzüne sahip bir nesneyi döndürmek olduğunu görebiliyoruz. Onu çağırdığımızda, bir modülü `imports` listesine sağlıyoruz; aynı şekilde statik durumdaki gibi bir modül sınıf adını listelerken yapardık. Başka bir de

yişle, dinamik modül API'si sadece bir modül döndürür, ancak `@Module()` dekoratörü aracılığıyla metadata özelliklerini sabitlemek yerine, bunları programlı bir şekilde belirtiriz.

Bu resmi tamamlamaya yardımcı olmak için kapsamamız gereken hala birkaç ayrıntı var:

1. Şimdi, `@Module()` dekoratörünün `imports` özelliğinin sadece bir modül sınıf adını (örneğin `imports: [UsersModule]`) değil, aynı zamanda bir dinamik modülü **döndüren bir işlevi** (örneğin `imports: [ConfigModule.register(...)]`) de alabileceğini söyleyebiliriz.
2. Bir dinamik modül kendisi diğer modülleri içe aktarabilir. Bu örnekte yapmayacağız, ancak dinamik modül diğer modüllerden sağlayıcılarına bağlıysa, bunları opsiyonel `imports` özelliğini kullanarak içe aktarırsınız. Yine, bu, bir statik modül için metadata bildirimi yaparken `@Module()` dekoratörünü kullanma şeklinin tam olarak benzeridir.

Bu anlayışla donatılmış olarak, dinamik `ConfigModule` deklarasyonumuzun nasıl görünmesi gerektiğine bir göz atabiliriz. Şimdi bunu anlamaya çalışalım.

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(): DynamicModule {
    return {
      module: ConfigModule,
      providers: [ConfigService],
      exports: [ConfigService],
    };
  }
}
```

Parçaların nasıl bir araya geldiğini artık net bir şekilde görmelisiniz. `ConfigModule.register(...)`'ı çağırdığımızda, `@Module()` dekoratöründeki metadata olarak sağladığımız özelliklere esasen aynı olan bir `DynamicModule` nesnesi döndirir.

> info **İpucu** `DynamicModule`'ü `@nestjs/common` modülünden içe aktarın.

Ancak, dinamik modülümüz henüz çok ilginç değil, çünkü onu **nasıl yapılandıracağımızı** söylediğimiz gibi bir yetenek eklememişiz. Şimdi bunu ele alalım.

#### Modül Yaplandırması

`ConfigModule`'un davranışını özelleştirmek için açık bir çözüm, yukarıda tahmin ettiğimiz gibi, statik `register()` metoduna bir `options` nesnesi iletmektir. Tüketen modülün `imports` özelliğine bir kez daha bakalım:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Bu, `options` nesnesini dinamik modülümüze iletmeyi güzel bir şekilde halleder. Peki sonra bu `options` nesnesini `ConfigModule` içinde nasıl kullanabiliriz? Bir dakika boyunca bunu düşünelim. `ConfigModule`'umuz, temelde, enjekte edilebilir bir servis olan - `ConfigService` - sağlamak ve diğer sağlayıcılar tarafından kullanılmak üzere bir ana bilgisayardır. Aslında, `ConfigService`'in davranışını özelleştirmek için `options` nesnesini okuma ihtiyacında olan kendisidir. Şu an için bu `options`'ın nasıl geçirileceğini henüz belirlemediğimizden, sadece `options`'ı sabit bir şekilde kodlayacağız. Bu durumu biraz sonra düzelteceğiz.

```typescript
import { Injectable } from '@nestjs/common';
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const options = { folder: './config' };

    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

Şimdi `ConfigService`'imiz, `options`'ı belirttiğimiz klasördeki `.env` dosyasını bulmayı nasıl bileceğini biliyor.

Kalıcı görevimiz, `options` nesnesini `register()` adımından `ConfigService`'imize somehow enjekte etmektir. Ve tabii ki bunu yapmak için _bağımlılık enjeksiyonunu_ kullanacağız. Bu önemli bir nokta, bu nedenle anladığınızdan emin olun. `ConfigModule`'umuz, `ConfigService`'i sağlıyor. `ConfigService` ise sadece çalışma zamanında sağlanan `options` nesnesine bağlıdır. Bu nedenle, çalışma zamanında önce `options` nesnesini Nest IoC konteynerine bağlamamız, ardından Nest'in bunu `ConfigService`'e enjekte etmesi gerekecek. **Özel sağlayıcılar** bölümünden hatırlayın ki, sağlayıcılar [yalnızca servisler değil, herhangi bir değeri içerebilir](https://docs.nestjs.com/fundamentals/custom-providers#non-service-based-providers), bu nedenle basit bir `options` nesnesini işlemek için bağımlılık enjeksiyonunu kullanmak sorun değildir.

İlk olarak, `options` nesnesini IoC konteynerine bağlamayı ele alalım. Bunu statik `register()` metodumuzda yapıyoruz. Unutmayın ki dinamik olarak bir modül oluşturuyoruz ve bir modülün özelliklerinden biri, sağlayıcılarının listesidir. Bu nedenle yapmamız gereken şey, `options` nesnesini bir sağlayıcı olarak tanımlamaktır. Bu, `ConfigService`'e enjekte edilebilir hale getirecektir ve bunu bir sonraki adımda kullanacağız. Aşağıdaki kodda `providers` dizisine dikkat edin:

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(options: Record<string, any>): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}
```

Şimdi süreci tamamlayabiliriz, `'CONFIG_OPTIONS'` sağlayıcısını `ConfigService`'e enjekte ederek. Sağlayıcıyı tanımlarken sınıf belirteci kullanmadığımızda, [burada açıklandığı gibi](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens), `@Inject()` dekoratörünü kullanmamız gerektiğini hatırlayın.

```typescript
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { Injectable, Inject } from '@nestjs/common';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor(@Inject('CONFIG_OPTIONS') private options: Record<string, any>) {
    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

Son bir not: basitlik için yukarıda string tabanlı bir enjeksiyon belirteci (`'CONFIG_OPTIONS'`) kullandık, ancak en iyi uygulama, bu belirteci ayrı bir dosyada bir sabit (veya `Symbol`) olarak tanımlamaktır ve o dosyayı içe aktarmaktır. Örneğin:

```typescript
export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';
```

#### Örnek

Bu bölümdeki kodun tam bir örneğini [burada](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules) bulabilirsiniz.

#### Topluluk İlkeleri

Muhtemelen `@nestjs/` paketlerinin bazılarında `forRoot`, `register` ve `forFeature` gibi yöntemlerin kullanıldığını görmüş olabilir ve bu yöntemler arasındaki farkın ne olduğunu merak ediyor olabilirsiniz. Bunun için kesin bir kural yoktur, ancak `@nestjs/` paketleri genellikle şu kuralları takip etmeye çalışır:

- `register` kullanarak bir modül oluşturduğunuzda, belirli bir yapılandırma ile bir dinamik modülü yapılandırmayı ve yalnızca çağıran modül tarafından kullanılmasını beklersiniz. Örneğin, Nest'in `@nestjs/axios`'u ile: `HttpModule.register({{ '{' }} baseUrl: 'someUrl' {{ '}' }})`. Başka bir modülde `HttpModule.register({{ '{' }} baseUrl: 'somewhere else' {{ '}' }})` kullanırsanız, farklı bir yapılandırmaya sahip olacaktır. Bu, istediğiniz kadar modül için yapabilirsiniz.

- `forRoot` kullanarak bir modül oluşturduğunuzda, bir dinamik modülü bir kez yapılandırmayı ve bu yapılandırmayı birden çok yerde yeniden kullanmayı beklersiniz (bunun bilinçli olarak olduğu kadar, soyutlanmış bir şekilde olabilir). Bu nedenle bir `GraphQLModule.forRoot()`, bir `TypeOrmModule.forRoot()` vb. olacaktır.

- `forFeature` kullanarak bir modülün `forRoot` yapılandırmasını kullanmayı beklersiniz, ancak çağıran modülün ihtiyaçlarına göre belirli yapılandırmayı değiştirmeniz gerekebilir (örneğin, bu modülün hangi depoya erişim sağlaması gerektiği veya bir günlükçünün hangi bağlamı kullanması gerektiği gibi).

Bunların hepsi genellikle aynı anlama gelir ve genellikle `registerAsync`, `forRootAsync` ve `forFeatureAsync` gibi `async` karşılıkları da vardır, ancak bunlar yapılandırmayı Nest'in Bağımlılık Enjeksiyonunu kullanarak yapar.

#### Yapılandırılabilir Modül Oluşturucu

Yüksek derecede yapılandırılabilir, dinamik modülleri manuel olarak oluşturmak (`registerAsync`, `forRootAsync`, vb.) oldukça karmaşık bir iş olduğundan, özellikle yeni başlayanlar için, Nest, bu süreci kolaylaştıran ve bir modül "taslağı" oluşturmanıza sadece birkaç satır kod ile olanak tanıyan `ConfigurableModuleBuilder` sınıfını sunar.

Örneğin, yukarıda kullandığımız örneği (`ConfigModule`) alalım ve onu `ConfigurableModuleBuilder`'ı kullanacak şekilde dönüştürelim. Başlamadan önce, `ConfigModule`'umuzun hangi seçenekleri aldığını temsil eden bir arayüz oluşturduğumuzdan emin olalım.

```typescript
export interface ConfigModuleOptions {
  folder: string;
}
```

Bunu yerine getirdikten sonra, varolan `config.module.ts` dosyasının yanına yeni bir dosya oluşturun ve adını `config.module-definition.ts` olarak adlandırın. Bu dosyada `ConfigurableModuleBuilder`'ı kullanarak `ConfigModule` tanımını oluşturalım.

```typescript
@@filename(config.module-definition)
import { ConfigurableModuleBuilder } from '@nestjs/common';
import { ConfigModuleOptions } from './interfaces/config-module-options.interface';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
@@switch
import { ConfigurableModuleBuilder } from '@nestjs/common';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().build();
```

Şimdi `config.module.ts` dosyasını açalım ve uygulamasını otomatik olarak oluşturulan `ConfigurableModuleClass`'ı kullanacak şekilde değiştirelim:

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {}
```

`ConfigurableModuleClass`'ı genişletmek, `ConfigModule`'un yalnızca önceki özel uygulama ile sağladığı `register` yöntemini değil, aynı zamanda tüketenlerin bu modülü asenkron olarak yapılandırmasına izin veren `registerAsync` yöntemini de sağlamasına olanak tanır. Örneğin, asenkron fabrikalar sağlayarak bu modülü asenkron olarak yapılandırmak mümkündür:

```typescript
@Module({
  imports: [
    ConfigModule.register({ folder: './config' }),
    // veya alternatif olarak:
    // ConfigModule.registerAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...herhangi ek bağımlılıklar...]
    // }),
  ],
})
export class AppModule {}
```

Son olarak, `ConfigService` sınıfını, şimdiye kadar kullandığımız `'CONFIG_OPTIONS'` yerine oluşturulan modül seçenekleri sağlayıcısını enjekte edecek şekilde güncelleyelim.

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) { ... }
}
```

#### Özel Yöntem Anahtarı

`ConfigurableModuleClass`, varsayılan olarak `register` ve karşılık gelen `registerAsync` yöntemlerini sağlar. Farklı bir yöntem adı kullanmak için `ConfigurableModuleBuilder#setClassMethodName` yöntemini aşağıdaki gibi kullanın:

```typescript
@@filename(config.module-definition)
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setClassMethodName('forRoot').build();
@@switch
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().setClassMethodName('forRoot').build();
```

Bu yapı, `ConfigurableModuleBuilder`'a `forRoot` ve `forRootAsync`'ı açığa çıkarması için bir sınıf oluşturmasını söyleyecektir. Örnek:

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ folder: './config' }), // <-- "register" yerine "forRoot" kullanımına dikkat edin
    // veya alternatif olarak:
    // ConfigModule.forRootAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...herhangi ek bağımlılıklar...]
    // }),
  ],
})
export class AppModule {}
```

#### Özel Seçenek Fabrika Sınıfı

`registerAsync` yöntemi (veya `forRootAsync` veya başka bir ad, yapılandırmaya bağlı olarak) tüketenin modül yapılandırmasını çözen bir sağlayıcı tanımlamasını geçmesine olanak tanır, bu nedenle bir kütüphane tüketicisi potansiyel olarak bir konfigürasyon nesnesi oluşturmak için kullanılacak bir sınıf sağlayabilir.

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory,
    }),
  ],
})
export class AppModule {}
```

Bu sınıf, varsayılan olarak bir modül konfigürasyon nesnesini döndüren `create()` yöntemini sağlamalıdır. Ancak, kütüphaneniz farklı bir adlandırma kuralları takip ediyorsa, bu davranışı değiştirebilir ve `ConfigurableModuleBuilder`'a farklı bir yöntem beklemesini, örneğin `createConfigOptions`, `ConfigurableModuleBuilder#setFactoryMethodName` yöntemini kullanarak bildirebilirsiniz:

```typescript
@@filename(config.module-definition)
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setFactoryMethodName('createConfigOptions').build();
@@switch
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().setFactoryMethodName('createConfigOptions').build();
```

Artık, `ConfigModuleOptionsFactory` sınıfının `createConfigOptions` yöntemini (ve `create` yerine) açığa çıkarması gerekir:

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory, // <-- bu sınıf "createConfigOptions" yöntemini sağlamalıdır
    }),
  ],
})
export class AppModule {}
```

#### Ekstra Seçenekler

Modülünüzün davranışını belirleyen ekstra seçeneklere ihtiyaç duyduğu durumlar olabilir (bu tür bir seçeneğin güzel bir örneği, `isGlobal` bayrağıdır - veya sadece `global`) ve aynı zamanda bu seçeneklerin (`ConfigService` gibi) modül içinde kayıtlı olan hizmetlere/bilgi sağlayıcılara dahil edilmemesi gerekir. Modül içinde kayıtlı hizmetler için ilgili değillerdir).

Bu tür durumlarda, `ConfigurableModuleBuilder#setExtras` yöntemi kullanılabilir. Aşağıdaki örneğe bakın:

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } = new ConfigurableModuleBuilder<ConfigModuleOptions>()
  .setExtras(
    {
      isGlobal: true,
    },
    (definition, extras) => ({
      ...definition,
      global: extras.isGlobal,
    }),
  )
  .build();
```

Yukarıdaki örnekte, `setExtras` yöntemine geçirilen ilk argüman, "ekstra" özellikler için varsayılan değerleri içeren bir nesnedir. İkinci argüman, otomatik oluşturulan modül tanımını (sağlayıcı, dışa aktarmalar, vb. ile birlikte) ve ekstra özellikleri temsil eden bir nesne olan bir işlevi alır. Bu işlevin döndürdüğü değer, değiştirilmiş bir modül tanımıdır. Bu özel örnekte, `extras.isGlobal` özelliğini alıp modül tanımının `global` özelliğine atıyoruz (ki bu da bir modülün global olup olmadığını belirler, daha fazla bilgi için [buraya](/docs/modules#dynamic-modules) bakın).

Şimdi bu modülü tüketirken, ek `isGlobal` bayrağı aşağıdaki gibi iletilir:

```typescript
@Module({
  imports: [
    ConfigModule.register({
      isGlobal: true,
      folder: './config',
    }),
  ],
})
export class AppModule {}
```

Ancak `isGlobal` ek bir özellik olarak tanımlandığından, `MODULE_OPTIONS_TOKEN` sağlayıcısında bulunmayacaktır:

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) {
    // "options" nesnesinde "isGlobal" özelliği bulunmayacak
    // ...
  }
}
```

#### Otomatik Oluşturulan Yöntemleri Genişletme

Otomatik olarak oluşturulan statik yöntemleri (`register`, `registerAsync`, vb.) ihtiyaca göre genişletebilirsiniz, aşağıdaki gibi:

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass, ASYNC_OPTIONS_TYPE, OPTIONS_TYPE } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {
  static register(options: typeof OPTIONS_TYPE): DynamicModule {
    return {
      // özel mantığınız burada
      ...super.register(options),
    };
  }

  static registerAsync(options: typeof ASYNC_OPTIONS_TYPE): DynamicModule {
    return {
      // özel mantığınız burada
      ...super.registerAsync(options),
    };
  }
}
```

`OPTIONS_TYPE` ve `ASYNC_OPTIONS_TYPE` türlerinin modül tanımlama dosyasından ihraç edilmesine dikkat edin:

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN, OPTIONS_TYPE, ASYNC_OPTIONS_TYPE } = new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
```