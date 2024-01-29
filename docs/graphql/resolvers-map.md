### Çözücüler

Çözücüler, bir [GraphQL](https://graphql.org/) işlemini (bir sorgu, mutasyon veya abonelik) veriye dönüştürme talimatlarını sağlar. Bu, belirttiğimiz şema şeklindeki veriyi ya senkron olarak döndürür ya da bu şekle çözülen bir sonuca çözülen bir söz verisine sahip olur. Genellikle bir **çözücü haritası** manuel olarak oluşturulur. Diğer yandan `@nestjs/graphql` paketi, sınıfları işaretlemek için kullandığınız dekoratörler aracılığıyla sağlanan metadata'yı kullanarak çözücü haritasını otomatik olarak oluşturur. Bu paket özelliklerini kullanarak bir GraphQL API oluşturmak için süreci göstermek için basit bir yazarlar API'si oluşturacağız.

#### Code First

Code first yaklaşımında, genellikle GraphQL SDL'yi el ile yazarak GraphQL şemamızı oluşturma tipik sürecini takip etmeyiz. Bunun yerine, TypeScript sınıf tanımlarından SDL'yi üretmek için TypeScript dekoratörlerini kullanırız. `@nestjs/graphql` paketi, dekoratörler aracılığıyla tanımlanan metadata'yı okur ve otomatik olarak şemayı sizin için oluşturur.

#### Object Types

GraphQL şemasındaki tanımların çoğu **object types**'tır. Tanımladığınız her bir object type, bir uygulama istemcisinin etkileşimde bulunabileceği bir alan nesnesi temsil etmelidir. Örneğin, örnek API'ımızın yazarların ve gönderilerinin listesini alabilmesi için `Author` ve `Post` türlerini tanımlamalıyız.

Eğer schema first yaklaşımını kullanıyor olsaydık, SDL kullanarak bu tür bir şemayı şu şekilde tanımlardık:

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post!]!
}
```

Bu durumda, code first yaklaşımını kullanarak, şemaları TypeScript sınıflarını kullanarak tanımlarız ve bu sınıfların alanlarını anlamlandırmak için TypeScript dekoratörlerini kullanırız. Yukarıdaki SDL'nin code first yaklaşımındaki karşılığı şudur:

```typescript
@@filename(authors/models/author.model)
import { Field, Int, ObjectType } from '@nestjs/graphql';
import { Post } from './post';

@ObjectType()
export class Author {
  @Field(type => Int)
  id: number;

  @Field({ nullable: true })
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  @Field(type => [Post])
  posts: Post[];
}
```

> bilgi **İpucu** TypeScript'in metadata yansıma sistemine birkaç kısıtlama bulunmaktadır. Bu kısıtlamaların neden olduğu problemlerden biri, bir sınıfın hangi özelliklere sahip olduğunu belirleme veya belirli bir özelliğin isteğe bağlı veya zorunlu olup olmadığını tanıma yeteneğinin olmamasıdır. Bu kısıtlamalardan dolayı şema tanım sınıflarımızda her bir alanın GraphQL türü ve isteğe bağlılığı hakkında metadata sağlamak için `@Field()` dekoratörünü açıkça kullanmamız veya bunları bizim için üreten bir [CLI eklentisi](/docs/graphql/cli-plugin) kullanmamız gerekmektedir.

`Author` object type, diğer sınıflar gibi, bir koleksiyon alanından oluşur ve her bir alan belirli bir türü ifade eder. Bir alanın türü, bir [GraphQL türüne](https://graphql.org/learn/schema/) karşılık gelir. Bir alanın GraphQL türü, başka bir nesne türü veya bir skaler tür olabilir. GraphQL skaler türü, tek bir değere çözülen bir temel

 veri türüdür (`ID`, `String`, `Boolean` veya `Int` gibi). 

> bilgi **İpucu** GraphQL'in yerleşik skaler türlerine ek olarak, özel skaler türleri tanımlayabilirsiniz (daha fazla bilgi için [buraya](/docs/graphql/scalars) bakın).

Yukarıdaki `Author` object type tanımı, Nest'in yukarıda gösterdiğimiz SDL'yi **üretmesine** neden olacaktır:

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post!]!
}
```

