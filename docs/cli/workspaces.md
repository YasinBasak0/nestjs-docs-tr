### Çalışma Alanları

Nest'in kodu düzenleme modları iki tanedir:

- **standart mod**: Kendi bağımlılıklarına ve ayarlarına sahip, modülleri paylaşmaya veya karmaşık derlemelere optimize etmeye ihtiyaç duymayan bireysel projelere odaklanan uygulamaları oluşturmak için kullanışlıdır. Bu, varsayılan modudur.
- **monorepo mod**: Bu mod, kod sanatlarını hafif bir **monorepo**nun bir parçası olarak ele alır ve bu, geliştirici ekipleri ve/veya çoklu proje ortamları için daha uygun olabilir. Bu, modüler bileşenleri oluşturmayı ve birleştirmeyi kolaylaştırmak için derleme sürecinin bazı bölümlerini otomatikleştirir, kodun yeniden kullanımını teşvik eder, entegrasyon testini kolaylaştırır, `eslint` kuralları gibi projeye özgü sanatları paylaşmayı kolaylaştırır ve github alt modüller gibi alternatifleri kullanmaktan daha kolaydır. Monorepo modu, monorepo'nun bileşenleri arasındaki ilişkiyi koordine etmek için `nest-cli.json` dosyasında temsil edilen bir **çalışma alanı** konseptini kullanır.

Bu seçimi yapmanın tek etkisi, projelerinizin nasıl oluşturulduğu ve derleme sanatının nasıl oluşturulduğudur. Diğer tüm işlevsellik, CLI'dan çekirdek modüllere kadar, her iki modda da aynı şekilde çalışır.

Ayrıca, **standart mod**dan **monorepo mod**'a herhangi bir zamanda kolayca geçiş yapabilirsiniz, bu nedenle bu seçimi bir süre için erteleyebilir ve bir yaklaşımın daha açık hale gelmesini bekleyebilirsiniz.

#### Standart Mod

`nest new` komutunu çalıştırdığınızda, size bir **proje** oluşturulur, bu da yerleşik bir şematik kullanılarak yapılır. Nest, şunları gerçekleştirir:

1. `nest new` komutuna sağladığınız `name` argümanına karşılık gelen yeni bir klasör oluşturur.
2. Bu klasörü, bir temel Nest uygulamasına karşılık gelen varsayılan dosyalarla doldurur. Bu dosyaları [typescript-starter](https://github.com/nestjs/typescript-starter) deposunda inceleyebilirsiniz.
3. Ayrıca, uygulamanızı derleme, test etme ve sunma işlemleri için çeşitli araçları yapılandıran ve etkinleştiren `nest-cli.json`, `package.json` ve `tsconfig.json` gibi ek dosyalar sağlar.

Buradan itibaren, başlangıç dosyalarını değiştirebilir, yeni bileşenler ekleyebilir, bağımlılıkları ekleyebilirsiniz (örneğin, `npm install`), ve uygulamanızı geri kalan bu belgelerde ele alındığı gibi geliştirebilirsiniz.

#### Monorepo Modu

Monorepo modunu etkinleştirmek için, bir _standart mod_ yapısı ile başlarsınız ve **projeleri** eklersiniz. Bir proje tam bir **uygulama** olabilir (bu projeyi `nest generate app` komutu ile çalışma alanına eklersiniz) veya bir **kütüphane** olabilir (bu projeyi `nest generate library` komutu ile çalışma alanına eklersiniz). Bu özel projelerin bileşen türleri hakkındaki detayları aşağıda tartışacağız. Şu anda dikkat etmeniz gereken temel nokta, mevcut bir standart mod yapısına bir projenin eklenmesinin, onu monorepo moduna **dönüştürdüğü**dir. Bir örneğe bir göz atalım.

Eğer şu komutu çalıştırırsak:

```bash
$ nest new my-project
```

Standart mod bir yapı oluşturduk ve klasör yapısı şu şekilde görünüyor:

<div class="file-tree">
  <div class="item">node_modules</div>
  <div class="item">src</div>
  <div class="children">
    <div class="item">app.controller.ts</div>
    <div class="item">app.module.ts</div>
    <div class="item">app.service.ts</div>
    <div class="item">main.ts</div>
  </div>
  <div class="item">nest-cli.json</div>
  <div class="item">package.json</div>
  <div class="item">tsconfig.json</div>
  <div class="item">.eslintrc.js</div>
</div>

Bu yapıyı bir monorepo modu yapısına dönüştürebiliriz:

```bash
$ cd my-project
$ nest generate app my-app
```

Bu noktada, `nest` mevcut yapıyı bir **monorepo modu** yapısına dönüştürür. Bu önemli birkaç değişikliği beraberinde getirir. Klasör yapısı şimdi şu şekilde:

<div class="file-tree">
  <div class="item">apps</div>
    <div class="children">
      <div class="item">my-app</div>
      <div class="children">
        <div class="item">src</div>
        <div class="children">
          <div class="item">app.controller.ts</div>
          <div class="item">app.module.ts</div>
          <div class="item">app.service.ts</div>
          <div class="item">main.ts</div>
        </div>
        <div class="item">tsconfig.app.json</div>
      </div>
      <div class="item">my-project</div>
      <div class="children">
        <div class="item">src</div>
        <div class="children">
          <div class="item">app.controller.ts</div>
          <div class="item">app.module.ts</div>
          <div class="item">app.service.ts</div>
          <div class="item">main.ts</div>
        </div>
        <div class="item">tsconfig.app.json</div>
      </div>
    </div>
  <div class="item">nest-cli.json</div>
  <div class="item">package.json</div>
  <div class="item">tsconfig.json</div>
  <div class="item">.eslintrc.js</div>
</div>

`generate app` şeması, kodu yeniden düzenledi - her **uygulama** projesini `apps` klasörünün altına taşıyarak ve her projenin kök klasöründe proje özel bir `tsconfig.app.json` dosyası ekleyerek. Orijinal `my-project` uygulamamız, monorepo için **varsayılan proje** haline geldi ve şimdi sadece eklenen `my-app` ile birlikte, `apps` klasörünün altında bulunan bir düzey oldu. Varsayılan projeleri aşağıda ele alacağız.

> Hata **Uyarı** Standart mod bir yapının monorepo'ya dönüştürülmesi, kanonik Nest proje yapısını takip eden projeler için çalışır. Özellikle dönüşüm sırasında şematik, projenin `src` ve `test` klasörlerini kökteki `apps` klasörünün altına taşımaya çalışır. Eğer bir proje bu yapısı kullanmıyorsa, dönüşüm başarısız olabilir veya güvenilmez sonuçlar üretebilir.

#### Çalışma Alanı Projeleri

Bir monorepo, üye varlıklarını yönetmek için bir çalışma alanı kavramını kullanır. Çalışma alanları **projelerden** oluşur. Bir proje ya:

- **uygulama**: Bir `main.ts` dosyasını içeren tam bir Nest uygulamasıdır. Derleme ve yapılandırma düşünceleri dışında, bir çalışma alanındaki bir uygulama türü proje, _standart mod_ yapısı içindeki bir uygulamayla işlevsel olarak aynıdır.
- **kütüphane**: Kütüphane, başka projeler içinde kullanılabilen genel amaçlı bir özellik setini (modüller, sağlayıcılar, denetleyiciler vb.) paketleme şeklidir. Bir kütüphane kendi başına çalışamaz ve `main.ts` dosyası yoktur. Kütüphaneler hakkında daha fazla bilgiyi [buradan](/docs/cli/libraries) alabilirsiniz.

Tüm çalışma alanlarının bir **varsayılan projesi** vardır (bu bir uygulama türü proje olmalıdır). Bu, `nest-cli.json` dosyasındaki üst düzey `"root"` özelliği tarafından tanımlanır ve varsayılan proje kökünü gösterir (daha fazla ayrıntı için aşağıdaki [CLI özellikleri](/docs/cli/monorepo#cli-properties)ne bakın). Genellikle bu, başladığınız _standart mod_ uygulamasıdır ve daha sonra `nest generate app` kullanarak monorepo'ya dönüştürülmüştür. Bu adımları takip ettiğinizde, bu özellik otomatik olarak doldurulur.

Varsayılan projeler, bir proje adı sağlanmadığında `nest build` ve `nest start` gibi `nest` komutları tarafından kullanılır.

Örneğin, yukarıdaki monorepo yapısında şu komutu çalıştırdığınızda:

```bash
$ nest start
```

`my-project` uygulamasını başlatacaktır. `my-app`'i başlatmak içinse şu komutu kullanırız:

```bash
$ nest start my-app
```

#### Uygulamalar

Uygulama türü projeler veya sadece "uygulamalar", çalıştırabilir ve dağıtabileceğiniz tam teşekküllü Nest uygulamalarıdır. Bir uygulama türü projesi, `nest generate app` ile oluşturulur.

Bu komut otomatik olarak bir proje iskeleti oluşturur ve [typescript starter](https://github.com/nestjs/typescript-starter) tarafından kullanılan standart `src` ve `test` klasörlerini içerir. Standart modun aksine, bir monorepo içindeki bir uygulama projesinde paket bağımlılıkları (`package.json`) veya `.prettierrc` ve `.eslintrc.js` gibi diğer proje yapılandırma araçları bulunmaz. Bunun yerine, monorepo genel bağımlılıklar ve yapılandırma dosyaları kullanılır.

Ancak, şematik, proje özel `tsconfig.app.json` dosyasını projenin kök klasöründe oluşturur. Bu yapılandırma dosyası derleme seçeneklerini otomatik olarak ayarlar ve derleme çıktı klasörünü düzgün bir şekilde ayarlar. Dosya, üst düzey (monorepo) `tsconfig.json` dosyasını genişletir, bu nedenle genel ayarları monorepo genelinde yönetebilir ve gerektiğinde projenin düzeyinde bunları geçersiz kılabilirsiniz.

#### Kütüphaneler

Daha önce belirtildiği gibi, kütüphane türü projeler veya sadece "kütüphaneler", çalıştırılabilmesi için Nest bileşenlerini içeren paketlerdir. Bir kütüphane türü projesi, `nest generate library` ile oluşturulur. Bir kütüphane içine hangi öğelerin dahil edileceğini belirlemek, bir mimari tasarım kararıdır. Kütüphaneler hakkında daha ayrıntılı bilgi için [kütüphaneler](/docs/cli/libraries) bölümünü inceleyebilirsiniz.

#### CLI Özellikleri

Nest, standart ve monorepo yapılandırılmış projeleri düzenlemek, derlemek ve dağıtmak için gereken meta verileri `nest-cli.json` dosyasında tutar. Nest, projeleri ekledikçe bu dosyaya otomatik olarak ekler ve günceller, bu nedenle genellikle bu dosya hakkında düşünmenize veya içeriğini düzenlemenize gerek yoktur. Ancak, manuel olarak değiştirmek isteyebileceğiniz bazı ayarlar vardır, bu nedenle dosyanın genel bir anlayışına sahip olmak faydalıdır.

Yukarıdaki adımları kullanarak bir monorepo oluşturduktan sonra, `nest-cli.json` dosyamız şu şekildedir:

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "apps/my-project/src",
  "monorepo": true,
  "root": "apps/my-project",
  "compilerOptions": {
    "webpack": true,
    "tsConfigPath": "apps/my-project/tsconfig.app.json"
  },
  "projects": {
    "my-project": {
      "type": "application",
      "root": "apps/my-project",
      "entryFile": "main",
      "sourceRoot": "apps/my-project/src",
      "compilerOptions": {
        "tsConfigPath": "apps/my-project/tsconfig.app.json"
      }
    },
    "my-app": {
      "type": "application",
      "root": "apps/my-app",
      "entryFile": "main",
      "sourceRoot": "apps/my-app/src",
      "compilerOptions": {
        "tsConfigPath": "apps/my-app/tsconfig.app.json"
      }
    }
  }
}
```

Dosya, şu bölümlere ayrılmıştır:

- standart ve monorepo genel ayarları kontrol eden üst düzey özelliklere sahip genel bir bölüm
- her bir projeye ait meta verileri içeren bir üst düzey özellik (`"projects"`). Bu bölüm, yalnızca monorepo modu yapıları için bulunur.

Üst düzey özellikler şunlardır:

- `"collection"`: bileşenleri oluşturmak için kullanılan şematik koleksiyonunu gösterir; genellikle bu değeri değiştirmemeniz gerekir
- `"sourceRoot"`: standart mod yapılarında tek projenin veya monorepo mod yapılarında _varsayılan proje_'nin köküne işaret eder
- `"compilerOptions"`: derleyici seçeneklerini belirten anahtarları olan bir harita ve bu seçeneğin ayarını belirten değerler içerir; ayrıntılar için aşağıya bakın
- `"generateOptions"`: anahtarları belirli genel üreteç seçeneklerini belirten bir harita ve bu seçeneğin ayarını belirten değerleri içerir; ayrıntılar için aşağıya bakın
- `"monorepo"`: (yalnızca monorepo) monorepo modu yapıları için, bu değer her zaman `true` olarak ayarlanır
- `"root"`: (yalnızca monorepo) _varsayılan proje_'nin proje köküne işaret eder

#### Genel Derleyici Seçenekleri

Bu özellikler, `nest build` veya `nest start`'ın bir parçası olarak gerçekleşen herhangi bir derleme adımını ve derleyiciyi etkileyen çeşitli seçenekleri belirtir. Ayrıca derleyici, `tsc` veya webpack olup olmadığına bakılmaksızın etkiler.

| Özellik Adı         | Özellik Değer Türü | Açıklama                                                                                                                                                                                                                                                                                                                                  |
| ------------------- | ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `webpack`           | boolean             | `true` ise [webpack derleyicisini](https://webpack.js.org/) kullanır. `false` veya belirtilmemişse, `tsc` kullanılır. Monorepo modunda, varsayılan değer `true` (webpack kullan), standart modda varsayılan değer `false` (tsc kullan). Ayrıntılar için aşağıya bakın. (deprecated: `builder` kullanın)                 |
| `tsConfigPath`      | string              | (**yalnızca monorepo**) `nest build` veya `nest start` çağrıldığında `project` seçeneği olmadan (örneğin, varsayılan proje derlendiğinde veya başlatıldığında) kullanılacak `tsconfig.json` ayarlarını içeren dosyaya işaret eder.                                                      |
| `webpackConfigPath` | string              | webpack seçenekleri dosyasına işaret eder. Belirtilmemişse, Nest, `webpack.config.js` dosyasını arar. Daha fazla ayrıntı için aşağıya bakın.                                                                                                                                                                                               |
| `deleteOutDir`      | boolean             | `true` ise, derleyici her çağrıldığında önce derleme çıkış dizinini kaldırır (`tsconfig.json` yapılandırmasında belirtildiği gibi, varsayılan olarak `./dist`).                                                                                                                                                                        |
| `assets`            | array               | Her derleme adımı başladığında (varlık dağıtımı, `--watch` modunda yapılan artımlı derlemelerde **gerçekleşmez**) otomatik olarak non-TypeScript varlıklarını dağıtmayı etkinleştirir. Ayrıntılar için aşağıya bakın.                                                                                                                       |
| `watchAssets`       | boolean             | `true` ise, **tüm** non-TypeScript varlıkları izleme modunda çalışır. (Varlıkları izlemek için daha ince kontrol için [Varlıklar](cli/monorepo#assets) bölümüne bakın).                                                                                                                                                          |
| `manualRestart`     | boolean             | `true` ise, elle sunucuyu yeniden başlatmak için `rs` kısayolunu etkinleştirir. Varsayılan değer `false`'dir.                                                                                                                                                                                                                           |
| `builder`           | string/object       | CLI'ya hangi `builder`'ın proje derlemek için kullanılacağını bildirir (`tsc`, `swc` veya `webpack`). Builder'ın davranışını özelleştirmek için, `type` (`tsc`, `swc` veya `webpack`) ve `options` içeren bir nesne iletebilirsiniz.                                           |
| `typeCheck`         | boolean             | `true` ise, SWC tarafından sürülen projeler için (builder `swc` olduğunda) tip kontrolünü etkinleştirir. Varsayılan değer `false`'dir.                                                                                                                                                                                              |

#### Genel Üretim Seçenekleri

Bu özellikler, `nest generate` komutu tarafından kullanılacak varsayılan üretim seçeneklerini belirtir.

| Özellik Adı | Özellik Değer Türü | Açıklama                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ------------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `spec`        | boolean _or_ object | Değer boolean ise, `true` varsayılan olarak `spec` üretimini etkinleştirir ve `false` varsayılan olarak devre dışı bırakır. CLI komut satırında geçirilen bir bayrak bu ayarı geçersiz kılar, aynı zamanda projeye özgü `generateOptions` ayarı (aşağıda daha fazla bilgi) bu ayarı geçersiz kılar. Değer bir nesne ise, her anahtar bir şematik adını temsil eder ve boolean değeri varsayılan olarak bu belirli şematik için spec üretimini etkinleştirir / devre dışı bırakır. |
| `flat`        | boolean             | `true` ise, tüm üretme komutları düz bir yapı oluşturur                                                                                                                                                                                                                                                                                                                                                                                 |

Aşağıdaki örnek, tüm projeler için varsayılan olarak spec dosyası oluşturmanın devre dışı bırakılması gerektiğini belirtmek için boolean bir değer kullanır:

```javascript
{
  "generateOptions": {
    "spec": false
  },
  ...
}
```

Aşağıdaki örnek, tüm projeler için varsayılan olarak düz dosya oluşturmanın devre dışı bırakılması gerektiğini belirtmek için boolean bir değer kullanır:

```javascript
{
  "generateOptions": {
    "flat": true
  },
  ...
}
```

Aşağıdaki örnekte, `spec` dosyası üretimi sadece `service` şematikleri için devre dışı bırakılmıştır (örneğin, `nest generate service...`):

```javascript
{
  "generateOptions": {
    "spec": {
      "service": false
    }
  },
  ...
}
```

> uyarı **Uyarı** `spec`'i bir nesne olarak belirtirken, üretim şematik adı için anahtarın otomatik takma ad işlemini desteklemediğini unutmayın. Bu, bir anahtarın örneğin `service: false` olarak belirtilmesi ve bir hizmeti `s` takma adıyla oluşturmaya çalıştığınızda,

 spec'in hala oluşturulacağı anlamına gelir. Normal şematik adı ve takma adın her ikisinin de amaçlandığı gibi çalışmasını sağlamak için, normal komut adını ve takma adını aşağıdaki gibi belirtin.
>
> ```javascript
> {
>   "generateOptions": {
>     "spec": {
>       "service": false,
>       "s": false
>     }
>   },
>   ...
> }
> ```

#### Proje Özel Üretim Seçenekleri

Genel üretim seçeneklerini belirtmenin yanı sıra, aynı zamanda proje özel üretim seçeneklerini de belirleyebilirsiniz. Proje özel üretim seçenekleri, genel üretim seçeneklerinin tam olarak aynı formatı izler, ancak doğrudan her projede belirtilir.

Proje özel üretim seçenekleri, genel üretim seçeneklerini geçersiz kılar.

```javascript
{
  "projects": {
    "cats-project": {
      "generateOptions": {
        "spec": {
          "service": false
        }
      },
      ...
    }
  },
  ...
}
```

> uyarı **Uyarı** Üretim seçenekleri için öncelik sırası şu şekildedir. CLI komut satırında belirtilen seçenekler, proje özel seçeneklerin üzerindedir. Proje özel seçenekler, genel seçenekleri geçersiz kılar.

#### Belirtilen derleyici

Farklı varsayılan derleyicilerin nedeni, daha büyük projelerde (örneğin, bir monorepo içinde daha tipik) webpack'ın derleme sürelerinde ve tüm proje bileşenlerini birleştiren tek bir dosyayı oluşturmada önemli avantajlara sahip olabilmesidir. İndividüel dosyalar oluşturmak istiyorsanız, `"webpack"`'ı `false` olarak ayarlayın, bu da derleme işleminin `tsc` (veya `swc`) kullanmasına neden olacaktır.

#### Webpack Seçenekleri

Webpack seçenekleri dosyası, standart [webpack yapılandırma seçeneklerini](https://webpack.js.org/configuration/) içerebilir. Örneğin, webpack'a varsayılan olarak hariç tutulan `node_modules`'ı paketlemesini söylemek için şunu `webpack.config.js`'ye ekleyin:

```javascript
module.exports = {
  externals: [],
};
```

Webpack yapılandırma dosyası bir JavaScript dosyası olduğundan, varsayılan seçenekleri alan ve değiştirilmiş bir nesne döndüren bir işlevi dışa açabilirsiniz:

```javascript
module.exports = function (options) {
  return {
    ...options,
    externals: [],
  };
};
```

#### Varlıklar

TypeScript derlemesi otomatik olarak derleyici çıktısını (`.js` ve `.d.ts` dosyalarını) belirtilen çıktı dizinine dağıtır. Aynı zamanda `.graphql` dosyaları, `resimler`, `.html` dosyaları ve diğer varlıklar gibi TypeScript olmayan dosyaların dağıtılması uygun olabilir. Bu, `nest build` (ve herhangi bir başlangıç derleme adımı) işlemini hafif bir **geliştirme derlemesi** adımı olarak düşünmenizi sağlar, burada TypeScript olmayan dosyaları düzenleyebilir ve tekrarlayan derleme ve test işlemleri gerçekleştirebilirsiniz.
Varlıkların `src` klasöründe bulunması gerekir, aksi takdirde kopyalanmazlar.

`assets` anahtarının değeri, dağıtılacak dosyaları belirten öğelerin bir dizisi olmalıdır. Öğeler, basit bir `glob` benzeri dosya özelliklerine sahip dizeler olabilir, örneğin:

```typescript
"assets": ["**/*.graphql"],
"watchAssets": true,
```

Daha ince kontrol için, öğeler, aşağıdaki anahtarları içeren nesneler olabilir:

- `"include"`: Dağıtılacak varlıklar için `glob` benzeri dosya özellikleri
- `"exclude"`: `include` listesinden **hariç tutulacak** varlıklar için `glob` benzeri dosya özellikleri
- `"outDir"`: Varlıkların dağıtılacağı yolun (kök klasöre göre) belirtildiği bir dize. Derleyici çıktısı için yapılandırılan aynı çıkış dizini için varsayılan değerler.
- `"watchAssets"`: boolean; `true` ise, belirtilen varlıkları izleme modunda çalıştır

Örneğin:

```typescript
"assets": [
  { "include": "**/*.graphql", "exclude": "**/omitted.graphql", "watchAssets": true },
]
```

> uyarı **Uyarı** `watchAssets` ayarlarını içeren bir `assets` özelliği içindeki herhangi bir `watchAssets` ayarını geçersiz kılar.
#### Proje Özellikleri

Bu öğe yalnızca monorepo modu yapıları için geçerlidir. Bu özellikleri genellikle düzenlememeniz gerekir, çünkü bunlar Nest'in monorepo içindeki projeleri ve yapılandırma seçeneklerini bulmak için kullandığı özelliklerdir.