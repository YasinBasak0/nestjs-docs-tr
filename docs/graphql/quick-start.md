### TypeScript ve GraphQL'in Gücünden Faydalanma

[GraphQL](https://graphql.org/), API'lar için güçlü bir sorgu dili ve mevcut verilerle bu sorguları yerine getirmek için bir çalışma zamanıdır. REST API'larıyla tipik olarak bulunan birçok sorunu çözen zarif bir yaklaşımdır. Arka plan bilgisi için GraphQL ve REST arasındaki bu [karşılaştırmayı](https://www.apollographql.com/blog/graphql-vs-rest) okumanızı öneririz. TypeScript ile birleştirildiğinde, GraphQL sorgularınız için daha iyi tür güvenliği geliştirmenize yardımcı olur ve size uçtan uca yazma sağlar.

Bu bölümde, temel bir GraphQL anlayışını varsayıyoruz ve `@nestjs/graphql` modülüyle nasıl çalışılacağına odaklanıyoruz. `GraphQLModule`, [Apollo](https://www.apollographql.com/) sunucusunu (`@nestjs/apollo` sürücüsü ile) ve [Mercurius](https://github.com/mercurius-js/mercurius)'u (`@nestjs/mercurius` ile) kullanmak üzere yapılandırılabilir. Bu kanıtlanmış GraphQL paketleri için resmi entegrasyonları sağlayarak Nest ile GraphQL'i kullanmanın basit bir yolunu sağlar (daha fazla entegrasyon için [buraya](https://docs.nestjs.com/graphql/quick-start#third-party-integrations) bakın).

Ayrıca kendi özel sürücünüzü oluşturabilirsiniz (bununla ilgili daha fazla bilgi için [buraya](/docs/graphql/other-features#creating-a-custom-driver) bakın).

#### Kurulum

İlk olarak, gerekli paketleri yükleyerek başlayın:

```bash
# Express ve Apollo için (varsayılan)
$ npm i @nestjs/graphql @nestjs/apollo @apollo/server graphql

# Fastify ve Apollo için
# npm i @nestjs/graphql @nestjs/apollo @apollo/server @as-integrations/fastify graphql

# Fastify ve Mercurius için
# npm i @nestjs/graphql @nestjs/mercurius graphql mercurius
```

> warning **Uyarı** `@nestjs/graphql@>=9` ve `@nestjs/apollo^10` paketleri, **Apollo v3** ile uyumludur (daha fazla ayrıntı için Apollo Server 3 [geçiş kılavuzuna](https://www.apollographql.com/docs/apollo-server/migration/) bakın), `@nestjs/graphql@^8` sadece **Apollo v2**'yi destekler (örneğin, `apollo-server-express@2.x.x` paketi).

#### Genel Bakış

Nest, GraphQL uygulamaları oluşturmanın iki yolunu sunar, **code first** ve **schema first** yöntemleri. Size en uygun olanı seçmelisiniz. Bu GraphQL bölümündeki çoğu bölüm, **code first**'i benimseyenlerin takip etmeleri gereken ana kısım ve **schema first**'i benimseyenlerin kullanmaları gereken diğer kısım olmak üzere iki ana bölüme ayrılmıştır.

**Code first** yaklaşımında, dekoratörleri ve TypeScript sınıflarını kullanarak karşılık gelen GraphQL şemalarını oluşturursunuz. Bu yaklaşım, yalnızca TypeScript ile çalışmayı ve dil sözdizimleri arasında geçiş yapmaktan kaçınmayı tercih ediyorsanız faydalıdır.

**Schema first** yaklaşımında, gerçeklik kaynağı GraphQL SDL (Schema Definition Language - Şema Tanımlama Dili) dosyalarıdır. SDL, farklı platformlar arasında şema dosyalarını paylaşmak için dil bağımsız bir yoldur. Nest, GraphQL şemalarınıza dayalı TypeScript tanımlamalarınızı (sınıflar veya arabirimler kullanarak) otomatik olarak oluşturur ve gereksiz tekrarlanan kalıp kodu yazma ihtiyacını azaltır.

<app-banner-courses-graphql-cf></app-banner-courses-graphql-cf>

#### GraphQL ve TypeScript ile Başlarken

> info **İpucu** İlerleyen bölümlerde `@nestjs/apollo` paketini entegre edeceğiz. Bunun yerine `mercurius` paketini kullanmak istiyorsanız, [bu bölüme](/docs/graphql/quick-start#mercurius-integration) gidin.

Paketler yüklendikten sonra, `GraphQLModule`'u içe aktarabilir ve `forRoot()` statik yöntemi ile yapılandırabiliriz.

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
    }),
  ],
})
export class AppModule {}
```

> info **İpucu** `mercurius` entegrasyonu için `MercuriusDriver` ve `MercuriusDriverConfig` kullanmalısınız. Her ikisi de `@nestjs/mercurius` paketinden içe aktarılır.

`forRoot()` yöntemi bir seçenek nesnesini argüman olarak alır. Bu seçenekler, altındaki sürücü örneğine iletildiği için ilgili sürücü örneğine [burada](https://www.apollographql.com/docs/apollo-server/v2/api/apollo-server.html#constructor-options-lt-ApolloServer-gt) ve [burada](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md#plugin-options) mevcut ayarlar hakkında daha fazla bilgi edinebilirsiniz. Örneğin, Apollo için `playground`'ı devre dışı bırakmak ve `debug` modunu kapatmak istiyorsanız, şu seçenekleri iletebilirsiniz:

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver

, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      playground: false,
    }),
  ],
})
export class AppModule {}
```

Bu durumda, bu seçenekler `ApolloServer` kurucuya iletilecektir.

#### GraphQL Oyun Alanı (Playground)

Oyun alanı, varsayılan olarak GraphQL sunucusu ile aynı URL'de bulunan grafiksel, etkileşimli, tarayıcı tabanlı bir GraphQL IDE'dir. Oyun alanına erişmek için temel bir GraphQL sunucunuzun yapılandırılmış ve çalışıyor olması gerekir. Şu anda görmek istiyorsanız, [işleyen örnek buraya](https://github.com/nestjs/nest/tree/master/sample/23-graphql-code-first) yükleyebilir ve derleyebilirsiniz. Aksi takdirde, bu kod örneklerini takip ediyorsanız, [Çözücüler bölümünü](/docs/graphql/resolvers-map) tamamladıktan sonra oyun alanına erişebilirsiniz.

Bu ayarlandığında ve uygulamanız arka planda çalışırken, ardından web tarayıcınızı açabilir ve `http://localhost:3000/graphql` (konak ve bağlantı noktası yapılandırmanıza bağlı olarak değişebilir) adresine giderek GraphQL oyun alanını görebilirsiniz.

<figure>
  <img src="/assets/playground.png" alt="" />
</figure>

> warning **Not** `@nestjs/mercurius` entegrasyonu, yerleşik GraphQL Playground entegrasyonu ile gelmez. Bunun yerine [GraphiQL](https://github.com/graphql/graphiql)'i kullanabilirsiniz (`graphiql: true` olarak ayarlanmışsa).

#### Birden Fazla Uç Nokta

`@nestjs/graphql` modülünün başka bir yararlı özelliği, aynı anda birden çok uç noktayı hizmete sunma yeteneğidir. Bu size hangi modüllerin hangi uç noktalara dahil edileceğini belirleme olanağı sağlar. Varsayılan olarak, `GraphQL` çözücüleri uygulama genelinde tarar. Bu taramayı yalnızca belirli modüllerin alt kümesine sınırlamak için `include` özelliğini kullanın.

```typescript
GraphQLModule.forRoot({
  include: [CatsModule],
}),
```

> warning **Uyarı** Eğer tek bir uygulama içinde `@apollo/server`'ı `@as-integrations/fastify` paketiyle birlikte birden çok GraphQL uç noktası kullanıyorsanız, `GraphQLModule` yapılandırmasındaki `disableHealthCheck` ayarını etkinleştirmeyi unutmayın.

#### Code First

**Code first** yaklaşımında, dekoratörleri ve TypeScript sınıflarını kullanarak karşılık gelen GraphQL şemalarını oluşturursunuz.

Code first yaklaşımını kullanmak için, `autoSchemaFile` özelliğini seçenek nesnesine ekleyerek başlayın:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
}),
```

`autoSchemaFile` özelliği değeri, otomatik olarak oluşturulan şemanın oluşturulacağı yoludur. Alternatif olarak, şema bellekte canlı olarak oluşturulabilir. Bunu etkinleştirmek için `autoSchemaFile` özelliğini `true` olarak ayarlayın:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: true,
}),
```

Varsayılan olarak, oluşturulan şemadaki tipler, dahil edilen modüllerde tanımlandıkları sıra ile olacaktır. Şemayı leksikografik olarak sıralamak için `sortSchema` özelliğini `true` olarak ayarlayın:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
  sortSchema: true,
}),
```

#### Örnek

Tamamen çalışan bir code first örneği [burada](https://github.com/nestjs/nest/tree/master/sample/23-graphql-code-first) bulunabilir.

#### Schema First

Schema first yaklaşımını kullanmak için, önce `typePaths` özelliğini seçenekler nesnesine ekleyin. `typePaths` özelliği, GraphQL SDL şema tanımı dosyalarınızın nereye bakılacağını gösterir. Bu dosyalar bellekte birleştirilecek; bu, şemalarınızı çözümleyicilerine yakın bir konumda birkaç dosyaya bölmeye ve bulmaya olanak tanır.

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
}),
```

