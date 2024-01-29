---
sidebar_position: 14
---

### Borular (Pipes)

Bir boru, `@Injectable()` dekoratörü ile işaretlenmiş ve `PipeTransform` arabirimini uygulayan bir sınıftır.

<figure>
  <img src="/assets/Pipe_1.png" />
</figure>

Boruların iki yaygın kullanım alanı vardır:

- **dönüşüm**: giriş verilerini istenen biçime dönüştürme (örneğin, dizeden tamsayıya)
- **doğrulama**: giriş verilerini değerlendirir ve geçerliyse değiştirilmeden geçirir; aksi takdirde bir istisna fırlatır

Her iki durumda da, borular bir <a href="controllers#route-parameters">denetleyici rota işleyicisi</a> tarafından işlenen `arguments` üzerinde çalışır. Nest, bir yöntem çağrılmadan hemen önce bir boru ekler ve boru, yöntem için amaçlanan argümanları alır ve üzerlerinde çalışır. Herhangi bir dönüşüm veya doğrulama işlemi bu sırada gerçekleşir, ardından rota işleyicisi, (potansiyel olarak) dönüştürülmüş argümanlarla çağrılır.

Nest, kullanıma hazır olan bir dizi yerleşik boru ile birlikte gelir. Ayrıca kendi özel borularınızı oluşturabilirsiniz. Bu bölümde, yerleşik boruları tanıtacak ve onları rota işleyicilerine nasıl bağlayacağınızı göstereceğiz. Daha sonra, sıfırdan bir boru nasıl oluşturulacağını göstermek için birkaç özel oluşturulmuş boruyu inceleyeceğiz.

> info **İpucu** Borular, istisnalar bölgesinde çalışır. Bu, bir Boru'nun bir istisna fırlatması durumunda, istisnalar katmanı tarafından (genel istisna filtresi ve geçerli bağlam için uygulanan [istisna filtreleri](/docs/exception-filters)) ele alındığı anlamına gelir. Yukarıdakilerden de anlaşılacağı gibi, Bir Boru'da bir istisna fırlatıldığında bir denetleyici yöntemi ardından çalıştırılmaz. Bu, uygulamaya dış kaynaklardan gelen verileri sistemin sınırında doğrulamak için bir en iyi uygulama tekniği sunar.

#### Yerleşik Borular

Nest, kullanıma hazır dokuz boru ile birlikte gelir:

- `ValidationPipe`
- `ParseIntPipe`
- `ParseFloatPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `ParseEnumPipe`
- `DefaultValuePipe`
- `ParseFilePipe`

Bunlar `@nestjs/common` paketinden içe aktarılır.

Hızlıca `ParseIntPipe` kullanımına bir göz atalım. Bu, **dönüşüm** kullanım durumunun bir örneğidir, burada boru, bir yöntem işleyici parametresinin bir JavaScript tamsayısına dönüştürülmesini sağlar (veya dönüştürme başarısız olursa bir istisna fırlatır). Bu bölümün ilerleyen kısımlarında, `ParseIntPipe` için basit bir özel uygulamayı göstereceğiz. Aşağıdaki örnek teknikleri, diğer yerleşik dönüşüm boruları (`ParseBoolPipe`, `ParseFloatPipe`, `ParseEnumPipe`, `ParseArrayPipe` ve `ParseUUIDPipe`, bu bölümde `Parse*` boruları olarak adlandıracağımız) için de geçerlidir.

#### Boruları Bağlama

Bir boruyu kullanmak için, boru sınıfının bir örneğini uygun bağlamla ilişkilendirmemiz gerekir. `ParseIntPipe` örneğimizde, bir boruyu belirli bir rota işleyici yöntemiyle ilişkilendirmek ve yöntem çağrılmadan önce çalışmasını sağlamak istiyoruz. Bunu, boruyu yöntem parametre seviyesinde bağlamak olarak adlandıracağımız aşağıdaki yapı ile yaparız:

```typescript
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```



Bu, `findOne()` yöntemine gelen parametrenin bir sayı olması (bu, `this.catsService.findOne()` çağrımızda beklenen) veya rota işleyicisi çağrılmadan önce bir istisna fırlatıldığı iki durumdan birinin doğru olduğunu sağlar.

Örneğin, rota şu şekilde çağrılırsa:

```bash
GET localhost:3000/abc
```

Nest, şu gibi bir istisna fırlatacaktır:

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

Bu istisna, `findOne()` yönteminin içeriğinin çalışmasını engelleyecektir.

Yukarıdaki örnekte bir sınıf (`ParseIntPipe`) geçiyoruz, bir örnek değil, örneğin oluşturma sorumluluğunu çerçeveye bırakarak ve bağımlılık enjeksiyonunu etkinleştirerek. Borunun ve koruyucuların yanı sıra, yerine yerinde bir örnek de geçebiliriz. Bir yerinde örnek geçmek, yerleşik borunun davranışını seçenekleri ile özelleştirmek istiyorsak kullanışlıdır:

```typescript
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

