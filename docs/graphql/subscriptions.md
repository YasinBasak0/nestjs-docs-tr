### Abonelikler

Sorguları kullanarak veri almanın ve mutasyonları kullanarak veriyi değiştirmenin yanı sıra, GraphQL spesifikasyonu üçüncü bir işlem türünü destekler, bu da `abonelik` (subscription) olarak adlandırılır. GraphQL abonelikleri, sunucudan gerçek zamanlı mesajları dinlemeyi seçen istemcilere veri göndermenin bir yoludur. Abonelikler, istemciye hemen tek bir yanıt döndürmek yerine, bir kanal açılır ve sunucuda belirli bir olay gerçekleştiğinde sonuç istemciye gönderilir.

Abonelikler, istemciyi belirli olaylar hakkında bilgilendirmenin yaygın bir kullanım durumudur; örneğin, yeni bir nesnenin oluşturulması, güncellenen alanlar vb. (daha fazla bilgi için [buraya](https://www.apollographql.com/docs/react/data/subscriptions) bakın).

#### Apollo sürücüsü ile abonelikleri etkinleştirme

Abonelikleri etkinleştirmek için `installSubscriptionHandlers` özelliğini `true` olarak ayarlayın.

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  installSubscriptionHandlers: true,
}),
```

> warning **Uyarı** `installSubscriptionHandlers` yapılandırma seçeneği, Apollo sunucunun en son sürümünden kaldırılmış ve bu pakette de yakında kullanımdan kaldırılacaktır. Varsayılan olarak, `installSubscriptionHandlers`, `subscriptions-transport-ws`'yi kullanmak üzere geri düşecektir ([daha fazlasını okuyun](https://github.com/apollographql/subscriptions-transport-ws)). Ancak, bunun yerine `graphql-ws` ([daha fazlasını okuyun](https://github.com/enisdenjo/graphql-ws)) kütüphanesini kullanmanızı şiddetle öneririz.

`graphql-ws` paketini kullanmak için şu konfigürasyonu kullanın:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'graphql-ws': true
  },
}),
```

> info **İpucu** Aynı zamanda her iki paketi (`subscriptions-transport-ws` ve `graphql-ws`) aynı anda kullanabilirsiniz, örneğin, geriye dönük uyumluluk için.

#### Kod ilk

Kod ilk yaklaşımını kullanarak bir abonelik oluşturmak için, `@Subscription()` dekoratörünü (`@nestjs/graphql` paketinden içe aktarılır) ve basit bir **yayın/abone API'si** sağlayan `graphql-subscriptions` paketinden `PubSub` sınıfını kullanırız.

Aşağıdaki abonelik işleyicisi, `PubSub#asyncIterator`'ı çağırarak bir olaya **abone olmayı** sağlar. Bu yöntem, bir argüman alır, `triggerName` adında bir olay konu adına karşılık gelir.

```typescript
const pubSub = new PubSub();

@Resolver((of) => Author)
export class AuthorResolver {
  // ...
  @Subscription((returns) => Comment)
  commentAdded() {
    return pubSub.asyncIterator('commentAdded');
  }
}
```

> info **İpucu** Tüm dekoratörler `@nestjs/graphql` paketinden içe aktarılırken, `PubSub` sınıfı `graphql-subscriptions` paketinden içe aktarılır.

> warning **Not** `PubSub`, basit bir `publish` ve `subscribe API`'sini açığa çıkaran bir sınıftır. [Buradan](https://www.apollographql.com/docs/graphql-subscriptions/setup.html) daha fazla bilgi edinin. Apollo dokümantasyonları, varsayılan uygulamanın üretime uygun olmadığını uyarır (daha fazla bilgi için [buraya](https://github.com/apollographql/graphql-subscriptions#getting-started-with-your-first-subscription)). Üretim uygulamaları, dış bir depo tarafından desteklenen bir `PubSub` uygulaması kullanmalıdır (daha fazla bilgi için [buraya](https://github.com/apollographql/graphql-subscriptions#pubsub-implementations)).

Bu, SDL'deki GraphQL şemasının aşağıdaki kısmını üretecektir:

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

Abonelikler, tanım gereği, tek bir üst düzey özelliğe sahip bir nesne döndürür ve

 bu özelliğin anahtarı abonelik adıdır. Bu ad, abonelik işleyici yönteminin adından miras alınır (yukarıdaki örnekte olduğu gibi `commentAdded`), veya `@Subscription()` dekoratörüne ikinci argüman olarak `name` anahtarı ile açıkça verilerek sağlanır.

