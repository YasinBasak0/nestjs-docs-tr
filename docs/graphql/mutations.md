### Mutasyonlar

GraphQL hakkındaki çoğu tartışma veri getirme üzerine odaklanır, ancak herhangi bir tam veri platformunun sunucu tarafındaki veriyi değiştirme yolu olması gerekir. REST'te herhangi bir istek sunucuda yan etkilere neden olabilir, ancak en iyi uygulama, veriyi GET isteklerinde değiştirmememiz gerektiğini söyler. GraphQL de benzer - teorik olarak herhangi bir sorgu bir veri yazma işlemine neden olabilir. Ancak REST gibi, yazma işlemine neden olan işlemlerin açıkça bir mutasyon aracılığıyla gönderilmesi önerilir (daha fazlası için [buraya](https://graphql.org/learn/queries/#mutations) bakın).

Resmi [Apollo](https://www.apollographql.com/docs/graphql-tools/generate-schema.html) belgeleri, bir `upvotePost()` mutasyonu örneğini kullanır. Bu mutasyon, bir gönderinin `votes` özelliğinin değerini artırmak için bir yöntem uygular. Nest'te eşdeğer bir mutasyon oluşturmak için `@Mutation()` dekoratörünü kullanacağız.

#### Kod odaklı

Önceki bölümde kullanılan `AuthorResolver`'a başka bir yöntem ekleyelim (bkz. [çözücüler](/docs/graphql/resolvers)).

```typescript
@Mutation(returns => Post)
async upvotePost(@Args({ name: 'postId', type: () => Int }) postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

> info **İpucu** Tüm dekoratörler (örneğin, `@Resolver`, `@ResolveField`, `@Args` vb.) `@nestjs/graphql` paketinden alınır.

Bu, SDL'de aşağıdaki şemayı oluşturur:

```graphql
type Mutation {
  upvotePost(postId: Int!): Post
}
```

`upvotePost()` yöntemi `postId` (`Int`)'yi bir argüman olarak alır ve güncellenmiş bir `Post` varlığını döndürür. [Çözücüler](/docs/graphql/resolvers) bölümünde açıklanan nedenlerden dolayı beklenen türü açıkça belirtmemiz gerektiğini unutmayın.

Eğer mutasyonun bir nesne almasını istiyorsak, bir **input tipi** oluşturabiliriz. Input tipi, bir argüman olarak iletilen özel bir nesne türüdür (daha fazlası için [buraya](https://graphql.org/learn/schema/#input-types) bakın). Bir giriş tipini bildirmek için `@InputType()` dekoratörünü kullanın.

```typescript
import { InputType, Field } from '@nestjs/graphql';

@InputType()
export class UpvotePostInput {
  @Field()
  postId: number;
}
```

> info **İpucu** `@InputType()` dekoratörüne bir seçenek nesnesi geçirildiğinden, örneğin, giriş türünün açıklamasını belirleyebilirsiniz. TypeScript'in meta veri yansıtma sistemi sınırlamaları nedeniyle, türü manuel olarak belirtmek için `@Field` dekoratörünü kullanmalısınız veya bir [CLI eklentisi](/docs/graphql/cli-plugin) kullanmalısınız.

Daha sonra bu türü çözücü sınıfında kullanabiliriz:

```typescript
@Mutation(returns => Post)
async upvotePost(
  @Args('upvotePostData') upvotePostData: UpvotePostInput,
) {}
```

#### Şema odaklı

Önceki bölümde kullanılan `AuthorResolver`'ı genişletelim (bkz. [çözücüler](/docs/graphql/resolvers)).

```typescript
@Mutation()
async upvotePost(@Args('postId') postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

Yukarıda, iş mantığının `PostsService`'e taşındığını varsaydık (gönderiyi sorgulama ve `votes` özelliğini artırma). `PostsService` sınıfındaki mantık, ihtiyaca bağlı olarak basit veya karmaşık olabilir. Bu örneğin ana noktası, çözücülerin diğer sağlayıcılarla nasıl etkileşimde bulunabileceğini göstermektir.

Son adım, mevcut türler tanımına mutasyonumuzu eklemektir.

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String
  votes: Int
}

type Query {
  author(id: Int!): Author
}

type Mutation {
  upvotePost(postId: Int!): Post
}
```

`upvotePost(postId: Int!): Post` mutasyonu artık uygulamanın GraphQL API'sinin bir parçası olarak çağrılabilir.