Diğer dönüşüm borularını bağlamak (tüm **Parse\*** boruları) benzer şekilde çalışır. Bu borular, rota parametrelerini, sorgu dizesi parametrelerini ve istek gövde değerlerini doğrulamanın bağlamında çalışırlar.

Örneğin, bir sorgu dizesi parametresiyle:

```typescript
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

İşte bir dize parametresini ayrıştırmak ve bunun bir UUID olup olmadığını doğrulamak için `ParseUUIDPipe` kullanma örneği:

```typescript
@@filename()
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
@@switch
@Get(':uuid')
@Bind(Param('uuid', new ParseUUIDPipe()))
async findOne(uuid) {
  return this.catsService.findOne(uuid);
}
```

> info **İpucu** `ParseUUIDPipe()` kullanırken, UUID'nin 3, 4 veya 5 sürümlerini ayrıştırıyorsunuzdur, yalnızca belirli bir UUID sürümünü gerekiyorsa bir sürümü boru seçenekleri ile iletebilirsiniz.

Yukarıda, çeşitli `Parse*` ailesi yerleşik borularını bağlama örneklerini gördük. Doğrulama borularını bağlamak biraz farklıdır; bunu bir sonraki bölümde tartışacağız.

> info **İpucu** Ayrıca, doğrulama borularıyla ilgili geniş kapsamlı örnekleri görmek için [Doğrulama Teknikleri](/docs/techniques/validation) bölümüne bakın.

#### Özel Borular

Belirtildiği gibi, kendi özel borularınızı oluşturabilirsiniz. Nest, sağlam bir yerleşik `ParseIntPipe` ve `ValidationPipe` sağlarken, her birini sıfırdan nasıl oluşturacağımızı görmek için bunların basit özel sürümlerini oluşturalım.

İlk olarak, basit bir `ValidationPipe` ile başlayalım. Başlangıçta, giriş değerini alacak ve hemen aynı değeri döndürecek şekilde davranacak bir kimlik işlevi gibi davranacaktır.

```typescript
@@filename(validation.pipe)
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class ValidationPipe {
  transform(value, metadata) {
    return value;
  }
}
```

> info **İpucu** `PipeTransform<T, R>` herhangi bir boru tarafından uygulanması gereken bir genel arabirimdir. Genel arabirim, giriş `value`'nun türünü belirtmek için `T`'yi ve `transform()` yönteminin dönüş türünü belirtmek için `R`'yi kullanır.

Her bir boru, `transform()` yöntemini `PipeTransform` arabirim sözleşmesini yerine getirmek için uygulamalıdır. Bu yöntemin iki parametresi vardır:

- `value`
- `metadata`

`value` parametresi, şu anda işlenmekte olan yöntem argümanıdır (rota işleyici yöntemi tarafından alınmadan önce), ve `metadata` şu anda işlenmekte olan yöntem argümanının metadata'sıdır. Metadata nesnesi şu özelliklere sahiptir:

```typescript
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

Bu özellikler, şu anda işlenen argümanı tanımlar.

