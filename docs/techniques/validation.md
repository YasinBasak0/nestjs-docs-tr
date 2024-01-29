### Doğrulama

Herhangi bir verinin web uygulamasına gönderildiğinin doğruluğunu kontrol etmek iyi bir uygulamadır. Nest, gelen istekleri otomatik olarak doğrulamak için doğrudan kullanıma hazır birkaç pipe sağlar:

- `ValidationPipe`
- `ParseIntPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`

`ValidationPipe`, güçlü [class-validator](https://github.com/typestack/class-validator) paketini ve onun deklaratif doğrulama dekoratörlerini kullanır. `ValidationPipe`, özel doğrulama kurallarını belirtmenin basit bir yolunu sunar. Bu kurallar, her modüldeki yerel sınıf/DTO bildirimlerinde basit açıklamalarla belirtilir.

#### Genel Bakış

[Pipes](/docs/pipes) bölümünde, basit pipe'lar oluşturma sürecini ve bu sürecin nasıl çalıştığını göstermek için bu pipe'ları denetleyicilere, yöntemlere veya uygulamaya genel olarak bağlama sürecini gözden geçirdik. Bu bölümün konularını en iyi anlamak için o bölümü inceleyin. Burada, `ValidationPipe`'nin çeşitli **gerçek dünya** kullanım durumlarına odaklanacağız ve bazı gelişmiş özelleştirme özelliklerini nasıl kullanacağımızı göstereceğiz.

### Yerleşik ValidationPipe Kullanımı

Kullanmaya başlamak için önce gerekli bağımlılığı yükleriz.

```bash
$ npm i --save class-validator class-transformer
```

> info **İpucu** `ValidationPipe`, `@nestjs/common` paketinden içe aktarılmıştır.

Bu pipe, [`class-validator`](https://github.com/typestack/class-validator) ve [`class-transformer`](https://github.com/typestack/class-transformer) kütüphanelerini kullanır, bu nedenle birçok seçenek mevcuttur. Bu ayarları, pipe'a iletilen bir yapılandırma nesnesi aracılığıyla yapılandırırsınız. İşte yerleşik seçenekler:

```typescript
export interface ValidationPipeOptions extends ValidatorOptions {
  transform?: boolean;
  disableErrorMessages?: boolean;
  exceptionFactory?: (errors: ValidationError[]) => any;
}
```

Bunlara ek olarak, `class-validator` seçeneklerinin tamamı (`ValidatorOptions` arayüzünden devralınanlar) kullanılabilir:

<table>
  <tr>
    <th>Seçenek</th>
    <th>Tür</th>
    <th>Açıklama</th>
  </tr>
  <tr>
    <td><code>enableDebugMessages</code></td>
    <td><code>boolean</code></td>
    <td>True olarak ayarlanırsa, doğrulayıcı bir şey doğru değilse konsola ekstra uyarı mesajları yazdırır.</td>
  </tr>
  <tr>
    <td><code>skipUndefinedProperties</code></td>
    <td><code>boolean</code></td>
    <td>True olarak ayarlanırsa, doğrulayıcı, doğrulanan nesnenin içinde tanımlanmamış tüm özelliklerin doğrulamasını atlar.</td>
  </tr>
  <tr>
    <td><code>skipNullProperties</code></td>
    <td><code>boolean</code></td>
    <td>True olarak ayarlanırsa, doğrulayıcı, doğrulanan nesnenin içinde null olan tüm özelliklerin doğrulamasını atlar.</td>
  </tr>
  <tr>
    <td><code>skipMissingProperties</code></td>
    <td><code>boolean</code></td>
    <td>True olarak ayarlanırsa, doğrulayıcı, doğrulanan nesnenin içinde null veya tanımlanmamış tüm özelliklerin doğrulamasını atlar.</td>
  </tr>
  <tr>
    <td><code>whitelist</code></td>
    <td><code>boolean</code></td>
    <td>True olarak ayarlanırsa, doğrulayıcı, doğrulanan (döndürülen) nesnenin hiçbir doğrulama dekoratörü kullanmayan özelliklerini kaldırır.</td>
  </tr>
  <tr>
    <td><code>forbidNonWhitelisted</code></td>
    <td><code>boolean</code></td>
    <td>True olarak ayarlanırsa, doğrulayıcı, doğrulama dekoratörü kullanmayan özellikleri kaldırmak yerine bir istisna fırlatır.</td>
  </tr>
  <tr>
    <td><code>forbidUnknownValues</code></td>
    <td><code>boolean</code></td>
    <td>True olarak ayarlanırsa, bilinmeyen nesnelerin doğrulanma girişimleri hemen başarısız olur.</td>
  </tr>
  <tr>
    <td><code>disableErrorMessages</code></td>
    <td><code>boolean</code></td>
    <td>True olarak ayarlanırsa, doğrulama hataları istemciye geri döndürülmez.</td>
  </tr>
  <tr>
    <td><code>errorHttpStatusCode</code></td>
    <td><code>number</code></td>
    <td>Bu ayar, bir hata oluştuğunda hangi istisna türünün kullanılacağını belirtmenizi sağlar. Varsayılan olarak <code>BadRequestException</code> fırlatır.</td>
  </tr>
  <tr>
    <td><code>exceptionFactory</code></td>
    <td><code>Fonksiyon</code></td>
    <td>Doğrulama hatalarının bir dizisini alır ve fırlatılacak bir istisna nesnesini döndürür.</td>
  </tr>
  <tr>
    <td><code>groups</code></td>
    <td><code>dizi</code></td>
    <td>Nesnenin doğrulanması sırasında kullanılacak gruplar.</td>
  </tr>
  <tr>
    <td><code>always</code></td>
    <td><code>boolean</code></td>
    <td>Decorator'ların <code>always</code> seçeneği için varsayılanı belirler. Varsayılan, dekoratör seçeneklerinde geçersiz kılınabilir.</td>
  </tr>

  <tr>
    <td><code>strictGroups</code></td>
    <td><code>boolean</code></td>
    <td>Belirtilmemiş veya boşsa, en az bir grup içeren dekoratörleri yoksayar.</td>
  </tr>
  <tr>
    <td><code>dismissDefaultMessages</code></td>
    <td><code>boolean</code></td>
    <td>True olarak ayarlanırsa, doğrulama varsayılan mesajları kullanmaz. Hata mesajı her zaman açıkça belirtilmezse <code>undefined</code> olacaktır.</td>
  </tr>
  <tr>
    <td><code>validationError.target</code></td>
    <td><code>boolean</code></td>
    <td><code>ValidationError</code>'da hedefin açıklayıp açıklanmayacağını belirtir.</td>
  </tr>
  <tr>
    <td><code>validationError.value</code></td>
    <td><code>boolean</code></td>
    <td><code>ValidationError</code>'da doğrulanan değerin açıklayıp açıklanmayacağını belirtir.</td>
  </tr>
  <tr>
    <td><code>stopAtFirstError</code></td>
    <td><code>boolean</code></td>
    <td>Belirtilen özelliğin doğrulamasında ilk hatayla karşılaşıldıktan sonra doğrulamayı durdurur. Varsayılan false'tur.</td>
  </tr>
</table>

> info **Dikkat** `class-validator` paketi hakkında daha fazla bilgiyi [repository](https://github.com/typestack/class-validator) üzerinde bulabilirsiniz.

#### Otomatik Doğrulama

İlk olarak, `ValidationPipe`'i uygulama düzeyinde bağlamaya başlayarak, tüm uç noktaların yanlış verileri almasına karşı korunduğunu sağlayacağız.

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

Pipe'ımızı test etmek için temel bir uç nokta oluşturalım.

```typescript
@Post()
create(@Body() createUserDto: CreateUserDto) {
  return 'Bu işlem yeni bir kullanıcı ekler';
}
```

> info **İpucu** TypeScript, **generikleri veya arabirimleri** hakkında metadata bilgisi saklamadığından, DTO'larınızda bunları kullanıyorsanız, `ValidationPipe` gelen verileri düzgün bir şekilde doğrulayamayabilir. Bu nedenle, DTO'larınızda somut sınıfları kullanmayı düşünmelisiniz.

> info **İpucu** DTO'larınızı içe aktarırken, bunu çalışma zamanında silineceği için yalnızca bir tip içeren içe aktarma kullanamazsınız, yani `import {{ '{' }} CreateUserDto {{ '}' }}` yerine `import type {{ '{' }} CreateUserDto {{ '}' }}` kullanmamız gerekiyor.

Şimdi `CreateUserDto`'muzda birkaç doğrulama kuralı ekleyebiliriz. Bu kuraları, `class-validator` paketi tarafından sağlanan dekoratörleri kullanarak [burada](https://github.com/typestack/class-validator#validation-decorators) ayrıntılı olarak açıklanan bir şekilde yaparız. Bu şekilde, `CreateUserDto`'yu kullanan herhangi bir rotanın otomatik olarak bu doğrulama kurallarını uygulayacağından emin olabiliriz.

```typescript
import { IsEmail, IsNotEmpty } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsNotEmpty()
  password: string;
}
```

Bu kuralların yerine getirilmemesi durumunda, isteği gelen bir isteğin gövdesinde geçersiz bir `email` özelliği varsa, uygulama otomatik olarak bir `400 Bad Request` kodu ile birlikte aşağıdaki yanıt gövdesiyle yanıt verecektir:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": ["email must be an email"]
}
```

İstek gövdelerini doğrulamakla kalmaz, `ValidationPipe` ayrıca diğer istek nesnesi özellikleriyle de kullanılabilir. Diyelim ki uç nokta yolunda `:id`'yi kabul etmek istiyoruz. Bu istek parametresi için yalnızca sayıların kabul edilmesini sağlamak için şu yapıyı kullanabiliriz:

```typescript
@Get(':id')
findOne(@Param() params: FindOneParams) {
  return 'Bu işlem bir kullanıcı döndürür';
}
```

`FindOneParams`, bir DTO gibi, sadece `class-validator`'ü kullanarak doğrulama kurallarını tanımlayan bir sınıftır. Aşağıdaki gibi görünecektir:

```typescript
import { IsNumberString } from 'class-validator';

export class FindOneParams {
  @IsNumberString()
  id: number;
}
```

#### Ayrıntılı hataları devre dışı bırakma

Hata mesajları, bir istekte neyin yanlış olduğunu açıklamak için yararlı olabilir. Ancak, bazı üretim ortamları ayrıntılı hataları devre dışı bırakmayı tercih edebilir. Bunun için bir seçenek nesnesini `ValidationPipe`'a ileterek bunu yapabilirsiniz:

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    disableErrorMessages: true,
  }),
);
```

Sonuç olarak, ayrıntılı hata mesajları yanıt gövdesinde görüntülenmeyecek.

#### Özellikleri filtreleme

`ValidationPipe`'ımız, yöntem işleyicisi tarafından alınmaması gereken özellikleri de filtreleyebilir. Bu durumda, kabul edilebilir özellikleri **beyaz listeye** alabilir ve beyaz listeye dahil edilmeyen her özellik otomatik olarak sonuç nesnesinden çıkartılır. Örneğin, işleyicimizin `email` ve `password` özelliklerini beklediği, ancak bir isteğin ayrıca bir `age` özelliği içerdiği durumda, bu özellik sonuç DTO'sundan otomatik olarak kaldırılabilir. Bu davranışı etkinleştirmek için `whitelist`'i `true` olarak ayarlayın.

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
  }),
);
```

`true` olarak ayarlandığında, bu, beyaz listeye dahil edilmeyen özellikleri (doğrulama sınıfındaki herhangi bir dekoratörü olmayanları) otomatik olarak kaldırır.

Alternatif olarak, beyaz listeye dahil edilmeyen özellikler bulunduğunda isteğin işlenmesini durdurabilir ve kullanıcıya bir hata yanıtı döndürebilirsiniz. Bunu etkinleştirmek için `forbidNonWhitelisted` seçeneğini `true` olarak ayarlayın ve `whitelist`'i `true` olarak ayarlayın.

<app-banner-courses></app-banner-courses>

#### Yük nesnelerini dönüştürme

Ağ üzerinden gelen yük nesneleri düz JavaScript nesneleridir. `ValidationPipe`, yükleri otomatik olarak DTO sınıflarına göre tip belirlenmiş nesneler haline dönüştürebilir. Otomatik dönüşümü etkinleştirmek için `transform`'u `true` olarak ayarlayın. Bu, bir yöntem düzeyinde şu şekilde yapılabilir:

```typescript
@@filename(cats.controller)
@Post()
@UsePipes(new ValidationPipe({ transform: true }))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Bu davranışı global olarak etkinleştirmek için seçeneği global bir pipe üzerine ayarlayın:

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,
  }),
);
```

Otomatik dönüşüm seçeneği etkinleştirildiğinde, `ValidationPipe`, aynı zamanda ilkel tiplerin dönüştürülmesini de gerçekleştirir. Aşağıdaki örnekte, `findOne()` yöntemi, bir çıkarılmış `id` yol parametresini temsil eden bir argüman alır:

```typescript
@Get(':id')
findOne(@Param('id') id: number) {
  console.log(typeof id === 'number'); // true
  return 'This action returns a user';
}
```

Varsayılan olarak, her yol parametresi ve sorgu parametresi ağ üzerinden bir `string` olarak gelir. Yukarıdaki örnekte, `id` tipini bir `number` olarak belirttik (yöntem imzasında). Bu nedenle, `ValidationPipe`, bir dize kimliğini otomatik olarak bir sayıya dönüştürmeye çalışacaktır.

#### Açık dönüşüm

Yukarıdaki bölümde, `ValidationPipe`'nin sorgu ve yol parametrelerini beklenen türe dayalı olarak örtük olarak nasıl dönüştürebileceğini gösterdik. Ancak, bu özellik otomatik dönüşümün etkinleştirilmesini gerektirir.

Alternatif olarak (otomatik dönüşüm devre dışı bırakıldığında), değerleri açıkça `ParseIntPipe` veya `ParseBoolPipe` kullanarak dönüştürebilirsiniz (unutmayın ki, her yol parametresi ve sorgu parametresi varsayılan olarak ağ üzerinden bir `string` olarak gelir).

```typescript
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('sort', ParseBoolPipe) sort: boolean,
) {
  console.log(typeof id === 'number'); // true
  console.log(typeof sort === 'boolean'); // true
  return 'This action returns a user';
}
```

> info **İpucu** `ParseIntPipe` ve `ParseBoolPipe`, `@nestjs/common` paketinden içe aktarılır.

#### Haritalama Türleri

**CRUD** (Create/Read/Update/Delete) gibi özellikleri oluştururken genellikle temel bir varlık türü üzerinde değişiklikler yapmak yararlı olabilir. Nest, bu görevi daha uygun hale getirmek için tip dönüşümleri gerçekleştiren bir dizi yardımcı işlev sağlar.

> **Uyarı** Uygulamanız `@nestjs/swagger` paketini kullanıyorsa, Mapped Types hakkında daha fazla bilgi için [bu bölüme](/docs/openapi/mapped-types) başvurun. Benzer şekilde, `@nestjs/graphql` paketini kullanıyorsanız [bu bölüme](/docs/graphql/mapped-types) başvurun. Her iki paket de ağırlıklı olarak tiplere bağlı oldukları için farklı bir içe aktarıma ihtiyaç duyarlar. Bu nedenle, uygun olmayan bir paket yerine (`@nestjs/swagger` veya `@nestjs/graphql`), `@nestjs/mapped-types`'i kullandıysanız, belgelenmemiş çeşitli yan etkilerle karşılaşabilirsiniz.

Giriş doğrulama tipleri (DTO'lar olarak da adlandırılır) oluştururken, aynı tip üzerinde **oluşturma** ve **güncelleme** varyasyonlarını oluşturmak genellikle yararlıdır. Örneğin, **oluşturma** varyantı tüm alanları gerektirebilirken, **güncelleme** varyantı tüm alanları isteğe bağlı yapabilir.

Nest, bu görevi daha kolay hale getirmek ve gereksiz tekrarı en aza indirmek için `PartialType()` yardımcı işlevini sağlar.

`PartialType()` işlevi, giriş tipinin tüm özelliklerini isteğe bağlı olarak ayarlanmış bir tipte (sınıfta) döndürür. Örneğin, aşağıdaki gibi bir **oluşturma** tipimiz varsa:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Varsayılan olarak, bu alanların hepsi zorunludur. Aynı alanlara sahip ancak her biri isteğe bağlı olan bir tip oluşturmak için `PartialType()`'ı kullanabilirsiniz ve bu işlevi çağırmak için sınıf referansını (`CreateCatDto`) bir argüman olarak iletebilirsiniz:

```typescript
export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

