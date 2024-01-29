### Apollo ile Eklentiler

Eklentiler, Apollo Server'ın temel işlevselliğini özel işlemler gerçekleştirerek genişletmenizi sağlar. Şu anda, bu etkinlikler GraphQL isteği yaşam döngüsünün bireysel aşamalarına ve Apollo Server'ın kendisinin başlatılmasına karşılık gelir (daha fazlası için [buraya](https://www.apollographql.com/docs/apollo-server/integrations/plugins/) bakın). Örneğin, temel bir günlük kaydı eklentisi, Apollo Server'a gönderilen her isteğin ilişkilendirildiği GraphQL sorgu dizisini kaydedebilir.

#### Özel eklentiler

Bir eklenti oluşturmak için, `@nestjs/apollo` paketinden alınan `@Plugin` dekoratörü ile işaretlenmiş bir sınıf bildirin. Ayrıca, daha iyi kod otomatik tamamlama için `@apollo/server` paketinden `ApolloServerPlugin` arabirimini uygulayın.

```typescript
import { ApolloServerPlugin, GraphQLRequestListener } from '@apollo/server';
import { Plugin } from '@nestjs/apollo';

@Plugin()
export class LoggingPlugin implements ApolloServerPlugin {
  async requestDidStart(): Promise<GraphQLRequestListener<any>> {
    console.log('Request started');
    return {
      async willSendResponse() {
        console.log('Will send response');
      },
    };
  }
}
```

Bu yapı ile, `LoggingPlugin`'i bir sağlayıcı olarak kaydedebiliriz.

```typescript
@Module({
  providers: [LoggingPlugin],
})
export class CommonModule {}
```

Nest otomatik olarak bir eklentiyi başlatır ve Apollo Server'a uygular.

#### Harici eklentilerin kullanımı

Birkaç eklenti zaten kutudan çıkartılmıştır. Mevcut bir eklentiyi kullanmak için, sadece onu içe aktarın ve `plugins` dizisine ekleyin:

```typescript
GraphQLModule.forRoot({
  // ...
  plugins: [ApolloServerOperationRegistry({ /* options */})]
}),
```

> info **İpucu** `ApolloServerOperationRegistry` eklentisi, `@apollo/server-plugin-operation-registry` paketinden alınır.

#### Mercurius ile Eklentiler

Mevcut mercurius özel Fastify eklentilerinin bazıları, eklenti ağacında mercurius eklentisinden sonra yüklenmelidir (daha fazlası için [buraya](https://mercurius.dev/#/docs/plugins) bakın).

> warning **Uyarı** [mercurius-upload](https://github.com/mercurius-js/mercurius-upload) istisnadır ve ana dosyada kaydedilmelidir.

Bunun için, `MercuriusDriver` isteğe bağlı bir `plugins` yapılandırma seçeneği sunar. Bu, `plugin` ve `options` olmak üzere iki öznitelikten oluşan bir dizi nesnesidir. Bu nedenle, [önbellek eklentisi](https://github.com/mercurius-js/cache) ni kaydetmek şu şekilde görünebilir:

```typescript
GraphQLModule.forRoot({
  driver: MercuriusDriver,
  // ...
  plugins: [
    {
      plugin: cache,
      options: {
        ttl: 10,
        policy: {
          Query: {
            add: true
          }
        }
      },
    }
  ]
}),
```