<table>
  <tr>
    <td>
      <code>type</code>
    </td>
    <td>Argümanın bir gövde
      <code>@Body()</code>, sorgu
      <code>@Query()</code>, parametre
      <code>@Param()</code> veya özel bir parametre olup olmadığını belirtir (daha fazla bilgi için
      <a routerLink="/custom-decorators">buraya</a> bakın).</td>
  </tr>
  <tr>
    <td>
      <code>metatype</code>
    </td>
    <td>
      Argümanın metatürünü sağlar, örneğin,
      <code>String</code>. Not: eğer rota işleyici yöntem imzasında bir tür bildirimi atlarsanız veya düz JavaScript kullanırsanız, değeri
      <code>undefined</code>'dir.
    </td>
  </tr>
  <tr>
    <td>
      <code>data</code>
    </td>
    <td>Dekoratöre iletilen dize, örneğin
      <code>@Body('string')</code>. Parantezi boş bırakırsanız,
      <code>undefined</code>'dir.</td>
  </tr>
</table>

> warning **Uyarı** TypeScript arayüzleri transpile sırasında kaybolur. Bu nedenle, bir yöntem parametresinin türü bir sınıf yerine bir arayüz olarak bildirilirse, `metatype` değeri `Object` olacaktır.

#### Şema Tabanlı Doğrulama

Doğrulama borumuzu biraz daha kullanışlı hale getirelim. `CatsController`'ın `create()` yöntemine daha yakından bakalım, burada servis yöntemimizi çalıştırmadan önce post gövdesi nesnesinin geçerli olmasını sağlamak istiyoruz.

```typescript
@@filename()
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
async create(@Body() createCatDto) {
  this.catsService.create(createCatDto);
}
```

`createCatDto` gövde parametresine odaklanalım. Türü `CreateCatDto`:

```typescript
@@filename(create-cat.dto)
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

`create` yöntemine gelen herhangi bir isteğin geçerli bir gövde içermesini istiyoruz. Bu nedenle, `createCatDto` nesnesinin üç üyesini doğrulamamız gerekiyor. Bu işlemi rota işleyici yöntemi içinde yapabilirdik, ancak bu, **tek sorumluluk ilkesini** (SRP) ihlal ederdi.

Başka bir yaklaşım, bir **doğrulayıcı sınıf** oluşturmak ve görevi buraya devretmek olabilir. Bu, bu doğrulayıcıyı her yöntemin başında çağırmayı hatırlamamız gerektiği dezavantajına sahiptir.

Doğrulama ara katmanı oluşturmak nasıl olurdu? Bu çalışabilir, ancak ne yazık ki, uygulamanın tüm bağlamlarında kullanılabilecek **genel bir ara katman oluşturmak mümkün değildir**. Bu, ara katmanın **yürütme bağlamından** (handler'ın çağrılacak ve herhangi bir parametresinin bilincinde olacak) haberdar olmamasından kaynaklanmaktadır.

Tabii ki, boruların tasarlandığı tam olarak bu kullanım durumu budur. Bu nedenle doğrulama borumuzu daha da geliştirelim.

<app-banner-courses></app-banner-courses>

#### Nesne Şeması Doğrulama

Temiz, [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) bir şekilde nesne doğrulama için birkaç yaklaşım bulunmaktadır. Bu konuda yaygın bir yaklaşım, **şema tabanlı** doğrulamadır. Hadi bu yaklaşımı deneyelim.

[Zod](https://zod.dev/) kütüphanesi, okunabilir bir API ile şemalar oluşturmanıza olanak tanır. Zod tabanlı şemaları kullanan bir doğrulama pipe'ı oluşturalım.

İlk olarak gerekli paketi yükleyerek başlayalım:

```bash
$ npm install --save zod
```

Aşağıdaki kod örneğinde, bir şemayı `constructor` argümanı olarak alan basit bir sınıf oluşturuyoruz. Ardından, gelen argümanı sağlanan şemaya karşı doğrularız, `schema.parse()` yöntemini kullanırız.

Daha önce belirtildiği gibi, bir **doğrulama pipe'ı**, ya değeri değiştirmeden geri döndürür ya da bir istisna fırlatır.

Sonraki bölümde, bir denetleyici yöntemi için uygun şemayı `@UsePipes()` dekoratörünü kullanarak nasıl sağladığımızı göreceksiniz. Bunu yapmak, doğrulama pipe'ımızı bağlam bağlam olacak şekilde kullanmamıza olanak tanır, tam olarak amaçladığımız gibi.

```typescript
@@filename()
import { PipeTransform, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ZodSchema  } from 'zod';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Doğrulama başarısız');
    }
  }
}
@@switch
import { BadRequestException } from '@nestjs/common';