> info **İpucu** `PartialType()` işlevi, `@nestjs/mapped-types` paketinden içe aktarılır.

`PickType()` işlevi, bir giriş tipinden belirli özellikleri seçerek yeni bir tip (sınıf) oluşturur. Örneğin, aşağıdaki gibi bir tip ile başladığımızı varsayalım:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Bu sınıftan belirli bir dizi özelliği seçerek bu sınıftan türetilmiş bir tip oluşturabiliriz:

```typescript
export class UpdateCatAgeDto extends PickType(CreateCatDto, ['age'] as const) {}
```

> info **İpucu** `PickType()` işlevi, `@nestjs/mapped-types` paketinden içe aktarılır.

`OmitType()` işlevi, bir giriş tipinden tüm özellikleri seçer ve ardından belirli bir anahtarı kaldırırak bir tip oluşturur. Örneğin, aşağıdaki gibi bir tip ile başladığımızı varsayalım:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Aşağıdaki gibi her özellik **hariç** `name`'i içeren türetilmiş bir tip oluşturabiliriz. Bu yapıda, `OmitType`'ın ikinci argümanı bir dizi özellik adıdır.

```typescript
export class UpdateCatDto extends OmitType(CreateCatDto, ['name'] as const) {}
```

> info **İpucu** `OmitType()` işlevi, `@nestjs/mapped-types` paketinden içe aktarılır.