```typescript
@Subscription(returns => Comment, {
  name: 'commentAdded',
})
subscribeToCommentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

Bu yapı, önceki kod örneğinde olduğu gibi aynı SDL'yi üretir, ancak bize yöntem adını abonelikten ayırmamıza olanak tanır.

#### Yayınlama

Şimdi, olayı yayınlamak için `PubSub#publish` yöntemini kullanıyoruz. Bu genellikle bir mutasyon içinde kullanılır ve nesne grafinin bir kısmı değiştiğinde istemci tarafında bir güncelleme tetiklemek için kullanılır. Örneğin:

```typescript
@@filename(posts/posts.resolver)
@Mutation(returns => Post)
async addComment(
  @Args('postId', { type: () => Int }) postId: number,
  @Args('comment', { type: () => Comment }) comment: CommentInput,
) {
  const newComment = this.commentsService.addComment({ id: postId, comment });
  pubSub.publish('commentAdded', { commentAdded: newComment });
  return newComment;
}
```

`PubSub#publish` yöntemi, birinci parametre olarak bir `triggerName` (yine, bu bir olay konu adı olarak düşünülebilir) ve ikinci parametre olarak bir olay yükünü alır. Bahsedildiği gibi, abonelik tanım gereği bir değer döndürür ve bu değerin bir şekli vardır. `commentAdded` aboneliğimizin ürettiği SDL'ye tekrar bakalım:

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

Bu bize abonelikten döndürülmesi beklenen bir `commentAdded` adında üst düzey bir özelliğe sahip bir nesne olması gerektiğini söyler. Önemli olan nokta, `PubSub#publish` yöntemi tarafından yayınlanan olay yükünün şeklinin, abonelikten döndürülmesi beklenen değerin şekline uyması gerektiğidir. Bu nedenle, yukarıdaki örneğimizde `pubSub.publish('commentAdded', {{ '{' }} commentAdded: newComment {{ '}' }})` ifadesi, uygun şekilli bir yük içeren `commentAdded` olayını yayınlamaktadır. Bu şekiller uyuşmazsa, abonelik GraphQL doğrulama aşamasında başarısız olacaktır.

#### Abonelikleri filtreleme

Belirli olayları filtrelemek için `filter` özelliğini bir filtre işlevine ayarlayın. Bu işlev, bir dizi `filter`'a geçirilen bir işleve benzer şekilde hareket eder. İki argüman alır: `payload` olay yükünü içerir (olay yayıncısı tarafından gönderilen şekilde), ve `variables` abonelik isteği sırasında iletilen herhangi bir argümanı alır. Bu işlev, bu olayın istemci dinleyicilerine yayımlanıp yayımlanmayacağını belirleyen bir boole değeri döndürür.

```typescript
@Subscription(returns => Comment, {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded(@Args('title') title: string) {
  return pubSub.asyncIterator('commentAdded');
}
```

#### Mutasyon Abonelik Yüklerini Değiştirme

Yayınlanan olay yükünü değiştirmek için `resolve` özelliğini bir işleve ayarlayın. Bu işlev, olay yükünü (olay yayıncısı tarafından gönderilen şekilde) alır ve uygun değeri döndürür.

