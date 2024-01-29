### Türler ve Parametreler

`SwaggerModule`, API belgesini oluşturmak için route handler'larındaki `@Body()`, `@Query()` ve `@Param()` dekoratörlerini arar. Ayrıca, yansıma özelliklerinden faydalanarak karşılık gelen model tanımlamalarını oluşturur. Aşağıdaki kodu düşünün:

```typescript
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **İpucu** Vücut tanımını açıkça belirtmek için `@ApiBody()` dekoratörünü kullanın (`@nestjs/swagger` paketinden alınmıştır).

`CreateCatDto`'ya dayanarak, Swagger UI'de aşağıdaki model tanımı oluşturulacaktır:

<figure><img src="/assets/swagger-dto.png" /></figure>

Görebileceğiniz gibi, tanım boştur, ancak sınıfta birkaç deklare edilmiş özellik bulunmaktadır. Sınıf özelliklerini `SwaggerModule`'e görünür kılmak için özellikleri `@ApiProperty()` dekoratörü ile işaretlememiz veya bunu otomatik olarak yapacak CLI eklentisini kullanmamız gerekmektedir (daha fazlasını **Eklenti** bölümünde okuyun):

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

> info **İpucu** Her bir özelliği manuel olarak işaretleme yerine, bunu otomatik olarak sağlayacak olan Swagger eklentisini kullanmayı düşünün (bkz. [Eklenti](/docs/openapi/cli-plugin) bölümü).

Hadi tarayıcıyı açalım ve oluşturulan `CreateCatDto` modelini doğrulayalım:

<figure><img src="/assets/swagger-dto2.png" /></figure>

Ayrıca, `@ApiProperty()` dekoratörü çeşitli [Schema Object](https://swagger.io/specification/#schemaObject) özelliklerini belirlemeye olanak tanır:

```typescript
@ApiProperty({
  description: 'Bir kedinin yaşı',
  minimum: 1,
  default: 1,
})
age: number;
```

> info **İpucu** `{{"@ApiProperty({ required: false })"}}`'yi açıkça yazmak yerine, `@ApiPropertyOptional()` kısa dekoratörünü kullanabilirsiniz.

Özelliğin türünü açıkça belirtmek için `type` anahtarını kullanın:

```typescript
@ApiProperty({
  type: Number,
})
age: number;
```

#### Diziler

Özellik bir dizi olduğunda, dizinin türünü aşağıdaki gibi manuel olarak belirtmemiz gerekmektedir:

```typescript
@ApiProperty({ type: [String] })
names: string[];
```

> info **İpucu** Dizileri otomatik olarak algılayacak olan Swagger eklentisini kullanmayı düşünün (bkz. [Eklenti](/docs/openapi/cli-plugin) bölümü).

Türü dizinin ilk elemanı olarak dahil edin (yukarıda gösterildiği gibi) veya `isArray` özelliğini `true` olarak ayarlayın.

<app-banner-enterprise></app-banner-enterprise>

#### Dairesel Bağımlılıklar

Sınıflar arasında dairesel bağımlılıklarınız olduğunda, `SwaggerModule`'e tür bilgisi sağlamak için tembel bir işlev kullanın:

```typescript
@ApiProperty({ type: () => Node })
node: Node;
```

> info **İpucu** Dairesel bağımlılıkları otomatik olarak algılayacak olan Swagger eklentisini kullanmayı düşünün (bkz. [Eklenti](/docs/openapi/cli-plugin) bölümü).

#### Generikler ve Arayüzler

TypeScript, generikler veya arayüzler hakkında metadata depolamaz, bu nedenle bunları DTO'larınızda kullandığınızda, `SwaggerModule`'ün çalışma zamanında model tanımlamalarını doğru bir şekilde oluşturamayabilir. Örneğin, aşağıdaki kod Swagger modülü tarafından doğru bir şekilde incelenmeyecektir:

```typescript
createBulk(@Body() usersDto: CreateUserDto[])
```

Bu sınırlamayı aşmak için türü açıkça belirleyebilirsiniz:

```typescript
@ApiBody({ type: [CreateUserDto] })
createBulk(@Body() usersDto: CreateUserDto[])
```

#### Enumlar

Bir `enum` tanımlamak için, `@ApiProperty` üzerinde `enum` özelliğini, bir değerler dizisi içeren bir dizi olarak manuel olarak ayarlamamız gerekmektedir.

```typescript
@ApiProperty({ enum: ['Admin', 'Moderator', 'User']})
role: UserRole;
```

Alternatif olarak, aşağıdaki gibi gerçek bir TypeScript enum tanımlayabilirsiniz:

```typescript
export enum UserRole {
  Admin = 'Admin',
  Moderator = 'Moderator',
  User = 'User',
}
```

Daha sonra enum'u, `@Query()` parametre dekoratörü ile birlikte `@ApiQuery()` dekoratörü kullanarak doğrudan kullanabilirsiniz.

```typescript
@ApiQuery({ name: 'role', enum: UserRole })
async filterByRole(@Query('role') role: UserRole = UserRole.User) {}
```

<figure><img src="/assets/enum_query.gif" /></figure>

`isArray` **true** olarak ayarlandığında, `enum` bir **multi-select** olarak seçilebilir:

<figure><img src="/assets/enum_query_array.gif" /></figure>

#### Enumlar Şeması

Varsayılan olarak, `enum` özelliği `parameter` üzerinde [Enum](https://swagger.io/docs/specification/data-models/enums/) için bir raw tanım ekler.

```yaml
- breed:
    type: 'string'
    enum:
      - Persian
      - Tabby
      - Siamese
