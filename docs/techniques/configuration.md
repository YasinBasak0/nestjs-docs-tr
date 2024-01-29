### Yapılandırma

Uygulamalar genellikle farklı **çevrelerde** çalışır. Çevreye bağlı olarak, farklı yapılandırma ayarları kullanılmalıdır. Örneğin, genellikle yerel ortam, yalnızca yerel DB örneği için geçerli olan belirli veritabanı kimlik bilgilerine dayanır. Üretim ortamı ayrı bir DB kimlik bilgisi kümesini kullanacaktır. Çünkü yapılandırma değişkenleri değişir, iyi bir uygulama [yapılandırma değişkenlerini](https://12factor.net/config) ortamda [saklamak](https://12factor.net/config) en iyisidir.

Dışarıdan tanımlanan ortam değişkenleri, Node.js içinde `process.env` globali aracılığıyla görünür. Bu problemi çözmek için ortam değişkenlerini her ortamda ayrı ayrı ayarlamaya çalışabiliriz. Bu, özellikle bu değerlerin kolayca taklit edilmesi ve/veya değiştirilmesi gereken geliştirme ve test ortamlarında hızlı bir şekilde karmaşık hale gelebilir.

Node.js uygulamalarında, her bir anahtarın belirli bir değeri temsil ettiği anahtar-değer çiftlerini içeren `.env` dosyalarını kullanmak yaygındır, her ortam için doğru `.env` dosyasını takas etmek sadece doğru `.env` dosyasını takas etmek meselesidir.

Bu tekniği Nest'te kullanmak için iyi bir yaklaşım, uygun `.env` dosyasını yükleyen bir `ConfigModule` oluşturmaktır. Kendi modülünüzü yazmayı seçebilirsiniz, ancak Nest, bu paketi kolaylık sağlamak için kutudan çıkar. Bu paketi mevcut bölümde ele alacağız.

#### Kurulum

Kullanmaya başlamak için önce gerekli bağımlılığı kurarız.

```bash
$ npm i --save @nestjs/config
```

> info **Hint** `@nestjs/config` paketi, dahili olarak [dotenv](https://github.com/motdotla/dotenv) kullanır.

> warning **Not** `@nestjs/config`, TypeScript 4.1 veya daha üstünü gerektirir.

#### Başlangıç

Kurulum işlemi tamamlandığında, `ConfigModule`'u içe aktarabiliriz. Genellikle, bunu kök `AppModule`'a içe aktarırız ve davranışını `.forRoot()` statik yöntemini kullanarak kontrol ederiz. Bu sırada, ortam değişkeni anahtar/değer çiftleri ayrıştırılır ve çözülür. Daha sonra, `ConfigModule`'un `ConfigService` sınıfına erişmek için diğer özellik modüllerimizde birkaç seçenek göreceğiz.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot()],
})
export class AppModule {}
```

Yukarıdaki kod, bir `.env` dosyasını (proje kök dizini), `.env` dosyasından gelen anahtar/değer çiftlerini çözer ve bir sonucu `ConfigService` üzerinden erişebileceğiniz özel bir yapıda `process.env` ile birleştirir. `forRoot()` yöntemi, bu ayrıştırılmış/birleştirilmiş yapıyı okumak için bir `get()` yöntemi sağlayan `ConfigService` sağlayıcısını kaydeder. `@nestjs/config`, [dotenv](https://github.com/motdotla/dotenv) paketinin çevre değişkeni adlarını çözme kurallarını kullanır. Bir anahtar, çalışma zamanı ortamında bir çevre değişkeni olarak (örneğin, `export DATABASE_USER=test` gibi OS kabuğu ihracatları aracılığıyla) ve bir `.env` dosyasında bulunursa, çalışma zamanı çevre değişkeni önceliklidir.

Örnek bir `.env` dosyası şöyle görünebilir:

```json
DATABASE_USER=test
DATABASE_PASSWORD=test
```

####

 Özel env dosya yolu

Paket, varsayılan olarak uygulamanın kök dizininde bir `.env` dosyası arar. `.env` dosyası için başka bir yol belirtmek için, `forRoot()`'a ilettiğiniz (isteğe bağlı) seçenekler nesnesinin `envFilePath` özelliğini aşağıdaki gibi ayarlayın:

```typescript
ConfigModule.forRoot({
  envFilePath: '.development.env',
});
```

Ayrıca şu şekilde birden çok `.env` dosyası yolu belirtebilirsiniz:

```typescript
ConfigModule.forRoot({
  envFilePath: ['.env.development.local', '.env.development'],
});
```

Bir değişken birden çok dosyada bulunursa, ilk dosya önceliklidir.

#### env değişkenlerini yükleme devre dışı bırakma

`.env` dosyasını yüklemek istemiyorsanız, ancak sadece çalışma zamanı ortam değişkenlerine erişmek istiyorsanız (örneğin, `export DATABASE_USER=test` gibi OS kabuğu ihracatları gibi), seçenekler nesnesinin `ignoreEnvFile` özelliğini `true` olarak ayarlayın:

```typescript
ConfigModule.forRoot({
  ignoreEnvFile: true,
});
```

#### Modülü global olarak kullanma

`ConfigModule`'u diğer modüllerde kullanmak istediğinizde, onu içe aktarmanız gerekecektir (herhangi bir Nest modülüyle standart olan budur). Alternatif olarak, seçenekler nesnesinin `isGlobal` özelliğini `true` olarak ayarlayarak onu [global bir modül](https://docs.nestjs.com/modules#global-modules) olarak bildirebilirsiniz. Bu durumda, kök modülde (örneğin, `AppModule`'da) yüklendikten sonra diğer modüllerde `ConfigModule`'u içe aktarmanıza gerek kalmaz.

```typescript
ConfigModule.forRoot({
  isGlobal: true,
});
```

#### Özel Yapılandırma Dosyaları

Daha karmaşık projeler için, iç içe geçmiş yapılandırma nesnelerini döndüren özel yapılandırma dosyalarını kullanabilirsiniz. Bu, ilgili yapılandırma ayarlarını işlev bazında (örneğin, veritabanı ile ilgili ayarlar) gruplamak ve bunları bağımsız olarak yönetmek için ayrı dosyalarda depolamak için olanak tanır.

Bir özel yapılandırma dosyası, bir yapılandırma nesnesini döndüren bir fabrika fonksiyonunu içe aktarır. Yapılandırma nesnesi, keyfi olarak iç içe geçmiş düz JavaScript nesnesi olabilir. `process.env` nesnesi, tamamen çözülmüş ortam değişkeni anahtar/değer çiftlerini içerecektir (yukarıda [açıklandığı gibi](#getting-started), `.env` dosyası ve dışarıdan tanımlanan değişkenler çözülüp birleştirilmiştir). Döndürülen yapılandırma nesnesini kontrol ettiğinizden, değerleri uygun bir türe dönüştürmek, varsayılan değerleri ayarlamak vb. gibi gereken mantığı ekleyebilirsiniz. Örneğin:

```typescript
@@filename(config/configuration)
export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT, 10) || 5432
  }
});
```

Bu dosyayı, `ConfigModule.forRoot()` yöntemine geçirdiğimiz seçenekler nesnesinin `load` özelliğini kullanarak yükleriz:

```typescript
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration],
    }),
  ],
})
export class AppModule {}
```

> info **Bilgi** `load` özelliğine atanan değer bir dizi olduğundan, birden çok yapılandırma dosyasını yükleyebilirsiniz (örneğin, `load: [databaseConfig, authConfig]`)

Özel yapılandırma dosyaları ile, YAML dosyaları gibi özel dosyaları da yönetebiliriz. İşte YAML formatını kullanarak yapılandırma örneği:

```yaml
http:
  host: 'localhost'
  port: 8080

