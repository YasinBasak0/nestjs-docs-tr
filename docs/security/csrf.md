### CSRF Koruması

Cross-Site Request Forgery (CSRF veya XSRF olarak da bilinen), bir web uygulamasının güvendiği bir kullanıcıdan iletilen **yetkisiz** komutların yapıldığı kötü niyetli bir saldırı türüdür. Bu tür saldırıları hafifletmek için [csurf](https://github.com/expressjs/csurf) paketini kullanabilirsiniz.

#### Express ile Kullanım (varsayılan)

İlk olarak gerekli paketi yükleyerek başlayın:

```bash
$ npm i --save csurf
```

> warning **Uyarı** Bu paket artık kullanılmıyor, daha fazla bilgi için [`csurf` belgelerine](https://github.com/expressjs/csurf#csurf) başvurun.

> warning **Uyarı** [`csurf` belgelerinde](https://github.com/expressjs/csurf#csurf) açıklandığı gibi, bu ara yazılımın etkinleştirilmesi için önce oturum ara yazılımının veya `cookie-parser`'ın başlatılması gerekmektedir. Lütfen daha fazla talimat için o belgelere bakın.

Yükleme tamamlandığında, `csurf` middleware'ini global middleware olarak uygulayın.

```typescript
import * as csurf from 'csurf';
// ...
// başlatma dosyanızın bir yerinde
app.use(csurf());
```

#### Fastify ile Kullanım

İlk olarak gerekli paketi yükleyerek başlayın:

```bash
$ npm i --save @fastify/csrf-protection
```

Yükleme tamamlandığında, `@fastify/csrf-protection` eklentisini şu şekilde kaydedin:

```typescript
import fastifyCsrf from '@fastify/csrf-protection';
// ...
// başlatma dosyanızın bir yerinde, bir depolama eklentisi kaydettikten sonra
await app.register(fastifyCsrf);
```

> warning **Uyarı** `@fastify/csrf-protection` belgelerinde [burada](https://github.com/fastify/csrf-protection#usage) açıklandığı gibi, bu eklentinin başlatılması için önce bir depolama eklentisinin başlatılması gerekmektedir. Lütfen daha fazla talimat için o belgelere bakın.