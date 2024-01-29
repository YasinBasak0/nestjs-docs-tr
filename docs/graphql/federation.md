### Federasyon

Federasyon, monolitik GraphQL sunucunuzu bağımsız mikro servislere ayırmak için bir yöntem sunar. İki bileşenden oluşur: bir ağ geçidi (gateway) ve bir veya daha fazla federasyon mikro servisi. Her mikro servis, şemanın bir kısmını içerir ve ağ geçidi şemaları birleştirerek istemci tarafından tüketilebilen tek bir şemaya dönüştürür.

[Apollo belgelerinden](https://blog.apollographql.com/apollo-federation-f260cf525d21) alıntı yapmak gerekirse, Federasyon şu temel prensiplerle tasarlanmıştır:

- Bir grafik oluşturmak **deklaratif** olmalıdır. Federasyon ile şemadan içeriden deklaratif bir şekilde bir grafik oluşturabilirsiniz, zorunlu şema birleştirme kodu yazmak yerine.
- Kod, **ilgi** ne göre ayrılmalıdır, türler ne göre değil. Genellikle önemli bir türün her yönünü tek bir ekip kontrol etmez, bu nedenle bu türlerin tanımı, ekipler ve kod tabanları arasında dağıtılmalı, merkezi değil.
- Grafik, istemcilerin tüketmesi için basit olmalıdır. Federasyon hizmetleri bir araya geldiğinde, tamamlayıcı, ürün odaklı bir grafik oluşturabilir ve istemcide nasıl tüketildiğini doğru bir şekilde yansıtabilir.
- Sadece **GraphQL** kullanılmalıdır, dilin spesifikasyona uygun özellikleri kullanılmalıdır. Sadece JavaScript değil, herhangi bir dil, federasyonu uygulayabilir.

> uyarı **Uyarı** Şu anda Federasyon, abonelikleri desteklememektedir.

Aşağıdaki bölümlerde, bir ağ geçidi ve iki federasyonlu uç nokta içeren bir demo uygulaması oluşturacağız: Kullanıcılar servisi ve Gönderiler servisi.

#### Apollo ile Federasyon

İlk olarak, gerekli bağımlılıkları yükleyerek başlayın:

```bash
$ npm install --save @apollo/federation @apollo/subgraph
```

#### Şema Tabanlı

"Kullanıcı servisi", basit bir şema sağlar. `@key` direktifine dikkat edin: bu, Apollo sorgu planlayıcısına belirli bir `id` belirtildiğinde belirli bir `User` örneğini alabileceğini söyler. Ayrıca, `Query` türünü `extend` ettiğimize dikkat edin.

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

extend type Query {
  getUser(id: ID!): User
}
```

Çözücü, `resolveReference()` adında bir ek yöntem sağlar. Bu yöntem, Apollo Gateway herhangi bir ilgili kaynağın bir Kullanıcı örneği gerektirdiğinde tetiklenir. Bu örneği daha sonra Gönderiler servisinde göreceğiz. Lütfen bu yöntemin `@ResolveReference()` dekoratörü ile işaretlenmesi gerektiğini unutmayın.

```typescript
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { UsersService } from './users.service';

@Resolver('User')
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query()
  getUser(@Args('id') id: string) {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: string }) {
    return this.usersService.findById(reference.id);
  }
}
```

Son olarak, her şeyi bağlamak için `GraphQLModule`'u kaydederek yapılandırma nesnesinde `ApolloFederationDriver` sürücüsünü geçerek:

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { UsersResolver } from './users.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [UsersResolver],
})
export class AppModule {}
```

#### Kod Tabanlı

İlk olarak, `User` varlığına bazı ek dekoratörler ekleyerek başlayın.

```typescript
import { Directive, Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field((type) => ID)
  id: number;

  @Field()
  name: string;
}
```

Çözücü, `resolveReference()` adında bir ek yöntem sağlar. Bu yöntem, Apollo Gateway herhangi bir ilgili kaynağın bir Kullanıcı örneği gerektirdiğinde tetiklenir. Bu örneği daha sonra Gönderiler servisinde göreceğiz. Lütfen bu yöntemin `@ResolveReference()` dekoratörü ile işaretlenmesi gerektiğini unutmayın.

