### Sıkıştırma

Sıkıştırma, yanıt gövdesinin boyutunu büyük ölçüde azaltabilir ve böylece bir web uygulamasının hızını artırabilir.

**Yüksek trafiğe** sahip üretim siteleri için, sıkıştırmayı genellikle uygulama sunucusundan dışa aktarmak - genellikle ters proxy'de (örneğin, Nginx) - kesinlikle önerilir. Bu durumda, sıkıştırma ortamını kullanmamalısınız.

#### Express ile Kullanım (varsayılan)

gzip sıkıştırmayı etkinleştirmek için [compression](https://github.com/expressjs/compression) ortam yazılımı paketini kullanın.

İlk olarak, gerekli paketi kurun:

```bash
$ npm i --save compression
```

Kurulum tamamlandığında, sıkıştırma ortamını genel ortam ortamı olarak uygulayın.

```typescript
import * as compression from 'compression';
// başlatma dosyanızın herhangi bir yerinde
app.use(compression());
```

#### Fastify ile Kullanım

`FastifyAdapter` kullanıyorsanız, [fastify-compress](https://github.com/fastify/fastify-compress) kullanmak isteyeceksiniz:

```bash
$ npm i --save @fastify/compress
```

Kurulum tamamlandığında, `@fastify/compress` ortamını genel ortam ortamı olarak uygulayın.

```typescript
import compression from '@fastify/compress';
// başlatma dosyanızın herhangi bir yerinde
await app.register(compression);
```

Varsayılan olarak, `@fastify/compress`, tarayıcılar sıkıştırma desteğini belirttiklerinde (Node >= 11.7.0 üzerinde) Brotli sıkıştırmayı kullanacaktır. Brotli, sıkıştırma oranı açısından oldukça verimli olabilir, ancak oldukça yavaş da olabilir. Varsayılan olarak, Brotli, 11 maksimum sıkıştırma kalitesi ayarlar, ancak sıkıştırma süresini sıkıştırma kalitesi yerine azaltmak için `BROTLI_PARAM_QUALITY`'yi 0 minimum ve 11 maksimum arasında ayarlayarak ayarlanabilir. Bu, alan/zaman performansını optimize etmek için ince ayar gerektirecektir. Kalite 4 örneği ile:

```typescript
import { constants } from 'zlib';
// başlatma dosyanızın herhangi bir yerinde
await app.register(compression, { brotliOptions: { params: { [constants.BROTLI_PARAM_QUALITY]: 4 } } });
```

Basitleştirmek için, `fastify-compress`'e yalnızca yanıtları sıkıştırmak için deflate ve gzip kullanmasını söyleyebilirsiniz; sonuçta potansiyel olarak daha büyük yanıtlarınız olacaktır, ancak çok daha hızlı bir şekilde teslim edilecektir.

Kodlamaları belirtmek için, `app.register`'a ikinci bir argüman sağlayın:

```typescript
await app.register(compression, { encodings: ['gzip', 'deflate'] });
```

Yukarıdaki, `fastify-compress`'e yalnızca gzip ve deflate kodlamalarını kullanmasını söyler, istemci her ikisini de destekliyorsa gzip'i tercih eder.