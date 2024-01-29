### Serverless

Serverless bilişim, bulut sağlayıcısının müşterileri adına sunucularla ilgilenerek talep üzerine makine kaynakları tahsis ettiği bir bulut bilişim yürütme modelidir. Bir uygulama kullanılmadığında, uygulamaya ayrılan bilişim kaynakları yoktur. Fiyatlandırma, bir uygulama tarafından tüketilen kaynak miktarına dayanmaktadır ([source](https://en.wikipedia.org/wiki/Serverless_computing)).

**Serverless mimarisi** ile uygulama kodunuzda yalnızca bireysel işlevlere odaklanırsınız. AWS Lambda, Google Cloud Functions ve Microsoft Azure Functions gibi hizmetler, tüm fiziksel donanım, sanal makine işletim sistemi ve web sunucusu yazılım yönetimini üstlenir.

> info **İpucu** Bu bölüm, serverless işlevlerin avantajları ve dezavantajları hakkında bilgi vermez ve hiçbir bulut sağlayıcının ayrıntılarına inmez.

#### Soğuk Başlatma

Soğuk başlatma, kodunuzun bir süre boyunca yürütülmemiş olması durumunu ifade eder. Kullandığınız bulut sağlayıcısına bağlı olarak, bu, kodu indirme ve çalışma zamanını başlatma işlemlerini içerebilir.

Bu süreç, dil, uygulamanızın gereksinim duyduğu paket sayısı gibi çeşitli faktörlere bağlı olarak **önemli bir gecikme** ekler.

Soğuk başlatma önemlidir ve kontrolümüz dışındaki şeyler olsa da, bu süreyi mümkün olduğunca kısaltmak için kendi tarafımızda yapabileceğimiz birçok şey bulunmaktadır.

Nest'i karmaşık, kurumsal uygulamalarda kullanılmak üzere tasarlanmış tam teşekküllü bir çerçeve olarak düşünebilirsiniz. Ancak, [Bağımsız uygulamalar](/docs/standalone-applications) özelliğini kullanarak, Nest'in DI sistemini basit işçilerde, CRON görevlerinde, CLIs veya serverless işlevlerinde kullanabilirsiniz.

#### Performans Testleri

Nest veya diğer popüler kütüphanelerin (`express` gibi) serverless fonksiyonlar bağlamında kullanımının maliyetini daha iyi anlamak için, aşağıdaki betikleri çalıştırmak için Node çalışma zamanının ne kadar süre gerektirdiğini karşılaştıralım:

```typescript
// #1 Express
import * as express from 'express';

async function bootstrap() {
  const app = express();
  app.get('/', (req, res) => res.send('Merhaba dünya!'));
  await new Promise<void>((resolve) => app.listen(3000, resolve));
}
bootstrap();

// #2 Nest (with @nestjs/platform-express)
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { logger: ['error'] });
  await app.listen(3000);
}
bootstrap();

// #3 Nest as a Standalone application (no HTTP server)
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AppService } from './app.service';

async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule, {
    logger: ['error'],
  });
  console.log(app.get(AppService).getHello());
}
bootstrap();

// #4 Raw Node.js script
async function bootstrap() {
  console.log('Merhaba dünya!');
}
bootstrap();
```

Tüm bu betikler için `tsc` (TypeScript) derleyicisini kullandık, bu nedenle kod paketlenmemiştir (`webpack` kullanılmamıştır).

|                                      |                   |
| ------------------------------------ | ----------------- |
| Express                              | 0.0079s (7.9ms)   |
| Nest with `@nestjs/platform-express` | 0.1974s (197.4ms) |
| Nest (standalone application)        | 0.1117s (111.7ms) |
| Raw Node.js script                   | 0.0071s (7.1ms)   |

> info **Not** Makine: MacBook Pro Mid 2014, 2.5 GHz Dört Çekirdekli Intel Core i7, 16 GB 1600 MHz DDR3, SSD.

Şimdi, tüm performans testlerini tekrarlayalım, ancak bu sefer uygulamamızı tek bir yürütülebilir JavaScript dosyasına paketlemek için (`Nest CLI` yüklüyse `nest build --webpack` komutunu çalıştırabilirsiniz) `webpack` kullanalım.
Ancak, Nest CLI'nin içinde bulunan varsayılan `webpack` yapılandırmasını kullanmak yerine, tüm bağımlılıkları (`node_modules`) bir araya getirmek için aşağıdaki gibi yapılandıracağız:

```javascript
module.exports = (options, webpack) => {
  const lazyImports = [
    '@nestjs/microservices/microservices-module',
    '@nestjs/websockets/socket-module',
  ];

  return {
    ...options,
    externals: [],
    plugins: [
      ...options.plugins,
      new webpack.IgnorePlugin({
        checkResource(resource) {
          if (lazyImports.includes(resource)) {
            try {
              require.resolve(resource);
            } catch (err) {
              return true;
            }
          }
          return false;
        },
      }),
    ],
  };
};
```

> info **İpucu** Nest CLI'ye bu yapılandırmayı kullanmasını söylemek için projenizin kök dizininde yeni bir `webpack.config.js` dosyası oluşturun.

Bu yapılandırmayla, aşağıdaki sonuçları elde ettik:

|                                      |                  |
| ------------------------------------ | ---------------- |
| Express                              | 0.0068s (6.8ms)  |
| Nest with `@nestjs/platform-express` | 0.0815s (81.5ms) |
| Nest (standalone application)        | 0.0319s (31.9ms) |
| Raw Node.js script                   | 0.0066s (6.6ms)  |

> info **Not** Makine: MacBook Pro Mid 2014, 2.5 GHz Dört Çekirdekli Intel Core i7, 16 GB 1600 MHz DDR3, SSD.

> info **İpucu** Ek kod sıkıştırma ve optimizasyon tekniklerini (webpack eklentileri kullanma, vb.) uygulayarak bunu daha da optimize edebilirsiniz.

Gördüğünüz gibi, nasıl derlediğiniz (ve kodunuzu paketleyip paketlemediğiniz) önemlidir ve genel başlatma süresi üzerinde önemli bir etkiye sahiptir. `webpack` ile, bir

 başlangıç Nest uygulamasının (bir modül, denetleyici ve servis içeren başlangıç projesi) başlangıç süresini ortalama ~32ms'ye ve düzenli bir HTTP, express tabanlı NestJS uygulaması için ~81.5ms'ye düşürebilirsiniz.

Daha karmaşık Nest uygulamaları için, örneğin, 10 kaynak içerenler ( `$ nest g resource` şematikle oluşturulan = 10 modül, 10 denetleyici, 10 servis, 20 DTO sınıfı, 50 HTTP uç noktası + `AppModule`) üzerinde genel başlatma süresi MacBook Pro Mid 2014, 2.5 GHz Dört Çekirdekli Intel Core i7, 16 GB 1600 MHz DDR3, SSD üzerinde yaklaşık olarak 0.1298s (129.8ms)'dir. Tipik olarak, monolitik bir uygulamayı serverless bir fonksiyon olarak çalıştırmak fazla anlam ifade etmeyebilir, bu nedenle bu performans testini uygulamanız büyüdükçe başlangıç süresinin nasıl artabileceğinin bir örneği olarak düşünün.

#### Çalışma Zamanı Optimizasyonları

Bu noktaya kadar derleme zamanı optimizasyonlarını ele aldık. Bunlar, sağlayıcıları nasıl tanımladığınız ve Nest modüllerini uygulamanıza nasıl yüklediğinizle ilgili değildir ve bu, uygulamanız büyüdükçe önemli bir rol oynar.

Örneğin, bir [asenkron sağlayıcı](/docs/fundamentals/async-providers) olarak tanımlanmış bir veritabanı bağlantısına sahip olduğunuzu düşünün. Asenkron sağlayıcılar, uygulamanın başlatılmasını bir veya daha fazla asenkron görev tamamlanana kadar geciktirmek için tasarlanmıştır.
Bu, sunucu fonksiyonunuzun ortalama olarak 2 saniye süren bir veritabanına bağlanma süresine ihtiyaç duyması durumunda, uygulamanız zaten çalışmıyorsa (soğuk başlatma) endpointinizin bir yanıt göndermesi için en az iki saniye daha gerekeceği anlamına gelir.

Görüldüğü gibi, sağlayıcılarınızı yapılandırma şekliniz **serverless ortamında** bootstrap zamanının önemli olduğu bir yerde biraz farklıdır.
Başka iyi bir örnek de önbellek için Redis kullanıyorsanız, ancak sadece belirli senaryolarda. Belki de bu durumda Redis bağlantısını bir asenkron sağlayıcı olarak tanımlamamalısınız, çünkü bu, soğuk başlatma süresini yavaşlatacaktır, özellikle bu belirli fonksiyon çağrısı için gerekmese bile.

Ayrıca bazen tam modülleri `LazyModuleLoader` sınıfını kullanarak tembel yükleyebilirsiniz, [bu bölümde](/docs/fundamentals/lazy-loading-modules) açıklandığı gibi. Önbellek de burada harika bir örnektir.
Uygulamanızın, örneğin, Redis ile bağlantı kurarak `CacheModule`'e sahip olduğunu düşünün ve ayrıca Redis depolama ile etkileşim için `CacheService`'i dışa aktarıyor. Eğer tüm potansiyel fonksiyon çağrıları için bunu ihtiyacınız yoksa,
sadece on-demand olarak tembel yükleyebilirsiniz. Bu şekilde, önbellek gerektirmeyen tüm çağrılar için (soğuk başlatma gerçekleştiğinde) daha hızlı bir başlangıç süresi elde edersiniz.

```typescript
if (request.method === RequestMethod[RequestMethod.GET]) {
  const { CacheModule } = await import('./cache.module');
  const moduleRef = await this.lazyModuleLoader.load(() => CacheModule);

  const { CacheService } = await import('./cache.service');
  const cacheService = moduleRef.get(CacheService);

  return cacheService.get(ENDPOINT_KEY);
}
```

Başka harika bir örnek de bazı özel koşullara bağlı olarak (örneğin, giriş argümanları), farklı işlemler gerçekleştirebilen bir webhook veya worker'dır.
Bu durumda, route işleyicinizde, belirli bir fonksiyon çağrısı için uygun bir modülü tembel yüklemek için bir koşul belirleyebilir ve sadece diğer tüm modülleri tembel yükleyebilirsiniz.

```typescript
if (workerType === WorkerType.A) {
  const { WorkerAModule } = await import('./worker-a.module');
  const moduleRef = await this.lazyModuleLoader.load(() => WorkerAModule);
  // ...
} else if (workerType === WorkerType.B) {
  const { WorkerBModule } = await import('./worker-b.module');
  const moduleRef = await this.lazyModuleLoader.load(() => WorkerBModule);
  // ...
}
```

#### Örnek Entegrasyon

Uygulamanızın giriş dosyasının (genellikle `main.ts` dosyası) nasıl görünmesi gerektiği, **çeşitli faktörlere bağlıdır** ve bu nedenle sadece her senaryo için işleyen tek bir şablon yoktur.
Örneğin, sunucu fonksiyonunuzu başlatmak için gerekli olan başlatma dosyası, kullanılan bulut sağlayıcısına (AWS, Azure, GCP vb.) göre değişir.
Ayrıca, tipik bir HTTP uygulamasını birden çok rota/endpoint ile mi yoksa yalnızca tek bir rotayı mı sağlamak istediğinize bağlı olarak (veya belirli bir kod parçasını yürütmek),
uygulamanızın kodu farklı olacaktır (örneğin, fonksiyon başına endpoint yaklaşımı için HTTP sunucusunu başlatmak yerine `NestFactory.createApplicationContext`'i kullanabilirsiniz).

Sadece görselleştirme amaçlı olarak, Nest'i (tam, tamamen işlevsel HTTP yönlendiricisi başlatan `@nestjs/platform-express` kullanarak)
[Serverless](https://www.serverless.com/) çerçevesiyle entegre edeceğiz (bu durumda, AWS Lambda'ya hedef). Daha önce belirttiğimiz gibi, kodunuz, seçtiğiniz bulut sağlayıcısına ve diğer birçok faktöre bağlı olarak değişecektir.

İlk olarak, gerekli paketleri yükleyelim:

```bash
$ npm i @codegenie/serverless-express aws-lambda
$ npm i -D @types/aws-lambda serverless-offline
```

> info **İpucu** Geliştirme döngülerini hızlandırmak için `serverless-offline` eklentisini yükleriz, bu eklenti AWS λ ve API Gateway'i taklit eder.

Kurulum işlemi tamamlandıktan sonra, Serverless çerçevesini yapılandırmak için `serverless.yml` dosyasını oluşturalım:

```yaml
service: serverless-example

plugins:
  - serverless-offline

provider:
  name: aws
  runtime: nodejs14.x

functions:
  main:
    handler: dist/main.handler
    events:
      - http:
          method: ANY
          path: /
      - http:
          method: ANY
          path: '{proxy+}'
```

> info **İpucu** Serverless çerçevesi hakkında daha fazla bilgi için [resmi belgelere](https://www.serverless.com/framework/docs/) göz atın.

Bu konumda, `main.ts` dosyasına gidip gerekli başlatma kodumuzu ilgili şablona güncelleyebiliriz:

```typescript
import { NestFactory } from '@nestjs/core';
import serverlessExpress from '@codegenie/serverless-express';
import { Callback, Context, Handler } from 'aws-lambda';
import { AppModule } from './app.module';

let server: Handler;

async function bootstrap(): Promise<Handler> {
  const app = await NestFactory.create(AppModule);
  await app.init();

  const expressApp = app.getHttpAdapter().getInstance();
  return serverlessExpress({ app: expressApp });
}

export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  server = server ?? (await bootstrap());
  return server(event, context, callback);
};
```

> info **İpucu** Birden çok serverless fonksiyonu oluşturmak ve bunlar arasında ortak modülleri paylaşmak için [CLI Monorepo modunu](/docs/cli/monorepo#monorepo-mode) kullanmanızı öneririz.

> warning **Uyarı** Eğer `@nestjs/swagger` paketini kullanıyorsanız, serverless fonksiyonun bağlamında düzgün çalışması için ek adımlar gerekmektedir. Daha fazla bilgi için bu [konuya](https://github.com/nestjs/swagger/issues/199) göz atın.

Ardından, `tsconfig.json` dosyasını açın ve `esModuleInterop` seçeneğini etkinleştirdiğinizden emin olun, bu, `@codegenie/serverless-express` paketinin düzgün şekilde yüklenmesini sağlar.

```json
{
  "compilerOptions": {
    ...
    "esModuleInterop": true
  }
}
```

Şimdi uygulamamızı oluşturabiliriz (`nest build` veya `tsc` kullanarak) ve lambda fonksiyonumuzu yerelde başlatmak için `serverless` CLI'ı kullanabiliriz:

```bash
$ npm run build
$ npx serverless offline
```

Uygulama çalıştığında tarayıcınızı açın ve `http://localhost:3000/dev/[ANY_ROUTE]` adresine gidin (`[ANY_ROUTE]`, uygulamanıza kayıtlı herhangi bir endpoint'i temsil eder).

Yukarıdaki bölümlerde, `webpack` kullanmanın ve uygulamanızı paketlemenin genel başlatma süresi üzerinde önemli bir etkiye sahip olabileceğini gösterdik.
Ancak, örneğimizle çalışması için `webpack.config.js` dosyanıza ek yapmanız gereken bazı ek yapılandırmalar vardır. Genellikle,
`handler` fonksiyonumuzun alınacağından emin olmak için `output.libraryTarget` özelliğini `commonjs2` olarak değiştirmemiz gerekir.

```javascript
return {
  ...options,
  externals: [],
  output: {
    ...options.output,
    libraryTarget: 'commonjs2',


  },
  // ... geri kalan yapılandırma
};
```

Bu yerine getirildiğinde, artık uygulamanızın kodunu derlemek için `$ nest build --webpack` kullanabilir ve ardından test etmek için `$ npx serverless offline` kullanabilirsiniz.

Ayrıca (ancak **gereksiz** olduğu için yapılması **zorunlu değildir** ve derleme sürecinizi yavaşlatabilir) `terser-webpack-plugin` paketini yüklemenizi ve üretim derlemenizi sıkıştırırken sınıf adlarını bozmamak için konfigürasyonunu geçersiz kılmanızı öneririz. Bu yapılmazsa, uygulamanız içinde `class-validator` kullanırken yanlış davranışlara neden olabilir.

```javascript
const TerserPlugin = require('terser-webpack-plugin');

return {
  ...options,
  externals: [],
  optimization: {
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          keep_classnames: true,
        },
      }),
    ],
  },
  output: {
    ...options.output,
    libraryTarget: 'commonjs2',
  },
  // ... geri kalan yapılandırma
};
```

#### Bağımsız Uygulama Özelliğini Kullanma

Alternatif olarak, fonksiyonunuzu çok hafif tutmak ve HTTP ile ilgili özelliklere (yönlendirme, aynı zamanda koruyucular, interceptor'lar, pipes vb. dahil) ihtiyaç duymuyorsanız,
tüm HTTP sunucusunu (ve altında `express`i) çalıştırmak yerine sadece `NestFactory.createApplicationContext`'i kullanabilirsiniz (yukarıda bahsedildiği gibi), şu şekilde:

```typescript
@@filename(main)
import { HttpStatus } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { Callback, Context, Handler } from 'aws-lambda';
import { AppModule } from './app.module';
import { AppService } from './app.service';

export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  const appContext = await NestFactory.createApplicationContext(AppModule);
  const appService = appContext.get(AppService);

  return {
    body: appService.getHello(),
    statusCode: HttpStatus.OK,
  };
};
```

> info **İpucu** `NestFactory.createApplicationContext`, denetleyici yöntemlerini geliştirmeyi (guard, interceptor vb. ile) sağlamaz. Bunun için `NestFactory.create` yöntemini kullanmalısınız.

Ayrıca, `event` nesnesini aşağıya, diyelim ki, işleyebilecek bir `EventsService` sağlayıcısına geçirebilir ve karşılık gelen bir değeri döndürebilir (giriş değerine ve iş mantığınıza bağlı olarak).

```typescript
export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  const appContext = await NestFactory.createApplicationContext(AppModule);
  const eventsService = appContext.get(EventsService);
  return eventsService.process(event);
};
```