db:
  postgres:
    url: 'localhost'
    port: 5432
    database: 'yaml-db'

  sqlite:
    database: 'sqlite.db'
```

YAML dosyalarını okumak ve ayrıştırmak için `js-yaml` paketini kullanabiliriz.

```bash
$ npm i js-yaml
$ npm i -D @types/js-yaml
```

Paket kurulduktan sonra, yukarıda oluşturduğumuz YAML dosyasını yüklemek için `yaml#load` fonksiyonunu kullanırız.

```typescript
@@filename(config/configuration)
import { readFileSync } from 'fs';
import * as yaml from 'js-yaml';
import { join } from 'path';

const YAML_CONFIG_FILENAME = 'config.yaml';

export default () => {
  return yaml.load(
    readFileSync(join(__dirname, YAML_CONFIG_FILENAME), 'utf8'),
  ) as Record<string, any>;
};
```

> warning **Not** Nest CLI, "assets" (TS olmayan dosyalar) dosyalarını derleme sırasında otomatik olarak `dist` klasörüne taşımaz. YAML dosyalarınızın kopyalandığından emin olmak için, `nest-cli.json` dosyasındaki `compilerOptions#assets` nesnesine bunu belirtmeniz gerekir. Örneğin, `config` klasörü `src` klasörü ile aynı düzeydeyse, `"assets": [{{ '{' }}"include": "../config/*.yaml", "outDir": "./dist/config"{{ '}' }}]` değerini ekleyin. Daha fazlasını [buradan](/docs/cli/monorepo#assets) okuyun.

<app-banner-devtools></app-banner-devtools>

#### `ConfigService` Kullanımı

`ConfigService`'den yapılandırma değerlerine erişmek için önce `ConfigService`'yi enjekte etmemiz gerekiyor. Herhangi bir sağlayıcı gibi, bunu kullanacak modül içine - yani `ConfigModule` - içe aktarmamız gerekiyor (eğer `ConfigModule.forRoot()` yöntemine geçilen seçenekler nesnesindeki `isGlobal` özelliğini `true` olarak ayarlamadıysanız). Aşağıdaki gibi bir özellik modülüne içe aktarın.

```typescript
@@filename(feature.module)
@Module({
  imports: [ConfigModule],
  // ...
})
```

Sonra, standard constructor enjeksiyonunu kullanarak onu enjekte edebiliriz:

```typescript
constructor(private configService: ConfigService) {}
```

> info **Hint** `ConfigService`, `@nestjs/config` paketinden içe aktarılır.

Ve sınıfımızda kullanabiliriz:

```typescript
// bir ortam değişkenini al
const dbUser = this.configService.get<string>('DATABASE_USER');

// özel yapılandırma değerini al
const dbHost = this.configService.get<string>('database.host');
```

Yukarıda gösterildiği gibi, bir basit ortam değişkenini almak için `configService.get()` yöntemini kullanın ve değişken adını ileterek. Yukarıda gösterildiği gibi (örneğin, `get<string>(...)`), türü geçirerek TypeScript tip belirleme yapabilirsiniz. `get()` yöntemi ayrıca bir <a href="#custom-configuration-files">Özel Yapılandırma Dosyası</a> ile oluşturulan iç içe geçmiş özel yapılandırma nesnesini de geçebilir, yukarıdaki ikinci örnekte gösterildiği gibi.

Bir tür ipucu olarak bir arabirim kullanarak tüm iç içe geçmiş özel yapılandırma nesnesini alabilirsiniz:

```typescript


interface DatabaseConfig {
  host: string;
  port: number;
}

const dbConfig = this.configService.get<DatabaseConfig>('database');

// şimdi `dbConfig.port` ve `dbConfig.host`'u kullanabilirsiniz
const port = dbConfig.port;
```

`get()` yöntemi ayrıca, aşağıda gösterildiği gibi anahtarı tanımlanmamış bir değer döndürecek şekilde tanımlanan bir ikinci argümanı da alır:

```typescript
// "database.host" tanımlanmamışsa "localhost" kullan
const dbHost = this.configService.get<string>('database.host', 'localhost');
```

`ConfigService`'nin iki opsiyonel generic'i (tür argümanı) vardır. İlk olarak, bir yapılandırma özelliğine erişmeyi engellemek için kullanılır. Aşağıdaki gibi kullanın:

```typescript
interface EnvironmentVariables {
  PORT: number;
  TIMEOUT: string;
}

// kodun bir yerinde
constructor(private configService: ConfigService<EnvironmentVariables>) {
  const port = this.configService.get('PORT', { infer: true });

  // TypeScript Hatası: URL özelliği EnvironmentVariables'da tanımlı olmadığından bu geçersizdir
  const url = this.configService.get('URL', { infer: true });
}
```

`infer` özelliği `true` olarak ayarlandığında, `ConfigService#get` yöntemi otomatik olarak arabirime dayalı olarak özelliğin türünü çıkaracaktır, bu nedenle örneğin, `typeof port === "number"` (eğer `strictNullChecks` bayrağını TypeScript'ten kullanmıyorsanız) çünkü `PORT`, `EnvironmentVariables` arabiriminde bir `number` türüne sahiptir.

Ayrıca, `infer` özelliği ile, nokta notasyonunu kullansanız bile, iç içe geçmiş bir özel yapılandırma nesnesinin özelliğinin türünü çıkarabilirsiniz, aşağıdaki gibi:

```typescript
constructor(private configService: ConfigService<{ database: { host: string } }>) {
  const dbHost = this.configService.get('database.host', { infer: true })!;
  // typeof dbHost === "string"                                          |
  //                                                                     +--> non-null assertion operator
}
```

İkinci generic, birinciye bağlıdır ve `strictNullChecks` açıkken `ConfigService`'nin yöntemlerinin döndürebileceği `undefined` türünü ortadan kaldırmak için bir tür iddiası görevi görür. Örneğin:

```typescript
// ...
constructor(private configService: ConfigService<{ PORT: number }, true>) {
  //                                                               ^^^^
  const port = this.configService.get('PORT', { infer: true });
  //    ^^^ 'number' türünde olacaktır ve artık TS tür iddialarına ihtiyaç duymazsınız
}
```

#### Yapılandırma Alanları

`ConfigModule`, yukarıda gösterildiği gibi <a href="#custom-configuration-files">Özel Yapılandırma Dosyaları</a> bölümünde gösterilen gibi birden çok özel yapılandırma dosyasını tanımlamanıza ve yüklemenize olanak tanır. Bu bölümde gösterildiği gibi iç içe geçmiş yapılandırma nesneleri ile karmaşık yapılandırma nesnesi hiyerarşilerini yönetebilirsiniz. Alternatif olarak, `registerAs()` fonksiyonu ile "isim alanlı" bir yapılandırma nesnesi döndürebilirsiniz:

```typescript
@@filename(config/database.config)
export default registerAs('database', () => ({
  host: process.env.DATABASE_HOST,
  port: process.env.DATABASE_PORT || 5432
}));
```

Özel yapılandırma dosyalarında olduğu gibi, `registerAs()` fabrika fonksiyonunuz içinde, `process.env` nesnesi tamamen çözülmüş ortam değişkeni anahtar/değer çiftlerini içerecektir (yukarıda [açıklandığı gibi](#getting-started), `.env` dosyası ve dışarıdan tanımlanan değişkenler çözülüp birleştirilmiştir).

> info **Hint** `registerAs` fonksiyonu, `@nestjs/config` paketinden içe aktarılır.

İsim alanlı bir yapılandırmayı, özel bir yapılandırma dosyasını yüklediğiniz gibi, `forRoot()` yönteminin seçenekler nesnesinin `load` özelliği ile yükleyin:

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [databaseConfig],
    }),
  ],
})
export class AppModule {}
```

Şimdi, `database` isim alanından `host` değerini almak için nokta notasyonunu kullanın. İsim alanının adına ( `registerAs()` fonksiyonuna geçirilen ilk argüman) uygun olan önek olarak `'database'`'yi kullanın:

```typescript
const dbHost = this.configService.get<string>('database.host');
```

Makul bir alternatif, `database` isim alanını doğrudan enjekte etmektir. Bu, güçlü yazım avantajlarından yararlanmamıza olanak tanır:

```typescript
constructor(
  @Inject(databaseConfig.KEY)
  private dbConfig: ConfigType<typeof databaseConfig>,
) {}
```

> info **Hint** `ConfigType`, `@nestjs/config` paketinden içe aktarılır.

#### Ortam değişkenlerini önbelleğe alın

`process.env`'e erişmek yavaş olabilir, bu nedenle `ConfigModule.forRoot()`'a geçilen seçenekler nesnesinin `cache` özelliğini kullanarak `ConfigService#get` yönteminin performansını artırmak için bu değişkenleri önbelleğe alabilirsiniz.