```

Yukarıdaki belgeleme çoğu durum için işe yarar. Ancak, belgelemeyi **giriş** olarak alan ve **istemci tarafı** kodu üreten bir araç kullanıyorsanız, üretilen kodun yinelemeli `enums` içermesi sorunuyla karşılaşabilirsiniz. Aşağıdaki kod örneğini düşünün:

```typescript
// üretilen istemci tarafı kodu
export class CatDetail {
  breed: CatDetailEnum;
}

export class CatInformation {
  breed: CatInformationEnum;
}

export enum CatDetailEnum {
  Persian = 'Persian',
  Tabby = 'Tabby',
  Siamese = 'Siamese',
}

export enum CatInformationEnum {
  Persian = 'Persian',
  Tabby = 'Tabby',
  Siamese = 'Siamese',
}
```

> info **İpucu** Yukarıdaki örnek, [NSwag](https://github.com/RicoSuter/NSwag) adlı bir araç kullanılarak üretilmiştir.

Görüldüğü gibi şimdi iki tamamen aynı `enum`'a sahipsiniz. Bu sorunu çözmek için dekoratörünüzde `enum` özelliği ile birlikte bir `enumName` iletebilirsiniz.

```typescript
export class CatDetail {
  @ApiProperty({ enum: CatBreed, enumName: 'CatBreed' })
  breed: CatBreed;
}
```

`enumName` özelliği, `@nestjs/swagger`'in `CatBreed`'i kendi `schema`'sına dönüştürmesine olanak tanır, bu da `CatBreed` enum'unun tekrar kullanılabilir hale gelmesini sağlar. Belgeleme şu şekilde görünecektir:

```yaml
CatDetail:
  type: 'object'
  properties:
    ...
    - breed:
        schema:
          $ref: '#/components/schemas/CatBreed'
CatBreed:
  type: string
  enum:
    - Persian
    - Tabby
    - Siamese
```

> info **İpucu** `enum`'u alan herhangi bir **dekoratör**, aynı zamanda `enumName` özelliğini de alacaktır.

#### Raw Tanımlamalar

Bazı özel senaryolarda (örneğin, derinlemesine gömülü diziler, matrisler) türünüzü elle tanımlamak isteyebilirsiniz.

```typescript
@ApiProperty({
  type: 'array',
  items: {
    type: 'array',
    items: {
      type: 'number',
    },
  },
})
coords: number[][];
```

Aynı şekilde, denetleyici sınıflarında giriş/çıkış içeriğinizi manuel olarak tanımlamak için `schema` özelliğini kullanın:

```typescript
@ApiBody({
  schema: {
    type: 'array',
    items: {
      type: 'array',
      items: {
        type: 'number',
      },
    },
  },
})
async create(@Body() coords: number[][]) {}
```

#### Ek Modeller

Kontrolcülerinizde doğrudan referans alınmayan ancak Swagger modülü tarafından incelenmesi gereken ek modelleri tanımlamak için `@ApiExtraModels()` dekoratörünü kullanın:

```typescript
@ApiExtraModels(ExtraModel)
export class CreateCatDto {}
```

> info **İpucu** Belirli bir model sınıfı için `@ApiExtraModels()`'u yalnızca bir kez kullanmanız yeterlidir.

Alternatif olarak, `SwaggerModule#createDocument()` yöntemine belirli `extraModels` özelliği belirtilmiş bir seçenek nesnesi geçebilirsiniz, aşağıdaki gibi:

```typescript
const document = SwaggerModule.createDocument(app, options, {
  extraModels: [ExtraModel],
});
```

Modelinize bir referans (`$ref`) almak için `getSchemaPath(ExtraModel)` fonksiyonunu kullanın:

```typescript
'application/vnd.api+json': {
   schema: { $ref: getSchemaPath(ExtraModel) },
},
```

#### oneOf, anyOf, allOf

Şemaları birleştirmek için `oneOf`, `anyOf` veya `allOf` anahtar kelimelerini kullanabilirsiniz ([daha fazlasını oku](https://swagger.io/docs/specification/data-models/oneof-anyof-allof-not/)).

```typescript
@ApiProperty({
  oneOf: [
    { $ref: getSchemaPath(Cat) },
    { $ref: getSchemaPath(Dog) },
  ],
})
pet: Cat | Dog;
```

Polimorfik bir dizi tanımlamak istiyorsanız (yani, üyeleri çoklu şemaları kapsayan bir dizi), tipinizi el ile tanımlamak için yukarıdaki örnekte gösterildiği gibi bir raw tanım kullanmalısınız.

```typescript
type Pet = Cat | Dog;

@ApiProperty({
  type: 'array',
  items: {
    oneOf: [
      { $ref: getSchemaPath(Cat) },
      { $ref: getSchemaPath(Dog) },
    ],
  },
})
pets: Pet[];
```

> info **İpucu** `getSchemaPath()` fonksiyonu `@nestjs/swagger` paketinden içe aktarılmıştır.

`Cat` ve `Dog` her ikisi de sınıf düzeyinde `@ApiExtraModels()` dekoratörü kullanılarak ek modeller olarak tanımlanmalıdır.