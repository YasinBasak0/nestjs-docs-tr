### CLI Eklentisi

> warning **Uyarı** Bu bölüm, yalnızca kod odaklı yaklaşıma uygundur.

TypeScript'in metadata yansıma sistemi, örneğin bir sınıfın hangi özelliklere sahip olduğunu belirlemenin veya belirli bir özelliğin isteğe bağlı mı yoksa zorunlu mu olduğunu tanımanın imkansız olduğu birkaç kısıtlamaya sahiptir. Ancak, bu kısıtlamaların bazıları derleme zamanında ele alınabilir. Nest, TypeScript derleme sürecini geliştiren bir eklenti sağlar ve gerekli olan çoğu tekrarlanan kod miktarını azaltır.

> info **İpucu** Bu eklenti **isteğe bağlıdır**. Tercih ederseniz, tüm dekoratörleri manuel olarak bildirebilir veya ihtiyaç duyduğunuz dekoratörleri belirleyebilirsiniz.

#### Genel Bakış

GraphQL eklentisi otomatik olarak:

- `@Field` kullanılmadığı sürece tüm giriş nesnesi, nesne türü ve argüman sınıfı özelliklerine `@Field` ekler
- soru işaretine bağlı olarak `nullable` özelliğini ayarlar (örneğin, `name?: string` `nullable: true` olarak ayarlanır)
- tür tipine bağlı olarak `type` özelliğini ayarlar (dizileri de destekler)
- yorumlara dayanarak özellikler için açıklamalar oluşturur (eğer `introspectComments` `true` olarak ayarlanmışsa)

Lütfen, dosya adlarınızın eklenti tarafından analiz edilmesi için aşağıdaki soneklerden birine sahip olması **gerektiğine dikkat edin**: `['.input.ts', '.args.ts', '.entity.ts', '.model.ts']` (örneğin, `author.entity.ts`). Farklı bir sonek kullanıyorsanız, eklentinin davranışını `typeFileNameSuffix` seçeneğini belirterek ayarlayabilirsiniz (aşağıya bakınız).

Şimdiye kadar öğrendiklerimizle, türünüzün GraphQL'de nasıl bildirilmesi gerektiğini pakete bildirmek için çok fazla kodu çoğaltmanız gerekebilir. Örneğin, basit bir `Author` sınıfını aşağıdaki gibi tanımlayabilirsiniz:

```typescript
@@filename(authors/models/author.model)
@ObjectType()
export class Author {
  @Field(type => ID)
  id: number;

  @Field({ nullable: true })
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  @Field(type => [Post])
  posts: Post[];
}
```

Orta büyüklükteki projelerle önemli bir sorun olmasa da, büyük bir sınıf kümesine sahip olduğunuzda bu sıkıcı ve bakımı zor hale gelir.

GraphQL eklentisi etkinleştirildiğinde, yukarıdaki sınıf tanımını basitleştirebilirsiniz:

```typescript
@@filename(authors/models/author.model)
@ObjectType()
export class Author {
  @Field(type => ID)
  id: number;
  firstName?: string;
  lastName?: string;
  posts: Post[];
}
```

Eklenti, **Abstract Syntax Tree**'ye dayanarak anında uygun dekoratörleri ekler. Bu nedenle, kodunuzun her tarafına yayılmış `@Field` dekoratörleri ile uğraşmanıza gerek kalmaz.

> info **İpucu** Eklenti, eksik GraphQL özelliklerini otomatik olarak oluşturacaktır, ancak bunları geçersiz kılmanız gerekiyorsa, basitçe `@Field()` aracılığıyla açıkça ayarlayın.

#### Yorumları İnceleme

Yorumlar inceleme özelliği etkinleştirildiğinde, CLI eklentisi, alanlar için açıklamalar oluşturmak için yorumlara dayanabilir.

Örneğin, `roles` özelliğinin bir örnek:

```typescript
/**
 * Kullanıcının rollerinin listesi
 */
@Field(() => [String], {
  description: `Kullanıcının rollerinin listesi`
})
roles: string[];
```

Açıklama değerlerini çoğaltmanız gerekir. `introspectComments` etkinleştirildiğinde, CLI eklentisi bu yorumları çıkarabilir ve otomatik olarak özellikler için açıklamalar sağlayabilir. Şimdi, yukarıdaki alan aşağıdaki gibi basitleştirilebilir:

```typescript
/**
 * Kullanıcının rollerinin listesi
 */
roles: string[];
```

#### CLI Eklentisini Kullanma