```typescript
ConfigModule.forRoot({
  cache: true,
});
```

#### Kısmi kayıt

Şu ana kadar, yapılandırma dosyalarını kök modülde (örneğin, `AppModule`) işledik, `forRoot()` yöntemi ile. Belki daha karmaşık bir proje yapınız var ve özellikle her bir özellik modülü ile ilişkilendirilmiş yapılandırma dosyalarını birden çok farklı dizinde bulun. Bu dosyaları kök modülde yüklemek yerine, `@nestjs/config` paketi, yalnızca her özellik modülü ile ilişkilendirilmiş yapılandırma dosyalarını referans alacak olan **kısmi kayıt** adlı bir özellik sağlar. Bu kısmi kayıtı gerçekleştirmek için, bir özellik modülü içinde `forFeature()` statik yöntemini kullanın, aşağıdaki gibi:

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [ConfigModule.forFeature(databaseConfig)],
})
export class DatabaseModule {}
```

> info **Uyarı** Bazı durumlarda, kısmi kayıt ile yüklenen

 özelliklere ait özellikleri bir kurucu içinde değil de `onModuleInit()` kancasında erişmeniz gerekebilir. Bu, `forFeature()` yöntemi modül başlatma sırasında çalıştırıldığından, modül başlatma sırasının belirlenmemiş olması nedeniyle olabilir. Eğer bu şekilde yüklenen değerlere başka bir modülde, bir kurucu içinde erişirseniz, yapılandırmaya bağlı modül henüz başlatılmamış olabilir. `onModuleInit()` metodu, ona bağlı tüm modüller başlatıldıktan sonra çalışır, bu nedenle bu teknik güvenlidir.

 #### Şema Doğrulama

Uygulama başlangıcında gerekli ortam değişkenleri sağlanmamışsa veya belirli doğrulama kurallarına uymuyorsa genellikle bir istisna fırlatmak standart bir uygulamadır. `@nestjs/config` paketi, bunu yapmanın iki farklı yolunu sağlar:

- [Joi](https://github.com/sideway/joi) yerleşik doğrulayıcı. Joi ile, bir nesne şemasını tanımlar ve JavaScript nesnelerini buna göre doğrularsınız.
- Ortam değişkenlerini giriş olarak alan özel bir `validate()` fonksiyonu.

Joi'yi kullanmak için Joi paketini kurmamız gerekmektedir:

```bash
$ npm install --save joi
```

Şimdi Joi doğrulama şemasını tanımlayabilir ve bu şemayı `forRoot()` yöntemi seçenek nesnesinin `validationSchema` özelliği aracılığıyla iletebiliriz, aşağıda gösterildiği gibi:

```typescript
@@filename(app.module)
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().default(3000),
      }),
    }),
  ],
})
export class AppModule {}
```

Varsayılan olarak, tüm şema anahtarları isteğe bağlı kabul edilir. Burada, `NODE_ENV` ve `PORT` için varsayılan değerleri ayarlıyoruz; bunlar, bu değişkenleri ortamda (`.env` dosyası veya işlem ortamı) sağlamazsak kullanılacaktır. Alternatif olarak, bir değerin ortamda (`.env` dosyası veya işlem ortamı) tanımlanmış olması gerektiğini belirtmek için `required()` doğrulama yöntemini kullanabiliriz. Bu durumda, değişkeni ortamda sağlamazsak, doğrulama adımı bir istisna fırlatacaktır. Daha fazla bilgi için [Joi doğrulama yöntemleri](https://joi.dev/api/?v=17.3.0#example)'ne bakın.

Varsayılan olarak, (şema içinde bulunmayan anahtarlar olan) bilinmeyen ortam değişkenlerine izin verilir ve doğrulama istisnası tetiklemez. Varsayılan olarak, tüm doğrulama hataları rapor edilir. Bu davranışları, `forRoot()` seçenekleri nesnesinin `validationOptions` anahtarını kullanarak bir seçenekler nesnesi ile değiştirebilirsiniz. Bu seçenekler nesnesi, [Joi doğrulama seçenekleri](https://joi.dev/api/?v=17.3.0#anyvalidatevalue-options) tarafından sağlanan standart doğrulama seçenekleri özelliklerinden herhangi birini içerebilir. Örneğin, yukarıdaki iki ayarı tersine çevirmek için aşağıdaki gibi seçenekler geçirin:

```typescript
@@filename(app.module)
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().default(3000),
      }),
      validationOptions: {
        allowUnknown: false,
        abortEarly: true,
      },
    }),
  ],
})
export class AppModule {}
```

`@nestjs/config` paketi, varsayılan olarak aşağıdaki ayarları kullanır:

- `allowUnknown`: ortam değişkenlerinde bilinmeyen anahtarların olup olmamasını kontrol eder. Varsayılan `true`'dir.
- `abortEarly`: true ise, ilk hatada doğrulamayı durdurur; false ise, tüm hataları döndürür. Varsayılan olarak `false`'dir.

Unutmayın ki bir kez `validationOptions` nesnesi geçmeye karar verdiğinizde, açıkça belirtmediğiniz tüm ayarlar `Joi`'nin standart varsayılanlarına ( `@nestjs/config` varsayılanları değil) varsayılan yapılır. Örneğin, özel nesnenizde `allowUnknowns`'ı belirtmezseniz, bu, `Joi`'nin varsayılan değeri olan `false` değerine sahip olacaktır. Bu nedenle, özel nesnenizde **her ikisini de** belirtmek en güvenlisidir.

#### Özel Doğrulama Fonksiyonu

Alternatif olarak, bir **senkron** `validate` fonksiyonu belirleyebilirsiniz. Bu fonksiyon, ortam değişkenlerini içeren bir nesneyi alır ve ihtiyaç duyulursa bunları dönüştürmek/mutasyona uğratmak için doğrulanmış ortam değişkenlerini içeren bir nesne döndürür. Eğer fonksiyon bir hata fırlatırsa, uygulamanın başlamasını engeller.

Bu örnekte, `class-transformer` ve `class-validator` paketlerini kullanacağız. İlk olarak şunları tanımlamamız gerekiyor:

- doğrulama kısıtlamalarına sahip bir sınıf,
- `plainToInstance` ve `validateSync` fonksiyonlarını kullanacak bir doğrulama fonksiyonu.

```typescript
@@filename(env.validation)
import { plainToInstance } from 'class-transformer';
import { IsEnum, IsNumber, validateSync } from 'class-validator';