export class ZodValidationPipe {
  constructor(private schema) {}

  transform(value, metadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Doğrulama başarısız');
    }
  }
}

```

#### Bağlama Doğrulama Pipe'ları

Daha önce, dönüşüm pipe'larını (`ParseIntPipe` ve diğer `Parse*` pipe'ları gibi) nasıl bağlayacağımızı görmüştük.

Doğrulama pipe'larını bağlamak da çok basittir.

Bu durumda, pipe'ı yöntem çağrı seviyesinde bağlamak istiyoruz. Mevcut örneğimizde `ZodValidationPipe`'ı kullanmak için aşağıdakileri yapmamız gerekiyor:

1. `ZodValidationPipe`'ın bir örneğini oluşturun
2. Şema bağlamak için pipe'ın sınıf yapısına bağlamayı yapın
3. Pipe'ı yönteme bağlayın

Zod şema örneği:

```typescript
import { z } from 'zod';

export const createCatSchema = z
  .object({
    name: z.string(),
    age: z.number(),
    breed: z.string(),
  })
  .required();

export type CreateCatDto = z.infer<typeof createCatSchema>;
```

Bunu, aşağıda gösterildiği gibi `@UsePipes()` dekoratörünü kullanarak yaparız:

```typescript
@@filename(cats.controller)
@Post()
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Bind(Body())
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **Hint** `@UsePipes()` dekoratörü, `@nestjs/common` paketinden alınır.

> warning **Warning** `zod` kütüphanesi, `tsconfig.json` dosyanızda `strictNullChecks` konfigürasyonunun etkinleştirilmesini gerektirir.

#### Class Validator

> warning **Uyarı** Bu bölümdeki teknikler TypeScript gerektirir ve uygulamanız saf JavaScript kullanılarak yazılmışsa kullanılamaz.

Doğrulama tekniğimiz için alternatif bir uygulamaya göz atalım.

