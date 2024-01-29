### CLI Eklentisi

TypeScript'in meta veri yansıma sisteminin birkaç kısıtlaması vardır ki bu da örneğin bir sınıfın hangi özelliklere sahip olduğunu belirlemeyi veya belirli bir özelliğin isteğe bağlı mı yoksa zorunlu mu olduğunu tanımamızı imkansız kılar. Bununla birlikte, bu kısıtlamaların bazıları derleme zamanında ele alınabilir. Nest, gerekli olan tekrarlayan kod miktarını azaltmak için TypeScript derleme sürecini geliştiren bir eklenti sağlar.

> info **İpucu** Bu eklenti **isteğe bağlıdır**. Tercih ederseniz tüm dekoratörleri manuel olarak bildirebilir veya yalnızca ihtiyacınız olan belirli dekoratörleri kullanabilirsiniz.

#### Genel Bakış

Swagger eklentisi otomatik olarak şunları yapacaktır:

- `@ApiProperty` kullanılmadıkça tüm DTO özelliklerini etiketler
- `required` özelliğini soru işaretine bağlı olarak ayarlar (örneğin, `name?: string` `required: false` olarak ayarlanır)
- Tip (dizileri de destekler) bağlı olarak `type` veya `enum` özelliğini ayarlar
- Atanmış varsayılan değere dayalı olarak `default` özelliğini ayarlar
- `class-validator` dekoratörlerine dayalı olarak bir dizi doğrulama kuralını ayarlar (`classValidatorShim` özelliği `true` olarak ayarlandıysa)
- Her uç noktaya uygun bir durum ve `type` (yanıt modeli) ile yanıt dekoratörü ekler
- Açıklamaları ve uç noktalar için açıklamaları ( `introspectComments` özelliği `true` olarak ayarlandıysa) temel alarak özellikler ve uç noktalar için örnek değerler oluşturur
- Lütfen dosya adlarınızın eklenti tarafından analiz edilebilmesi için şu ek sone sahip olmalıdır: `['.dto.ts', '.entity.ts']` (örneğin, `create-user.dto.ts` gibi).

Farklı bir sone kullanıyorsanız, eklentinin davranışını belirleyebilmek için `dtoFileNameSuffix` seçeneğini belirterek ayarları yapabilirsiniz (aşağıya bakınız).

Daha önce, Swagger UI ile etkileşimli bir deneyim sağlamak istiyorsanız, modellerinizin/bileşenlerinizin nasıl bildirileceğini pakete bildirmek için birçok kodu çoğaltmanız gerekiyordu. Örneğin, basit bir `CreateUserDto` sınıfını şu şekilde tanımlayabilirdiniz:

```typescript
export class CreateUserDto {
  @ApiProperty()
  email: string;

  @ApiProperty()
  password: string;

  @ApiProperty({ enum: RoleEnum, default: [], isArray: true })
  roles: RoleEnum[] = [];

  @ApiProperty({ required: false, default: true })
  isEnabled?: boolean = true;
}
```

Orta büyüklükteki projelerle önemli bir sorun olmasa da, büyük bir sınıf kümeniz olduğunda bakımı zor hale gelir ve uzun hale gelir.

[Swagger eklentisi etkinleştirildiğinde](/docs/openapi/cli-plugin#using-the-cli-plugin), yukarıdaki sınıf tanımını basitçe bildirebilirsiniz:

```typescript
export class CreateUserDto {
  email: string;
  password: string;
  roles: RoleEnum[] = [];
  isEnabled?: boolean = true;
}
```

> info **Not** Swagger eklentisi, TypeScript türlerinden ve class-validator dekoratörlerinden @ApiProperty() açıklamalarını türetecektir. Bu, üretilen Swagger UI belgeleri için API'nizi açıkça tanımlamanıza yardımcı olur. Ancak, çalışma zamanında doğrulama hala class-validator dekoratörleri tarafından ele alınacaktır. Bu nedenle, otomatik açıklamalara dayalı belge oluşturmak ve hala çalışma zamanı doğrulamalarını istiyorsanız, class-validator dekoratörlerini kullanmaya devam etmek gereklidir.

> info **İpucu** [Haritalanmış tipler yardımcı programlarını](https://docs.nestjs.com/openapi/mapped-types) (örneğin, `PartialType`) DTO'larda kullanırken, şemayı almak için `@nestjs/mapped-types` yerine `@nestjs/swagger`'ı içe aktarın.

Eklenti, **Abstract Syntax Tree**'e dayanarak gerekli dekoratörleri dinamik olarak ekler. Bu nedenle, kodunuzun her tarafına yayılmış `@ApiProperty` dekoratörleriyle uğraşmanız gerekmez.

> info **İpucu** Eklenti, eksik Swagger özelliklerini otomatik olarak oluşturacaktır, ancak bunları geçersiz kılmanız gerekiyorsa, bunları basitçe `@ApiProperty()` aracılığıyla açıkça ayarlarsınız.

#### Yorumlar İncelemesi

Yorumlar inceleme özelliği etkinleştirildiğinde, CLI eklentisi, açıklamalara dayalı olarak özellikler ve uç noktalar için açıklamalar ve örnek değerler oluşturacaktır.

Örneğin, bir örnek `roles` özelliğine sahipseniz:

```typescript
/**
 * A list of user's roles
 * @example ['admin']
 */
@ApiProperty({
  description: `A list of user's roles`,
  example: ['admin'],
})
roles: RoleEnum[] = [];
```

Açıklama ve örnek değerlerin her ikisini de çoğaltmanız gerekir. `introspectComments` etkin olduğunda, CLI e

klentisi bu yorumları çıkarabilir ve özellikler için açıklamaları (ve tanımlanmışse örnekleri) otomatik olarak sağlayabilir. Şimdi, yukarıdaki özellik şu şekilde basitçe bildirilebilir:

```typescript
/**
 * A list of user's roles
 * @example ['admin']
 */
