### Skaler Tipleri

Bir GraphQL nesne türü bir ada ve alanlara sahiptir, ancak bu alanların bir noktada somut verilere çözünmesi gerekir. İşte skaler tipler devreye girer: Bunlar, sorgunun yapraklarını temsil eder (daha fazla bilgi için [buraya](https://graphql.org/learn/schema/#scalar-types) bakın). GraphQL'in varsayılan tipleri şunları içerir: `Int`, `Float`, `String`, `Boolean` ve `ID`. Bu yerleşik tiplerin yanı sıra özel atomik veri tiplerini (örneğin, `Date`) desteklemeniz gerekebilir.

#### Kod ile Başla

Code-first yaklaşımı, üç tanesi mevcut GraphQL tipleri için basit takma ad olan beş skaler ile birlikte gelir.

- `ID` (`GraphQLID` için takma ad) - genellikle bir nesneyi tekrar getirmek veya bir önbellek için anahtar olarak kullanılmak üzere benzersiz bir tanımlayıcıyı temsil eder
- `Int` (`GraphQLInt` için takma ad) - işaretli 32-bit bir tamsayı
- `Float` (`GraphQLFloat` için takma ad) - işaretli çift hassasiyetli kayan noktalı bir değer
- `GraphQLISODateTime` - UTC'de bir tarih-zaman dizesi (varsayılan olarak `Date` türünü temsil etmek için kullanılır)
- `GraphQLTimestamp` - UNIX epokunun başlangıcından itibaren milisaniye cinsinden tarih ve saati temsil eden işaretli bir tamsayı

`GraphQLISODateTime` (örneğin, `2019-12-03T09:54:33Z`), varsayılan olarak `Date` türünü temsil etmek için kullanılır. Bunun yerine `GraphQLTimestamp`'i kullanmak için, `buildSchemaOptions` nesnesinin `dateScalarMode` öğesini şu şekilde `'timestamp'` olarak ayarlayın:

```typescript
GraphQLModule.forRoot({
  buildSchemaOptions: {
    dateScalarMode: 'timestamp',
  }
}),
```

Benzer şekilde, varsayılan olarak `number` türünü temsil etmek için `GraphQLFloat` kullanılır. Bunun yerine `GraphQLInt`'i kullanmak için, `buildSchemaOptions` nesnesinin `numberScalarMode` öğesini şu şekilde `'integer'` olarak ayarlayın:

```typescript
GraphQLModule.forRoot({
  buildSchemaOptions: {
    numberScalarMode: 'integer',
  }
}),
```

Ayrıca özel skaler tipler oluşturabilirsiniz.

#### Varsayılan bir skaleri geçersiz kılma

`Date` skaleri için özel bir uygulama oluşturmak için basitçe yeni bir sınıf oluşturun.

```typescript
import { Scalar, CustomScalar } from '@nestjs/graphql';
import { Kind, ValueNode } from 'graphql';

@Scalar('Date', (type) => Date)
export class DateScalar implements CustomScalar<number, Date> {
  description = 'Date custom scalar type';

  parseValue(value: number): Date {
    return new Date(value); // müşteriden gelen değer
  }

  serialize(value: Date): number {
    return value.getTime(); // müşteriye gönderilen değer
  }

  parseLiteral(ast: ValueNode): Date {
    if (ast.kind === Kind.INT) {
      return new Date(ast.value);
    }
    return null;
  }
}
```

Bunu yerleştirdikten sonra, `DateScalar`'ı bir sağlayıcı olarak kaydedin.

```typescript
@Module({
  providers: [DateScalar],
})
export class CommonModule {}
```

Şimdi `Date` türünü sınıflarımızda kullanabiliriz.

```typescript
@Field()
creationDate: Date;
```

#### Özel bir skaleri içe aktarma

Özel bir skaleri kullanmak için, onu içe aktarın ve bir çözümleyici olarak kaydedin. Bu amaçla `graphql-type-json` paketini kullanacağız. Bu npm paketi, `JSON` GraphQL skaler türünü tanımlar.

Paketi yüklemeye başlayın:

```bash
$ npm i --save graphql-type-json
```

Paket yüklendikten sonra, `forRoot()` yöntemine özel bir çözümleyici ileterek bir çözümleyici olarak kaydedin:

```typescript
import GraphQLJSON from 'graphql-type-json';

@Module({
  imports: [
    GraphQLModule.forRoot({
      resolvers: { JSON: GraphQLJSON },
    }),
  ],
})
export class AppModule {}
```

Artık sınıflarımızda `JSON` türünü kullanabiliriz.

```typescript
@Field((type) => GraphQLJSON)
info: JSON;
```

Yararlı bir dizi skaler için [graphql-scalars](https://www.npmjs.com/package/graphql-scalars) paketine göz atın.

#### Özel bir skaler oluşturma

Özel bir skaleri tanımlamak için yeni bir `GraphQLScalarType` örneği oluşturun. Özel bir `UUID` skaleri oluşturacağız.

```typescript
const regex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

function validate(uuid: unknown): string | never {
  if (typeof uuid !== "string" || !regex.test(uuid)) {
    throw new Error("invalid uuid");
  }
  return uuid;
}

export const CustomUuidScalar = new GraphQLScalarType({
  name: 'UUID',
  description: 'A simple UUID parser',
  serialize: (value) => validate(value),
  parseValue: (value) => validate(value),
  parseLiteral: (ast) => validate(ast.value)
})
```

`forRoot()` yöntemine özel bir çözümleyici ileterek bir çözümleyici olarak kaydedin:

```typescript
@Module({
  imports: [
    GraphQLModule.forRoot({
      resolvers: { UUID: CustomUuidScalar },
    }),
  ],
})
export class AppModule {}
```

Artık sınıflarımızda `UUID` türünü kullanabiliriz.

```typescript
@Field((type) => CustomUuidScalar)
uuid: string;
```

#### Şema İlk

Özel bir skaler tanımlamak için (skalerler hakkında daha fazla bilgi için [buraya](https://www.apollographql.com/docs/graphql-tools/scalars.html) bakabilirsiniz), bir tür tanımı ve özel bir çözümleyici oluşturun. İşte (resmi belgelerde olduğu gibi) bu amaçla `graphql-type-json` paketini kullanacağımız bir örnek. Bu npm paketi, bir `JSON` GraphQL skaler türü tanımlar.

İlk olarak, paketi yükleyin:

```bash
$ npm i --save graphql-type-json
```

Paket yüklendikten sonra, `forRoot()` yöntemine özel bir çözümleyiciyi iletiyoruz:

```typescript
import GraphQLJSON from 'graphql-type-json';

@Module({
  imports: [
    GraphQLModule.forRoot({
      typePaths: ['./**/*.graphql'],
      resolvers: { JSON: GraphQLJSON },
    }),
  ],
})
export class AppModule {}
```

Şimdi `JSON` skalerini tür tanımlarımızda kullanabiliriz:

```graphql
scalar JSON

type Foo {
  field: JSON
}
```

Bir skaler türü tanımlamak için başka bir yöntem de basit bir sınıf oluşturmaktır. Diyelim ki şemamızı `Date` türü ile geliştirmek istiyoruz.

```typescript
import { Scalar, CustomScalar } from '@nestjs/graphql';
import { Kind, ValueNode } from 'graphql';

@Scalar('Date')
export class DateScalar implements CustomScalar<number, Date> {
  description = 'Date custom scalar type';

  parseValue(value: number): Date {
    return new Date(value); // istemciden gelen değer
  }

  serialize(value: Date): number {
    return value.getTime(); // istemciye gönderilen değer
  }

  parseLiteral(ast: ValueNode): Date {
    if (ast.kind === Kind.INT) {
      return new Date(ast.value);
    }
    return null;
  }
}
```

Bunu sağladıktan sonra, `DateScalar`'ı bir sağlayıcı olarak kaydedin.

```typescript
@Module({
  providers: [DateScalar],
})
export class CommonModule {}
```

Artık `Date` skalerini tür tanımlarımızda kullanabiliriz.

```graphql
scalar Date
```

Varsayılan olarak, tüm skaler türler için oluşturulan TypeScript tanımı `any` şeklindedir - bu özellikle tip güvenliği açısından pek uygun değildir. Ancak, özel skaler türleriniz için Nest'in nasıl tanıma yapacağını belirleyebilirsiniz. Bunun için türleri nasıl oluşturacağınızı belirlediğinizde yapmanız gerekenler:

```typescript
import { GraphQLDefinitionsFactory } from '@nestjs/graphql';
import { join } from 'path';

const definitionsFactory = new GraphQLDefinitionsFactory();

definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
  defaultScalarType: 'unknown',
  customScalarTypeMapping: {
    DateTime: 'Date',
    BigNumber: '_BigNumber',
  },
  additionalHeader: "import _BigNumber from 'bignumber.js'",
});
```

> info **İpucu** Alternatif olarak, bir tür referansı kullanabilirsiniz; örneğin: `DateTime: Date`. Bu durumda, `GraphQLDefinitionsFactory`, belirtilen türün adını (`Date.name`) çıkarmak için kullanılacaktır. Not: (özel türler gibi) yerleşik olmayan türler için bir import ifadesi eklemek gereklidir.

Şimdi, aşağıdaki GraphQL özel skaler türleri verildiğinde:

```graphql
scalar DateTime
scalar BigNumber
scalar Payload
```

`src/graphql.ts` dosyasında aşağıdaki oluşturulan TypeScript tanımlarını göreceğiz:

```typescript
import _BigNumber from 'bignumber.js';

export type DateTime = Date;
export type BigNumber = _BigNumber;
export type Payload = unknown;
```

Burada, `customScalarTypeMapping` özelliğini kullanarak özel skaler türlerimiz için hangi türleri bildirmek istediğimizi belirttik. Ayrıca, bu tür tanımlar için gereken herhangi bir import ifadesini eklemek için `additionalHeader` özelliğini sağladık. Son olarak, belirli bir `defaultScalarType` olarak `'unknown'` ekledik; böylece `customScalarTypeMapping` içinde belirtilmeyen herhangi bir özel skaler, `any` yerine (`TypeScript 3.0`'dan itibaren önerilen) ekstra tip güvenliği için `unknown` olarak altyapılır.

>

 info **İpucu** `_BigNumber`'ı `bignumber.js`'den içe aktardık; bu, [döngüsel tür referansları](https://github.com/Microsoft/TypeScript/issues/12525#issuecomment-263166239)'ı önlemek için yapıldı.