Nest, [class-validator](https://github.com/typestack/class-validator) kütüphanesi ile iyi çalışır. Bu güçlü kütüphane, dekoratör tabanlı doğrulama kullanmanıza olanak tanır. Dekoratör tabanlı doğrulama, özellikle Nest'in **Pipe** yetenekleri ile birleştirildiğinde son derece güçlüdür, çünkü işlenen özelliğin `metatype`'ine erişim sağlarız. Başlamadan önce gerekli paketleri yüklememiz gerekiyor:

```bash
$ npm i --save class-validator class-transformer
```

Bunlar yüklendikten sonra, `CreateCatDto` sınıfına birkaç dekoratör ekleyebiliriz. Bu teknikteki önemli avantajlardan birini burada görüyoruz: `CreateCatDto` sınıfı, Post body nesnemiz için (ayrı bir doğrulama sınıfı oluşturmak zorunda kalmadan) tek doğru kaynak olarak kalır.

```typescript
@@filename(create-cat.dto)
import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

> info **Hint** class-validator dekoratörleri hakkında daha fazla bilgi için [buraya](https://github.com/typestack/class-validator#usage) bakın.

Şimdi bu dekoratörleri kullanan bir `ValidationPipe` sınıfı oluşturabiliriz.

```typescript
@@filename(validation.pipe)
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

> info **Hint** Hatırlatma olarak, özel bir doğrulama pipe'ı oluşturmanıza gerek yoktur çünkü `ValidationPipe`, Nest tarafından hazır bir şekilde sağlanmaktadır. Dahili `ValidationPipe`, bu bölümde oluşturduğumuz örneğe göre daha fazla seçenek sunar ve özel olarak oluşturulan bir pipe'ın mekanizmasını açıklamak amacıyla temel tutulmuştur. Tam ayrıntıları ve birçok örneği [burada](/docs/techniques/validation) bulabilirsiniz.

> warning **Notice** Yukarıda [class-transformer](https://github.com/typestack/class-transformer) kütüphanesini kullandık ve bu kütüphane, **class-validator** kütüphanesinin aynı yazarı tarafından yapıldığı için birbirleriyle çok iyi çalışır.

Bu kodu gözden geçirelim. İlk olarak, `transform()` methodunun `async` olarak işaretlendiğine dikkat edin. Bu, Nest'in hem senkron hem de **asenkron** pipe'ları desteklediği anlamına gelir. Bu methodu `async` yapabiliyoruz çünkü [class-validator doğrulamalarının bazıları async olabilir](https://github.com/typestack/class-validator#custom-validation-classes) (Promiseleri kullanır).

Sonraki adımda, metatype alanını (bir `ArgumentMetadata`'den sadece bu üyeyi çıkarmak) `metatype` parametresine atamak için yıkıcı atamayı kullanıyoruz. Bu, sadece tam `ArgumentMetadata`'yi elde etmek ve ardından metatype değişkenine ek bir atama yapmak yerine yapılmış bir kısaltmadır.

Sonraki adımda, `toValidate()` adlı yardımcı fonksiyonumuza dikkat edin. Bu fonksiyon, işlenen argümanın doğal JavaScript türü olduğunda (bu türlerin doğrulama dekoratörleri eklenemez, bu nedenle bunları doğrulama adımından geçirmenin bir anlamı yoktur) doğrulama adımını atlamaktan sorumludur.

Sonraki olarak, class-transformer fonksiyonu `plainToInstance()`'yi kullanarak basit JavaScript argüman nesnemizi tip belirtilmiş bir nesneye dönüştürüyoruz, böylece doğrulama uygulayabiliriz. Bunun nedenini şöyle açıklayabiliriz: ağ isteğinden deserialize edilen gelen post body nesnesinin **herhangi bir tür bilgisi yoktur** (bu, Express gibi altta yatan platformun çalışma şeklidir). Class-validator, DTO'muz için önceden tanımladığımız doğrulama dekoratörlerini kullanmak istiyor, bu nedenle bu dönüşümü yapmamız gerekiyor, böylece gelen body'yi sadece düz bir vanilya nesne olarak değil, uygun şekilde dekore edilmiş bir nesne olarak işleyebiliriz.

Son olarak, yukarıda belirtildiği gibi, bu bir **doğrulama pipe'ı** olduğundan ya değeri değişmeden döndürür ya da bir istisna fırlatır.

Son adım, `ValidationPipe`'ı bağlamaktır. Pipe'lar parametre bazlı, yöntem bazlı, denetleyici bazlı veya genel bazlı olabilir. D

aha önce, Zod tabanlı doğrulama pipe'ımızda, pipe'ı yöntem seviyesinde bağlama örneğini gördük.
Aşağıdaki örnekte, pipe örneğini route handler `@Body()` dekoratörüne bağlıyoruz, böylece pipe'ımız post body'yi doğrulamak için çağrılır.

```typescript
@@filename(cats.controller)
@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

Parametre bazlı pipe'lar, doğrulama mantığı yalnızca belirtilen bir parametre ile ilgili olduğunda faydalıdır.

#### Global Scoped Pipes

`ValidationPipe`'i mümkün olduğunca genel amaçlı olacak şekilde oluşturduğumuzdan, bu pipe'ı tüm uygulama genelinde her route handler'a uygulanacak şekilde **global-scoped** bir pipe olarak yapılandırabiliriz.

```typescript
@@filename(main)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

> warning **Notice** <a href="faq/hybrid-application">Hybrid uygulama</a> durumunda, `useGlobalPipes()` yöntemi gateway'ler ve mikro servisler için boruları kurmaz. "Standart" (hybrid olmayan) mikroservis uygulamaları için, `useGlobalPipes()` boruları genel olarak bağlar.

Global borular, tüm uygulama genelinde, her denetleyici ve her route handler için kullanılır.

Dikkat edilmesi gereken bir diğer önemli nokta, dış bir modül içinde (yukarıdaki örnekte olduğu gibi) kaydedilen global boruların bağımlılık enjeksiyonu açısından bağlam dışında yapıldığıdır. Bu sorunu çözmek için, şu yapının kullanıldığı bir modül içinden global bir pipe **doğrudan kurabilirsiniz**:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

> info **Hint** Pipe için bağımlılık enjeksiyonunu gerçekleştirmek için bu yaklaşımı kullanırken, bu yapının kullanıldığı modülün her durumda global olduğuna dikkat edin (`ValidationPipe` yukarıdaki örnekte olduğu gibi). Ayrıca, `useClass`, özel bir sağlayıcı kaydıyla başa çıkmanın tek yolu değildir. Daha fazla bilgi için [buraya](/docs/fundamentals/custom-providers) bakın.

#### Dahili ValidationPipe

Hatırlatma olarak, `ValidationPipe`'i kendi başınıza oluşturmanıza gerek yoktur, çünkü Nest, bunu kutudan çıkan bir özellik olarak sağlar. Dahili `ValidationPipe`, bu bölümde oluşturduğumuz örnekten daha fazla seçenek sunar ve özel bir pipe'ın mekaniklerini açıklamak amacıyla temel tutulmuştur. Tam detayları ve birçok örneği [burada](/docs/techniques/validation) bulabilirsiniz.

#### Dönüşüm Kullanım Durumu

Özel boruların sadece bir kullanım durumu doğrulama değildir. Bu bölümün başında, bir borunun aynı zamanda giriş verilerini istenen formata **dönüştürebileceğini** belirttik. Bu, `transform` fonksiyonundan dönen değerin, argümanın önceki değerini tamamen geçersiz kılabilmesi nedeniyle mümkündür.

Bu ne zaman faydalıdır? Bazı durumlarda, istemciden gelen verilerin, örneğin bir dizeyi bir tamsayıya dönüştürme gibi, işlenebilmesi için önce bir değişiklik geçirmesi gerekebilir. Ayrıca, bazı zorunlu veri alanları eksik olabilir ve varsayılan değerleri uygulamak isteriz. **Dönüşüm boruları**, istemci isteği ile istek işleyici arasına bir işleme işlevi sokarak bu işlevleri yerine getirebilir.

İşte bir dizeyi bir tamsayı değerine çevirmekten sorumlu olan basit bir `ParseIntPipe` örneği (Yukarıda belirtildiği gibi, Nest'in daha karmaşık bir yerleşik `ParseIntPipe`'ı bulunmaktadır; bunu özel bir dönüşüm borusunun basit bir örneği olarak dahil ediyoruz).

```typescript
@@filename(parse-int.pipe)
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
@@switch
import { Injectable, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe {
  transform(value, metadata) {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

Daha sonra bu borusu aşağıdaki gibi seçilen parametre ile bağlayabiliriz:

```typescript
@@filename()
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id) {
  return this.catsService.findOne(id);
}
@@switch
@Get(':id')
@Bind(Param('id', new ParseIntPipe()))
async findOne(id) {
  return this.catsService.findOne(id);
}
```

Başka bir kullanışlı dönüşüm durumu, istekte sağlanan bir id kullanarak veritabanından **mevcut bir kullanıcı** varlığını seçmek olacaktır:

```typescript
@@filename()
@Get(':id')
findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
  return userEntity;
}
@@switch
@Get(':id')
@Bind(Param('id', UserByIdPipe))
findOne(userEntity) {
  return userEntity;
}
```

Bu borusunun uygulanışını okuyucuya bırakıyoruz, ancak tıpkı diğer dönüşüm boruları gibi, bir giriş değeri (bir `id`) alır ve bir çıkış değeri (bir `UserEntity` nesnesi) döndürür. Bu, kodunuzu daha bildirimsel ve [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) yapabilir, tekrarlayan kodu işleyicinizden çıkararak ortak bir boruya geçirebilirsiniz.

#### Varsayılan Değerlerin Sağlanması

`Parse*` boruları, bir parametrenin değerinin tanımlanmış olmasını bekler. `null` veya `undefined` değerleri aldığında bir istisna fırlatırlar. Bir uç noktanın eksik sorgu dizisi parametre değerlerini ele almasına izin vermek için, bu değerlere işlem yapmadan önce bir varsayılan değer sağlamamız gerekiyor. Bu amaçla `DefaultValuePipe` kullanılır. İlgili `Parse*` borusundan önce `@Query()` dekoratöründe `DefaultValuePipe` örneğini basitçe instantiate edin, aşağıdaki gibi:

```typescript
@@filename()
@Get()
async findAll(
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
) {
  return this.catsService.findAll({ activeOnly, page });
}
```