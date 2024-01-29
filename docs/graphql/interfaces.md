### Arayüzler

GraphQL, birçok tür sistem gibi arayüzleri destekler. Bir **Arayüz**, bir türün arayüzü uygulamak için içermesi gereken belirli bir seti temsil eden soyut bir tiptir (daha fazla bilgi için [buraya](https://graphql.org/learn/schema/#interfaces) bakın).

#### Kod tabanlı yaklaşım

Kod tabanlı yaklaşım kullanılırken, `@nestjs/graphql` tarafından sağlanan `@InterfaceType()` dekoratörü ile işaretlenmiş soyut bir sınıf oluşturarak GraphQL arayüzünü tanımlarsınız.

```typescript
import { Field, ID, InterfaceType } from '@nestjs/graphql';

@InterfaceType()
export abstract class Character {
  @Field((type) => ID)
  id: string;

  @Field()
  name: string;
}
```

> **Uyarı** TypeScript arayüzleri GraphQL arayüzlerini tanımlamak için kullanılamaz.

Bu, SDL'de şu kısmı üretir:

```graphql
interface Character {
  id: ID!
  name: String!
}
```

Şimdi, `Character` arayüzünü uygulamak için `implements` anahtarını kullanabilirsiniz:

```typescript
@ObjectType({
  implements: () => [Character],
})
export class Human implements Character {
  id: string;
  name: string;
}
```

> ℹ️ **İpucu** `@ObjectType()` dekoratörü `@nestjs/graphql` paketinden gelir.

Varsayılan olarak, kütüphane tarafından oluşturulan `resolveType()` işlevi, çözümleyici metodundan dönen değere dayanarak türü çıkarır. Bu, sınıf örnekleri döndürmeniz gerektiği anlamına gelir (doğrudan JavaScript nesnelerini döndüremezsiniz).

Özelleştirilmiş bir `resolveType()` işlevi sağlamak için, `@InterfaceType()` dekoratörüne geçirilen seçenekler nesnesine `resolveType` özelliğini iletebilirsiniz:

```typescript
@InterfaceType({
  resolveType(book) {
    if (book.colors) {
      return ColoringBook;
    }
    return TextBook;
  },
})
export abstract class Book {
  @Field((type) => ID)
  id: string;

  @Field()
  title: string;
}
```

#### Arayüz çözücüler

Şimdiye kadar, arayüzleri kullanarak yalnızca nesne türleriyle alan tanımlamalarını paylaşabilirdiniz. Gerçek alan çözücü uygulamasını da paylaşmak istiyorsanız, şu şekilde özel bir arayüz çözücüsü oluşturabilirsiniz:

```typescript
import { Resolver, ResolveField, Parent, Info } from '@nestjs/graphql';

@Resolver(type => Character) // Hatırlatma: Character bir arayüzdür
export class CharacterInterfaceResolver {
  @ResolveField(() => [Character])
  friends(
    @Parent() character, // Character arayüzünü uygulayan çözülen nesne
    @Info() { parentType }, // Character arayüzünü uygulayan nesnenin türü
    @Args('search', { type: () => String }) searchTerm: string,
  ) {
    // Karakterin arkadaşlarını al
    return [];
  }
}
```

Artık `friends` alan çözücüsü, `Character` arayüzünü uygulayan tüm nesne türleri için otomatik olarak kaydedilir.

#### Şema tabanlı yaklaşım

Şema tabanlı yaklaşım kullanılırken, basitçe bir GraphQL arayüzü oluşturun ve SDL ile tanımlayın.

```graphql
interface Character {
  id: ID!
  name: String!
}
```

Daha sonra, ilgili TypeScript tanımlamalarını oluşturmak için [hızlı başlangıç](/docs/graphql/quick-start) bölümünde gösterildiği gibi tip oluşturma özelliğini kullanabilirsiniz:

```typescript
export interface Character {
  id: string;
  name: string;
}
```

Arayüzler, hangi türün çözülmesi gerektiğini belirlemek için çözücü haritasında ek bir `__resolveType` alanına ihtiyaç duyar. Şimdi, bir `CharactersResolver` sınıfı oluşturup `__resolveType` metodunu tanımlayalım:

```typescript
@Resolver('Character')
export class CharactersResolver {
  @ResolveField()
  __resolveType(value) {
    if ('age' in value) {
      return Person;
    }
    return null;
  }
}
```

> ℹ️ **İpucu** Tüm dekoratörler `@nestjs/graphql` paketinden gelmektedir.