Genellikle ayrıca, GraphQL SDL tiplerine karşılık gelen TypeScript tanımlarına (sınıflar ve arayüzler) ihtiyacınız olacaktır. Karşılık gelen TypeScript tanımlarını elle oluşturmak tekrarlı ve sıkıcıdır. Bize bir tek doğrulama kaynağı bırakmaz - SDL içinde yapılan her değişiklik, TypeScript tanımlarını da ayarlamamızı gerektirir. Bu sorunu çözmek için, `@nestjs/graphql` paketi, abstract syntax tree ([AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree)) üzerinden TypeScript tanımlarını **otomatik olarak oluşturabilir**. Bu özelliği etkinleştirmek için, `GraphQLModule`'u yapılandırırken `definitions` seçenek özelliğini ekleyin.

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
  },
}),
```

`definitions` nesnesinin `path` özelliği, oluşturulan TypeScript çıktısını nereye kaydedeceğimizi gösterir. Varsayılan olarak, oluşturulan tüm TypeScript tipleri arayüz olarak oluşturulur. Bunun yerine sınıflar oluşturmak için `outputAs` özelliğini `'class'` değeri ile belirtin.

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
    outputAs: 'class',
  },
}),
```

Yukarıdaki yaklaşım uygulama başladığında dinamik olarak TypeScript tanımlarını oluşturur. Alternatif olarak, bunları isteğe bağlı olarak oluşturan basit bir komut dosyası oluşturmak da tercih edilebilir. Örneğin, aşağıdaki komut dosyasını `generate-typings.ts` olarak oluşturduğumuzu varsayalım:

```typescript
import { GraphQLDefinitionsFactory } from '@nestjs/graphql';
import { join } from 'path';

const definitionsFactory = new GraphQLDefinitionsFactory();
definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
});
```

Şimdi bu komut dosyasını isteğe bağlı olarak çalıştırabilirsiniz:

```bash
$ ts-node generate-typings
```

> info **İpucu** Komut dosyasını önceden derleyebilirsiniz (örneğin, `tsc` ile) ve `node` kullanarak çalıştırabilirsiniz.

Komut dosyası için izleme modunu etkinleştirmek için (herhangi bir `.graphql` dosyası değiştiğinde otomatik olarak yazı tiplerini oluşturmak için), `watch` seçeneğini `generate()` yöntemine iletin.

```typescript
definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
  watch: true,
});
```

Her nesne tipi için ek `__typename` alanını otomatik olarak oluşturmak için `emitTypenameField` seçeneğini etkinleştirin.

```typescript
definitionsFactory.generate({
  // ...,
  emitTypenameField: true,
});
```

Sorgu çözücülerini (queries), mutasyonları (mutations) ve abonelikleri (subscriptions) sadece alanlar olarak, argümanlar olmadan oluşturmak için `skipResolverArgs` seçeneğini etkinleştirin.

```typescript
definitionsFactory.generate({
  // ...,
  skipResolverArgs: true,
});
```

#### Apollo Sandbox

