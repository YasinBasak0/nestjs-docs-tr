---
sidebar_position: 1
---

### Bağımsız Uygulamalar

Nest uygulamasını başlatmanın birkaç yolu vardır. Bir web uygulaması, bir mikro hizmet veya yalnızca bir çıplak Nest **bağımsız uygulaması** (herhangi bir ağ dinleyicisi olmadan) oluşturabilirsiniz. Nest bağımsız uygulaması, tüm örneklendirilmiş sınıfları içeren Nest **IoC konteynırı** etrafında bir sarmalayıcıdır. Herhangi bir içe aktarılan modülden doğrudan bağımsız uygulama nesnesini kullanarak mevcut herhangi bir örneğe başvurabiliriz. Bu sayede Nest çerçevesinden, örneğin betikli **CRON** görevlerinden bile herhangi bir yerde yararlanabilirsiniz. Üstelik bunun üzerine bir **CLI** bile inşa edebilirsiniz.

#### Başlarken

Bir Nest bağımsız uygulaması oluşturmak için şu yapımı kullanın:

```typescript
@@filename()
async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  // uygulama mantığı...
}
bootstrap();
```

Bağımsız uygulama nesnesi, Nest uygulaması içinde kayıtlı herhangi bir örneğe başvurmanıza olanak tanır. Diyelim ki `TasksModule` içinde bir `TasksService`'imiz var. Bu sınıf, bir CRON görevinden çağırmak istediğimiz bir dizi yöntem sağlar.

```typescript
@@filename()
const app = await NestFactory.createApplicationContext(AppModule);
const tasksService = app.get(TasksService);
```

`get()` yöntemini kullanarak `TasksService` örneğine erişiyoruz. `get()` yöntemi, her bir kayıtlı modülde bir örneği arayan bir **sorgu** gibi davranır. Alternatif olarak, kesin bağlam kontrolü için `strict: true` özelliğine sahip bir seçenek nesnesi ile geçin. Bu seçenek etkin olduğunda, belirli bir bağlamdan belirli bir örneği almak için belirli modüller arasında gezinmelisiniz.

```typescript
@@filename()
const app = await NestFactory.createApplicationContext(AppModule);
const tasksService = app.select(TasksModule).get(TasksService, { strict: true });
```

İşte bağımsız uygulama nesnesinden örnek başvurularını almak için kullanılabilen yöntemlerin bir özeti:

<table>
  <tr>
    <td>
      <code>get()</code>
    </td>
    <td>
      Uygulama bağlamında bulunan bir denetleyici veya sağlayıcının (koruyucular, filtreler vb. dahil) bir örneğini alır.
    </td>
  </tr>
  <tr>
    <td>
      <code>select()</code>
    </td>
    <td>
      Belirli bir modülün seçilen örneğini çekmek için modül grafiği üzerinde gezinir (yukarıda açıklandığı gibi sıkı mod ile birlikte kullanılır).
    </td>
  </tr>
</table>

> info **İpucu** Sıkı modda, varsayılan olarak kök modül seçilir. Başka bir modülü seçmek için modülleri grafiğinde manuel olarak adım adım gezmeniz gerekir.

Eğer betik CRON görevlerini çalıştıran bir betik için uygulamanın bitmesini istiyorsanız, `bootstrap` fonksiyonunuzun sonuna `await app.close()` ekleyin:

```typescript
@@filename()
async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  // uygulama mantığı...
  await app.close();
}
bootstrap();
```

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/18-context) bulunabilir.