enum Environment {
  Development = "development",
  Production = "production",
  Test = "test",
  Provision = "provision",
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsNumber()
  PORT: number;
}

export function validate(config: Record

<string, unknown>) {
  const validatedConfig = plainToInstance(
    EnvironmentVariables,
    config,
    { enableImplicitConversion: true },
  );
  const errors = validateSync(validatedConfig, { skipMissingProperties: false });

  if (errors.length > 0) {
    throw new Error(errors.toString());
  }
  return validatedConfig;
}
```

Bu hazır olduğunda, `validate` fonksiyonunu `ConfigModule` konfigürasyon seçeneği olarak aşağıdaki gibi kullanın:

```typescript
@@filename(app.module)
import { validate } from './env.validation';

@Module({
  imports: [
    ConfigModule.forRoot({
      validate,
    }),
  ],
})
export class AppModule {}
```

#### Özel Getter Fonksiyonları

`ConfigService`, bir anahtarla bir yapılandırma değerini almak için genel bir `get()` yöntemi tanımlar. Ayrıca, biraz daha doğal bir kodlama stili sağlamak için `getter` fonksiyonları ekleyebiliriz:

```typescript
@@filename()
@Injectable()
export class ApiConfigService {
  constructor(private configService: ConfigService) {}

  get isAuthEnabled(): boolean {
    return this.configService.get('AUTH_ENABLED') === 'true';
  }
}
@@switch
@Dependencies(ConfigService)
@Injectable()
export class ApiConfigService {
  constructor(configService) {
    this.configService = configService;
  }

