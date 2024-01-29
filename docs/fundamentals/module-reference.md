### Modül Referansı

Nest, iç servislerin listesinde gezinmek ve bir sağlayıcıya herhangi bir sağlayıcıya enjeksiyon belirteci olarak kullanarak başvurmak için `ModuleRef` sınıfını sağlar. `ModuleRef` sınıfı, hem statik hem de kapsamlı sağlayıcıları dinamik olarak örneklemenin bir yolunu sağlar. `ModuleRef`, normal bir şekilde bir sınıfa enjekte edilebilir:

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(private moduleRef: ModuleRef) {}
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }
}
```

> info **Hint** `ModuleRef` sınıfı, `@nestjs/core` paketinden içe aktarılır.

#### Örnekleri Almak

`ModuleRef` örneği (bundan sonra bunu **modül referansı** olarak adlandıracağız) bir `get()` yöntemine sahiptir. Bu yöntem, enjeksiyon belirteci/sınıf adı kullanarak **mevcut** modülde var olan (öğrenilmiş) bir sağlayıcıyı, denetleyiciyi veya enjekte edilebilir öğeyi alır.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  private service: Service;
  constructor(private moduleRef: ModuleRef) {}

  onModuleInit() {
    this.service = this.moduleRef.get(Service);
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  onModuleInit() {
    this.service = this.moduleRef.get(Service);
  }
}
```

> warning **Uyarı** `get()` yöntemi ile kapsamlı sağlayıcıları (geçici veya istek kapsamlı) alamazsınız. Bunun yerine <a href="https://docs.nestjs.com/fundamentals/module-ref#resolving-scoped-providers">aşağıda</a> açıklanan teknikleri kullanın. Kapsamları nasıl kontrol edeceğinizi öğrenmek için [buraya](/docs/fundamentals/injection-scopes) bakın.

Bir sağlayıcıyı genel bağlamdan almak için (örneğin, sağlayıcı başka bir modülde enjekte edilmişse), `get()` yöntemine ikinci bir argüman olarak `{{ '{' }} strict: false {{ '}' }}` seçeneğini geçirin.

```typescript
this.moduleRef.get(Service, { strict: false });
```

#### Kapsamlı Sağlayıcıları Çözmek

Bir kapsamlı sağlayıcıyı (geçici veya istek kapsamlı) dinamik olarak çözmek için `resolve()` yöntemini kullanın ve sağlayıcının enjeksiyon belirtecini bir argüman olarak geçirin.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  private transientService: TransientService;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.transientService = await this.moduleRef.resolve(TransientService);
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    this.transientService = await this.moduleRef.resolve(TransientService);
  }
}
```

`resolve()` yöntemi, sağlayıcının kendi **DI konteyner alt-ağacından** benzersiz bir örneği döndürür. Her alt-ağaç benzersiz bir **bağlam tanımlayıcısı** içerir. Bu nedenle, bu yöntemi birden fazla kez çağırırsanız ve örnek başvurularını karşılaştırırsanız, eşit olmadıklarını göreceksiniz.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService),
      this.moduleRef.resolve(TransientService),
    ]);
    console.log(transientServices[0] === transientServices[1]); // false
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService),
      this.moduleRef.resolve(TransientService),
    ]);
    console.log(transientServices[0] === transientServices[1]); // false
  }
}
```

Birden fazla `resolve()` çağrısında tek bir örnek oluşturmak ve bunların aynı oluşturulan DI konteyner alt-ağacını paylaştığından emin olmak için, `resolve()` yöntemine bir bağlam tanımlayıcı geçirebilirsiniz. `ContextIdFactory` sınıfını kullanarak bir bağlam tanımlayıcı oluşturan bu sınıf, uygun bir benzersiz tanımlayıcı döndüren bir `create()` yöntemi sağlar.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const contextId = ContextIdFactory.create();
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService, contextId),
      this.moduleRef.resolve(TransientService, contextId),
    ]);
    console.log(transientServices[0] === transientServices[1]); // true
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    const contextId = ContextIdFactory.create();
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService, contextId),
      this.moduleRef.resolve(TransientService, contextId),
    ]);
    console.log(transientServices[0] === transientServices[1]); // true
  }
}
```

> info **Hint** `ContextIdFactory` sınıfı, `@nestjs/core` paketinden içe aktarılır.

#### `REQUEST` Sağlayıcısını Kaydetme

Manuel olarak oluşturulan bağlam tanımlayıcıları (`ContextIdFactory.create()` ile) `REQUEST` sağlayıcısının, çünkü bunlar Nest bağımlılık enjeksiyon sistemi tarafından oluşturulmaz ve yönetilmez, `undefined` olduğu DI alt-ağaçlarını temsil eder.

Manuel olarak oluşturulan bir DI alt-ağacı için özel bir `REQUEST` nesnesini kaydetmek için `ModuleRef#registerRequestByContextId()` yöntemini kullanın, aşağıdaki gibi:

```typescript
const contextId = ContextIdFactory.create();
this.moduleRef.registerRequestByContextId(/docs/* SENIN_REQUEST_NESNEN */, contextId);
```

#### Mevcut Alt-Ağacı Almak

Bazen bir **istek bağlamı** içinde bir isteğe bağlı sağlayıcının bir örneğini çözmek isteyebilirsiniz. Diyelim ki `CatsService` bir istek kapsamlıdır ve aynı zamanda bir istek kapsamlı sağlayıcı olarak işaretlenmiş olan `CatsRepository` örneğini çözmek istiyorsunuz. Aynı DI konteyner alt-ağacını paylaşmak için, yeni bir tane oluşturmak yerine (örneğin, yukarıda gösterildiği gibi `ContextIdFactory.create()` fonksiyonu ile), mevcut bağlam tanımlayıcısını almanız gerekir. Mevcut bağlam tanımlayıcısını almak için, isteği `@Inject()` dekoratörünü kullanarak enjekte etmeye başlayın.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    @Inject(REQUEST) private request: Record<string, unknown>,
  ) {}
}
@@switch
@Injectable()
@Dependencies(REQUEST)
export class CatsService {
  constructor(request) {
    this.request = request;
  }
}
```

> info **Hint** İstek sağlayıcısı hakkında daha fazla bilgi için [buraya](https://docs.nestjs.com/fundamentals/injection-scopes#request-provider) bakın.

Şimdi, `ContextIdFactory` sınıfının `getByRequest()` yöntemini kullanarak bir bağlam tanımlayıcı oluşturun ve bunu `resolve()` çağrısına geçirin:

```typescript
const contextId = ContextIdFactory.getByRequest(this.request);
const catsRepository = await this.moduleRef.resolve(CatsRepository, contextId);
```

#### Özel Sınıfları Dinamik Olarak Oluşturma

Bir sınıfı **önceki** olarak **kaydedilmemiş bir sağlayıcı** olarak dinamik olarak oluşturmak için, modül referansının `create()` yöntemini kullanın.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  private catsFactory: CatsFactory;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.catsFactory = await this.moduleRef.create(CatsFactory);
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    this.catsFactory = await this.moduleRef.create(CatsFactory);
  }
}
```

Bu teknik, çerçeve konteynerinin dışında koşullu olarak farklı sınıfları instantiate etmenizi sağlar.

<app-banner-devtools></app-banner-devtools>