```typescript
@Subscription(returns => Comment, {
  resolve: value => value,
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

> warning **Not** Eğer `resolve` seçeneğini kullanıyorsanız, geri döndürdüğünüz değerin (örneğin, bizim örneğimizde `{{ '{' }} commentAdded: newComment {{ '}' }}` nesnesi değil) düzleştirilmiş olay yükü olması gerekir.

Eğer enjekte edilmiş sağlayıcıları (örneğin, verileri doğrulamak için bir harici servisi kullanmak) erişmeniz gerekiyorsa, aşağıdaki yapıyı kullanın.

```typescript
@Subscription(returns => Comment, {
  resolve(this: AuthorResolver, value) {
    // "this", bir "AuthorResolver" örneğine referans verir
    return value;
  }
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

Aynı yapı filtrelerle de çalışır:

```typescript
@Subscription(returns => Comment, {
  filter(this: AuthorResolver, payload, variables) {
    // "this", bir "AuthorResolver" örneğine referans verir
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

#### Şema İlk Yaklaşımı

Nest'te eşdeğer bir abonelik oluşturmak için `@Subscription()` dekoratörünü kullanacağız.

```typescript
const pubSub = new PubSub();

@Resolver('Author')
export class AuthorResolver {
  // ...
  @Subscription()
  commentAdded() {
    return pubSub.asyncIterator('commentAdded');
  }
}
```

Bağlam ve argümanlara dayalı olarak belirli olayları filtrelemek için `filter` özelliğini ayarlayın.

```typescript
@Subscription('commentAdded', {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

Yayınlanan yükü değiştirmek için bir `resolve` işlevini kullanabiliriz.

```typescript
@Subscription('commentAdded', {
  resolve: value => value,
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

Enjekte edilmiş sağlayıcılara erişmeniz gerekiyorsa (örneğin, verileri doğrulamak için harici bir servisi kullanmak), aşağıdaki yapısıyla kullanabilirsiniz:

```typescript
@Subscription('commentAdded', {
  resolve(this: AuthorResolver, value) {
    // "this", bir "AuthorResolver" örneğine referans verir
    return value;
  }
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

Aynı yapı filtrelerle de çalışır:

```typescript
@Subscription('commentAdded', {
  filter(this: AuthorResolver, payload, variables) {
    // "this", bir "AuthorResolver" örneğine referans verir
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

Son adım, tip tanımları dosyasını güncellemektir.

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

type Comment {
  id: String
  content: String
}

type Subscription {
  commentAdded(title: String!): Comment
}
```

Bu şekilde, `commentAdded(title: String!): Comment` adlı tek bir abonelik oluşturduk. Tam bir örnek uygulamayı [buradan](https://github.com/nestjs/nest/blob/master/sample/12-graphql-schema-first) bulabilirsiniz.

#### PubSub

Yukarıda yerel bir `PubSub` örneği oluşturduk. Tercih edilen yaklaşım, `PubSub`'ı bir [sağlayıcı](/docs/fundamentals/custom-providers) olarak tanımlamak ve gerekli yerlerde (`@Inject()` dekoratörünü kullanarak) enjekte etmektir. Bu, örneği tüm uygulama boyunca yeniden kullanmamıza olanak tanır. Örneğin, aşağıdaki gibi bir sağlayıcıyı tanımlayın ve gerektiğinde `'PUB_SUB'`'ı enjekte edin.

```typescript
{
  provide: 'PUB_SUB',
  useValue: new PubSub(),
}
```

#### Abonelik sunucusunu özelleştirme

Abonelik sunucusunu özelleştirmek için (örneğin, yolu değiştirmek) `subscriptions` seçenek özelliğini kullanın.

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'subscriptions-transport-ws': {
      path: '/graphql'
    },
  }
}),
```

Abonelikler için `graphql-ws` paketini kullanıyorsanız, `subscriptions-transport-ws` anahtarını `graphql-ws` ile değiştirin, aşağıdaki gibi:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'graphql-ws': {
      path: '/graphql'
    },
  }
}),
```

#### WebSockets üzerinden Kimlik Doğrulama

Kullanıcının kimlik doğrulamasını kontrol etmek, `subscriptions` seçeneklerinde belirtebileceğiniz `onConnect` geri çağrı fonksiyonu içinde yapılabilir.

`onConnect`, `SubscriptionClient`'a iletilen `connectionParams`'ı birinci argüman olarak alacaktır (daha fazla bilgi için [buraya](https://www.apollographql.com/docs/react/data/subscriptions/#5-authenticate-over-websocket-optional) bakın).

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'subscriptions-transport-ws': {
      onConnect: (connectionParams) => {
        const authToken = connectionParams.authToken;
        if (!isValid(authToken)) {
          throw new Error('Token is not valid');
        }
        // token'dan kullanıcı bilgilerini çıkar
        const user = parseToken(authToken);
        // kullanıcı bilgilerini daha sonra bağlamak için geri döndür
        return { user };
      },
    }
  },
  context: ({ connection }) => {
    // connection.context, "onConnect" geri çağrısı tarafından döndürülen değere eşit olacaktır
  },
}),
```

Bu örnekteki `authToken`, istemcinin bağlantıyı ilk kez kurduğunda yalnızca bir kez gönderilir.
Bu bağlantı ile yapılan tüm abonelikler aynı `authToken`'a sahip olacak ve bu nedenle aynı kullanıcı bilgilerine sahip olacaktır.

> ⚠️ **Not:** `subscriptions-transport-ws` paketinde, bağlantıların `onConnect` aşamasını atlamasına izin veren bir hata bulunmaktadır (daha fazla bilgi için [buraya](https://github.com/apollographql/subscriptions-transport-ws/issues/349) bakın). Kullanıcının bir aboneliği başlattığında her zaman `onConnect`'in çağrıldığını varsaymamalı ve her zaman `context`'in doldurulup doldurulmadığını kontrol etmelisiniz.

`graphql-ws` paketini kullanıyorsanız, `onConnect` geri çağrısının imzası biraz farklı olacaktır:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'graphql-ws': {
      onConnect: (context: Context<any>) => {
        const { connectionParams, extra } = context;
        // kullanıcı doğrulaması yukarıdaki örnekteki gibi kalacaktır
        // graphql-ws ile kullanırken, ek bağlam değeri extra alanında saklanmalıdır
        extra.user = { user: {} };
      },
    },
  },
  context: ({ extra }) => {
    // artık ek bağlam değerinize extra alanı üzerinden erişebilirsiniz
  },
});
```

**Mercurius Sürücüsü ile Abonelikleri Etkinleştirme**

Abonelikleri etkinleştirmek için, `subscription` özelliğini `true` olarak ayarlayın.

```typescript
GraphQLModule.forRoot<MercuriusDriverConfig>({
  driver: MercuriusDriver,
  subscription: true,
}),
```

> ℹ️ **İpucu:** Ayrıca özel bir yayıcıyı, gelen bağlantıları doğrulamayı vb. yapılandırmak için seçenekleri geçirebilirsiniz. Daha fazla bilgi için [buraya](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md#plugin-options) bakın (bkz. `subscription`).

#### Code First (Kod Odaklı)

Kod odaklı yaklaşımı kullanarak bir abonelik oluşturmak için, `@nestjs/graphql` paketinden ihraç edilen `@Subscription()` dekoratörünü ve `mercurius` paketinden sağlanan basit bir **yayın/abonelik API**'sı olan `PubSub` sınıfını kullanıyoruz.

Aşağıdaki abonelik işleyicisi, bir olaya **abone olma** işini `PubSub#asyncIterator`'ı çağırarak gerçekleştirir. Bu yöntem, birinci argümanı olan `triggerName` ile bir olay konu adına karşılık gelir.

```typescript
@Resolver((of) => Author)
export class AuthorResolver {
  // ...
  @Subscription((returns) => Comment)
  commentAdded(@Context('pubsub') pubSub: PubSub) {
    return pubSub.subscribe('commentAdded');
  }
}
```

> ℹ️ **İpucu:** Yukarıdaki örnekte kullanılan tüm dekoratörler `@nestjs/graphql` paketinden ihraç edilmektedir, `PubSub` sınıfı ise `mercurius` paketinden ihraç edilmektedir.

> ⚠️ **Not:** `PubSub`, basit bir `publish` ve `subscribe` API'sını açığa çıkaran bir sınıftır. Özel bir `PubSub` sınıfını kaydetme hakkında daha fazla bilgi için [bu bölüme](https://github.com/mercurius-js/mercurius/blob/master/docs/subscriptions.md#subscriptions-with-custom-pubsub) göz atın.

Bu, SDL'de aşağıdaki GraphQL şemasının bir kısmını üretecektir:

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

Aboneliklerin tanımı gereği, adı abonelik adı olan tek bir üst düzey özellik içeren bir nesneyi döndürür. Bu ad, abonelik işleyici yönteminin adından miras alınabilir (yani yukarıdaki örnekteki `commentAdded`), veya `@Subscription()` dekoratörüne ikinci argüman olarak `name` anahtarını geçirerek açıkça sağlanabilir, aşağıdaki gibi gösterildiği gibi.

```typescript
@Subscription(returns => Comment, {
  name: 'commentAdded',
})
subscribeToCommentAdded(@Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

Bu yapı, önceki kod örneğinden SDL'nin aynısını üretir, ancak aboneliği yöntem adından bağımsız hale getirmemize olanak tanır.

**Yayınlama**

Şimdi, etkinliği yayınlamak için `PubSub#publish` yöntemini kullanıyoruz. Bu genellikle bir mutasyon içinde kullanılır ve nesne grafiğinin bir kısmı değiştiğinde istemci tarafında bir güncelleme tetiklemek için kullanılır. Örneğin:

```typescript
@@filename(posts/posts.resolver)
@Mutation(returns => Post)
async addComment(
  @Args('postId', { type: () => Int }) postId: number,
  @Args('comment', { type: () => Comment }) comment: CommentInput,
  @Context('pubsub') pubSub: PubSub,
) {
  const newComment = this.commentsService.addComment({ id: postId, comment });
  await pubSub.publish({
    topic: 'commentAdded',
    payload: {
      commentAdded: newComment
    }
  });
  return newComment;
}
```

Bahsedildiği gibi, abonelik, tanım gereği bir değer döndürür ve bu değerin bir şekli vardır. `commentAdded` aboneliğimiz için oluşturulan SDL'ye tekrar bakalım:

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

Bu bize aboneliğin, değeri `commentAdded` adında üst düzey bir özellik içeren bir nesne döndürmesi gerektiğini söyler. Önemli olan nokta, `PubSub#publish` yöntemi tarafından gönderilen etkinlik payload'unun şeklinin, abonelikten dönmeyi beklenen değerin şekline uygun olması gerektiğidir. Bu nedenle yukarıdaki örnekte, `pubSub.publish({{ '{' }} topic: 'commentAdded', payload: {{ '{' }} commentAdded: newComment {{ '}' }} {{ '}' }})` ifadesi, şekil açısından uygun bir payload ile `commentAdded` etkinliğini yayınlar. Bu şekiller eşleşmezse, abonelik GraphQL doğrulama aşamasında başarısız olacaktır.

#### Abonelikleri Filtreleme

Belirli olayları filtrelemek için `filter` özelliğini bir filtre işlevine ayarlayın. Bu işlev, dizi `filter`'a iletilen işlev gibi davranır. İki argüman alır: olay payload'unu (etkinlik yayıncısı tarafından gönderilen) içeren `payload` ve abonelik isteği sırasında iletilen herhangi bir argümanı alan `variables`. Bu işlev, bu etkinliğin istemci dinleyicilerine gönderilip gönderilmeyeceğini belirleyen bir boole değeri döndürür.

```typescript
@Subscription(returns => Comment, {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded(@Args('title') title: string, @Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

Eğer enjekte edilmiş sağlayıcılara erişim sağlamanız gerekiyorsa (örneğin, veriyi doğrulamak için harici bir servis kullanma), aşağıdaki yapıyı kullanın.

```typescript
@Subscription(returns => Comment, {
  filter(this: AuthorResolver, payload, variables) {
    // "this" bir "AuthorResolver" örneğine referans yapar
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded(@Args('title') title: string, @Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

**Schema First**

Nest içinde eşdeğer bir abonelik oluşturmak için, `@Subscription()` dekoratörünü kullanacağız.

```typescript
const pubSub = new PubSub();

@Resolver('Author')
export class AuthorResolver {
  // ...
  @Subscription()
  commentAdded(@Context('pubsub') pubSub: PubSub) {
    return pubSub.subscribe('commentAdded');
  }
}
```

Bağlam ve argümanlara dayalı olarak belirli olayları filtrelemek için `filter` özelliğini ayarlayın.

```typescript
@Subscription('commentAdded', {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded(@Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

Enjekte edilmiş sağlayıcılara erişim sağlamanız gerekiyorsa (örneğin, veriyi doğrulamak için harici bir servis kullanma), aşağıdaki yapısı kullanılır.

```typescript
@Subscription('commentAdded', {
  filter(this: AuthorResolver, payload, variables) {
    // "this" bir "AuthorResolver" örneğine referans yapar
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded(@Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

Son adım, tip tanımları dosyasını güncellemektir.

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

type Comment {
  id: String
  content: String
}

type Subscription {
  commentAdded(title: String!): Comment
}
```

Bu şekilde, tek bir `commentAdded(title: String!): Comment` aboneliği oluşturduk.

### PubSub

Yukarıdaki örneklerde, varsayılan `PubSub` yayıcısını ([mqemitter](https://github.com/mcollina/mqemitter)) kullandık. Tercih edilen yaklaşım (prodüksiyon için) `mqemitter-redis` kullanmaktır. Ayrıca, özel bir `PubSub` uygulaması sağlanabilir (daha fazla bilgi için [buraya bakın](https://github.com/mercurius-js/mercurius/blob/master/docs/subscriptions.md)).

```typescript
GraphQLModule.forRoot<MercuriusDriverConfig>({
  driver: MercuriusDriver,
  subscription: {
    emitter: require('mqemitter-redis')({
      port: 6579,
      host: '127.0.0.1',
    }),
  },
});
```

### WebSockets Üzerinden Kimlik Doğrulama

Kullanıcının kimlik doğrulamasını kontrol etmek, `subscription` seçeneklerinde belirtilebilen `verifyClient` geri çağrı fonksiyonu içinde yapılabilir.

`verifyClient` fonksiyonu, `info` nesnesini birinci argüman olarak alır ve bu nesneyi kullanarak isteğin başlıklarını alabilirsiniz.

```typescript
GraphQLModule.forRoot<MercuriusDriverConfig>({
  driver: MercuriusDriver,
  subscription: {
    verifyClient: (info, next) => {
      const authorization = info.req.headers?.authorization as string;
      if (!authorization?.startsWith('Bearer ')) {
        return next(false);
      }
      next(true);
    },
  }
}),
```