### Birleşim Tipleri (Unions)

Birleşim tipleri, arayüzlerle çok benzerdir, ancak tipler arasında ortak alanları belirleme yetenekleri yoktur (daha fazla bilgi için [buraya bakın](https://graphql.org/learn/schema/#union-types)). Birleşim tipleri, tek bir alanından ayrı data tiplerini döndürmek için kullanışlıdır.

#### Code first

Bir GraphQL birleşim tipi tanımlamak için, bu birleşimin oluştuğu sınıfları tanımlamamız gerekir. Apollo belgelerindeki [örnek](https://www.apollographql.com/docs/apollo-server/schema/unions-interfaces/#union-type) takip edilerek, önce `Book` sınıfını oluşturacağız:

```typescript
import { Field, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Book {
  @Field()
  title: string;
}
```

Ve ardından `Author` sınıfını:

```typescript
import { Field, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Author {
  @Field()
  name: string;
}
```

Bu yapılandırmayla, `@nestjs/graphql` paketinden ihraç edilen `createUnionType` fonksiyonunu kullanarak `ResultUnion` birleşimini kaydedin:

```typescript
export const ResultUnion = createUnionType({
  name: 'ResultUnion',
  types: () => [Author, Book] as const,
});
```

> warning **Uyarı** `createUnionType` fonksiyonunun `types` özelliği tarafından döndürülen diziye bir `const` assertion eklenmelidir. Const assertion eklenmezse, derleme zamanında yanlış bir bildirim dosyası oluşturulur ve başka bir projeden kullanılırken bir hata oluşur.

Şimdi, sorgumuzda `ResultUnion`'ı referans alabiliriz:

```typescript
@Query(returns => [ResultUnion])
search(): Array<typeof ResultUnion> {
  return [new Author(), new Book()];
}
```

Bu, SDL'de şu bölümün oluşturulmasına neden olacaktır:

```graphql
type Author {
  name: String!
}

type Book {
  title: String!
}

union ResultUnion = Author | Book

type Query {
  search: [ResultUnion!]!
}
```

Kütüphane tarafından oluşturulan varsayılan `resolveType()` fonksiyonu, çözümleme yönteminden dönen değere dayanarak tipi çıkaracaktır. Bu, doğrudan JavaScript nesneleri yerine sınıf örneklerini döndürmek zorunlu olduğu anlamına gelir.

Özel bir `resolveType()` fonksiyonu sağlamak için, `createUnionType()` fonksiyonuna iletilen seçenekler nesnesine `resolveType` özelliğini aşağıdaki gibi ekleyin:

```typescript
export const ResultUnion = createUnionType({
  name: 'ResultUnion',
  types: () => [Author, Book] as const,
  resolveType(value) {
    if (value.name) {
      return Author;
    }
    if (value.title) {
      return Book;
    }
    return null;
  },
});
```

#### Schema first

Schema first yaklaşımını kullanarak bir birleşim tipi tanımlamak için basitçe bir SDL ile bir GraphQL birleşim tipi oluşturun.

```graphql
type Author {
  name: String!
}

type Book {
  title: String!
}

union ResultUnion = Author | Book
```

Ardından, karşılık gelen TypeScript tanımlamalarını oluşturmak için yazım oluşturma özelliğini ( [hızlı başlangıç](/docs/graphql/quick-start) bölümünde gösterildiği gibi) kullanabilirsiniz:

```typescript
export class Author {
  name: string;
}

export class Book {
  title: string;
}

export type ResultUnion = Author | Book;
```

Birleşim tipleri, birleşim tipinin hangi tipe çözüleceğini belirlemek için resolver haritasında ek bir `__resolveType` alanını gerektirir. Ayrıca, `ResultUnionResolver` sınıfının herhangi bir modülde bir sağlayıcı olarak kaydedilmiş olması gerektiğini unutmayın. Bir `ResultUnionResolver` sınıfı oluşturalım ve `__resolveType` yöntemini tanımlayalım.

```typescript
@Resolver('ResultUnion')
export class ResultUnionResolver {
  @ResolveField()
  __resolveType(value) {
    if (value.name) {
      return 'Author';
    }
    if (value.title) {
      return 'Book';
    }
    return null;
  }
}
```

> info **Hint** Tüm dekoratörler `@nestjs/graphql` paketinden ihraç edilir.

### Enums

Numaralandırma tipleri, belirli bir küme izin verilen değerle sınırlı olan özel bir skalar türüdür (daha fazlasını [buradan](https://graphql.org/learn/schema/#enumeration-types) okuyabilirsiniz). Bu, şunları yapmanıza olanak tanır:

- Bu türdeki herhangi bir argümanın izin verilen değerlerden biri olup olmadığını doğrulayın
- Bir alanın her zaman bir değer kümesinin bir elemanı olacağını tip sistemi aracılığıyla iletişim kurun

#### Code first

Code first yaklaşımını kullandığınızda, bir GraphQL numaralandırma türünü basitçe bir TypeScript numarası oluşturarak tanımlarsınız.

```typescript
export enum AllowedColor {
  RED,
  GREEN,
  BLUE,
}
```

Bunu yerine getirdiğinizde, `@nestjs/graphql` paketinden ihraç edilen `registerEnumType` işlevini kullanarak `AllowedColor` numaralandırmasını kaydedebilirsiniz:

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
});
```

Şimdi `AllowedColor`'ı tiplerimizde referans verebilirsiniz:

```typescript
@Field(type => AllowedColor)
favoriteColor: AllowedColor;
```

Bu, SDL'de aşağıdaki GraphQL şemasının bir kısmını üretir:

```graphql
enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

Numaralandırmaya bir açıklama sağlamak için, `registerEnumType()` işlevine `description` özelliğini geçirin.

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
  description: 'The supported colors.',
});
```

Numaralandırma değerleri için bir açıklama sağlamak veya bir değeri kullanım dışı bırakmak için `valuesMap` özelliğini geçirin, aşağıdaki gibi:

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
  description: 'The supported colors.',
  valuesMap: {
    RED: {
      description: 'The default color.',
    },
    BLUE: {
      deprecationReason: 'Too blue.',
    },
  },
});
```

Bu, SDL'de aşağıdaki GraphQL şemasını üretir:

```graphql
"""
The supported colors.
"""
enum AllowedColor {
  """
  The default color.
  """
  RED
  GREEN
  BLUE @deprecated(reason: "Too blue.")
}
```

#### Schema first

Schema first yaklaşımını kullanarak bir numaralandırıcıyı tanımlamak için basitçe bir GraphQL numarası oluşturun.

```graphql
enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

Ardından, karşılık gelen TypeScript tanımlamalarını oluşturmak için yazım oluşturma özelliğini ( [hızlı başlangıç](/docs/graphql/quick-start) bölümünde gösterildiği gibi) kullanabilirsiniz:

```typescript
export enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

Bazen bir backend, API'deki içsel değerlerden farklı bir değeri zorlar. Bu örnekte API `RED` içeriyorsa, çözücülerde bunun yerine `#f00` kullanabiliriz ([buradan](https://www.apollographql.com/docs/apollo-server/schema/scalars-enums/#internal-values) daha fazla bilgi alın). Bunu başarmak için, `AllowedColor` numaralandırması için bir çözücü nesnesi bildirin:

```typescript
export const allowedColorResolver: Record<keyof typeof AllowedColor, any> = {
  RED: '#f00',
};
```

> info **Hint** Tüm dekoratörler `@nestjs/graphql` paketinden ihraç edilir.

Daha sonra bu çözücü nesneyi `GraphQLModule#forRoot()` yönteminin `resolvers` özelliği ile birlikte kullanın, aşağıdaki gibi:

```typescript
GraphQLModule.forRoot({
  resolvers: {
    AllowedColor: allowedColorResolver,
  },
});
```