roles: RoleEnum[] = [];
```

Eklenti seçeneklerini belirlemeniz için `dtoKeyOfComment` ve `controllerKeyOfComment` adlı seçenekler bulunmaktadır. Aşağıdaki örneğe bakın:

```typescript
export class SomeController {
  /**
   * Create some resource
   */
  @Post()
  create() {}
}
```

Varsayılan olarak, bu seçenekler `"description"` olarak ayarlanmıştır. Bu, eklentinin `ApiOperation` operatörü üzerinde `description` anahtarına `"Create some resource"` değerini atayacağı anlamına gelir. Şu şekilde:

```ts
@ApiOperation({ description: "Create some resource" })
```

> info **İpucu** Modeller için aynı mantık geçerlidir ancak bunun yerine `ApiProperty` dekoratörü kullanılır.

#### CLI Eklentisini Kullanma

Eklentiyi etkinleştirmek için `nest-cli.json` dosyasını (eğer [Nest CLI](/docs/cli/overview) kullanıyorsanız) açın ve aşağıdaki `plugins` yapılandırmasını ekleyin:

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/swagger"]
  }
}
```

`options` özelliğini kullanarak eklentinin davranışını özelleştirebilirsiniz.

```javascript
"plugins": [
  {
    "name": "@nestjs/swagger",
    "options": {
      "classValidatorShim": false,
      "introspectComments": true
    }
  }
]
```

`options` özelliği aşağıdaki arayüzü karşılamak zorundadır:

```typescript
export interface PluginOptions {
  dtoFileNameSuffix?: string[];
  controllerFileNameSuffix?: string[];
  classValidatorShim?: boolean;
  dtoKeyOfComment?: string;
  controllerKeyOfComment?: string;
  introspectComments?: boolean;
}
```

<table>
  <tr>
    <th>Seçenek</th>
    <th>Varsayılan</th>
    <th>Açıklama</th>
  </tr>
  <tr>
    <td><code>dtoFileNameSuffix</code></td>
    <td><code>['.dto.ts', '.entity.ts']</code></td>
    <td>DTO (Veri Transfer Nesnesi) dosyalarının sone</td>
  </tr>
  <tr>
    <td><code>controllerFileNameSuffix</code></td>
    <td><code>.controller.ts</code></td>
    <td>Denetleyici dosyalarının sone</td>
  </tr>
  <tr>
    <td><code>classValidatorShim</code></td>
    <td><code>true</code></td>
    <td>True olarak ayarlanırsa, modül <code>class-validator</code> doğrulama dekoratörlerini yeniden kullanacaktır (örneğin, <code>@Max(10)</code>, şema tanımına <code>max: 10</code> ekleyecektir)</td>
  </tr>
  <tr>
    <td><code>dtoKeyOfComment</code></td>
    <td><code>'description'</code></td>
    <td><code>ApiProperty</code> üzerinde açıklama metnini ayarlamak için kullanılacak özellik anahtarı.</td>
  </tr>
  <tr>
    <td><code>controllerKeyOfComment</code></td>
    <td><code>'description'</code></td>
    <td><code>ApiOperation</code> üzerinde açıklama metnini ayarlamak için kullanılacak özellik anahtarı.</td>
  </tr>
  <tr>
    <td><code>introspectComments</code></td>
    <td><code>false</code></td>
    <td>True olarak ayarlanırsa, eklenti açıklamalara dayalı olarak özellikler ve uç noktalar için açıklamalar ve örnek değerler oluşturacaktır</td>
  </tr>
