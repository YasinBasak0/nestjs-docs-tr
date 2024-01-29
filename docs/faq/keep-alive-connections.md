### Keep alive bağlantıları

NestJS'nin varsayılan olarak HTTP adaptörleri, yanıt tamamlanana kadar uygulamayı kapatmayı bekler. Ancak bazen bu davranış istenmeyen veya beklenmeyen olabilir. `Connection: Keep-Alive` başlıklarını kullanan ve uzun süre yaşayan bazı istekler olabilir.

Uygulamanızın isteklerin bitmesini beklemeden her zaman çıkmasını istediğiniz senaryolarda, NestJS uygulamanızı oluştururken `forceCloseConnections` seçeneğini etkinleştirebilirsiniz.

> warning **İpucu** Çoğu kullanıcı bu seçeneği etkinleştirmeye ihtiyaç duymayacaktır. Ancak bu seçeneği etmeniz gereken durumun belirtisi, uygulamanızın beklediğiniz gibi çıkmamasıdır. Genellikle `app.enableShutdownHooks()` etkinleştirildiğinde ve uygulamanın yeniden başlamadığını/fark edildiğinde. Muhtemelen `--watch` ile NestJS uygulamasını geliştirme sırasında.

#### Kullanım

`main.ts` dosyanızda, NestJS uygulamanızı oluştururken seçeneği etkinleştirin:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    forceCloseConnections: true,
  });
  await app.listen(3000);
}

bootstrap();
```