```typescript
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { User } from './user.entity';
import { UsersService } from './users.service';

@Resolver((of) => User)
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query((returns) => User)
  getUser(@Args('id') id: number): User {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: number }): User {
    return this.usersService.findById(reference.id);
  }
}
```

Son olarak, her şeyi bağlamak için `GraphQLModule`'u kaydederek yapılandırma nesnesinde `ApolloFederationDriver` sürücüsünü geçerek:

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // Bu örneğe dahil edilmemiştir

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: true,
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

Kod tabanlı modda çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/users-application) ve şema tabanlı modda çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/users-application) bulunmaktadır.

#### Federasyon Örneği: Gönderiler

Gönderi servisi, `getPosts` sorgusu aracılığıyla toplu gönderiler sunmayı amaçlamaktadır, aynı zamanda `User` türümüzü `user.posts` alanıyla genişletmeyi hedefler.

#### Şema Tabanlı

"Gönderiler servisi", şemasında `extend` anahtar kelimesi ile `User` türüne referans verir ve aynı zamanda `User` türüne (`posts`) ek bir özellik ekler. User türünün örneklerini eşleştirmek için kullanılan `@key` direktifine ve `id` alanının başka bir yerde yönetildiğini belirten `@external` direktifine dikkat edin.

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post]
}

extend type Query {
  getPosts: [Post]
}
```

Aşağıdaki örnekte, `PostsResolver`'de, `getUser()` yöntemini sağlar, bu yöntem bir referans içeren `__typename` ve uygulamanızın referansı çözmek için ihtiyaç duyabileceği bazı ek özellikleri (`id` in bu durumda) döndürür. `__typename`, GraphQL Ağ Geçidi tarafından User türünden sorumlu mikro servisi belirlemek ve ilgili örneği almak için kullanılır. Yukarıda açıklanan "Kullanıcı servisi"nin `resolveReference()` yöntemi çağrıldığında talep edilecektir.

```typescript
import { Query, Resolver, Parent, ResolveField } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './posts.interfaces';

@Resolver('Post')
export class PostsResolver {
  constructor(private postsService: PostsService) {}

  @Query('getPosts')
  getPosts() {
    return this.postsService.findAll();
  }

  @ResolveField('user')
  getUser(@Parent() post: Post) {
    return { __typename: 'User', id: post.userId };
  }
}
```

Son olarak, `GraphQLModule`'u, yukarıdaki "Kullanıcı servisi" bölümünde yaptığımız gibi kaydetmeliyiz.

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { PostsResolver } from './posts.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [PostsResolver],
})
export class AppModule {}
```

#### Kod Tabanlı

İlk olarak, `User` varlığını temsil eden bir sınıfı bildirmemiz gerekecek. Varlık kendisi başka bir serviste bulunsa da, burada kullanacağız (tanımını genişleterek). `@extends` ve `@external` direktiflerine dikkat edin.

```typescript
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@extends')
@Directive('@key(fields: "id")')
export class User {
  @Field((type) => ID)
  @Directive('@external')
  id: number;

  @Field((type) => [Post])
  posts?: Post[];
}
```

Şimdi, `User` varlığımızın genişletilmesi üzerindeki çözücümüzü aşağıdaki gibi oluşturalım:

```typescript
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver((of) => User)
export class UsersResolver {
  constructor(private readonly postsService: PostsService) {}

  @ResolveField((of) => [Post])
  public posts(@Parent() user: User): Post[] {
    return this.postsService.forAuthor(user.id);
  }
}
```

Ayrıca, `Post` varlık sınıfımızı tanımlamamız gerekiyor:

```typescript
import { Directive, Field, ID, Int, ObjectType } from '@nestjs/graphql';
import { User } from './user.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class Post {
  @Field((type) => ID)
  id: number;

  @Field()
  title: string;

  @Field((type) => Int)
  authorId: number;

  @Field((type) => User)
  user?: User;
}
```

Ve onun çözücüsü:

```typescript
import { Query, Args, ResolveField, Resolver, Parent } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver((of) => Post)
export class PostsResolver {
  constructor(private readonly postsService: PostsService) {}

  @Query((returns) => Post)
  findPost(@Args('id') id: number): Post {
    return this.postsService.findOne(id);
  }

  @Query((returns) => [Post])
  getPosts(): Post[] {
    return this.postsService.all();
  }

  @ResolveField((of) => User)
  user(@Parent() post: Post): any {
    return { __typename: 'User', id: post.authorId };
  }
}
```

Ve son olarak, modülde birleştirelim. `User`'ın yetim (external) bir tip olduğunu belirttiğimiz şema derleme seçeneklerine dikkat edin.

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // Bu örneğe dahil edilmemiştir

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: true,
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```

Kod tabanlı modda çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/posts-application) ve şema tabanlı modda çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/posts-application) bulunmaktadır.

#### Federasyon Örneği: Gateway

İlk olarak gerekli bağımlılığı yükleyerek başlayın:

```bash
$ npm install --save @apollo/gateway
```

Ağ geçidi, belirtilen bir dizi uca ihtiyaç duyar ve karşılık gelen şemaları otomatik olarak keşfeder. Bu nedenle, ağ geçidi servisinin uygulaması kod tabanlı ve şema tabanlı yaklaşımlar için aynı kalacaktır.

```typescript
import { IntrospectAndCompose } from '@apollo/gateway';
import { ApolloGatewayDriver, ApolloGatewayDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloGatewayDriverConfig>({
      driver: ApolloGatewayDriver,
      server: {
        // ... Apollo server options
        cors: true,
      },
      gateway: {
        supergraphSdl: new IntrospectAndCompose({
          subgraphs: [
            { name: 'users', url: 'http://user-service/graphql' },
            { name: 'posts', url: 'http://post-service/graphql' },
          ],
        }),
      },
    }),
  ],
})
export class AppModule {}
```

Kod tabanlı mod için [burada](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/gateway) ve şema tabanlı mod için [burada](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/gateway) çalışan bir örnek bulunmaktadır.

#### Mercurius ile Federasyon

Gerekli bağımlılıkları yükleyerek başlayın:

```bash
$ npm install --save @apollo/subgraph @nestjs/mercurius
```

> info **Not** `@apollo/subgraph` paketi, bir alt grafik şeması oluşturmak için gereklidir (`buildSubgraphSchema`, `printSubgraphSchema` fonksiyonları).

#### Şema Tabanlı

"Kullanıcı servisi", basit bir şema sağlar. `@key` direktifine dikkat edin: belirli bir `User` örneğini almak istiyorsanız, `id`'sini belirtirseniz alınabilir. Ayrıca, `Query` türünü `extend` ettiğimize dikkat edin.

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

extend type Query {
  getUser(id: ID!): User
}
```

Çözücü, `resolveReference()` adlı bir ek yöntem sağlar. Bu yöntem, Mercurius Gateway tarafından ilişkili bir kaynak, bir User örneği gerektirdiğinde tetiklenir. Bu yöntemin, `@ResolveReference()` dekoratörü ile işaretlenmiş olması gerektiğine dikkat edin.

```typescript
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { UsersService } from './users.service';

@Resolver('User')
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query()
  getUser(@Args('id') id: string) {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: string }) {
    return this.usersService.findById(reference.id);
  }
}
```

