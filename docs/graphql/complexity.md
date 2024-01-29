### Complexity

Karmaşıklık (complexity), belirli alanların ne kadar karmaşık olduğunu tanımlamanıza ve bir **maksimum karmaşıklık** ile sorguları sınırlamanıza olanak tanır. Her bir alanın karmaşıklığını belirlemek için basit bir sayı kullanarak her alanın karmaşıklığını tanımlamaktır. Genellikle, her bir alana varsayılan olarak `1` karmaşıklık vermek yaygındır. Ayrıca, GraphQL sorgusunun karmaşıklık hesaplaması, karmaşıklık tahmincileri olarak adlandırılan işlevlerle özelleştirilebilir. Bir karmaşıklık tahmincisi, bir alan için karmaşıklığı hesaplayan basit bir işlevdir. Kurala her seferinde sırayla çalıştırılacak birden çok karmaşıklık tahmincisi ekleyebilirsiniz. İlk sayısal karmaşıklık değerini döndüren ilk tahminci, o alanın karmaşıklığını belirler.

`@nestjs/graphql` paketi, maliyet analizine dayalı bir çözüm sunan [graphql-query-complexity](https://github.com/slicknode/graphql-query-complexity) gibi araçlarla çok iyi entegre olmuştur. Bu kütüphane ile, GraphQL sunucunuza maliyeti yüksek olduğu düşünülen sorguları reddedebilirsiniz.

#### Kurulum

Kullanmaya başlamak için önce gerekli bağımlılığı yükleriz.

```bash
$ npm install --save graphql-query-complexity
```

#### Başlangıç

Kurulum işlemi tamamlandığında, `ComplexityPlugin` sınıfını tanımlayabiliriz:

```typescript
import { GraphQLSchemaHost } from "@nestjs/graphql";
import { Plugin } from "@nestjs/apollo";
import {
  ApolloServerPlugin,
  GraphQLRequestListener,
} from 'apollo-server-plugin-base';
import { GraphQLError } from 'graphql';
import {
  fieldExtensionsEstimator,
  getComplexity,
  simpleEstimator,
} from 'graphql-query-complexity';

@Plugin()
export class ComplexityPlugin implements ApolloServerPlugin {
  constructor(private gqlSchemaHost: GraphQLSchemaHost) {}

  async requestDidStart(): Promise<GraphQLRequestListener> {
    const maxComplexity = 20;
    const { schema } = this.gqlSchemaHost;

    return {
      async didResolveOperation({ request, document }) {
        const complexity = getComplexity({
          schema,
          operationName: request.operationName,
          query: document,
          variables: request.variables,
          estimators: [
            fieldExtensionsEstimator(),
            simpleEstimator({ defaultComplexity: 1 }),
          ],
        });
        if (complexity > maxComplexity) {
          throw new GraphQLError(
            `Query is too complex: ${complexity}. Maximum allowed complexity: ${maxComplexity}`,
          );
        }
        console.log('Query Complexity:', complexity);
      },
    };
  }
}
```

Demonstrasyon amaçlı olarak, maksimum izin verilen karmaşıklığı `20` olarak belirttik. Yukarıdaki örnekte, `simpleEstimator` ve `fieldExtensionsEstimator` olmak üzere 2 tahminci kullandık.

- `simpleEstimator`: basit tahminci, her alan için sabit bir karmaşıklık döndürür.
- `fieldExtensionsEstimator`: alan uzantıları tahmincisi, şemanızın her alanının karmaşıklık değerini çıkarır.

> info **İpucu** Bu sınıfı herhangi bir modülde `providers` dizisine eklemeyi unutmayın.

#### Alan düzeyinde karmaşıklık

Bu eklenti ile herhangi bir alandaki karmaşıklığı belirlemek için `@Field()` dekoratörüne geçirilen seçenekler nesnesindeki `complexity` özelliğini belirleyebiliriz, örneğin:

```typescript
@Field({ complexity: 3 })
title: string;
```

Alternatif olarak, tahminci işlevini tanımlayabilirsiniz:

```typescript
@Field({ complexity: (options: ComplexityEstimatorArgs) => ... })
title: string;
```

#### Sorgu/Mutasyon düzeyinde karmaşıklık

Ayrıca, `@Query()` ve `@Mutation()` dekoratörlerine şu şekilde belirtilen bir `complexity` özelliği olabilir:

```typescript
@Query({ complexity: (options: ComplexityEstimatorArgs) => options.args.count * options.childComplexity })
items(@Args('count') count: number) {
  return this.itemsService.getItems({ count });
}
```