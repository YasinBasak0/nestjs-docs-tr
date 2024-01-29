### Read-Eval-Print-Loop (REPL)

REPL, "Read-Eval-Print-Loop," basit bir etkileşimli ortamdır ve tek kullanıcı girişlerini alır, bunları yürütür ve sonucu kullanıcıya döndürür. REPL özelliği, bağımlılık grafinizi incelemenize ve sağlayıcılarınızın (ve denetleyicilerinizin) yöntemlerini doğrudan terminalinizden çağırmanıza olanak tanır.

#### Kullanım

NestJS uygulamanızı REPL modunda çalıştırmak için, yeni bir `repl.ts` dosyası oluşturun (mevcut `main.ts` dosyasının yanına) ve içine aşağıdaki kodu ekleyin:

```typescript
@@filename(repl)
import { repl } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  await repl(AppModule);
}
bootstrap();
@@switch
import { repl } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  await repl(AppModule);
}
bootstrap();
```

Şimdi terminalinizde, aşağıdaki komutla REPL'yi başlatın:

```bash
$ npm run start -- --entryFile repl
```

> info **İpucu** `repl`, bir [Node.js REPL sunucu](https://nodejs.org/api/repl.html) nesnesini döndürür.

Çalıştığında, konsolunuzda aşağıdaki mesajı görmelisiniz:

```bash
LOG [NestFactory] Starting Nest application...
LOG [InstanceLoader] AppModule dependencies initialized
LOG REPL initialized
```

Ve şimdi bağımlılıklar grafiğinizle etkileşime geçebilirsiniz. Örneğin, bir `AppService` (burada bir örnek olarak başlangıç projesini kullanıyoruz) alabilir ve `getHello()` yöntemini çağırabilirsiniz:

```typescript
> get(AppService).getHello()
'Hello World!'
```

Terminalinizden herhangi bir JavaScript kodunu yürütebilirsiniz; örneğin, bir `AppController` örneğini yerel bir değişkene atayabilir ve bir asenkron yöntemi çağırmak için `await` kullanabilirsiniz:

```typescript
> appController = get(AppController)
AppController { appService: AppService {} }
> await appController.getHello()
'Hello World!'
```

Belirli bir sağlayıcı veya denetleyici üzerindeki tüm genel yöntemleri görüntülemek için `methods()` işlevini kullanabilirsiniz:

```typescript
> methods(AppController)

Methods:
 ◻ getHello
```

Tüm kayıtlı modülleri, denetleyicileri ve sağlayıcıları bir liste halinde ve birlikte görüntülemek için `debug()` kullanın.

```typescript
> debug()

AppModule:
 - controllers:
  ◻ AppController
 - providers:
  ◻ AppService
```

Hızlı bir demo:

<figure><img src="/assets/repl.gif" alt="REPL örneği" /></figure>

Mevcut, önceden tanımlanmış yerel yöntemler hakkında daha fazla bilgi için aşağıdaki bölüme bakabilirsiniz.

#### Yerel Fonksiyonlar

Dahili NestJS REPL, REPL başladığında genel olarak kullanılabilen bazı yerel fonksiyonlarla birlikte gelir. Bunları listelemek için `help()`'i çağırabilirsiniz.

Bir işlevin imzasını (yani beklenen parametreler ve dönüş türü) hatırlamıyorsanız, `<function_name>.help`'i çağırabilirsiniz.
Örneğin:

```text
> $.help
Bir enjekte edilebilir veya denetleyici örneğini alır, aksi takdirde istisna fırlatır.
Arayüz: $(token: InjectionToken) => any
```

> info **İpucu** Bu işlev arayüzleri [TypeScript işlev türü ifade sözdizimi](https://www.typescriptlang.org/docs/handbook/2/functions.html#function-type-expressions) ile yazılmıştır.

| Fonksiyon   | Açıklama                                                      | İmza                                                                 |
| ----------- | ------------------------------------------------------------- | --------------------------------------------------------------------- |
| `debug`     | Kayıtlı tüm modülleri, denetleyicileri ve sağlayıcıları bir liste halinde yazdırın. | `debug(moduleCls?: ClassRef \| string) => void`                       |
| `get` veya `$` | Enjekte edilebilir veya denetleyici örneğini alır, aksi takdirde istisna fırlatır.    | `get(token: InjectionToken) => any`                                   |
| `methods`   | Verilen sağlayıcı veya denetleyicinin üzerindeki tüm genel yöntemleri görüntüler.       | `methods(token: ClassRef \| string) => void`                          |
| `resolve`   | Belirli bir bağımlılığın, aksi takdirde istisna fırlatır, geçici veya istek temelli örneğini çözer. | `resolve(token: InjectionToken, contextId: any) => Promise<any>`      |
| `select`    | Modül ağacı içinde gezinmeye olanak tanır, örneğin, seçilen modülden belirli bir örneği çıkarmak için. | `select(token: DynamicModule \| ClassRef) => INestApplicationContext` |

#### İzleme modu

Geliştirme sırasında tüm kod değişikliklerini otomatik olarak yansıtmak için REPL'yi izleme modunda çalıştırmak faydalıdır:

```bash
$ npm run start -- --watch --entryFile repl
```

Bu, REPL'nin komut geçmişinin her yeniden yüklemeden sonra atıldığı bir kusuru vardır ki bu da zahmetli olabilir.
Neyse ki, çok basit bir çözüm var. `bootstrap` işlevinizi şu şekilde değiştirin:

```typescript
async function bootstrap() {
  const replServer = await repl(AppModule);
  replServer.setupHistory(".nestjs_repl_history", (err) => {
    if (err) {
      console.error(err

);
    }
  });
}
```

Şimdi geçmiş, çalışmalar/aralar arasında korunur.