Son olarak, `GraphQLModule`'u kaydederek her şeyi birleştiriyoruz ve yapılandırma nesnesinde `MercuriusFederationDriver` sürücüsünü geçiriyoruz:

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { UsersResolver } from './users.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      typePaths: ['**/*.graphql'],
      federationMetadata: true,
    }),
  ],
  providers: [UsersResolver],
})
export class AppModule {}
```

#### Kod Tabanlı

İlk olarak, `User` varlığına ek dekoratörler ekleyerek başlayın.

```ts
import { Directive, Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field((type) => ID)
  id: number;

  @Field()
  name: string;
}
```

Çözücü, `resolveReference()` adlı bir ek yöntem sağlar. Bu yöntem, Mercurius Gateway tarafından ilişkili bir kaynak, bir User örneği gerektirdiğinde tetiklenir. Bu yöntemin, `@ResolveReference()` dekoratörü ile işaretlenmiş olması gerektiğine dikkat edin.

```ts
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { User } from './user.entity';
import { UsersService } from './users.service';

@Resolver((of) => User)
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query((returns) => User)
  getUser(@Args('id') id: number): User {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: number }): User {
    return this.usersService.findById(reference.id);
  }
}
```

Son olarak, `GraphQLModule`'u kaydederek her şeyi birleştiriyoruz ve yapılandırma nesnesinde `MercuriusFederationDriver` sürücüsünü geçiriyoruz:

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // Bu örnekte dahil edilmemiştir

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      autoSchemaFile: true,
      federationMetadata: true,
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

#### Federasyon Örneği: Postlar

Post servisi, `getPosts` sorgusu aracılığıyla toplu gönderiler sağlamakla kalmayacak, aynı zamanda `User` türümüzü `user.posts` alanıyla genişletecek.

#### Şema Tabanlı

"Post servisi", şemasında `User` türüne başka bir `extend` anahtar kelimesi ile referans yapar. Ayrıca `User` türü üzerinde bir ek özellik (`posts`) bildirir. User örneklerini eşleştirmek için kullanılan `@key` direktifi ve `id` alanının başka bir yerde yönetildiğini belirten `@external` direktifi dikkat çeker.

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post]
}

extend type Query {
  getPosts: [Post]
}
```

Aşağıdaki örnekte, `PostsResolver` sınıfı, `__typename` ve bu referansı çözme ihtiyacınız olabilecek ek özellikleri içeren bir referans döndüren `getUser()` yöntemini sağlar. `__typename`, GraphQL Gateway tarafından, User türünden sorumlu mikroservisi işaret etmek ve karşılık gelen örneği almak için kullanılır. Yukarıda açıklanan "Kullanıcı servisi", `resolveReference()` yöntemi çalıştırıldığında istenecektir.

```typescript
import { Query, Resolver, Parent, ResolveField } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './posts.interfaces';

@Resolver('Post')
export class PostsResolver {
  constructor(private postsService: PostsService) {}

  @Query('getPosts')
  getPosts() {
    return this.postsService.findAll();
  }

  @ResolveField('user')
  getUser(@Parent() post: Post) {
    return { __typename: 'User', id: post.userId };
  }
}
```

Son olarak, `GraphQLModule`'u kaydederek, "Kullanıcı servisi" bölümünde yaptığımız gibi benzer bir şekilde kayıt yapmamız gerekmektedir.

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { PostsResolver } from './posts.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      federationMetadata: true,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [PostsResolvers],
})
export class AppModule {}
```

#### Code First

İlk olarak, `User` varlığını temsil eden bir sınıf bildirmemiz gerekecek. Varlık kendisi başka bir serviste bulunsa da, burada (tanımını genişleterek) kullanacağız. `@extends` ve `@external` direktiflerine dikkat edin.

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@extends')
@Directive('@key(fields: "id")')
export class User {
  @Field((type) => ID)
  @Directive('@external')
  id: number;

  @Field((type) => [Post])
  posts?: Post[];
}
```

Şimdi, `User` varlığımızdaki bu genişletme için karşılık gelen çözücüyü oluşturalım:

```ts
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver((of) => User)
export class UsersResolver {
  constructor(private readonly postsService: PostsService) {}

  @ResolveField((of) => [Post])
  public posts(@Parent() user: User): Post[] {
    return this.postsService.forAuthor(user.id);
  }
}
```

Ayrıca, `Post` varlık sınıfını tanımlamamız gerekiyor:

```ts
import { Directive, Field, ID, Int, ObjectType } from '@nestjs/graphql';
import { User } from './user.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class Post {
  @Field((type) => ID)
  id: number;

  @Field()
  title: string;

  @Field((type) => Int)
  authorId: number;

  @Field((type) => User)
  user?: User;
}
```

Ve onun çözücüsü:

```ts
import { Query, Args, ResolveField, Resolver, Parent } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver((of) => Post)
export class PostsResolver {
  constructor(private readonly postsService: PostsService) {}

  @Query((returns) => Post)
  findPost(@Args('id') id: number): Post {
    return this.postsService.findOne(id);
  }

  @Query((returns) => [Post])
  getPosts(): Post[] {
    return this.postsService.all();
  }

  @ResolveField((of) => User)
  user(@Parent() post: Post): any {
    return { __typename: 'User', id: post.authorId };
  }
}
```

Ve son olarak, bunu bir modülde birleştirelim. `User`'ın öksüz (external) bir tür olduğunu belirttiğimiz şema oluşturma seçeneklerine dikkat edin.

```ts
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // Örneğe dahil edilmemiştir

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      autoSchemaFile: true,
      federationMetadata: true,
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```

#### Federasyon Örneği: Gateway

Gateway (Ara Birim) belirli bir listeye sahip olmalıdır ve ilgili şemaları otomatik olarak keşfedecektir. Bu nedenle gateway servisinin uygulanması, hem kod bazlı hem de şema bazlı yaklaşımlar için aynı kalacaktır.

```typescript
import {
  MercuriusGatewayDriver,
  MercuriusGatewayDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusGatewayDriverConfig>({
      driver: MercuriusGatewayDriver,
      gateway: {
        services: [
          { name: 'users', url: 'http://user-service/graphql' },
          { name: 'posts', url: 'http://post-service/graphql' },
        ],
      },
    }),
  ],
})
export class AppModule {}
```

#### Federasyon 2

[Apollo Federation 2 belgelerinden](https://www.apollographql.com/docs/federation/federation-2/new-in-federation-2) alıntı yapacak olursak, Federation 2, geliştirici deneyimini orijinal Apollo Federation'dan (bu belgede Federation 1 olarak adlandırılır) iyileştirir ve çoğu orijinal supergraph ile geriye dönük uyumludur.

> ⚠️ **Uyarı** Mercurius, Federation 2'yi tam olarak desteklememektedir. Federation 2'yi destekleyen kütüphanelerin listesini [buradan](https://www.apollographql.com/docs/federation/supported-subgraphs#javascript--typescript) görebilirsiniz.

Aşağıdaki bölümlerde, önceki örneği Federation 2'ye yükselteceğiz.

#### Federasyon Örneği: Kullanıcılar

Federation 2'deki bir değişiklik, varlıkların kaynak alt grafiğe sahip olmamasıdır, bu nedenle artık `Query`'yi genişletmemize gerek yoktur. Daha fazla ayrıntı için lütfen [Apollo Federation 2 belgelerindeki entities konusuna](https://www.apollographql.com/docs/federation/federation-2/new-in-federation-2#entities) başvurun.

#### Şema İlk Olarak

Şemadan sadece `extend` anahtar kelimesini kaldırabiliriz.

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

type Query {
  getUser(id: ID!): User
}
```

#### Kod İlk Olarak

Federation 2'yi kullanmak için `autoSchemaFile` seçeneğinde federasyon sürümünü belirtmemiz gerekmektedir.

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // Bu örnekte yer almamıştır

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: {
        federation: 2,
      },
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

#### Federasyon Örneği: Gönderiler

Yukarıda belirtildiği gibi, artık `User` ve `Query`'yi genişletmemiz gerekmiyor.

#### Şema İlk Olarak

Şemadan sadece `extend` ve `external` direktiflerini kaldırabiliriz.

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

type User @key(fields: "id") {
  id: ID!
  posts: [Post]
}

type Query {
  getPosts: [Post]
}
```

#### Kod İlk Olarak

Artık `User` varlığını genişletmediğimizden, `User`'daki `extends` ve `external` direktiflerini sadece kaldırabiliriz.

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field((type) => ID)
  id: number;

  @Field((type) => [Post])
  posts?: Post[];
}
```

Ayrıca, User servisi için olduğu gibi, `GraphQLModule` içinde Federation 2'yi kullanmasını belirtmemiz gerekmektedir.

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // Bu örnekte yer almamıştır

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: {
        federation: 2,
      },
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```
