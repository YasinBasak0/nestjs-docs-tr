### Direktifler

Bir direktif, bir alan veya parça dahil etmeye bağlanabilir ve sunucunun isteği herhangi bir şekilde etkileyebilir (daha fazlasını [buradan](https://graphql.org/learn/queries/#directives) okuyabilirsiniz). GraphQL spesifikasyonu birkaç varsayılan direktif sağlar:

- `@include(if: Boolean)` - sadece bu alanı sonuçta dahil et, eğer argüman true ise
- `@skip(if: Boolean)` - eğer argüman true ise bu alanı atla
- `@deprecated(reason: String)` - alanı belirtilen nedenle eskimiş olarak işaretle

Bir direktif, `@` karakteri ile başlayan bir tanımlayıcıdır ve isteğe bağlı olarak neredeyse GraphQL sorgusundaki herhangi bir öğeden sonra görünebilen adlandırılmış argüman listesini takip edebilir.

#### Özel direktifler

Apollo/Mercurius direktifinizle karşılaştığında ne yapılması gerektiğini belirtmek için bir dönüştürücü fonksiyon oluşturabilirsiniz. Bu fonksiyon, şemanızdaki konumları (alan tanımları, tip tanımları vb.) dolaşmak ve ilgili dönüşümleri gerçekleştirmek için `mapSchema` fonksiyonunu kullanır.

```typescript
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

        // Orijinal çözücüyü, *ilk* önce orijinal çözücüyü çağıran ve ardından sonucunu büyük harfe dönüştüren bir işlevle değiştirin
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

Şimdi, `GraphQLModule#forRoot` yönteminde `transformSchema` fonksiyonunu kullanarak `upperDirectiveTransformer` dönüşüm fonksiyonunu uygulayın:

```typescript
GraphQLModule.forRoot({
  // ...
  transformSchema: (schema) => upperDirectiveTransformer(schema, 'upper'),
});
```

Bir kez kaydedildikten sonra, `@upper` direktifi şemamızda kullanılabilir hale gelir. Ancak direktifi nasıl uyguladığınız, kullanılan yaklaşıma bağlı olarak değişecektir (kod tabanlı veya şema tabanlı).

#### Kod tabanlı

Kod tabanlı yaklaşımda, direktifi uygulamak için `@Directive()` dekoratörünü kullanın.

```typescript
@Directive('@upper')
@Field()
title: string;
```

> info **İpucu** `@Directive()` dekoratörü, `@nestjs/graphql` paketinden alınır.

Directives, alanlara, alan çözücülere, giriş ve nesne tiplerine, aynı zamanda sorgulara, mutasyonlara ve aboneliklere uygulanabilir. İşte direktifin sorgu işleyici seviyesine uygulandığı bir örneği:

```typescript
@Directive('@deprecated(reason: "Bu sorgu bir sonraki sürümde kaldırılacaktır")')
@Query(returns => Author, { name: 'author' })
async getAuthor(@Args({ name: 'id', type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

> uyarı **Uyarı** `@Directive()` dekoratörü aracılığıyla uygulanan direktifler, oluşturulan şema tanım dosyasına yansıtılmayacaktır.

Son olarak, direktifleri `GraphQLModule` içinde bildiğinizden emin olun, aşağıdaki gibi:

```typescript
GraphQLModule.forRoot({
  // ...,
  transformSchema: schema => upperDirectiveTransformer(schema, 'upper'),
  buildSchemaOptions: {
    directives: [
      new GraphQLDirective({
        name: 'upper',
        locations: [DirectiveLocation.FIELD_DEFINITION],
      }),
    ],
  },
}),
```

> info **İpucu** Hem `GraphQLDirective` hem de `DirectiveLocation`, `graphql` paketinden alınır.

#### Şema tabanlı

Şema tabanlı yaklaşımda, direktifleri doğrudan SDL'de uygulayın.

```graphql
directive @upper on FIELD_DEFINITION

type Post {
  id: Int!
  title: String! @upper
  votes: Int
}
```