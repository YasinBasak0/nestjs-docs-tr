### Eşleştirilmiş Türler

> warning **Uyarı** Bu bölüm, yalnızca kod odaklı yaklaşım için geçerlidir.

CRUD (Create/Read/Update/Delete) gibi özellikleri oluştururken, genellikle temel bir varlık türü üzerinde değişiklikler yapmak faydalıdır. Nest, bu görevi daha pratik hale getirmek ve tekrarlanan kodu en aza indirmek için bazı yardımcı işlevler sağlar.

#### Partial

Giriş doğrulama türleri oluştururken (ayrıca Veri Transfer Objesi veya DTO olarak adlandırılır), aynı türde **oluşturma** ve **güncelleme** varyasyonları oluşturmak genellikle faydalıdır. Örneğin, **oluşturma** varyantı tüm alanları gerektirebilirken, **güncelleme** varyantı her alanı isteğe bağlı yapabilir.

Nest, bu görevi kolaylaştırmak ve tekrarlanan kodu en aza indirmek için `PartialType()` yardımcı işlevini sağlar.

`PartialType()` işlevi, giriş türündeki tüm özellikleri isteğe bağlı olarak ayarlanmış bir tür (sınıf) döndürür. Örneğin, aşağıdaki gibi bir **oluşturma** türümüz olduğunu varsayalım:

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

Varsayılan olarak, bu alanların hepsi zorunludur. Aynı alanlara sahip ancak her biri isteğe bağlı olan bir tür oluşturmak için, `PartialType()`'ı kullanın ve birinci argüman olarak sınıf referansını (`CreateUserInput`) geçirin:

```typescript
@InputType()
export class UpdateUserInput extends PartialType(CreateUserInput) {}
```

> info **İpucu** `PartialType()` fonksiyonu, `@nestjs/graphql` paketinden içe aktarılır.

`PartialType()` fonksiyonu, ikinci bir argüman olarak bir dekoratör fabrikası referansı alan isteğe bağlı bir ikinci argüman alır. Bu argüman, (çocuk) sınıfa uygulanan dekoratör işlevini değiştirmek için kullanılabilir. Belirtilmezse, çocuk sınıf, etkili olarak birinci argümandaki **ebeveyn** sınıf ile aynı dekoratörü kullanır (ilk argümanda referans alınan sınıf). Yukarıdaki örnekte, `@InputType()` ile işaretlenmiş `CreateUserInput`'i genişletiyoruz. `UpdateUserInput`'in de sanki `@InputType()` ile işaretlenmiş gibi işlem görmesini istediğimizden, ikinci argüman olarak `InputType`'ı geçirmemize gerek yoktu. Eğer ana ve alt türler farklıysa (örneğin, ana tür `@ObjectType` ile işaretlenmişse), ikinci argüman olarak `InputType`'ı geçirebilirdik. Örneğin:

```typescript
@InputType()
export class UpdateUserInput extends PartialType(User, InputType) {}
```

#### Pick

`PickType()` fonksiyonu, bir giriş türünden belirli bir özellik kümesini seçerek yeni bir tür (sınıf) oluşturur. Örneğin, aşağıdaki gibi bir türle başlıyorsak:

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

`PickType()` yardımcı işlevini kullanarak bu sınıftan bir özellik kümesi seçebiliriz:

```typescript
@InputType()
export class UpdateEmailInput extends PickType(CreateUserInput, [
  'email',
] as const) {}
```

> info **İpucu** `PickType()` fonksiyonu, `@nestjs/graphql` paketinden içe aktarılır.

#### Omit

`OmitType()` fonksiyonu, bir giriş türünden tüm özellikleri seçer ve ardından belirli bir anahtar kümesini kaldırarak bir tür oluşturur. Örneğin, aşağıdaki gibi bir türle başlıyorsak:

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

Aşağıdaki gibi, bu sınıftan `email` dışındaki tüm özelliklere sahip yeni bir tür oluşturabiliriz. Bu oluşturmada, `OmitType`'ın ikinci argümanı bir özellik adları dizisidir.

```typescript
@InputType()
export class UpdateUserInput extends OmitType(CreateUserInput, [
  'email',
] as const) {}
```

> info **İpucu** `OmitType()` fonksiyonu, `@nestjs/graphql` paketinden içe aktarılır.

#### Kesişim (Intersection)

`IntersectionType()` fonksiyonu, iki türü birleştirerek yeni bir tür (sınıf) oluşturur. Örneğin, aşağıdaki gibi iki türle başlıyorsak:

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;
}

@ObjectType()
export class AdditionalUserInfo {
  @Field()
  firstName: string;

  @Field()
  lastName: string;
}
```

Her iki türdeki tüm özellikleri birleştiren yeni bir tür oluşturabiliriz.

```typescript
@InputType()
export class UpdateUserInput extends IntersectionType(
  CreateUserInput,
  AdditionalUserInfo,
) {}
```

> info **İpucu** `IntersectionType()` fonksiyonu, `@nestjs/graphql` paketinden içe aktarılır

.

#### Kompozisyon

Tür eşleme yardımcı işlevleri birbirine eklenabilir. Örneğin, aşağıdaki, `CreateUserInput` türünün tüm özelliklerine sahip ancak `email` hariç, ve bu özellikler isteğe bağlı olarak ayarlanmış bir tür (sınıf) üretecektir:

```typescript
@InputType()
export class UpdateUserInput extends PartialType(
  OmitType(CreateUserInput, ['email'] as const),
) {}
```