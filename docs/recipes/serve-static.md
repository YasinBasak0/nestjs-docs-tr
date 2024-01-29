### Serve Static

Statik içerik, bir Tek Sayfa Uygulaması'nı (SPA) sunmak için `@nestjs/serve-static` paketinden gelen `ServeStaticModule` kullanılabilir.

#### Kurulum

İlk olarak, gerekli paketi kurmamız gerekiyor:

```bash
$ npm install --save @nestjs/serve-static
```

#### Başlatma

Kurulum işlemi tamamlandığında, `ServeStaticModule`'u kök `AppModule`'a ekleyebilir ve `forRoot()` yöntemine bir yapılandırma nesnesi geçirerek yapılandırabiliriz.

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'path';

@Module({
  imports: [
    ServeStaticModule.forRoot({
      rootPath: join(__dirname, '..', 'client'),
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Bunu yerine getirdiğinizde, statik web sitesini oluşturun ve içeriğini `rootPath` özelliği tarafından belirtilen konuma yerleştirin.

#### Yapılandırma

[ServeStaticModule](https://github.com/nestjs/serve-static), davranışını özelleştirmek için çeşitli seçeneklerle yapılandırılabilir.
Statik uygulamanızı render etmek için yol belirleyebilir, hariç tutulan yolları belirtebilir, Cache-Control yanıt başlığını etkinleştirebilir veya devre dışı bırakabilirsiniz. Tüm seçenekleri [burada](https://github.com/nestjs/serve-static/blob/master/lib/interfaces/serve-static-options.interface.ts) görebilirsiniz.

> uyarı **Dikkat** Statik Uygulamanın varsayılan `renderPath`'ı `*` (tüm yollar) ve modül, yanıt olarak "index.html" dosyalarını gönderecektir.
> Bu, SPA'nız için İstemci Tarafı yönlendirmesi oluşturmanıza olanak tanır. Denetleyicilerinizde belirtilen yollar, sunucuya geri düşer.
> Bu davranışı değiştirebilirsiniz `serveRoot`, `renderPath`'i diğer seçeneklerle birleştirerek ayarlayarak.

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/24-serve-static) bulunmaktadır.