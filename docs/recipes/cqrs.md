### CQRS

Basit [CRUD](https://tr.wikipedia.org/wiki/CRUD) (Create, Read, Update ve Delete) uygulamalarının akışı aşağıdaki gibi açıklanabilir:

1. Kontrolcü katmanı HTTP isteklerini ele alır ve görevleri hizmet katmanına delegeler.
2. Hizmet katmanı, iş mantığının büyük çoğunluğunun bulunduğu yerdir.
3. Hizmetler, varlıkları değiştirmek / kalıcı hale getirmek için depoları / DAO'ları kullanır.
4. Varlıklar, değerleri depolayan, setter ve getter'lara sahip konteynerler olarak hareket eder.

Bu model genellikle küçük ve orta ölçekli uygulamalar için yeterli olabilir, ancak daha büyük, daha karmaşık uygulamalar için en iyi seçenek olmayabilir. Bu durumlarda, **CQRS** (Command and Query Responsibility Segregation - Komut ve Sorgu Sorumluluk Ayrımı) modeli, uygulamanın gereksinimlerine bağlı olarak daha uygun ve ölçeklenebilir olabilir. Bu modelin avantajları şunlardır:

- **İlgili Konuların Ayrılması**: Model, okuma ve yazma işlemlerini ayrı modellere ayırır.
- **Ölçeklenebilirlik**: Okuma ve yazma işlemleri bağımsız bir şekilde ölçeklendirilebilir.
- **Esneklik**: Model, okuma ve yazma işlemleri için farklı veri depolarının kullanılmasına izin verir.
- **Performans**: Model, okuma ve yazma işlemleri için optimize edilmiş farklı veri depolarının kullanılmasına izin verir.

Bu modeli kolaylaştırmak için, Nest hafif bir [CQRS modülü](https://github.com/nestjs/cqrs) sağlar. Bu bölüm, onu nasıl kullanacağınızı açıklar.

#### Kurulum

İlk olarak gerekli paketi yükleyin:

```bash
$ npm install --save @nestjs/cqrs
```

#### Komutlar

Komutlar, uygulama durumunu değiştirmek için kullanılır. Bunlar, veri merkezli olmaktan ziyade görev odaklı olmalıdır. Bir komut gönderildiğinde, karşılık gelen bir **Komut İşleyici** tarafından işlenir. İşleyici, uygulama durumunu güncelleme sorumluluğuna sahiptir.

```typescript
@@filename(heroes-game.service)
@Injectable()
export class HeroesGameService {
  constructor(private commandBus: CommandBus) {}

  async killDragon(heroId: string, killDragonDto: KillDragonDto) {
    return this.commandBus.execute(
      new KillDragonCommand(heroId, killDragonDto.dragonId)
    );
  }
}
@@switch
@Injectable()
@Dependencies(CommandBus)
export class HeroesGameService {
  constructor(commandBus) {
    this.commandBus = commandBus;
  }

  async killDragon(heroId, killDragonDto) {
    return this.commandBus.execute(
      new KillDragonCommand(heroId, killDragonDto.dragonId)
    );
  }
}
```

Yukarıdaki kod örneğinde, `KillDragonCommand` sınıfını örnekleriz ve onu `CommandBus`'un `execute()` metoduna geçiririz. İşte gösterilen komut sınıfı:

```typescript
@@filename(kill-dragon.command)
export class KillDragonCommand {
  constructor(
    public readonly heroId: string,
    public readonly dragonId: string,
  ) {}
}
@@switch
export class KillDragonCommand {
  constructor(heroId, dragonId) {
    this.heroId = heroId;
    this.dragonId = dragonId;
  }
}
```

`CommandBus`, komutların bir **akışını** temsil eder. Komutları uygun işleyicilere göndermekten sorumludur. `execute()` metodu, işleyici tarafından döndürülen değere çözünen bir promise döndürür.

`KillDragonCommand` komutu için bir işleyici oluşturalım.

```typescript
@@filename(kill-dragon.handler)
@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
  constructor(private repository: HeroRepository) {}

  async execute(command: KillDragonCommand) {
    const { heroId, dragonId } = command;
    const hero = this.repository.findOneById(+heroId);

    hero.killEnemy(dragonId);
    await this.repository.persist(hero);
  }
}
@@switch
@CommandHandler(KillDragonCommand)
@Dependencies(HeroRepository)
export class KillDragonHandler {
  constructor(repository) {
    this.repository = repository;
  }

  async execute(command) {
    const { heroId, dragonId } = command;
    const hero = this.repository.findOneById(+heroId);

    hero.killEnemy(dragonId);
    await this.repository.persist(hero);
  }
}
```

Bu işleyici, depodan `Hero` varlığını alır, `killEnemy()` metodunu çağırır ve ardından değişiklikleri kalıcı hale getirir. `KillDragonHandler` sınıfı, `ICommandHandler` arabirimini uygular ve `execute()` metodunu uygular. `execute()` metodu, komut nesnesini bir argüman olarak alır.

#### Sorgular

Sorgular, uygulama durumundan veri almak için kullanılır. Bunlar görev odaklı olmaktan ziyade veri odaklı olmalıdır. Bir sorgu gönderildiğinde, karşılık gelen bir **Sorgu İşleyici** tarafından işlenir. İşleyici, verilerin alınmasından sorumludur.

`QueryBus`, `CommandBus` ile aynı kalıbı takip eder. Sorgu işleyicileri `IQueryHandler` arabirimini uygulamalı ve `@QueryHandler()` dekoratörü ile işaretlenmelidir.

#### Olaylar

Olaylar, uygulama durumundaki değişiklikleri diğer uygulama bileşenlerine bildirmek için kullanılır. **Modeller** tarafından veya doğrudan `EventBus` kullanılarak gönderilirler. Bir olay gönderildiğinde, karşılık gelen **Olay İşleyicileri** tarafından işlenir. İşleyiciler, örneğin okuma modelini güncelleyebilir.

Demonstrasyon amaçlı olarak, bir olay sınıfı oluşturalım:

```typescript
@@filename(hero-killed-dragon.event)
export class HeroKilledDragonEvent {
  constructor(
    public readonly heroId: string,
    public readonly dragonId: string,
  ) {}
}
@@switch
export class HeroKilledDragonEvent {
  constructor(heroId, dragonId) {
    this.heroId = heroId;
    this.dragonId = dragonId;
  }
}
```

Şimdi, olaylar doğrudan `EventBus.publish()` yöntemi kullanılarak gönderilebildiği gibi, aynı zamanda model tarafından da gönderilebilir. `Hero` modelini, `killEnemy()` yöntemi çağrıldığında `HeroKilledDragonEvent` olayını gönderecek şekilde güncelleyelim.

```typescript
@@filename(hero.model)
export class Hero extends AggregateRoot {
  constructor(private id: string) {
    super();
  }

  killEnemy(enemyId: string) {
    // İş mantığı
    this.apply(new HeroKilledDragonEvent(this.id, enemyId));
  }
}
@@switch
export class Hero extends AggregateRoot {
  constructor(id) {
    super();
    this.id = id;
  }

  killEnemy(enemyId) {
    // İş mantığı
    this.apply(new HeroKilledDragonEvent(this.id, enemyId));
  }
}
```

`apply()` yöntemi, olayları göndermek için kullanılır. Bir olay nesnesini argüman olarak alır. Ancak, modelimiz `EventBus`'tan haberdar olmadığından, onu modelle ilişkilendirmemiz gerekir. Bunu, `EventPublisher` sınıfını kullanarak yapabiliriz.

```typescript
@@filename(kill-dragon.handler)
@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
  constructor(
    private repository: HeroRepository,
    private publisher: EventPublisher,
  ) {}

  async execute(command: KillDragonCommand) {
    const { heroId, dragonId } = command;
    const hero = this.publisher.mergeObjectContext(
      await this.repository.findOneById(+heroId),
    );
    hero.killEnemy(dragonId);
    hero.commit();
  }
}
@@switch
@CommandHandler(KillDragonCommand)
@Dependencies(HeroRepository, EventPublisher)
export class KillDragonHandler {
  constructor(repository, publisher) {
    this.repository = repository;
    this.publisher = publisher;
  }

  async execute(command) {
    const { heroId, dragonId } = command;
    const hero = this.publisher.mergeObjectContext(
      await this.repository.findOneById(+heroId),
    );
    hero.killEnemy(dragonId);
    hero.commit();
  }
}
```

`EventPublisher#mergeObjectContext` yöntemi, olay yayıncısını sağlanan nesneye birleştirir, bu da demektir ki nesne artık olayları olay akışına gönderebilecek.

Bu örnekte dikkat edilmesi gereken bir başka nokta da model üzerinde `commit()` yöntemini çağırıyor olmamızdır. Bu yöntem, bekleyen olayları göndermek için kullanılır. Olayları otomatik olarak göndermek için, `autoCommit` özelliğini `true` olarak ayarlayabiliriz:

```typescript
export class Hero extends AggregateRoot {
  constructor(private id: string) {
    super();
    this.autoCommit = true;
  }
}
```

Eğer olay yayıncısını mevcut olmayan bir nesne yerine, bir sınıfa birleştirmek istersek, `EventPublisher#mergeClassContext` yöntemini kullanabiliriz:

```typescript
const HeroModel = this.publisher.mergeClassContext(Hero);
const hero = new HeroModel('id'); // <-- HeroModel bir sınıf
```

Artık `HeroModel` sınıfının her örneği, `mergeObjectContext()` yöntemini kullanmadan olayları gönderebilecek.

Ayrıca, `EventBus` kullanarak olayları manuel olarak da gönderebiliriz:

```typescript
this.eventBus.publish(new HeroKilledDragonEvent());
```

> info **İpucu** `EventBus`, enjekte edilebilir bir sınıftır.

Her olayın birden fazla **Olay İşleyicisi** olabilir.

```typescript
@@filename(hero-killed-dragon.handler)
@EventsHandler(HeroKilledDragonEvent)
export class HeroKilledDragonHandler implements IEventHandler<HeroKilledDragonEvent> {
  constructor(private repository: HeroRepository) {}

  handle(event: HeroKilledDragonEvent) {
    // İş mantığı
  }
}
```

> info **İpucu** Olay işleyicilerini kullanmaya başladığınızda, geleneksel HTTP web bağlamından çıkarsınız.
>
> - `CommandHandlers` içindeki hatalar hala yerleşik [İstisna filtreleri](/docs/exception-filters) tarafından yakalanabilir.
> - `EventHandlers` içindeki hatalar İstisna filtreleri tarafından yakalanamaz: bunları manuel olarak ele almanız gerekecektir. Basit bir `try/catch` kullanarak, [Sagas](/docs/recipes/cqrs#sagas) kullanarak dengeleme olayını tetikleyerek veya başka bir çözüm kullanarak.
> - `CommandHandlers` içinde HTTP Yanıtları hala istemciye gönderilebilir.
> - `EventHandlers` içindeki HTTP Yanıtları gönderilemez. İstemciye bilgi göndermek istiyorsanız [WebSocket](/docs/websockets/gateways), [SSE](/docs/techniques/server-sent-events) veya seçtiğiniz başka bir çözümü kullanabilirsiniz.

#### Sagalar

Saga, olayları dinleyen ve yeni komutları tetikleyebilen uzun süre çalışan bir süreçtir. Genellikle uygulamadaki karmaşık iş akışlarını yönetmek için kullanılır. Örneğin, bir kullanıcı kaydolduğunda, bir saga `UserRegisteredEvent`'i dinleyebilir ve kullanıcıya hoş geldin e-postası gönderebilir.

Sagalar son derece güçlü bir özelliktir. Tek bir saga, 1..\* olayı dinleyebilir. [RxJS](https://github.com/ReactiveX/rxjs) kütüphanesini kullanarak, olay akışlarını filtreleyebilir, eşleyebilir, çatallayabilir ve birleştirebiliriz, böylece karmaşık iş akışları oluşturabiliriz. Her saga, bir komut örneği üreten bir `Observable` döndürür. Bu komut daha sonra `CommandBus` tarafından **asenkron olarak** gönderilir.

Hadi, `HeroKilledDragonEvent`'i dinleyen ve `DropAncientItemCommand` komutunu gönderen bir saga oluşturalım.

```typescript
@@filename(heroes-game.saga)
@Injectable()
export class HeroesGameSagas {
  @Saga()
  dragonKilled = (events$: Observable<any>): Observable<ICommand> => {
    return events$.pipe(
      ofType(HeroKilledDragonEvent),
      map((event) => new DropAncientItemCommand(event.heroId, fakeItemID)),
    );
  }
}
@@switch
@Injectable()
export class HeroesGameSagas {
  @Saga()
  dragonKilled = (events$) => {
    return events$.pipe(
      ofType(HeroKilledDragonEvent),
      map((event) => new DropAncientItemCommand(event.heroId, fakeItemID)),
    );
  }
}
```

> info **İpucu** `ofType` operatörü ve `@Saga()` dekoratörü, `@nestjs/cqrs` paketinden alınır.

`@Saga()` dekoratörü, yöntemi bir saga olarak işaretler. `events$` argümanı, tüm olayların bir `Observable` akışıdır. `ofType` operatörü, belirtilen olay türüne göre akışı filtreler. `map` operatörü, olayı yeni bir komut örneğine eşler.

Bu örnekte, `HeroKilledDragonEvent`'i `DropAncientItemCommand` komutuna eşliyoruz. `DropAncientItemCommand` komutu daha sonra `CommandBus` tarafından otomatik olarak gönderilir.

#### Kurulum

Son olarak, tüm komut işleyicilerini, olay işleyicilerini ve sagaları `HeroesGameModule`'a kaydetmemiz gerekiyor:

```typescript
@@filename(heroes-game.module)
export const CommandHandlers = [KillDragonHandler, DropAncientItemHandler];
export const EventHandlers =  [HeroKilledDragonHandler, HeroFoundItemHandler];

@Module({
  imports: [CqrsModule],
  controllers: [HeroesGameController],
  providers: [
    HeroesGameService,
    HeroesGameSagas,
    ...CommandHandlers,
    ...EventHandlers,
    HeroRepository,
  ]
})
export class HeroesGameModule {}
```

#### İşlenmemiş istisnalar

Olay işleyicileri asenkron bir şekilde çalıştırılır. Bu, uygulamanın tutarsız bir duruma girmesini önlemek için tüm istisnaları her zaman ele alması gerektiği anlamına gelir. Ancak bir istisna ele alınmazsa, `EventBus`, `UnhandledExceptionInfo` nesnesini oluşturur ve bunu `UnhandledExceptionBus` akışına iter. Bu akış, işlenmemiş istisnaları işlemek için kullanılabilen bir `Observable`'dir.

```typescript
private destroy$ = new Subject<void>();

constructor(private unhandledExceptionsBus: UnhandledExceptionBus) {
  this.unhandledExceptionsBus
    .pipe(takeUntil(this.destroy$))
    .subscribe((exceptionInfo) => {
      // İstisnanın burada ele alınması
      // örneğin, bunu harici bir servise göndermek, işlemi sona erdirmek veya yeni bir olayı yayınlamak
    });
}

onModuleDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

İstisnaları filtrelemek için `ofType` operatörünü kullanabiliriz, örneğin:

```typescript
this.unhandledExceptionsBus.pipe(takeUntil(this.destroy$), UnhandledExceptionBus.ofType(TransactionNotAllowedException)).subscribe((exceptionInfo) => {
  // İstisnanın burada ele alınması
});
```

Burada `TransactionNotAllowedException` filtrelemek istediğimiz istisnadır.

`UnhandledExceptionInfo` nesnesi aşağıdaki özelliklere sahiptir:

```typescript
export interface UnhandledExceptionInfo<Cause = IEvent | ICommand, Exception = any> {
  /**
   * Atılan istisna.
   */
  exception: Exception;
  /**
   * İstisna nedeni (olay veya komut referansı).
   */
  cause: Cause;
}
```

#### Tüm olaylara abone olma

`CommandBus`, `QueryBus` ve `EventBus` hepsi de **Observables**'dir. Bu, tüm akışa abone olabileceğimiz ve örneğin tüm olayları işleyebileceğimiz anlamına gelir. Örneğin, tüm olayları konsola kaydedebilir veya olayları olay deposuna kaydedebiliriz.

```typescript
private destroy$ = new Subject<void>();

constructor(private eventBus: EventBus) {
  this.eventBus
    .pipe(takeUntil(this.destroy$))
    .subscribe((event) => {
      // Olayları veritabanına kaydet
    });
}

onModuleDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

#### Örnek

Çalışan bir örnek [burada](https://github.com/kamilmysliwiec/nest-cqrs-example) bulunmaktadır.