### Execution Context

Nest, çeşitli uygulama bağlamlarında çalışan uygulamaları yazmayı kolaylaştırmak için bir dizi yardımcı sınıf sağlar (örneğin, Nest HTTP sunucu tabanlı, mikroservis ve WebSocket uygulama bağlamları). Bu yardımcılar, geniş bir setteki denetleyiciler, yöntemler ve yürütme bağlamları üzerinde çalışabilen genel koruyucuları, filtreleri ve interceptor'ları oluşturmak için kullanılabilen geçerli yürütme bağlamı hakkında bilgi sağlar.

Bu bölümde iki tane bu tür sınıfa değineceğiz: `ArgumentsHost` ve `ExecutionContext`.

#### ArgumentsHost Sınıfı

`ArgumentsHost` sınıfı, bir işleyiciye iletilen argümanları almak için yöntemler sağlar. İlgili bağlamdan (örneğin, HTTP, RPC (mikroservis) veya WebSocket) argümanları almak için kullanılabilir. Framework, genellikle bir `ArgumentsHost` örneğini (`host` parametresi olarak başvurulan) erişmek istediğiniz yerlerde sağlar. Örneğin, [istisna filtresinin](https://docs.nestjs.com/exception-filters#arguments-host) `catch()` yöntemi, bir `ArgumentsHost` örneği ile çağrılır.

`ArgumentsHost`, temelde bir işleyicinin argümanları üzerinde bir soyutlama olarak hareket eder. Örneğin, HTTP sunucu uygulamaları için (`@nestjs/platform-express` kullanıldığında), `host` nesnesi Express'in `[request, response, next]` dizisini içerir, burada `request` istek nesnesini, `response` yanıt nesnesini ve `next` işleminin talep-yanıt döngüsünü kontrol eden bir işlevi içerir. Öte yandan, [GraphQL](/docs/graphql/quick-start) uygulamaları için, `host` nesnesi `[root, args, context, info]` dizisini içerir.

#### Geçerli Uygulama Bağlamı

Çoklu uygulama bağlamalarında çalışacak genel [koruyucular](/docs/guards), [filtreler](/docs/exception-filters) ve [interceptor'ları](/docs/interceptors) yazarken, yöntemimizin şu anda hangi türde bir uygulama içinde çalıştığını belirlemek için bir yol bulmamız gerekmektedir. Bunun için `ArgumentsHost`'un `getType()` yöntemini kullanın:

```typescript
if (host.getType() === 'http') {
  // yalnızca normal HTTP istekleri bağlamında önemli olan bir şey yap
} else if (host.getType() === 'rpc') {
  // yalnızca Mikroservis istekleri bağlamında önemli olan bir şey yap
} else if (host.getType<GqlContextType>() === 'graphql') {
  // yalnızca GraphQL istekleri bağlamında önemli olan bir şey yap
```

> info **İpucu** `GqlContextType`, `@nestjs/graphql` paketinden içe aktarılır.

Uygulama türü elde edildikten sonra, aşağıda gösterildiği gibi daha genel bileşenler yazabiliriz.

#### Host İşleyici Argümanları

İşleyiciye iletilen argüman dizisini almak için bir yaklaşım, `getArgs()` yöntemini kullanmaktır.

```typescript
const [req, res, next] = host.getArgs();
```

Belirli bir argümanı indeks kullanarak alabilirsiniz `getArgByIndex()` yöntemi:

```typescript
const request = host.getArgByIndex(0);
const response = host.getArgByIndex(1);
```

Bu örneklerde, isteği ve yanıtı indeksine göre aldık, bu da uygulamayı belirli bir yürütme bağlamına bağlar, bu genellikle önerilmez çünkü uygulamayı belirli bir yürütme bağlamına bağlar. Bunun yerine, kodunuzu daha sağlam ve yeniden kullanılabilir hale getirebilirsiniz, `host` nesnesinin uygun uygulama bağlamına geçmek için bir dizi yardımcı yöntemden birini kullanarak. Aşağıda, bağlam değiştirme yardımcı yöntemleri gösterilmiştir.

```typescript
/**
 * Bağlamı RPC'ye değiştir.
 */
switchToRpc(): RpcArgumentsHost;
/**
 * Bağlamı HTTP'ye değiştir.
 */
switchToHttp(): HttpArgumentsHost;
/**
 * Bağlamı WebSocket'e değiştir.
 */
switchToWs(): WsArgumentsHost;
```

Önceki örneği `switchToHttp()` yöntemini kullanarak yeniden yazalım. `host.switchToHttp()` yardımcı çağrısı, HTTP uygulama bağlamı için uygun olan bir `HttpArgumentsHost` nesnesi döndürür. `HttpArgumentsHost` nesnesinde, istenen nesneleri çıkarmak için kullanabileceğimiz iki kullan

ışlı yöntem bulunmaktadır. Ayrıca bu durumda Express tip belirtmelerini kullanarak native Express tip nesnelerini döndürüyoruz:

```typescript
const ctx = host.switchToHttp();
const request = ctx.getRequest<Request>();
const response = ctx.getResponse<Response>();
```

Benzer şekilde, `WsArgumentsHost` ve `RpcArgumentsHost` için, mikroservis ve WebSocket bağlamlarında uygun nesneleri döndürmek için yöntemlere sahiptir. İşte `WsArgumentsHost` için yöntemler:

```typescript
export interface WsArgumentsHost {
  /**
   * Veri nesnesini döndürür.
   */
  getData<T>(): T;
  /**
   * İstemci nesnesini döndürür.
   */
  getClient<T>(): T;
}
```

`RpcArgumentsHost` için yöntemler aşağıdaki gibidir:

```typescript
export interface RpcArgumentsHost {
  /**
   * Veri nesnesini döndürür.
   */
  getData<T>(): T;

  /**
   * Bağlam nesnesini döndürür.
   */
  getContext<T>(): T;
}
```

#### ExecutionContext Sınıfı

`ExecutionContext`, `ArgumentsHost` sınıfını genişleterek, geçerli yürütme işlemi hakkında ek bilgiler sağlar. `ArgumentsHost` gibi, Nest, ihtiyaç duyabileceğiniz yerlerde (`canActivate()` yöntemi gibi) bir `ExecutionContext` örneği sağlar. İlgili [guard](https://docs.nestjs.com/guards#execution-context)'ın `intercept()` yöntemi ve [interceptor](https://docs.nestjs.com/interceptors#execution-context) içinde. Aşağıdaki yöntemleri sağlar:

```typescript
export interface ExecutionContext extends ArgumentsHost {
  /**
   * Geçerli işleyicinin ait olduğu denetleyici sınıfının türünü döndürür.
   */
  getClass<T>(): Type<T>;
  /**
   * İsteğin işleme alınan bir sonraki adımda çağrılacak olan işleyici (yöntem) için bir referansı döndürür.
   */
  getHandler(): Function;
}
```

`getHandler()` yöntemi, çağrılacak işleyiciye bir referans döndürür. `getClass()` yöntemi, bu belirli işleyicinin ait olduğu `Controller` sınıfının türünü döndürür. Örneğin, bir HTTP bağlamında, şu anda işlenen istek bir `POST` isteği ise ve `CatsController`'daki `create()` yöntemine bağlı ise, `getHandler()` `create()` yöntemine bir referans döndürür ve `getClass()` `CatsController` **sınıfını** (örnek değil) döndürür.

```typescript
const methodKey = ctx.getHandler().name; // "create"
const className = ctx.getClass().name; // "CatsController"
```

Geçerli sınıfa ve işleyici yöntemine referanslara erişim sağlama yeteneği büyük esneklik sağlar. En önemlisi, `Reflector#createDecorator` ile oluşturulan dekoratörler aracılığıyla veya `@SetMetadata()` içinden, koruyucular veya interceptor'lar içinden bu metadatalara erişme olanağı sağlar. Bu kullanım durumunu aşağıda ele alıyoruz.

#### Yansıma ve Metadata

Nest, özel metadata'ları `Reflector#createDecorator` yöntemi aracılığıyla oluşturulan dekoratörler ve yerleşik `@SetMetadata()` dekoratörü aracılığıyla route handler'larına eklemek için bir yetenek sağlar. Bu bölümde, iki yaklaşımı karşılaştırarak metadata'ya nasıl erişileceğini bir koruyucu veya interceptor içinden görelim.

`Reflector#createDecorator` kullanarak güçlü türde dekoratörler oluşturmak için, tür argümanını belirtmemiz gerekiyor. Örneğin, dizi tipinde bir argüman alan `Roles` adında bir dekoratör oluşturalım.

```ts
@@filename(roles.decorator)
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

Burada `Roles` dekoratörü, `string[]` türünde tek bir argümanı olan bir fonksiyondur.

Şimdi, bu dekoratörü kullanmak için, basitçe işleyiciye onu ekleyelim:

```typescript
@@filename(cats.controller)
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles(['admin'])
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

Burada, `create()` yöntemine `Roles` dekoratör metadata'sını ekledik ve bu rotaya sadece `admin` rolüne sahip kullanıcıların erişmesine izin verildiğini belirttik.

Rotanın rol(ler)ine (özel metadata) erişmek için, `Reflector` yardımcı sınıfını yine kullanacağız. `Reflector`, bir sınıfa normal bir şekilde enjekte edilebilir:

```typescript
@@filename(roles.guard)
@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
@@switch
@Injectable()
@Dependencies(Reflector)
export class CatsService {
  constructor(reflector) {
    this.reflector = reflector;
  }
}
```

> info **İpucu** `Reflector` sınıfı, `@nestjs/core` paketinden içe aktarılır.

Şimdi, işleyici metadata'sını okumak için `get()` yöntemini kullanabiliriz:

```typescript
const roles = this.reflector.get(Roles, context.getHandler());
```

`Reflector#get` yöntemi, metadata'ya erişmemizi sağlar ve iki argümanı alır: bir dekoratör referansı ve metadata'yı almak için bir **context** (dekoratör hedefi). Bu örnekte belirtilen **dekoratör** `Roles`'dir (yukarıdaki `roles.decorator.ts` dosyasına geri dönün). Context, `context.getHandler()` çağrısı tarafından sağlanır ve bu da şu anda işlenen rota işleyicisinin metadata'sını çıkarmak anlamına gelir. Unutmayın, `getHandler()` bize rota işleyici fonksiyonuna bir **referans** verir.

Alternatif olarak, metadata'yı tüm rotalara uygulayarak kontrolör düzeyinde organize edebiliriz.

```typescript
@@filename(cats.controller)
@Roles(['admin'])
@Controller('cats')
export class CatsController {}
@@switch
@Roles(['admin'])
@Controller('cats')
export class CatsController {}
```

Bu durumda, kontrolör metadata'sını çıkarmak için ikinci argüman olarak `context.getClass()`'i (metadata çıkarmak için kontrolör sınıfını sağlamak için) kullanırız, `context.getHandler()` yerine:

```typescript
@@filename(roles.guard)
const roles = this.reflector.get(Roles, context.getClass());
```

Birden çok seviyede metadata sağlama yeteneğiniz olduğundan, birkaç bağlamadan metadata çıkarmak ve birleştirmek için iki yardımcı yöntem sunar `Reflector` sınıfı. Bu yöntemler, metadata'yı **hem** kontrolör hem de yöntem seviyelerinde birleştirir ve farklı yollarla birleştirir.

Aşağıdaki senaryoyu düşünün, `Roles` metadata'sını her iki seviyede de sağladınız.

```typescript
@@filename(cats.controller)
@Roles(['user'])
@Controller('cats')
export class CatsController {
  @Post()
  @Roles(['admin'])
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }
}
@@switch
@Roles(['user'])
@Controller('cats')
export class CatsController {}
  @Post()
  @Roles(['admin'])
  @Bind(Body())
  async create(createCatDto) {
    this.catsService.create(createCatDto);
  }
}
```

Eğer amacınız `'user'`'ı varsayılan rol olarak belirtmek ve belirli yöntemler için seçmeli olarak üzerine yazmaksa, muhtemelen `getAllAndOverride()` yöntemini kullanırsınız.

```typescript
const roles = this.reflector.getAllAndOverride(Roles, [context.getHandler(), context.getClass()]);
```

Bu kodu içeren bir koruyucu, yukarıdaki metadata ile `create()` yöntemi bağlamında çalışırken, `roles`'ün `['admin']` içerdiği sonucuna varacaktır.

Her ikisini birleştirmek ve metadata'yı birleştirmek için (bu yöntem hem dizileri hem de nesneleri birleştirir), `getAllAndMerge()` yöntemini kullanın:

```typescript
const roles = this.reflector.getAllAndMerge(Roles, [context.getHandler(), context.getClass()]);
```

Bu, `roles`'ün `['user', 'admin']` içerdiği bir sonuca yol açacaktır.

Her iki birleştirme yöntemi için, metadata anahtarını birinci argüman olarak, metadata hedef bağlamlarının bir dizisini (yani `getHandler()` ve/veya `getClass())` yöntemlerine yapılan çağrılar) ikinci argüman olarak iletilir.

#### Düşük Seviyeli Yaklaşım

Daha önce bahsedildiği gibi, `Reflector#createDecorator` kullanmak yerine, yerleşik `@SetMetadata()` dekoratörünü bir işleyiciye metadata eklemek için de kullanabilirsiniz.

```typescript
@@filename(cats.controller)
@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@SetMetadata('roles', ['admin'])
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **İpucu** `@SetMetadata()` dekoratörü, `@nestjs/common` paketinden içe aktarılır.

Yukarıdaki yapıyla, `create()` yöntemine `roles` metadata'sini (roles bir metadata anahtarıdır ve `['admin']` buna ilişkin değerdir) ekledik. Bu çalışsa da, rotalarınızda `@SetMetadata()`'yi doğrudan kullanmak iyi bir uygulama değildir. Bunun yerine, aşağıda gösterildiği gibi kendi dekoratörlerinizi oluşturabilirsiniz:

```typescript
@@filename(roles.decorator)
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
@@switch
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles) => SetMetadata('roles', roles);
```

Bu yaklaşım çok daha temiz ve okunabilir, ve biraz da `Reflector#createDecorator` yaklaşımına benzer. Fark, `@SetMetadata` ile metadata anahtarını ve değerini daha fazla kontrol edebilmenizdir ve ayrıca birden fazla argüman alan dekoratörler oluşturabilirsiniz.

Artık özel `@Roles()` dekoratörümüz olduğuna göre, bunu `create()` yöntemini dekore etmek için kullanabiliriz.

```typescript
@@filename(cats.controller)
@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles('admin')
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

Rotanın rol(ler)ine (özel metadata) erişmek için, yine `Reflector` yardımcı sınıfını kullanacağız:

```typescript
@@filename(roles.guard)
@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
@@switch
@Injectable()
@Dependencies(Reflector)
export class CatsService {
  constructor(reflector) {
    this.reflector = reflector;
  }
}
```

> info **İpucu** `Reflector` sınıfı, `@nestjs/core` paketinden içe aktarılır.

Şimdi, işleyici metadata'sini okumak için `get()` yöntemini kullanabiliriz.

```typescript
const roles = this.reflector.get<string[]>('roles', context.getHandler());
```

Burada bir dekoratör referansı yerine, ilk argüman olarak metadata **anahtarını** (ki bizim durumumuzda `'roles'`) geçiriyoruz. Diğer her şey, `Reflector#createDecorator` örneğindeki gibi aynı kalır.