  get isAuthEnabled() {
    return this.configService.get('AUTH_ENABLED') === 'true';
  }
}
```

Şimdi getter fonksiyonunu aşağıdaki gibi kullanabiliriz:

```typescript
@@filename(app.service)
@Injectable()
export class AppService {
  constructor(apiConfigService: ApiConfigService) {
    if (apiConfigService.isAuthEnabled) {
      // Kimlik doğrulama etkindir
    }
  }
}
@@switch
@Dependencies(ApiConfigService)
@Injectable()
export class AppService {
  constructor(apiConfigService) {
    if (apiConfigService.isAuthEnabled) {
      // Kimlik doğrulama etkindir
    }
  }
}
```

#### Ortam Değişkenleri Yüklendiğinde Kancası

Eğer bir modül konfigürasyonu ortam değişkenlerine bağlıysa ve bu değişkenler `.env` dosyasından yükleniyorsa, `ConfigModule.envVariablesLoaded` kancasını kullanarak `process.env` objesiyle etkileşime geçmeden önce dosyanın yüklendiğinden emin olabilirsiniz. Aşağıdaki örneğe bakın:

```typescript
export async function getStorageModule() {
  await ConfigModule.envVariablesLoaded;
  return process.env.STORAGE === 'S3' ? S3StorageModule : DefaultStorageModule;
}
```

Bu yapının, `ConfigModule.envVariablesLoaded` Promise çözüldükten sonra tüm yapılandırma değişkenlerinin yüklendiğini garanti ettiği bir durumdur.

#### Koşullu Modül Konfigürasyonu

Bir modülü koşullu olarak yüklemek ve koşulu bir ortam değişkeninde belirtmek istediğiniz durumlar olabilir. Neyse ki, `@nestjs/config` buna izin veren bir `ConditionalModule` sağlar.

```typescript
@Module({
  imports: [ConfigModule.forRoot(), ConditionalModule.registerWhen(FooModule, 'USE_FOO')],
})
export class AppModule {}
```

Yukarıdaki modül, `.env` dosyasında `USE_FOO` ortam değişkeni için `false` bir değer olmadığı sürece `FooModule`'u yükleyecektir. Ayrıca, `ConditionalModule`'a işlemi gerçekleştirecek bir `process.env` referansını alan ve bir boolean döndüren bir fonksiyon da geçebilirsiniz:

```typescript
@Module({
  imports: [ConfigModule.forRoot(), ConditionalModule.registerWhen(FooBarModule, (env: NodeJS.ProcessEnv) => !!env['foo'] && !!env['bar'])],
})
export class AppModule {}
```

`ConditionalModule`'u kullanırken, uygulamada aynı zamanda `ConfigModule`'un yüklü olduğundan emin olmalısınız, böylece `ConfigModule.envVariablesLoaded` kancası düzgün bir şekilde referans alınıp kullanılabilir. Eğer kancası 5 saniye içinde veya `registerWhen` yönteminin üçüncü seçenek parametresinde kullanıcı tarafından belirlenen bir zaman aşımı süresinde ters çevrilmezse, `ConditionalModule` bir hata fırlatacak ve Nest uygulamanın başlamasını durduracaktır.

#### Genişletilebilir Değişkenler

`@nestjs/config` paketi ortam değişkeni genişletmeyi destekler. Bu teknikle, bir değişkenin tanımı içinde başka bir değişkenin referansını kullanarak iç içe geçmiş ortam değişkenleri oluşturabilirsiniz. Örneğin:

```json
APP_URL=mywebsite.com
SUPPORT_EMAIL=support@${APP_URL}
```

Bu yapıyla, `SUPPORT_EMAIL` değişkeni `'support@mywebsite.com'` değerine çözünür. `SUPPORT_EMAIL`'in tanımı içinde `APP_URL` değişkeninin değerini çözmek için `${{ '{' }}...{{ '}' }}` sözdizimini kullanın.

> info **Hint** Bu özellik için `@nestjs/config` paketi, [dotenv-expand](https://github.com/motdotla/dotenv-expand) paketini dahili olarak kullanır.

Ortam değişkeni genişletmeyi etkinleştirmek için `ConfigModule`'un `forRoot()` yöntemine geçilen seçenekler nesnesinde `expandVariables` özelliğini kullanın, aşağıdaki gibi:

```typescript
@@filename(app.module)
@Module({
  imports: [
    ConfigModule.forRoot({
      // ...
      expandVariables: true,
    }),
  ],
})
export class AppModule {}
```

#### `main.ts` Dosyasında Kullanımı

Konfigürasyonumuz bir serviste depolandığından, hala `main.ts` dosyasında kullanılabilir. Bu şekilde, uygulama portu veya CORS ana bilgisini gibi değişkenleri depolamak için kullanabilirsiniz.

Buna erişmek için `app.get()` yöntemini ve ardından servis referansını kullanmalısınız:

```typescript
const configService = app.get(ConfigService);
```

Ardından, `get` yöntemini kullanarak konfigürasyon anahtarını çağırarak onu normal şekilde kullanabilirsiniz:

```typescript
const port = configService.get('PORT');
```