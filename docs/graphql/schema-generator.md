#### SDL Oluşturma

> warning **Uyarı** Bu bölüm yalnızca kod ilk yaklaşımına uygulanır.

GraphQL SDL şemasını (yani, bir uygulama çalıştırılmadan, veritabanına bağlanmadan, çözücülerle bağlantı kurmadan vb.) manuel olarak oluşturmak için `GraphQLSchemaBuilderModule`'u kullanın.

```typescript
async function generateSchema() {
  const app = await NestFactory.create(GraphQLSchemaBuilderModule);
  await app.init();

  const gqlSchemaFactory = app.get(GraphQLSchemaFactory);
  const schema = await gqlSchemaFactory.create([RecipesResolver]);
  console.log(printSchema(schema));
}
```

> info **İpucu** `GraphQLSchemaBuilderModule` ve `GraphQLSchemaFactory` `@nestjs/graphql` paketinden içe aktarılır. `printSchema` fonksiyonu `graphql` paketinden içe aktarılır.

#### Kullanım

`gqlSchemaFactory.create()` yöntemi bir dizi çözücü sınıf referansını alır. Örneğin:

```typescript
const schema = await gqlSchemaFactory.create([
  RecipesResolver,
  AuthorsResolver,
  PostsResolvers,
]);
```

İkinci, isteğe bağlı bir argüman olarak bir dizi skaler sınıfları içeren bir dizi alır:

```typescript
const schema = await gqlSchemaFactory.create(
  [RecipesResolver, AuthorsResolver, PostsResolvers],
  [DurationScalar, DateScalar],
);
```

Son olarak, bir seçenekler nesnesi geçirebilirsiniz:

```typescript
const schema = await gqlSchemaFactory.create([RecipesResolver], {
  skipCheck: true,
  orphanedTypes: [],
});
```

- `skipCheck`: şema doğrulamasını yok say; boolean, varsayılan değeri `false`
- `orphanedTypes`: açıkça başvurulmayan (nesne grafiğinin bir parçası olmayan) sınıfların üretilmesi için bir liste. Normalde, bir sınıf bildirilmişse ancak başka şekilde grafiğe referans edilmiyorsa, atlanır. Özellik değeri bir sınıf referanslarını içeren bir dizi.