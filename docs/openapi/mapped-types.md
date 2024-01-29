### Eşleştirilmiş Tipler

**CRUD** (Oluştur/Oku/Güncelle/Sil) gibi özellikleri oluştururken, genellikle temel bir varlık türü üzerinde varyantlar oluşturmak faydalıdır. Nest, bu görevi daha uygun hale getirmek için tip dönüşümleri gerçekleştiren birkaç yardımcı işlev sağlar.

#### Partial

Giriş doğrulama tipleri oluştururken (DTO'lar olarak da adlandırılır), aynı tür üzerinde **oluştur** ve **güncelle** varyasyonlarını oluşturmak sıkça kullanılır. Örneğin, **oluştur** varyantı tüm alanları gerektirebilir, ancak **güncelle** varyantı her alanı isteğe bağlı yapabilir.

Nest, bu görevi kolaylaştırmak ve tekrarlayan kodu en aza indirmek için `PartialType()` yardımcı işlevini sağlar.

`PartialType()` işlevi, giriş türündeki tüm özellikleri isteğe bağlı olarak ayarlanmış bir tür (sınıf) döndürür. Örneğin, aşağıdaki gibi bir **oluştur** türümüz olduğunu varsayalım:

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

Varsayılan olarak, bu alanların hepsi zorunludur. Aynı alanlara sahip, ancak her birini isteğe bağlı hale getiren bir tür oluşturmak için `PartialType()`'ı kullanabiliriz ve sınıf referansını (`CreateCatDto`) bir argüman olarak geçirebiliriz:

```typescript
export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

> info **Hint** `PartialType()` işlevi, `@nestjs/swagger` paketinden içe aktarılır.

#### Pick

`PickType()` işlevi, bir giriş türünden belirli bir özellik kümesini seçerek yeni bir tür (sınıf) oluşturur. Örneğin, aşağıdaki gibi bir türle başlıyorsak:

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

Bu sınıftan belirli bir özellik kümesini seçmek için `PickType()` yardımcı işlevini kullanabiliriz:

```typescript
export class UpdateCatAgeDto extends PickType(CreateCatDto, ['age'] as const) {}
```

> info **Hint** `PickType()` işlevi, `@nestjs/swagger` paketinden içe aktarılır.

#### Omit

`OmitType()` işlevi, bir giriş türünden tüm özellikleri seçer ve belirli bir anahtarı kaldırarak bir tür oluşturur. Örneğin, aşağıdaki gibi bir türle başlıyorsak:

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

Aşağıda gösterildiği gibi `name` dışında her özelliğe sahip bir türe sahip türe sahip bir türe sahip bir türe sahip bir türe sahip bir türe sahip bir tür oluşturabiliriz. Bu yapıda, `OmitType`'a ikinci argüman, özellik adlarının bir dizisidir.

```typescript
export class UpdateCatDto extends OmitType(CreateCatDto, ['name'] as const) {}
```

> info **Hint** `OmitType()` işlevi, `@nestjs/swagger` paketinden içe aktarılır.

#### Intersection

`IntersectionType()` işlevi iki türü tek bir yeni tür (sınıf) olarak birleştirir. Örneğin, aşağıdaki gibi iki türle başlarsak:

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  breed: string;
}

export class AdditionalCatInfo {
  @ApiProperty()
  color: string;
}
```

Her iki türdeki tüm özellikleri birleştiren yeni bir tür oluşturabiliriz.

```typescript
export class UpdateCatDto extends IntersectionType(
  CreateCatDto,
  AdditionalCatInfo,
) {}
```

> info **Hint** `IntersectionType()` işlevi, `@nestjs/swagger` paketinden içe aktarılır.

#### Kompozisyon

Tür eşleme yardımcı işlevleri birbirine eklenebilir. Örneğin, aşağıdaki, `CreateCatDto` türünün tüm özelliklerine sahip ancak `name` hariç tüm özellikleri isteğe bağlı olarak ayarlanmış bir tür (sınıf) üretecektir:

```typescript
export class UpdateCatDto extends PartialType(
  OmitType(CreateCatDto, ['name'] as const),
) {}
```