Yerel geliştirmeler için bir GraphQL IDE olarak [Apollo Sandbox](https://www.apollographql.com/blog/announcement/platform/apollo-sandbox-an-open-graphql-ide-for-local-development/) kullanmak istiyorsanız, aşağıdaki yapılandırmayı kullanın:

```typescript
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloServerPluginLandingPageLocalDefault } from '@apollo/server/plugin/landingPage/default';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      playground: false,
      plugins: [ApolloServerPluginLandingPageLocalDefault()],
    }),
  ],
})
export class AppModule {}
```

#### Örnek

Tamamen çalışan bir schema first örneği [burada](https://github.com/nestjs/nest/tree/master/sample/12-graphql-schema-first) bulunabilir.

#### Oluşturulan şemaya erişme

Bazı durumlarda (örneğin, uçtan uca testlerde), oluşturulan şema nesnesine bir referans almak isteyebilirsiniz. Uçtan uca testlerde, ardından HTTP dinleyicilerini kullanmadan `graphql` nesnesini kullanarak sorguları çalıştırabilirsiniz.

Oluşturulan şemaya (hem code first hem de schema first yaklaşımında) `GraphQLSchemaHost` sınıfını kullanarak erişebilirsiniz:

```typescript
const { schema } = app.get(GraphQLSchemaHost);
```

> info **İpucu** `GraphQLSchemaHost#schema` getter'ını uygulama başlatıldıktan sonra ( `app.listen()` veya `app.init()` yöntemiyle tetiklenen `onModuleInit` kancası tarafından) çağırmanız gerekir.

#### Asenkron yapılandırma

Modül seçeneklerini statik olarak değil de asenkron olarak iletmek istediğinizde, `forRootAsync()` yöntemini kullanın. Çoğu dinamik modül gibi, Nest asenkron yapılandırma ile başa çıkmak için birkaç teknik sunar.

Bir teknik, bir fabrika işlevi kullanmaktır:

```typescript
 GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  useFactory: () => ({
    typePaths: ['./**/*.graphql'],
  }),
}),
```

Diğer fabrika sağlayıcıları gibi, fabrika işlevimiz <a href="https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory">asenkron</a> olabilir ve `inject` aracılığıyla bağımlılıkları enjekte edebilir.

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    typePaths: configService.get<string>('GRAPHQL_TYPE_PATHS'),
  }),
  inject: [ConfigService],
}),
```

Alternatif olarak, bir sınıf yerine bir fabrika kullanarak `GraphQLModule`'u yapılandırabilirsiniz, aşağıda gösterildiği gibi:

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  useClass: GqlConfigService,
}),
```

Yukarıdaki yapı, `GqlConfigService`'i `GraphQLModule` içinde örnekleyerek, onu kullanarak bir seçenek nesnesi oluşturur. Bu örnekte dikkat edilmesi gereken bir husus, `GqlConfigService`'in `GqlOptionsFactory` arabirimini uygulamasıdır. `GraphQLModule`, sağlanan sınıfın örneği üzerinde `createGqlOptions()` yöntemini çağırır.

```typescript
@Injectable()
class GqlConfigService implements GqlOptionsFactory {
  createGqlOptions(): ApolloDriverConfig {
    return {
      typePaths: ['./**/*.graphql'],
    };
  }
}
```

`GraphQLModule` içinde özel bir kopya oluşturmak yerine mevcut bir seçenek sağlayıcısını yeniden kullanmak istiyorsanız, `useExisting` sözdizimini kullanın.

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  imports: [ConfigModule],
  useExisting: ConfigService,
}),
```

#### Mercurius Entegrasyonu

Apollo yerine, Fastify kullanıcıları (daha fazla bilgi için [buraya](/docs/techniques/performance) bakın) alternatif olarak `@nestjs/mercurius` sürücüsünü kullanabilirler.

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { MercuriusDriver, MercuriusDriverConfig } from '@nestjs/mercurius';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusDriverConfig>({
      driver: MercuriusDriver,
      graphiql: true,
    }),
  ],
})
export class AppModule {}
```

> bilgi **İpucu** Uygulama çalıştığında, tarayıcınızı açın ve `http://localhost:3000/graphiql` adresine gidin. [GraphQL IDE](https://github.com/graphql/graphiql)'yi görmelisiniz.

`forRoot()` yöntemi bir seçenek nesnesini argüman olarak alır. Bu seçenekler, altta yatan sürücü örneğine iletilir. Kullanılabilir ayarlar hakkında daha fazla bilgiyi [buradan](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md#plugin-options) edinin.

#### Üçüncü Taraf Entegrasyonları

- [GraphQL Yoga](https://github.com/charlypoly/graphql-yoga-nestjs)

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/33-graphql-mercurius) bulunabilir.