Eklentiyi etkinleştirmek için `nest-cli.json` dosyasını açın (eğer [Nest CLI](/docs/cli/overview) kullanıyorsanız) ve aşağıdaki `plugins` yapılandırmasını ekleyin:

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/graphql"]
  }
}
```

Eklentinin davranışını özelleştirmek için `options` özelliğini kullanabilirsiniz.

```javascript
"plugins": [
  {
    "name": "@nestjs/graphql",
    "options": {
      "typeFileNameSuffix": [".input.ts", ".args.ts"],
      "introspectComments": true
    }
  }
]
```

`options` özelliği aşağıdaki arayüzü karşılamalıdır:

```typescript
export interface PluginOptions {
  typeFileNameSuffix?: string[];
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
    <td><code>typeFileNameSuffix</code></td>
    <td><code>['.input.ts', '.args.ts', '.entity.ts', '.model.ts']</code></td>
    <td>GraphQL tür dosyalarının son eki</td>
  </tr>
  <tr>
    <td><code>introspectComments</code></td>
      <td><code>false</code></td>
      <td>True olarak ayarlanırsa, eklenti özellikler için yorumlara dayalı açıklamalar oluşturacaktır</td>
  </tr>
</table>

Eğer CLI kullanmıyorsanız ve özel bir `webpack` yapılandırmanız varsa, bu eklentiyi `ts-loader` ile birleştirerek kullanabilirsiniz:

```javascript
getCustomTransformers: (program: any) => ({
  before: [require('@nestjs/graphql/plugin').before({}, program)]
}),
```

#### SWC builder

Standart kurulumlar için (monorepo olmayanlar), SWC derleyicisi ile CLI Eklentilerini kullanmak için tür kontrolünü etkinleştirmeniz gerekiyor, [burada](/docs/recipes/swc#type-checking) açıklandığı gibi.

```bash
$ nest start -b swc --type-check
```

Monorepo kurulumları için, [buradaki](/docs/recipes/swc#monorepo-and-cli-plugins) talimatları izleyin.

```bash
$ npx ts-node src/generate-metadata.ts
# VEYA npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

Şimdi, `GraphQLModule` yöntemi tarafından oluşturulan seri hale getirilmiş metadata dosyası aşağıdaki gibi yüklenmelidir:

```typescript
import metadata from './metadata'; // <-- "PluginMetadataGenerator" tarafından otomatik oluşturulan dosya

GraphQLModule.forRoot<...>({
  ..., // diğer seçenekler
  metadata,
}),
```

#### `ts-jest` ile entegrasyon (e2e testleri)

E2E testleri bu eklenti etkinleştirilmişken çalıştırırken, şema derlemesiyle ilgili sorunlarla karşılaşabilirsiniz. Örneğin, en yaygın hatalardan biri şudur:

```json
Object type <name> must define one or more fields.
```

Bu, `jest` yapılandırmasının `@nestjs/graphql/plugin` eklentisini herhangi bir yerde içe aktarmamasından kaynaklanır.

Bunu düzeltmek için, e2e testlerinizin dizininde aşağıdaki dosyayı oluşturun:

```javascript
const transformer = require('@nestjs/graphql/plugin');

module.exports.name = 'nestjs-graphql-transformer';
// Herhangi bir yapılandırmayı değiştirdiğinizde sürüm numarasını her zaman değiştirmeniz gerekmektedir. Aksi takdirde, jest değişiklikleri algılamayabilir.
module.exports.version = 1;

module.exports.factory = (cs) => {
  return transformer.before(
    {
      // @nestjs/graphql/plugin seçenekleri (boş olabilir)
    },
    cs.program, // Jest'in daha eski sürümleri için "cs.tsCompiler.program" (<= v27)
  );
};
```

Bununla birlikte, AST dönüştürücüsünü `jest` yapılandırma dosyanız içinde içe aktarın. Varsayılan olarak (başlangıç uygulamasında), e2e testleri yapılandırma dosyanızı `test` klasörü altında bulabilir ve `jest-e2e.json` olarak adlandırabilirsiniz.

```json
{
  ... // diğer yapılandırmalar
  "globals": {
    "ts-jest": {
      "astTransformers": {
        "before": ["<oluşturulan dosyanın yolu>"]
      }
    }
  }
}
```

Eğer `jest@^29` sürümünü kullanıyorsanız, önceki yöntem kullanımdan kalktığından aşağıdaki kesimi kullanın.

```json
{
  ... // diğer yapılandırmalar
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