`IntersectionType()` işlevi iki tipi yeni bir tip (sınıf) haline getirir. Örneğin, aşağıdaki gibi iki tip ile başladığımızı varsayalım:

```typescript
export class CreateCatDto {
  name: string;
  breed: string;
}

export class AdditionalCatInfo {
  color: string;
}
```

Bu iki tipin tüm özelliklerini birleştiren yeni bir tip oluşturan aşağıdaki gibi bir tip oluşturabiliriz.

```typescript
export class UpdateCatDto extends IntersectionType(
  CreateCatDto,
  AdditionalCatInfo,
) {}
```

> info **İpucu** `IntersectionType()` işlevi, `@nestjs/mapped-types` paketinden içe aktarılır.

Tip eşleme yardımcı işlevleri birbirine bağlanabilir. Örneğin, aşağıdaki, `CreateCatDto` tipinin `name` hariç tüm özelliklerine sahip olan ve bu özellikleri isteğe bağlı hale getiren bir tip (sınıf) oluşturacaktır:

```typescript
export class UpdateCatDto extends PartialType(
  OmitType(CreateCatDto, ['name'] as const),
) {}
```

#### Dizileri Ayrıştırma ve Doğrulama

TypeScript, genel olarak, genişletilebilir nesneler hakkında metadata saklamaz. Bu nedenle, DTO'larınızda genellemeleri veya arabirimleri kullandığınızda, `ValidationPipe` gelen verileri doğru bir şekilde doğrulayamayabilir. Örneğin, aşağıdaki kodda `createUserDtos` doğru bir şekilde doğrulanmaz:

```typescript
@Post()
createBulk(@Body() createUserDtos: CreateUserDto[]) {
  return 'Bu işlem yeni kullanıcılar ekler';
}
```

Diziyi doğrulamak için, diziyi saran bir özelliğe sahip olan ve bu özellikleri içeren bir sınıf oluşturun veya `ParseArrayPipe`'ı kullanın.

```typescript
@Post()
createBulk(
  @Body(new ParseArrayPipe({ items: CreateUserDto }))
  createUserDtos: CreateUserDto[],
) {
  return 'Bu işlem yeni kullanıcılar ekler';
}
```

Ayrıca, `ParseArrayPipe`, sorgu parametrelerini ayrıştırmak için kullanışlı olabilir. İd'leri sorgu parametreleri olarak iletilen bir `findByIds()` yöntemini düşünelim.

```typescript
@Get()
findByIds(
  @Query('ids', new ParseArrayPipe({ items: Number, separator: ',' }))
  ids: number[],
) {
  return 'Bu işlem, kimliklere göre kullanıcıları döndürür';
}
```

Bu yapım, bir HTTP `GET` isteğinden gelen sorgu parametrelerini aşağıdaki gibi doğrular:

```bash
GET /?ids=1,2,3
```

#### WebSockets ve Microservices

Bu bölüm, HTTP tarzı uygulamaları (örneğin, Express veya Fastify gibi) kullanarak örnekler gösterse de, `ValidationPipe`, ulaştırma yöntemi ne olursa olsun, WebSockets ve mikroservisler için aynı şekilde çalışır.

#### Daha Fazla Bilgi

`class-validator` paketi tarafından sağlanan özel doğrulayıcılar, hata mesajları ve mevcut dekoratörler hakkında daha fazla bilgi için [buraya](https://github.com/typestack/class-validator) göz atın.