`@Field()` dekoratörü, isteğe bağlı bir tür fonksiyonunu (örneğin, `type => Int`) ve opsiyonel olarak bir seçenek nesnesini kabul eder.

Tür fonksiyonu, TypeScript tür sistemi ile GraphQL tür sistemi arasında belirsizlik potansiyeli olduğunda gereklidir. Özellikle: `string` ve `boolean` türleri için **gerekli değildir**; `number` için (ki bu da GraphQL `Int` veya `Float`'a eşlenmelidir) **gereklidir**. Tür fonksiyonu, istenen GraphQL türünü döndürmelidir (bu, bu bölümlerdeki çeşitli örneklerde gösterildiği gibi).

Seçenekler nesnesi şu anahtar/değer çiftlerinden herhangi birini içerebilir:

- `nullable`: bir alanın null olup olmadığını belirtmek için (SDL'de, her bir alan varsayılan olarak null olamayan bir alandır); `boolean`
- `description`: bir alan açıklaması ayarlamak için; `string`
- `deprecationReason`: bir alanı kullanım dışı bırakmak için; `string`

Örneğin:

```typescript
@Field({ description: `Kitap başlığı`, deprecationReason: 'v2 şemasında kullanışlı değil' })
title: string;
```

> bilgi **İpucu** Tüm nesne türüne bir açıklama ekleyebilir veya onu kullanımdan kaldırabilirsiniz: `@ObjectType({{ '{' }} description: 'Yazar modeli' {{ '}' }})`.

Eğer alan bir dizi ise, `Field()` dekoratörünün tür fonksiyonunda dizinin türünü manuel olarak belirtmemiz gerekir, aşağıda gösterildiği gibi:

```typescript
@Field(type => [Post])
posts: Post[];
```

> bilgi **İpucu** Dizi gösterim notasyonunu (`[ ]`) kullanarak dizinin derinliğini belirtebiliriz. Örneğin, `[[Int]]` kullanmak bir tamsayı matrisini temsil eder.

Bir dizinin öğelerinin (dizi kendisi değil) null olabileceğini belirtmek için `Field()` dekoratörünün `nullable` özelliğini aşağıda gösterildiği gibi `'items'` olarak ayarlamamız gerekir:

```typescript
@Field(type => [Post], { nullable: 'items' })
posts: Post[];
```

> bilgi **İpucu** Eğer hem dizi hem de öğeleri nullable ise, `nullable`'ı `'itemsAndList'` olarak ayarlayın.

Şimdi `Author` object type oluşturulduğuna göre, `Post` object type'ını tanımlayalım.

```typescript
@@filename(posts/models/post.model)
import { Field, Int, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Post {
  @Field(type => Int)
  id: number;

  @Field()
  title: string;

  @Field(type => Int, { nullable: true })
  votes?: number;
}
```

`Post` object type, SDL'de aşağıdaki şema kısmını üretecektir:

```graphql
type Post {
  id: Int!
  title: String!
  votes: Int
}
```

#### Code First Resolver

Bu noktada, veri grafiğimizde var olabilecek nesneleri (tür tanımlamalarını) tanımladık, ancak istemciler henüz bu nesnelerle etkileşimde bulunmanın bir yoluna sahip değiller. Bu sorunu çözmek için bir çözücü sınıfı oluşturmamız gerekiyor. Code first yönteminde, bir çözücü sınıfı hem çözücü işlevlerini tanımlar **hem de** **Query türünü oluşturur**. Bu, aşağıdaki örneği çalıştırdıkça açıklık kazanacaktır:

```typescript
@@filename(authors/authors.resolver)
@Resolver(of => Author)
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query(returns => Author)
  async author(@Args('id', { type: () => Int }) id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField()
  async posts(@Parent() author: Author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> bilgi **İpucu** Tüm dekoratörler (örneğin, `@Resolver`, `@ResolveField`, `@Args`, vb.) `@nestjs/graphql` paketinden içe aktarılmaktadır.

Birden çok çözücü sınıfı tanımlayabilirsiniz. Nest çalışma zamanında bunları birleştirecektir. Kod düzenleme hakkında daha fazla bilgi için aşağıdaki [module](/docs/graphql/resolvers-map#module) bölümüne bakın.

> uyarı **Not** `AuthorsService` ve `PostsService` sınıflarındaki mantık isteğe bağlı veya karmaşık olabilir. Bu örnek, çözücüleri nasıl oluşturacağımızı ve diğer sağlayıcılarla nasıl etkileşim kurabileceğimizi göstermek amacını taşımaktadır.

Yukarıdaki örnekte, `AuthorsResolver`'ı oluşturduk ve bir sorgu çözücü işlevi ve bir alan çözücü işlevi tanımladık. Bir çözücü oluşturmak için, çözücü işlevlerini yöntem olarak içeren bir sınıf oluştururuz ve sınıfı `@Resolver()` dekoratörü ile işaretleriz.

Bu örnekte, isteğe bağlı bir parent nesne sağlamak için kullanılan `@Resolver()` dekoratörüne geçirilen argüman opsiyoneldir, ancak grafimiz karmaşık hale geldiğinde devreye girer. Bu, bir nesne grafiği üzerinde aşağı doğru hareket ederken alan çözücü işlevleri tarafından kullanılan bir üst nesneyi sağlamak için kullanılır.

Örneğimizde, çünkü sınıf bir **alan çözücü** işlevi içeriyor (Author nesnesinin `posts` özelliği için), bu sınıf içinde tanımlanan tüm alan çözücü işlevleri için hangi sınıfın ana türü olduğunu (yani, karşılık gelen `ObjectType` sınıf adını) belirtmek **gereklidir**. Örnekte açık olduğu gibi, bir alan çözücü işlevi yazarken, ebeveyn nesneye (çözülen alanın bir üyesi olan nesne) erişim sağlamak gerekir. Bu örnekte, bir yazarın gönderiler dizisini bir alan çözücüsü aracılığıyla doldururuz ve bu servisi bir yazarın `id`'si ile çağırırız. Bu nedenle, `@Resolver()` dekoratöründe ebeveyn nesneyi tanımlamak gereklidir. Ayrıca, ardından bu alan çözücüsünde bu ebeveyn nesnesine bir referans çıkarmak için `@Parent()` yöntem parametre dekoratörünü kullanma ihtiyacı vardır.

Birden çok `@Query()` çözücü işlevi tanımlayabiliriz (hem bu sınıf içinde hem de başka bir çözücü sınıfında), ve bunlar SDL'de tek bir **Query türü** tanımına ve çözücü haritasındaki uygun girişlere bir araya getirilecektir. Bu, sorguları, kullandıkları modeller ve hizmetlere yakın bir şekilde tanımlamamıza ve modüller içinde iyi düzenlenmiş tutmamıza olanak tanır.

> bilgi **İpucu** Nest CLI, bize **tüm tekrarlayan kodu** otomatik olarak üreten bir jeneratör (şematik) sağlar. Bu sayede tüm bu işlemleri yapmaktan kaçınabilir ve geliştirici deneyimini çok daha basit hale getirebiliriz. Bu özellik hakkında daha fazla bilgi için [buraya](/docs/recipes/crud-generator) bakın.

#### Query türü adları

Yukarıdaki örneklerde, `@Query()` dekoratörü, metod adına dayalı olarak bir GraphQL şema sorgu türü adı oluşturur. Örneğin, yukarıdaki örnekten şu oluşturma işlemini düşünün:

```typescript
@Query(returns => Author)
async author(@Args('id', { type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

Bu, şemamızdaki `author` sorgusu için aşağıdaki girişi oluşturur (sorgu türü, metod adıyla aynı adı kullanır):

```graphql
type Query {
  author(id: Int!): Author
}
```

> bilgi **İpucu** GraphQL sorguları hakkında daha fazla bilgi için [buraya](https://graphql.org/learn/queries/) bakın.

Geleneksel olarak, bu adları bağlantısız kullanmayı tercih ederiz; örneğin, sorgu işleyici metodumuz için `getAuthor()` gibi bir isim kullanmayı tercih ederiz, ancak yine de sorgu türü adımız için `author`'ı kullanırız. Aynı şey alan çözücülerimiz için de geçerlidir. Bu adları `@Query()` ve `@ResolveField()` dekoratörlerinin argümanları olarak geçirerek kolayca yapabiliriz, aşağıda gösterildiği gibi:

```typescript
@@filename(authors/authors.resolver)
@Resolver(of => Author)
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query(returns => Author, { name: 'author' })
  async getAuthor(@Args('id', { type: () => Int }) id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField('posts', returns => [Post])
  async getPosts(@Parent() author: Author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

Yukarıdaki `getAuthor` işleyici metodunun sonucunda, GraphQL şemamızdaki aşağıdaki kısım oluşturulacaktır:

```graphql
type Query {
  author(id: Int!): Author
}
```

#### Query dekoratör seçenekleri

`@Query()` dekoratörünün (yukarıda `{{ '{' }}name: 'author'{{ '}' }}` olarak ilettiğimiz yer) seçenekler nesnesi bir dizi anahtar/değer çiftini kabul eder:

- `name`: sorgunun adı; bir `string`
- `description`: GraphQL şema belgelendirmesi oluşturmak için kullanılacak açıklama; bir `string`
- `deprecationReason`: sorguyu GraphQL oyun alanında (örneğin) kullanılan bir meta bilgi olarak işaretleme; bir `string`
- `nullable`: sorgunun null bir veri yanıtı döndürüp döndüremeyeceği; `boolean` veya `'items'` veya `'itemsAndList'` (yukarıdaki `'items'` ve `'itemsAndList'` ayrıntıları için bakınız)

#### Args dekoratör seçenekleri

`@Args()` dekoratörünü kullanarak, bir talepten argümanları çıkarmak için kullanılır. Bu, [REST rotası parametre argümanı çıkarma](/docs/controllers#route-parameters) ile çok benzer bir şekilde çalışır.

Genellikle `@Args()` dekoratörünüz basit olacak ve yukarıdaki `getAuthor()` metodunda gördüğünüz gibi bir nesne argümanını gerektirmeyecek. Örneğin, bir tanımlayıcının türü string ise, aşağıdaki oluşturma yeterlidir ve isimlendirilmiş alanı gelen GraphQL isteğinden metod argümanı olarak kullanır.

```typescript
@Args('id') id: string
```

`getAuthor()` durumunda, `number` türü kullanıldığından bir zorluk ortaya çıkar. `number` TypeScript türü, bize beklenen GraphQL temsil hakkında yeterli bilgi vermez (örneğin, `Int`'e karşı `Float`). Bu nedenle, **kesinlikle** tür referansını geçirmemiz gerekiyor. Bunun için `Args()` dekoratörüne bir ikinci argüman geçiriyoruz ve bu argüman seçeneklerini içeren bir nesne içeriyor, aşağıda gösterildiği gibi:

```typescript
@Query(returns => Author, { name: 'author' })
async getAuthor(@Args('id', { type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

Seçenekler nesnesi, aşağıdaki isteğe bağlı anahtar değer çiftlerini belirlemenize olanak tanır:

- `type`: GraphQL türünü döndüren bir fonksiyon
- `defaultValue`: bir varsayılan değer; `any`
- `description`: açıklama meta verileri; `string`
- `deprecationReason`: bir alanı işaretleme ve nedenini açıklayan meta veri sağlama; `string`
- `nullable`: alanın nullable olup olmadığı

#### Özel argüman sınıfı

Yukarıdaki örnekte olduğu gibi iç içe geçmiş `@Args()` çağrılarıyla, kod şişer. Bunun yerine, özel bir `GetAuthorArgs` argüman sınıfı oluşturabilir ve işlem yöntemine şu şekilde erişebilirsiniz:

```typescript
@Args() args: GetAuthorArgs
```

Aşağıda gösterildiği gibi `@ArgsType()` kullanarak `GetAuthorArgs` sınıfını oluşturun:

```typescript
@@filename(authors/dto/get-author.args)
import { MinLength } from 'class-validator';
import { Field, ArgsType } from '@nestjs/graphql';

@ArgsType()
class GetAuthorArgs {
  @Field({ nullable: true })
  firstName?: string;

  @Field({ defaultValue: '' })
  @MinLength(3)
  lastName: string;
}
```

> info **Hint** Yine de TypeScript'in metadata yansıtma sistemine ilişkin sınırlamalar nedeniyle, tür ve opsiyonelliği manuel olarak belirtmek için `@Field` dekoratörünü kullanmak veya [CLI eklentisi](/docs/graphql/cli-plugin) kullanmak gereklidir.

Bu, aşağıdaki GraphQL şemasının bir parçasını oluşturacaktır:

```graphql
type Query {
  author(firstName: String, lastName: String = ''): Author
}
```

> info **Hint** `GetAuthorArgs` gibi argüman sınıfları, `ValidationPipe` ile çok iyi çalışır (daha fazlasını okumak için [buraya](/docs/techniques/validation) bakın).

#### Sınıf Mirası

Standard TypeScript sınıf mirası kullanarak genişletilebilen genel tipli özelliklere (alanlar ve alan özellikleri, doğrulamalar, vb.) sahip temel sınıflar oluşturabilirsiniz. Örneğin, her zaman standart `offset` ve `limit` alanlarını içeren ancak aynı zamanda tür özel olan diğer indeks alanlarını da içeren bir dizi sayfalama ile ilgili argümanınız olabilir. Aşağıda gösterildiği gibi bir sınıf hiyerarşisi kurabilirsiniz.

Temel `@ArgsType()` sınıfı:

```typescript
@ArgsType()
class PaginationArgs {
  @Field((type) => Int)
  offset: number = 0;

  @Field((type) => Int)
  limit: number = 10;
}
```

Temel `@ArgsType()` sınıfının tür özel alt sınıfı:

```typescript
@ArgsType()
class GetAuthorArgs extends PaginationArgs {
  @Field({ nullable: true })
  firstName?: string;

  @Field({ defaultValue: '' })
  @MinLength(3)
  lastName: string;
}
```

Aynı yaklaşım, `@ObjectType()` nesneleriyle de kullanılabilir. Temel sınıfa genel özellikler ekleyin:

```typescript
@ObjectType()
class Character {
  @Field((type) => Int)
  id: number;

  @Field()
  name: string;
}
```

Alt sınıflara tür özel özellikleri ekleyin:

```typescript
@ObjectType()
class Warrior extends Character {
  @Field()
  level: number;
}
```

Ayrıca bir çözücü ile de mirası kullanabilirsiniz. Tür güvenliğini sağlamak için miras ve TypeScript generic'leri birleştirebilirsiniz. Örneğin, generic bir `findAll` sorgusu içeren bir temel sınıf oluşturmak için aşağıdaki gibi bir yapı kullanabilirsiniz:

```typescript
function BaseResolver<T extends Type<unknown>>(classRef: T): any {
  @Resolver({ isAbstract: true })
  abstract class BaseResolverHost {
    @Query((type) => [classRef], { name: `findAll${classRef.name}` })
    async findAll(): Promise<T[]> {
      return [];
    }
  }
  return BaseResolverHost;
}
```

Dikkat edilmesi gerekenler:

- Açık bir dönüş türü (`any` yukarıda) gereklidir: aksi takdirde TypeScript, özel bir sınıf tanımının kullanımıyla ilgili olarak şikayet eder. Tavsiye edilen: `any` kullanmak yerine bir arayüz tanımlayın.
- `Type`, `@nestjs/common` paketinden alınır.
- `isAbstract: true` özelliği, bu sınıf için SDL (Schema Definition Language ifadeleri) üretilmemesi gerektiğini gösterir. Unutmayın, bu özelliği diğer tipler için de ayarlayabilirsiniz ve SDL üretimini engellemek için kullanılabilir.

İşte `BaseResolver`'ın somut bir alt sınıfını nasıl oluşturabileceğinizi gösteren bir örnek:

```typescript
@Resolver((of) => Recipe)
export class RecipesResolver extends BaseResolver(Recipe) {
  constructor(private recipesService: RecipesService) {
    super();
  }
}
```

Bu yapı, aşağıdaki SDL'yi üretecektir:

```graphql
type Query {
  findAllRecipe: [Recipe!]!
}
```

#### Generics

Yukarıda generiklerin bir kullanımını gördük. Bu güçlü TypeScript özelliği, yararlı soyutlamalar oluşturmak için kullanılabilir. Örneğin, [bu belgede](https://graphql.org/learn/pagination/#pagination-and-edges) temellendirilen bir örnek cursor tabanlı sayfalama uygulaması:

```typescript
import { Field, ObjectType, Int } from '@nestjs/graphql';
import { Type } from '@nestjs/common';

interface IEdgeType<T> {
  cursor: string;
  node: T;
}

export interface IPaginatedType<T> {
  edges: IEdgeType<T>[];
  nodes: T[];
  totalCount: number;
  hasNextPage: boolean;
}

export function Paginated<T>(classRef: Type<T>): Type<IPaginatedType<T>> {
  @ObjectType(`${classRef.name}Edge`)
  abstract class EdgeType {
    @Field((type) => String)
    cursor: string;

    @Field((type) => classRef)
    node: T;
  }

  @ObjectType({ isAbstract: true })
  abstract class PaginatedType implements IPaginatedType<T> {
    @Field((type) => [EdgeType], { nullable: true })
    edges: EdgeType[];

    @Field((type) => [classRef], { nullable: true })
    nodes: T[];

    @Field((type) => Int)
    totalCount: number;

    @Field()
    hasNextPage: boolean;
  }
  return PaginatedType as Type<IPaginatedType<T>>;
}
```

Yukarıdaki temel sınıf tanımlandığında, şimdi bu davranışı miras alan özel tipleri kolayca oluşturabiliriz. Örneğin:

```typescript
@ObjectType()
class PaginatedAuthor extends Paginated(Author) {}
```

#### Schema first

[Önceki](/docs/graphql/quick-start) bölümde belirtildiği gibi, şema önce yaklaşımında, SDL'de şema türlerini manuel olarak tanımlamaya başlarız (daha fazla bilgi için [buraya bakın](https://graphql.org/learn/schema/#type-language)). Aşağıdaki SDL tür tanımlamalarını düşünün.

> info **Hint** Bu bölümde kolaylık sağlamak için, SDL'yi tek bir konumda (örneğin, aşağıda gösterildiği gibi tek bir `.graphql` dosyası) topladık. Uygulamada, kodunuzu modüler bir şekilde düzenlemeniz uygun olabilir. Örneğin, her etki alanını temsil eden tip tanımlarını içeren bireysel SDL dosyaları oluşturmak ve bu etki alanına özgü hizmetlerle, çözücü koduyla ve Nest modül tanımlama sınıfıyla birlikte ilgili bir dizinde organize etmek faydalı olabilir. Nest, tüm bireysel şema tip tanımlarını çalışma zamanında bir araya getirecektir.

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String!
  votes: Int
}

type Query {
  author(id: Int!): Author
}
```

#### Schema first resolver

Yukarıdaki şema, yalnızca bir sorgu - `author(id: Int!): Author` - sunmaktadır.

> info **Hint** GraphQL sorguları hakkında daha fazla bilgi edinin [burada](https://graphql.org/learn/queries/).

Şimdi `AuthorsResolver` adında bir sınıf oluşturalım ve bu sınıfın yazar sorgularını çözdüğünü gösterelim:

```typescript
@@filename(authors/authors.resolver)
@Resolver('Author')
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query()
  async author(@Args('id') id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField()
  async posts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> info **Hint** Tüm dekoratörler (örneğin, `@Resolver`, `@ResolveField`, `@Args` vb.) `@nestjs/graphql` paketinden içe aktarılır.

> warning **Note** `AuthorsService` ve `PostsService` sınıflarındaki mantık istendiği kadar basit veya karmaşık olabilir. Bu örneğin ana noktası çözücülerin nasıl oluşturulacağını ve diğer sağlayıcılarla nasıl etkileşimde bulunabileceklerini göstermektir.

`@Resolver()` dekoratörü zorunludur. İsteğe bağlı bir dize argüman alır ve bir sınıfın adını içerir. Bu sınıf adı, sınıfın içinde `@ResolveField()` dekoratörlerini içeriyorsa, dekore edilmiş yöntemin bir üst türle (şu anda örneğimizde `Author` türü) ilişkilendirildiğini Nest'e bildirmek için gereklidir. Aksi takdirde, `@Resolver()`'ı sınıfın en üst düzeyinde ayarlamak yerine bunu her yöntem için yapabiliriz:

```typescript
@Resolver('Author')
@ResolveField()
async posts(@Parent() author) {
  const { id } = author;
  return this.postsService.findAll({ authorId: id });
}
```

Bu durumda (`@Resolver()` dekoratörünü yöntem düzeyinde kullanmak), bir sınıf içinde birden çok `@ResolveField()` dekoratörünüz varsa, hepsine `@Resolver()` eklemelisiniz. Bu, ekstra iş yükü yarattığından en iyi uygulama olarak kabul edilmez.

> info **Hint** `@Resolver()`'a geçilen herhangi bir sınıf adı argümanı, sorguları (`@Query()` dekoratörü) veya mutasyonları (`@Mutation()` dekoratörü) etkilemez.

> warning **Warning** Yöntem düzeyinde `@Resolver` dekoratörünü kullanmak, **code first** yaklaşımıyla desteklenmez.

Yukarıdaki örneklerde `@Query()` ve `@ResolveField()` dekoratörleri, yöntem adına dayanarak GraphQL şema türleriyle ilişkilidir. Örneğin, yukarıdaki örnekte olduğu gibi:

```typescript
@Query()
async author(@Args('id') id: number) {
  return this.authorsService.findOneById(id);
}
```

Bu, şemamızdaki yazar sorgusu için aşağıdaki girişin oluşturulmasına neden olur (sorgu türü, yöntem adıyla aynı adı kullanır):

```graphql
type Query {
  author(id: Int!): Author
}
```

Geleneksel olarak, bu isimleri, çözücü yöntemlerimiz için `getAuthor()` veya `getPosts()` gibi isimler kullanmayı tercih ederiz. Bu, aşağıda gösterildiği gibi dekoratöre bir argüman olarak eşleme adını geçirerek kolayca yapılabilir:

```typescript
@@filename(authors/authors.resolver)
@Resolver('Author')
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query('author')
  async getAuthor(@Args('id') id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField('posts')
  async getPosts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> info **Hint** Nest CLI, tüm bu işlemleri otomatik olarak gerçekleştiren bir üreteç (şema) sağlar ve bunu yapmaktan kaçınmamıza ve geliştirici deneyimini çok daha basit hale getirmemize yardımcı olur. Bu özellik hakkında daha fazla bilgi için [buraya](/docs/recipes/crud-generator) bakın.

#### Türleri Oluşturma

Şema ilk yaklaşımını kullandığımızı ve (önceki bölümde gösterildiği gibi) tiplerin oluşturma özelliğini etkinleştirdiysek (örneğin, `outputAs: 'class'`), uygulamayı çalıştırdığınızda aşağıdaki dosyayı (belirttiğiniz konumda, `GraphQLModule.forRoot()` yönteminde gösterildiği gibi) oluşturacaktır. Örneğin, `src/graphql.ts` dosyasında:

```typescript
@@filename(graphql)
export (class Author {
  id: number;
  firstName?: string;
  lastName?: string;
  posts?: Post[];
})
export class Post {
  id: number;
  title: string;
  votes?: number;
}

export abstract class IQuery {
  abstract author(id: number): Author | Promise<Author>;
}
```

Sınıfları oluşturarak (varsayılan teknik olan arayüzleri oluşturmak yerine), şema ilk yaklaşımı ile deklaratif doğrulama **dekoratörleri** kullanabilir ve bu, son derece kullanışlı bir tekniktir (daha fazla bilgi için [buraya](/docs/techniques/validation) bakın). Örneğin, `CreatePostInput` sınıfına `title` alanında minimum ve maksimum dize uzunluklarını zorlamak için aşağıdaki gibi `class-validator` dekoratörlerini ekleyebilirsiniz:

```typescript
import { MinLength, MaxLength } from 'class-validator';

export class CreatePostInput {
  @MinLength(3)
  @MaxLength(50)
  title: string;
}
```

> warning **Notice** Girişlerinizin (ve parametrelerinizin) otomatik doğrulamasını etkinleştirmek için `ValidationPipe`'i kullanın. Doğrulama hakkında daha fazla bilgi için [buraya](/docs/techniques/validation) ve özellikle borular hakkında [buraya](/docs/pipes) bakın.

Ancak, dekoratörleri doğrudan otomatik olarak oluşturulan dosyaya eklerseniz, dosya her oluşturulduğunda **üzerine yazılacaktır**. Bunun yerine, ayrı bir dosya oluşturun ve sadece oluşturulan sınıfı genişletin.

```typescript
import { MinLength, MaxLength } from 'class-validator';
import { Post } from '../../graphql.ts';

export class CreatePostInput extends Post {
  @MinLength(3)
  @MaxLength(50)
  title: string;
}
```

#### GraphQL Argüman Dekoratörleri

Standart GraphQL çözücü argümanlarına özel dekoratörler kullanarak bu argümanlara erişebiliriz. Aşağıda, Nest dekoratörleri ile temsil edilen Apollo parametreleri ile karşılaştırma bulunmaktadır.

<table>
  <tbody>
    <tr>
      <td><code>@Root()</code> ve <code>@Parent()</code></td>
      <td><code>root</code>/<code>parent</code></td>
    </tr>
    <tr>
      <td><code>@Context(param?: string)</code></td>
      <td><code>context</code> / <code>context[param]</code></td>
    </tr>
    <tr>
      <td><code>@Info(param?: string)</code></td>
      <td><code>info</code> / <code>info[param]</code></td>
    </tr>
    <tr>
      <td><code>@Args(param?: string)</code></td>
      <td><code>args</code> / <code>args[param]</code></td>
    </tr>
  </tbody>
</table>

Bu argümanların aşağıdaki anlamları bulunmaktadır:

- `root`: ebeveyn alan çözücüsünden dönen sonucu içeren bir nesne veya, üst düzey bir `Query` alanı durumunda, sunucu yapılandırmasından geçen `rootValue`.
- `context`: belirli bir sorguda bulunan tüm çözücüler arasında paylaşılan bir nesne; genellikle her istek için durumu içermek için kullanılır.
- `info`: sorgunun yürütme durumu hakkında bilgi içeren bir nesne.
- `args`: sorguda alana iletilen argümanları içeren bir nesne.

<app-banner-devtools></app-banner-devtools>

#### Modül

Yukarıdaki adımları tamamladığımızda, `GraphQLModule` için bir çözücü haritası oluşturmak için gereken tüm bilgileri deklaratif olarak belirtmiş oluruz. `GraphQLModule`, dekoratörler aracılığıyla sağlanan meta verileri incelemek ve sınıfları otomatik olarak doğru çözücü haritasına dönüştürmek için yansıma kullanır.

Yapmanız gereken tek diğer şey, çözücü sınıf(lar)ını sağlamak (yani, bir modülde bir `provider` olarak listelemek) ve modülü (`AuthorsModule`) bir yerde içe aktarmak, böylece Nest bunu kullanabilir.

Örneğin, bunu bir `AuthorsModule` içinde yapabiliriz ve bu bağlamda gereken diğer servisleri de sağlayabiliriz. `AuthorsModule`'ü bir yerde içe aktardığınızdan emin olun (örneğin, kök modülde veya kök modül tarafından içe aktarılan başka bir modülde).

```typescript
@@filename(authors/authors.module)
@Module({
  imports: [PostsModule],
  providers: [AuthorsService, AuthorsResolver],
})
export class AuthorsModule {}
```

> info **Hint** Kodunuzu kendi **domain modeliniz** olarak adlandırılan (bir REST API'deki giriş noktalarını nasıl düzenleyeceğinizi düzenler gibi) ile düzenlemek faydalıdır. Bu yaklaşımda, modellerinizi (`ObjectType` sınıfları), çözücüleri ve servisleri bir arada tutmak için bir Nest modülünü kullanın. Bu bileşenleri her bir modül için tek bir klasörde tutun. Bunu yaptığınızda ve her öğeyi oluşturmak için [Nest CLI](/docs/cli/overview)’yi kullandığınızda, Nest tüm bu parçaları otomatik olarak birbirine bağlar (dosyaları uygun klasörlere yerleştirme, `provider` ve `imports` dizilerinde girişler oluşturma vb.).