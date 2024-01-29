### Kütüphaneler

Birçok uygulama genelde aynı problemleri çözmeye ihtiyaç duyar veya bir modüler bileşeni farklı bağlamlarda kullanmak ister. Nest, bu sorunları çözmek için farklı seviyelerde çalışan birkaç yaklaşım sunar, ancak her biri farklı mimari ve organizasyonel hedeflere ulaşmada yardımcı olan bir şekilde çalışır.

Nest [modülleri](/docs/modules), bir uygulama içinde bileşenleri paylaşmayı mümkün kılarak bir yürütme bağlamı sağlamak için kullanışlıdır. Modüller aynı zamanda [npm](https://npmjs.com) ile paketlenerek farklı projelere kurulabilen yeniden kullanılabilir bir kütüphane oluşturmak için de kullanılabilir. Bu, farklı, gevşek bağlantılı veya bağlantısız organizasyonlar tarafından kullanılabilen yapılandırılabilir, yeniden kullanılabilir kütüphaneleri dağıtmak için etkili bir yoldur (örneğin, 3. taraf kütüphaneleri dağıtarak/kurarak).

Kod paylaşımı, yakından organize edilmiş gruplar arasında (örneğin, şirket/proje sınırları içinde) gerçekleştiğinde, bileşenleri paylaşmak için daha hafif bir yaklaşımın olması faydalı olabilir. Monorepo'lar, bunu mümkün kılan bir yapı olarak ortaya çıkmış ve bir monorepo içinde bir **kütüphane**, kodu kolay ve hafif bir şekilde paylaşma imkanı sağlar. Nest monorepo'larında, kütüphanelerin kullanılması, bileşenleri paylaşarak uygulamaların kolayca birleştirilmesini sağlar. Aslında, bu, monolitik uygulamaların ve geliştirme süreçlerinin parçalara ayrılmasını teşvik eder, modüler bileşenlerin oluşturulması ve birleştirilmesine odaklanmayı sağlar.

#### Nest Kütüphaneleri

Bir Nest kütüphanesi, bir uygulamadan farklı olarak kendi başına çalışamayan bir Nest projesidir. Bir kütüphane, kodunun yürütülmesi için içinde bulunduğu uygulamaya içe aktarılmak zorundadır. Bu bölümde açıklanan yerleşik kütüphane desteği yalnızca **monorepo'lar** için geçerlidir (standart mod projeleri, benzer işlevselliği npm paketleri kullanarak elde edebilir).

Örneğin, bir kuruluş, tüm iç uygulamaları kapsayan şirket politikalarını uygulayarak kimlik doğrulama yöneten bir `AuthModule` geliştirebilir. Bu modülü her uygulama için ayrı ayrı oluşturmak yerine veya kodu npm ile paketleyip her projenin bunu kurmasını gerektirmek yerine, bir monorepo bu modülü bir kütüphane olarak tanımlayabilir. Bu şekilde düzenlendiğinde, kütüphanenin modülünü kullanan tüm taraflar, taahhüt edildiği gibi güncel bir `AuthModule` sürümünü görebilir. Bu, bileşen geliştirmeyi, birleştirmeyi kolaylaştırabilir ve uçtan uca testi basitleştirebilir.

#### Kütüphane Oluşturma

Yeniden kullanıma uygun herhangi bir işlevsellik, bir kütüphane olarak yönetilmek üzere adaydır. Bir şeyin kütüphane olup olmamasını ve bir uygulamanın parçası olup olmamasını belirlemek, mimari bir tasarım kararıdır. Kütüphaneler oluşturmak, mevcut bir uygulamadan kodu sadece yeni bir kütüphaneye kopyalamaktan daha fazlasını gerektirir. Bir kütüphane olarak paketlendiğinde, kütüphane kodu uygulamadan ayrılmalıdır. Bu, uygulamayla daha sıkı bir şekilde bağlı olmayan kodlarla karşılaşmayabilirsiniz, ancak bu ek çaba, kütüphanenin birden çok uygulama üzerinde daha hızlı uygulama birleştirmesine olanak tanıdığında karşılığını verebilir.

Bir kütüphane oluşturmak için aşağıdaki komutu çalıştırın:

```bash
$ nest g library my-library
```

Komutu çalıştırdığınızda, `library` şeması size kütüphane için bir önek (AKA takma ad) sorar:

```bash
Kütüphane için hangi öneği (varsayılan: @app) kullanmak istersiniz?
```

Bu, çalışma alanınıza `my-library` adında yeni bir proje oluşturur. Kütüphane tipi bir proje, uygulama tipi bir proje gibi bir şemaya göre adlandırılmış bir klasöre üretilir. Kütüphaneler, monorepo kökü altındaki `libs` klasörü altında yönetilir. Nest, ilk kez bir kütüphane oluşturulduğunda `libs` klasörünü oluşturur.

Bir kütüphane için oluşturulan dosyalar, bir uygulama için oluşturulan dosyalardan biraz farklıdır. Yukarıdaki komutu çalıştırdıktan sonra `libs` klasörünün

 içeriği şu şekildedir:

<div class="file-tree">
  <div class="item">libs</div>
  <div class="children">
    <div class="item">my-library</div>
    <div class="children">
      <div class="item">src</div>
      <div class="children">
        <div class="item">index.ts</div>
        <div class="item">my-library.module.ts</div>
        <div class="item">my-library.service.ts</div>
      </div>
      <div class="item">tsconfig.lib.json</div>
    </div>
  </div>
</div>

`nest-cli.json` dosyasına, kütüphanenin `"projects"` anahtarı altında yeni bir giriş eklenir:

```javascript
...
{
    "my-library": {
      "type": "library",
      "root": "libs/my-library",
      "entryFile": "index",
      "sourceRoot": "libs/my-library/src",
      "compilerOptions": {
        "tsConfigPath": "libs/my-library/tsconfig.lib.json"
      }
}
...
```

Kütüphaneler ve uygulamalar arasındaki `nest-cli.json` metaverilerinde iki fark vardır:

- `"type"` özelliği `"library"` olarak ayarlanmıştır, `"application"` yerine.
- `"entryFile"` özelliği `"index"` olarak ayarlanmıştır, `"main"` yerine.

Bu farklılıklar, kütüphaneleri uygun şekilde işlemek için derleme sürecini ayarlar. Örneğin, bir kütüphane, fonksiyonlarını `index.js` dosyası üzerinden dışa aktarır.

Uygulama tipi projelerle olduğu gibi, her bir kütüphane kendi `tsconfig.lib.json` dosyasına sahiptir ve bu dosya, kök (monorepo genelinde) `tsconfig.json` dosyasını genişletir. Gerekirse bu dosyayı değiştirebilir ve kütübünlere özgü derleyici seçenekleri sağlayabilirsiniz.

Aşağıdaki CLI komutuyla kütüphaneyi oluşturabilirsiniz:

```bash
$ nest build my-library
```

#### Kütüphaneleri Kullanma

Otomatik olarak oluşturulan konfigürasyon dosyalarıyla, kütüphaneleri kullanmak oldukça basittir. `my-library` kütüphanesinden `MyLibraryService`'i `my-project` uygulamasına nasıl içe aktarırız?

İlk olarak, kütüphane modüllerini kullanmak, herhangi diğer Nest modülünü kullanmakla aynıdır. Monorepo'nun yaptığı, kütübleri içe aktarmak ve yapıları şeffaf bir şekilde oluşturmak için yolları yönetmektir. `MyLibraryService`'i kullanmak için, ilgili modülü içe aktarmamız gerekiyor. `my-project/src/app.module.ts` dosyasını şu şekilde değiştirebiliriz:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { MyLibraryModule } from '@app/my-library';

@Module({
  imports: [MyLibraryModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Yukarıda dikkat edilmesi gereken, ES modül `import` satırında `@app` adında bir yol takma adı kullandık. Bu, yukarıda `nest g library` komutuyla sağladığımız `prefix` idi. Nest, bu işlemi tsconfig yolu eşlemeleri aracılığıyla yönetir. Bir kütüphane eklediğimizde, Nest, küresel (monorepo) `tsconfig.json` dosyasındaki `"paths"` anahtarını aşağıdaki gibi günceller:

```javascript
"paths": {
    "@app/my-library": [
        "libs/my-library/src"
    ],
    "@app/my-library/*": [
        "libs/my-library/src/*"
    ]
}
```

Bu nedenle, monorepo ve kütüphane özelliklerinin birleşimi, kütüphane modüllerini uygulamalara dahil etmeyi kolay ve sezgisel hale getirmiştir.

Bu aynı mekanizma, kütüphaneleri içeren uygulamaların oluşturulmasını ve dağıtılmasını sağlar. `MyLibraryModule`'i içe aktardıktan sonra, `nest build` komutunu çalıştırmak, tüm modül çözünürlüğünü otomatik olarak ele alır ve uygulamayı herhangi bir kütüphane bağımlılığıyla birlikte paketler, dağıtmak için. Monorepo için varsayılan derleyici **webpack**'tir, bu nedenle oluşturulan dağıtım dosyası, derlenmiş tüm JavaScript dosyalarını tek bir dosyaya paketleyen tek bir dosyadır. Ayrıca, <a href="https://docs.nestjs.com/cli/monorepo#global-compiler-options">burada</a> açıklandığı gibi `tsc`'ye geçiş yapabilirsiniz.