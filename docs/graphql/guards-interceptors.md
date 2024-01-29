### Diğer Özellikler

GraphQL dünyasında, **kimlik doğrulama**, veya işlemlerin **yan etkileri** gibi konuları ele alma konusunda birçok tartışma var. Bu tür konuları iş mantığı içinde mi halletmeliyiz? İşlemlere yetkilendirme mantığı eklemek için bir yüksek-sıra fonksiyon kullanmalı mıyız? Yoksa [şema direktifleri](https://www.apollographql.com/docs/apollo-server/schema/directives/) mi kullanmalıyız? Bu sorulara tek bir her duruma uyan bir cevap yok.

Nest, [guard](/docs/guards) ve [interceptor](/docs/interceptors) gibi çeşitli çapraz-platform özellikleri ile bu sorunları ele almaya yardımcı olur. Felsefe, tekrarı azaltmak ve iyi yapılandırılmış, okunabilir ve tutarlı uygulamalar oluşturmanıza yardımcı olan araçlar sağlamaktır.

#### Genel Bakış

GraphQL ile, standart [guard](/docs/guards), [interceptor](/docs/interceptors), [filter](/docs/exception-filters) ve [pipe](/docs/pipes) gibi araçları, bir RESTful uygulama ile aynı şekilde kullanabilirsiniz. Ayrıca, [custom decorators](/docs/custom-decorators) özelliğini kullanarak kendi dekoratörlerinizi kolayca oluşturabilirsiniz. Bir örnek GraphQL sorgu işleyicisine bir göz atalım.

```typescript
@Query('author')
@UseGuards(AuthGuard)
async getAuthor(@Args('id', ParseIntPipe) id: number) {
  return this.authorsService.findOneById(id);
}
```

Görüldüğü gibi, GraphQL, guard ve pipe'ları HTTP REST işleyicileriyle aynı şekilde kullanır. Bu nedenle, kimlik doğrulama mantığını bir güvenliğe taşıyabilirsiniz; hatta aynı guard sınıfını REST ve GraphQL API arayüzü üzerinde yeniden kullanabilirsiniz. Benzer şekilde, interceptor'lar her iki tür uygulama üzerinde aynı şekilde çalışır:

```typescript
@Mutation()
@UseInterceptors(EventsInterceptor)
async upvotePost(@Args('postId') postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

#### Yürütme Bağlamı

GraphQL, gelen isteğe farklı bir veri türü alır, bu nedenle hem guard'lar hem de interceptor'lar tarafından alınan yürütme bağlamı REST ile karşılaştırıldığında biraz farklıdır. GraphQL çözücülerinin belirli bir dizi argümanı vardır: `root`, `args`, `context` ve `info`. Bu nedenle guard'lar ve interceptor'lar, genel `ExecutionContext`'ı bir `GqlExecutionContext`'a dönüştürmelidir. Bu oldukça basittir:

```typescript
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const ctx = GqlExecutionContext.create(context);
    return true;
  }
}
```

`GqlExecutionContext.create()` tarafından döndürülen GraphQL bağlamı nesnesi, mevcut isteğin her GraphQL argümanını kolayca seçmemize olanak tanıyan `getArgs()`, `getContext()`, vb. için bir **get** yöntemi sunar.

#### İstisna Filtreleri

Nest standart [exception filters](/docs/exception-filters), GraphQL uygulamaları ile uyumludur. `ExecutionContext` gibi, GraphQL uygulamalarının `ArgumentsHost` nesnesini bir `GqlArgumentsHost` nesnesine dönüştürmesi gerekir.

```typescript
@Catch(HttpException)
export class HttpExceptionFilter implements GqlExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const gqlHost = GqlArgumentsHost.create(host);
    return exception;
  }
}
```

> ℹ️ **İpucu** Hem `GqlExceptionFilter` hem de `GqlArgumentsHost`, `@nestjs/graphql` paketinden içe aktarılır.

REST durumuyla karşılaştırıldığında, yanıt oluşturmak için doğrudan `response` nesnesini kullanmazsınız.

#### Özel Dekoratörler

Belirtildiği gibi, [custom decorators](/docs/custom-decorators) özelliği GraphQL çözücülerle beklenildiği gibi çalışır.

```typescript
export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) =>
    GqlExecutionContext.create(ctx).getContext().user,
);
```

`@User()` özel dekoratörünü şu şekilde kullanın:

```typescript
@Mutation()
async upvotePost(
  @User() user: UserEntity,
  @Args('postId') postId: number,
) {}
```

> ℹ️ **İpucu** Yukarıdaki örnekte, `user` nesnesinin GraphQL uygulamanızın bağlamına atanmış olduğunu varsaymış bulunmaktayız.

#### Field Resolver Seviyesinde Enhancer'ları Çalıştırma

GraphQL bağlamında, Nest, **enhancer'ları** (interceptor, guard ve filter için genel isim) alan düzeyinde çalıştırmaz [bu sorun için bakınız](https://github.com/nestjs/graphql/issues/320#issuecomment-511193229): yalnızca `@Query()`/`@Mutation()` metodları için en üst düzeyde çalışırlar. `@ResolveField()` ile işaretlenmiş metodlar için interceptor'ları, guard'ları veya filtreleri çalıştırması için Nest'e `GqlModuleOptions` içinde `fieldResolverEnhancers` seçeneğini ayarlamasını söyleyebilirsiniz. Ona uygun olarak `'interceptors'`, `'guards'` ve/veya `'filters'` listesini geçirin:

```typescript
GraphQLModule.forRoot({
  fieldResolverEnhancers: ['interceptors']
}),
```

> **Uyarı** Field resolver'lar için enhancer'ları etkinleştirmek, çok sayıda kayıt döndürüyorsanız ve field resolver'ınız binlerce kez çalıştırılıyorsa performans sorunlarına neden olabilir. Bu nedenle `fieldResolverEnhancers`'ı etkinleştirdiğinizde, field resolver'larınız için mutlaka gerekli olmayan enhancer'ların çalıştırılmasını atlamamanızı tavsiye ederiz. Bu işlemi aşağıdaki yardımcı işlevi kullanarak gerçekleştirebilirsiniz:

```typescript
export function isResolvingGraphQLField(context: ExecutionContext): boolean {
  if (context.getType<GqlContextType>() === 'graphql') {
    const gqlContext = GqlExecutionContext.create(context);
    const info = gqlContext.getInfo();
    const parentType = info.parentType.name;
    return parentType !== 'Query' && parentType !== 'Mutation';
  }
  return false;
}
```

#### Özel Sürücü Oluşturma

Nest, iki resmi sürücü sağlar: `@nestjs/apollo` ve `@nestjs/mercurius`, ayrıca geliştiricilere **custom drivers** oluşturmalarını sağlayan bir API sunar. Özel bir sürücü ile herhangi bir GraphQL kütüphanesini entegre edebilir veya mevcut entegrasyonu genişleterek üstüne ekstra özellikler ekleyebilirsiniz.

Örneğin, `express-graphql` paketini entegre etmek için aşağıdaki sürücü sınıfını oluşturabilirsiniz:

```typescript
import { AbstractGraphQLDriver, GqlModuleOptions } from '@nestjs/graphql';
import { graphqlHTTP } from 'express-graphql';

class ExpressGraphQLDriver extends AbstractGraphQLDriver {
  async start(options: GqlModuleOptions<any>): Promise<void> {
    options = await this.graphQlFactory.mergeWithSchema(options);

    const { httpAdapter } = this.httpAdapterHost;
    httpAdapter.use(
      '/graphql',
      graphqlHTTP({
        schema: options.schema,
        graphiql: true,
      }),
    );
  }

  async stop() {}
}
```

Ardından şu şekilde kullanabilirsiniz:

```typescript
GraphQLModule.forRoot({
  driver: ExpressGraphQLDriver,
});
```