</table>

Eklenti seçenekleri güncellendiğinde, lütfen uygulamanızın `/dist` klasörünü silin ve yeniden derleyin.
Eğer CLI kullanmıyorsanız ve özel bir `webpack` yapılandırmanız varsa, bu eklentiyi `ts-loader` ile birlikte kullanabilirsiniz:

```javascript
getCustomTransformers: (program: any) => ({
  before: [require('@nestjs/swagger/plugin').before({}, program)]
}),
```

#### SWC Derleyicisi

Standart kurulumlar için (monorepo olmayanlar), SWC derleyicisi ile CLI Eklentilerini kullanmak için tip kontrolünü etkinleştirmeniz gerekmektedir, [burada](/docs/recipes/swc#type-checking) açıklandığı şekilde.

```bash
$ nest start -b swc --type-check
```

Monorepo kurulumları için, [buradaki](/docs/recipes/swc#monorepo-and-cli-plugins) talimatları takip edin.

```bash
$ npx ts-node src/generate-metadata.ts
# VEYA npx ts-node apps/{SİZİN_UYGULAMANIZ}/src/generate-metadata.ts
```

Şimdi, serileştirilmiş meta veri dosyası `SwaggerModule#loadPluginMetadata` yöntemi tarafından yüklenmelidir, aşağıdaki gibi:

```typescript
import metadata from './metadata'; // <-- "PluginMetadataGenerator" tarafından otomatik olarak oluşturulan dosya

await SwaggerModule.loadPluginMetadata(metadata); // <-- burada
const document = SwaggerModule.createDocument(app, config);
```

#### `ts-jest` ile Entegrasyon (e2e testleri)

E2E testlerini çalıştırmak için, `ts-jest` bellek içinde kaynak kodu dosyalarını anında derler. Bu, Nest CLI derleyicisini kullanmaz ve herhangi bir eklenti uygulamaz veya AST dönüşümleri yapmaz.

Eklentiyi etkinleştirmek için, e2e testlerinizin dizininde aşağıdaki dosyayı oluşturun:

```javascript
const transformer = require('@nestjs/swagger/plugin');

module.exports.name = 'nestjs-swagger-transformer';
// yapılandırmayı değiştirdiğinizde her zaman versiyon numarasını değiştirmelisiniz - aksi takdirde jest değişiklikleri algılamaz
module.exports.version = 1;

module.exports.factory = (cs) => {
  return transformer.before(
    {
      // @nestjs/swagger/plugin seçenekleri (boş olabilir)
    },
    cs.program, // Jest'in eski sürümleri için "cs.tsCompiler.program" (v27'ye kadar)
  );
};
```

Bunu yerine getirdiğinizde, jest konfigürasyon dosyanız içinde AST dönüştürücüyü içe aktarın. Varsayılan olarak (başlangıç uygulamasında), e2e testleri yapılandırma dosyası `test` klasörü altında `jest-e2e.json` olarak adlandırılmıştır.

```json
{
  ... // diğer konfigürasyonlar
  "globals": {
    "ts-jest": {
      "astTransformers": {
        "before": ["<oluşturulan dosyanın yolu>"]
      }
    }
  }
}
```

Eğer `jest@^29` kullanıyorsanız, önceki yaklaşımın kaldırıldığı için aşağıdaki kodu kullanın.

```json
{
  ... // diğer konfigürasyonlar
  "transform": {
    "^.+\\.(t|j)s$": [
      "ts-jest",
      {
        "astTransformers": {
          "before": ["<oluşturulan dosyanın yolu>"]
        }
      }
    ]
  }
}
```

#### `jest` Sorun Giderme (e2e testleri)

Eğer `jest` değişikliklerinizi algılamıyorsa, Jest'in zaten **önbelleğe aldığından** şüphelenilebilir. Yeni yapılandırmayı uygulamak için Jest'in önbellek dizinini temizlemeniz gerekir.

Önbellek dizinini temizlemek için aşağıdaki komutu NestJS proje klasörünüzde çalıştırın:

```bash
$ npx jest --clearCache
```

Otomatik önbellek temizleme başarısız olursa, önbellek klasörünü aşağıdaki komutlarla manuel olarak kaldırabilirsiniz:

```bash
# Jest önbellek dizinini bulun (genellikle /tmp/jest_rs)
# NestJS proje kökünde aşağıdaki komutu çalıştırarak
$ npx jest --showConfig | grep cache
# örnek sonuç:
#   "cache": true,
#   "cacheDirectory": "/tmp/jest_rs"

# Jest önbellek dizinini silin veya boşaltın
$ rm -rf  <cacheDirectory değeri>
# örnek:
# rm -rf /tmp/jest_rs
```
