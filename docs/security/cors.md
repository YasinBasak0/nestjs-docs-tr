### CORS

Cross-Origin Resource Sharing (CORS), başka bir etki alanından kaynakların talep edilmesine izin veren bir mekanizmadır. Altyapıda, Nest Express [cors](https://github.com/expressjs/cors) veya Fastify [@fastify/cors](https://github.com/fastify/fastify-cors) paketlerini kullanır. Bu paketler, ihtiyaçlarınıza göre özelleştirebileceğiniz çeşitli seçenekler sağlar.

#### Başlangıç

CORS'u etkinleştirmek için, Nest uygulama nesnesi üzerinde `enableCors()` yöntemini çağırın.

```typescript
const app = await NestFactory.create(AppModule);
app.enableCors();
await app.listen(3000);
```

`enableCors()` yöntemi, isteğe bağlı bir yapılandırma nesnesi argümanını alır. Bu nesnenin kullanılabilir özellikleri resmi [CORS](https://github.com/expressjs/cors#configuration-options) belgelerinde açıklanmıştır. Başka bir yol, isteğe bağlı olarak (uçakta) yapılandırma nesnesini tanımlamanıza izin veren bir [geri çağırma işlevi](https://github.com/expressjs/cors#configuring-cors-asynchronously) sağlamaktır.

Alternatif olarak, CORS'u `create()` yönteminin seçenekler nesnesi üzerinden etkinleştirin. CORS'u varsayılan ayarlarla etkinleştirmek için `cors` özelliğini `true` olarak ayarlayın.
Veya davranışını özelleştirmek için `cors` özelliğine [CORS yapılandırma nesnesi](https://github.com/expressjs/cors#configuration-options) veya [geri çağırma işlevi](https://github.com/expressjs/cors#configuring-cors-asynchronously) olarak bir değer geçirin.

```typescript
const app = await NestFactory.create(AppModule, { cors: true });
await app.listen(3000);
```