### v10'dan v11'e Geçiş

Bu bölüm, `@nestjs/graphql` sürüm 10'dan sürüm 11'e geçiş için bir dizi kılavuz sağlar. Bu önemli sürümün bir parçası olarak, Apollo sürücüsünü Apollo Server v4 ile uyumlu hale getirdik (v3 yerine). Not: Apollo Server v4 çerçevesinde (özellikle eklentiler ve ekosistem paketleri etrafında) bir dizi önemli değişiklik bulunmaktadır, bu nedenle kod tabanınızı buna göre güncellemeniz gerekecektir. Daha fazla bilgi için [Apollo Server v4 geçiş rehberine](https://www.apollographql.com/docs/apollo-server/migration/) bakın.

#### Apollo paketleri

`apollo-server-express` paketini yüklemek yerine, artık `@apollo/server` paketini yüklemeniz gerekecek:

```bash
$ npm uninstall apollo-server-express
$ npm install @apollo/server
```

Fastify adaptörünü kullanıyorsanız, `@as-integrations/fastify` paketini yüklemeniz gerekecek:

```bash
$ npm uninstall apollo-server-fastify
$ npm install @apollo/server @as-integrations/fastify
```

#### Mercurius paketleri

Mercurius ağ geçidi artık `mercurius` paketinin bir parçası değil. Bunun yerine, `@mercuriusjs/gateway` paketini ayrıca yüklemeniz gerekecek:

```bash
$ npm install @mercuriusjs/gateway
```

Benzer şekilde, federasyon şemaları oluşturmak için `@mercuriusjs/federation` paketini yüklemeniz gerekecek:

```bash
$ npm install @mercuriusjs/federation
```

### v9'dan v10'a Geçiş

Bu bölüm, `@nestjs/graphql` sürüm 9'dan sürüm 10'a geçiş için bir dizi kılavuz sağlar. Bu major sürümün odak noktası, daha hafif, platformdan bağımsız bir çekirdek kitabı sağlamaktır.

#### "driver" Paketlerini Tanıtma

En son sürümde, `@nestjs/graphql` paketini Apollo (`@nestjs/apollo`), Mercurius (`@nestjs/mercurius`) veya projenizde başka bir GraphQL kütüphanesini kullanıp kullanmamaya karar verme olanağı sağlamak için bu paketi birkaç ayrı kütüphaneye bölmeye karar verdik.

Bu, şimdi uygulamanızın hangi sürücüyü kullanacağınızı açıkça belirtmeniz gerektiği anlamına gelir.

```typescript
// Önce
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot({
      autoSchemaFile: 'schema.gql',
    }),
  ],
})
export class AppModule {}

// Sonra
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: 'schema.gql',
    }),
  ],
})
export class AppModule {}
```

#### Eklentiler (Plugins)

Apollo Server eklentileri, belirli olaylara yanıt olarak özel işlemleri gerçekleştirmenize olanak tanır. Bu özel bir Apollo özelliği olduğu için, bunu `@nestjs/graphql` paketinden ayrılan yeni oluşturulan `@nestjs/apollo` paketine taşıdık, bu nedenle uygulamanızdaki içe aktarmaları güncellemeniz gerekecek.

```typescript
// Önce
import { Plugin } from '@nestjs/graphql';

// Sonra
import { Plugin } from '@nestjs/apollo';
```

#### Direktifler

`schemaDirectives` özelliği, `@graphql-tools/schema` paketinin v8 sürümünde yeni [Schema directives API](https://www.graphql-tools.com/docs/schema-directives) ile değiştirilmiştir.

```typescript
// Önce
import { SchemaDirectiveVisitor } from '@graphql-tools/utils';
import { defaultFieldResolver, GraphQLField } from 'graphql';

export class UpperCaseDirective extends SchemaDirectiveVisitor {
  visitFieldDefinition(field: GraphQLField<any, any>) {
    const { resolve = defaultFieldResolver } = field;
    field.resolve = async function (...args) {
      const result = await resolve.apply(this, args);
      if (typeof result === 'string') {
        return result.toUpperCase();
      }
      return result;
    };
  }
}

// Sonra
import { getDirective, MapperKind, mapSchema } from '@graphql-tools/utils';
import { defaultFieldResolver, GraphQLSchema } from 'graphql';

export function upperDirectiveTransformer(
  schema: GraphQLSchema,
  directiveName: string,
) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const upperDirective = getDirective(
        schema,
        fieldConfig,
        directiveName,
      )?.[0];

      if (upperDirective) {
        const { resolve = defaultFieldResolver } = fieldConfig;

        // Orjinal çözümleyiciyi, sonuçları büyük harfe çeviren bir işlevle *ilk* çağıran
        // orjinal çözümleyiciyi değiştir
        fieldConfig.resolve = async function (source, args, context, info) {
          const result = await resolve(source, args, context, info);
          if (typeof result === 'string') {
            return result.toUpperCase();
          }
          return result;
        };
        return fieldConfig;
      }
    },
  });
}
```

Bu direktif uygulamasını içeren bir şemaya `@upper` direktifleri eklemek için `transformSchema` işlevini kullanın:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  ...
  transformSchema: schema => upperDirectiveTransformer(schema, 'upper'),
})
```

#### Federasyon

`GraphQLFederationModule` kaldırıldı ve karşılık gelen sürücü sınıfıyla değiştirildi:

```typescript
// Önce
GraphQLFederationModule.forRoot({
  autoSchemaFile: true,
});

// Sonra
GraphQLModule.forRoot<ApolloFederationDriverConfig>({
  driver: ApolloFederationDriver,
  autoSchemaFile: true,
});
```

> info **Hint** Hem `ApolloFederationDriver` sınıfı hem de `ApolloFederationDriverConfig` `@nestjs/apollo` paketinden alınır.

Benzer şekilde, özel bir `GraphQLGatewayModule` kullanmak yerine, `GraphQLModule` ayarlarınıza uygun `driver` sınıfını geçirin:

```typescript
// Önce
GraphQLGatewayModule.forRoot({
  gateway: {
    supergraphSdl: new IntrospectAndCompose({
      subgraphs: [
        { name: 'users', url: 'http://localhost:3000/graphql' },
        { name: 'posts', url: 'http://localhost:3001/graphql' },
      ],
    }),
  },
});

// Sonra
GraphQLModule.forRoot<ApolloGatewayDriverConfig>({
  driver: ApolloGatewayDriver,
  gateway: {
    supergraphSdl: new IntrospectAndCompose({
      subgraphs: [
        { name: 'users', url: 'http://localhost:3000/graphql' },
        { name: 'posts', url: 'http://localhost:3001/graphql' },
      ],
    }),
  },
});
```

> info **Hint** Hem `ApolloGatewayDriver` sınıfı hem de `ApolloGatewayDriverConfig` `@nestjs/apollo` paketinden alınır.
