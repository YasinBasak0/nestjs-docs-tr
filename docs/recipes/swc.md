### SWC

[SWC](https://swc.rs/) (Speedy Web Compiler), derleme ve paketleme için kullanılabilen genişletilebilir bir Rust tabanlı platformdur.
SWC'yi Nest CLI ile kullanmak, geliştirme sürecinizi önemli ölçüde hızlandırmak için harika ve basit bir yoldur.

> info **İpucu** SWC, varsayılan TypeScript derleyicisinden yaklaşık **x20 kat daha hızlıdır**.

#### Kurulum

Başlamak için önce birkaç paket kurun:

```bash
$ npm i --save-dev @swc/cli @swc/core
```

#### Başlarken

Kurulum işlemi tamamlandığında, Nest CLI ile `swc` derleyicisini aşağıdaki gibi kullanabilirsiniz:

```bash
$ nest start -b swc
# VEYA nest start --builder swc
```

> info **İpucu** Eğer depo bir monorepo ise, [bu bölüme](/docs/recipes/swc#monorepo) göz atın.

`-b` bayrağını iletmek yerine, aynı zamanda `nest-cli.json` dosyanızda `compilerOptions.builder` özelliğini `"swc"` olarak ayarlayabilirsiniz, şu şekilde:

```json
{
  "compilerOptions": {
    "builder": "swc"
  }
}
```

Derleyicinin davranışını özelleştirmek için, aşağıdaki gibi iki özelliği içeren bir nesne iletebilirsiniz: `type` (`"swc"`) ve `options`:

```json
"compilerOptions": {
  "builder": {
    "type": "swc",
    "options": {
      "swcrcPath": "infrastructure/.swcrc",
    }
  }
}
```

Uygulamayı izleme modunda çalıştırmak için şu komutu kullanın:

```bash
$ nest start -b swc -w
# VEYA nest start --builder swc --watch
```

#### Tür denetimi

SWC, varsayılan TypeScript derleyicisinin aksine kendisi herhangi bir tür denetimi yapmaz, bu nedenle bunu açmak için `--type-check` bayrağını kullanmanız gerekir:

```bash
$ nest start -b swc --type-check
```

Bu komut, Nest CLI'nin SWC ile birlikte `tsc`'yi `noEmit` modunda çalıştırmasını sağlayacak ve bu, tür denetimi yapacaktır. Yine de `--type-check` bayrağını iletmek yerine, aynı zamanda `nest-cli.json` dosyanızda `compilerOptions.typeCheck` özelliğini `true` olarak ayarlayabilirsiniz, şu şekilde:

```json
{
  "compilerOptions": {
    "builder": "swc",
    "typeCheck": true
  }
}
```

#### CLI Eklentileri (SWC)

`--type-check` bayrağı otomatik olarak **NestJS CLI eklentilerini** yürütür ve ardından uygulama çalışma zamanında bu eklentiler tarafından yüklenen seri hale getirilmiş bir meta veri dosyası oluşturur.

#### SWC Konfigürasyonu

SWC derleyicisi, NestJS uygulamalarının gereksinimlerine uyacak şekilde önceden yapılandırılmıştır. Bununla birlikte, `.swcrc` adında bir dosya oluşturarak seçenekleri istediğiniz gibi özelleştirebilirsiniz.

```json
{
  "$schema": "https://json.schemastore.org/swcrc",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

#### Monorepo

Eğer depo bir monorepo ise, o zaman `swc` derleyicisini kullanmak yerine `webpack`'ı `swc-loader`'ı kullanacak şekilde yapılandırmalısınız.

İlk olarak, gerekli paketi kurun:

```bash
$ npm i --save-dev swc-loader
```

Kurulum tamamlandığında, uygulamanızın kök dizininde aşağıdaki içeriğe sahip bir `webpack.config.js` dosyası oluşturun:

```js
const swcDefaultConfig = require('@nestjs/cli/lib/compiler/defaults/swc-defaults').swcDefaultsFactory().swcOptions;

module.exports = {
  module: {
    rules: [
      {
        test: /\.ts$/,
        exclude: /node_modules/,
        use: {
          loader: 'swc-loader',
          options: swcDefaultConfig,
        },
      },
    ],
  },
};
```

#### Monorepo ve CLI Eklentileri

Şimdi eğer CLI eklentilerini kullanıyorsanız, `swc-loader` bunları otomatik olarak yüklemeyecek. Bunun yerine, bunları manuel olarak yükleyen ayrı bir dosya oluşturmalısınız.
Bunu yapmak için, `main.ts` dosyanızın yanına `generate-metadata.ts` adında bir dosya oluşturun ve aşağıdaki içeriğe sahip olsun:

```ts
import { PluginMetadataGenerator } from '@nestjs/cli/lib/compiler/plugins';
import { ReadonlyVisitor } from '@nestjs/swagger/dist/plugin';

const generator = new PluginMetadataGenerator();
generator.generate({
  visitors: [new ReadonlyVisitor({ introspectComments: true, pathToSource: __dirname })],
  outputDir: __dirname,
  watch: true,
  tsconfigPath: 'apps/<name>/tsconfig.app.json',
});
```

> info **İpucu** Bu örnekte `@nestjs/swagger` eklentisini kullandık, ancak tercihinize göre başka bir eklenti kullanabilirsiniz.

`generate()` yöntemi aşağıdaki seçenekleri kabul eder:

|                    |                                                                                                |
| ------------------ | ---------------------------------------------------------------------------------------------- |
| `watch`            | Projeyi değişikliklere karşı izleyip izlemediği.                                                 |
| `tsconfigPath`     | `tsconfig.json` dosyasının yolunu. Geçerli çalışma dizinine (`process.cwd()`) göreli.        |
| `outputDir`        | Meta veri dosyasının kaydedileceği dizinin yolu.                                               |
| `visitors`         | Meta veri oluşturmak için kullanılacak ziyaretçilerin bir dizisi.                               |
| `filename`         | Meta veri dosyasının adı. Varsayılan olarak `metadata.ts`.                                     |
| `printDiagnostics` | Tanılama sonuçlarını konsola yazdırılıp yazdırılmayacağı. Varsayılan olarak `true`.             |

Son olarak, `generate-metadata` betiğini aşağıdaki komutla ayrı bir terminal penceresinde çalıştırabilirsiniz:

```bash
$ npx ts-node src/generate-metadata.ts
# VEYA npx ts-node apps/{SIZIN_UYGULAMANIZ}/src/generate-metadata.ts
```

#### Yaygın Sorunlar

Uygulamanızda TypeORM/MikroORM veya başka bir ORM kullanıyorsanız, döngüsel import sorunlarıyla karşılaşabilirsiniz. SWC, **döngüsel importları** iyi bir şekilde işlemez, bu nedenle şu çalışma çözümünü kullanmalısınız:

```typescript
@Entity()
export class User {
  @OneToOne(() => Profile, (profile) => profile.user)
  profile: Relation<Profile>; // <--- sadece "Profile" yerine "Relation<>" tipini görün
}
```

> info **İpucu** `Relation` tipi, `typeorm` paketinden alınır.

Bu, özelliğin türünün döngüsel bağımlılık sorunlarını önlemek için transpile edilmiş kodun özelliğin meta verilerine kaydedilmemesini sağlar.

Eğer ORM'niz benzer bir çalışma çözümü sağlamıyorsa, kendiniz wrapper tipi tanımlayabilirsiniz:

```typescript
/**
 * ESM modülleri döngüsel bağımlılık sorununu çözmek için kullanılan wrapper tip.
 * Özelliğin türünün reflection metadata tarafından kaydedilmesinden kaynaklanan
 * döngüsel bağımlılık sorununu önler.
 */
export type WrapperType<T> = T; // WrapperType === Relation
```

Projedeki tüm [döngüsel bağımlılık enjeksiyonları](/docs/fundamentals/circular-dependency) için yukarıda tanımlanan özel wrapper tipini kullanmanız gerekecektir:

```typescript
@Injectable()
export class UserService {
  constructor(
    @Inject(forwardRef(() => ProfileService))
    private readonly profileService: WrapperType<ProfileService>,
  ) {};
}
```

### Jest + SWC

Jest kullanarak SWC'yi kullanmak için aşağıdaki paketleri kurmanız gerekir:

```bash
$ npm i --save-dev jest @swc/core @swc/jest
```

Kurulum tamamlandığında, `package.json`/`jest.config.js` dosyanızı (konfigürasyonunuza bağlı olarak) aşağıdaki içerikle güncelleyin:

```json
{
  "jest": {
    "transform": {
      "^.+\\.(t|j)s?$": ["@swc/jest"]
    }
  }
}
```

Ayrıca, `.swcrc` dosyanıza şu `transform` özelliklerini eklemeniz gerekecek: `legacyDecorator`, `decoratorMetadata`:

```json
{
  "$schema": "https://json.schemastore.org/swcrc",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "transform": {
      "legacyDecorator": true,
      "decoratorMetadata": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

Eğer projenizde NestJS CLI Eklentilerini kullanıyorsanız, `PluginMetadataGenerator`'ü manuel olarak çalıştırmanız gerekecektir. Daha fazla bilgi için [bu bölüme](/docs/recipes/swc#monorepo-and-cli-plugins) göz atın.

### Vitest

[Vitest](https://vitest.dev/), Vite ile birlikte çalışmak üzere tasarlanmış hızlı ve hafif bir test çalıştırıcısıdır. NestJS projeleriyle entegre edilebilen modern, hızlı ve kullanımı kolay bir test çözümü sunar.

#### Kurulum

Başlamak için önce gerekli paketleri kurun:

```bash
$ npm i --save-dev vitest unplugin-swc @swc/core @vitest/coverage-v8
```

#### Yapılandırma

Uygulamanızın kök dizininde aşağıdaki içerikte bir `vitest.config.ts` dosyası oluşturun:

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    root: './',
  },
  plugins: [
    // Bu, test dosyalarını SWC ile derlemek için gereklidir
    swc.vite({
      // Modül türünü açıkça belirtmek için, bu değeri .swcrc yapılandırma dosyasından devralmamak için
      module: { type: 'es6' },
    }),
  ],
});
```

Bu yapılandırma dosyası, Vitest ortamını, kök dizini ve SWC eklentisini kurar. Ayrıca, ek bir `include` alanını belirten e2e testleri için ayrı bir yapılandırma dosyası oluşturmalısınız:

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    root: './',
  },
  plugins: [swc.vite()],
});
```

Ayrıca, testlerinizde TypeScript yollarını desteklemek için `alias` seçeneklerini ayarlayabilirsiniz:

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    alias: {
      '@src': './src',
      '@test': './test',
    },
    root: './',
  },
  resolve: {
    alias: {
      '@src': './src',
      '@test': './test',
    },
  },
  plugins: [swc.vite()],
});
```

#### E2E testlerindeki importları güncelleme

`import * as request from 'supertest'` gibi E2E test importlarını `import request from 'supertest'` olarak değiştirin. Bu, Vite ile paketlendiğinde Vitest'in supertest için varsayılan bir import beklemesinden kaynaklanmaktadır. Bu özel kurulumda namespace import kullanımı sorunlara neden olabilir.

Son olarak, package.json dosyanızdaki test betiklerini aşağıdaki gibi güncelleyin:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:cov": "vitest run --coverage",
    "test:debug": "vitest --inspect-brk --inspect --logHeapUsage --threads=false",
    "test:e2e": "vitest run --config ./vitest.config.e2e.ts"
  }
}
```

Bu betikler, testleri çalıştırmak, değişiklikleri izlemek, kod kapsam raporları oluşturmak ve hata ayıklamak için Vitest'i yapılandırır. test:e2e betiği özellikle özel bir yapılandırma dosyası ile E2E testlerini çalıştırmak içindir.

Bu kurulum ile NestJS projenizde Vitest kullanmanın avantajlarının tadını çıkarabilirsiniz, bu da daha hızlı test yürütme ve daha modern bir test deneyimi anlamına gelir.

> info **İpucu** Çalışan bir örnek için [bu depoya](https://github.com/TrilonIO/nest-vitest) göz atabilirsiniz.