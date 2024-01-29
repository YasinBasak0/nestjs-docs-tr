### MikroORM

Bu reçete, MikroORM'u Nest içinde kullanmaya başlamak isteyen kullanıcılara yardımcı olmak için burada bulunmaktadır. MikroORM, Data Mapper, Unit of Work ve Identity Map desenlerine dayanan Node.js için TypeScript ORM'dir. TypeORM için harika bir alternatiftir ve TypeORM'den geçiş oldukça kolay olmalıdır. MikroORM hakkındaki tam belgeler [burada](https://mikro-orm.io/docs) bulunabilir.

> ℹ️ **Bilgi** `@mikro-orm/nestjs`, NestJS çekirdek ekibi tarafından yönetilmeyen üçüncü taraf bir pakettir. Lütfen kütüphane ile ilgili bulduğunuz herhangi bir sorunu [ilgili depoda](https://github.com/mikro-orm/nestjs) bildirin.

#### Kurulum

MikroORM'u Nest'e entegre etmenin en kolay yolu [`@mikro-orm/nestjs` modülü](https://github.com/mikro-orm/nestjs) aracılığıyla yapmaktır.
Sadece Nest, MikroORM ve altındaki sürücüyü yan yana kurun:

```bash
$ npm i @mikro-orm/core @mikro-orm/nestjs @mikro-orm/mysql # mysql/mariadb için
```

MikroORM ayrıca `postgres`, `sqlite` ve `mongo`'yu destekler. Tüm sürücüler için [resmi belgelere](https://mikro-orm.io/docs/usage-with-sql/) bakın.

Kurulum işlemi tamamlandıktan sonra, `MikroOrmModule`'u kök `AppModule`'a ekleyebiliriz.

```typescript
@Module({
  imports: [
    MikroOrmModule.forRoot({
      entities: ['./dist/entities'],
      entitiesTs: ['./src/entities'],
      dbName: 'my-db-name.sqlite3',
      type: 'sqlite',
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`forRoot()` yöntemi, MikroORM paketinden `init()` ile aynı yapılandırma nesnesini kabul eder. Tam yapılandırma belgesi için [bu sayfaya](https://mikro-orm.io/docs/configuration) bakın.

Alternatif olarak [CLI'yi yapılandırabilirsiniz](https://mikro-orm.io/docs/installation#setting-up-the-commandline-tool) ve ardından herhangi bir argüman olmadan `forRoot()`'u çağırabilirsiniz. Bu, ağaç sallama kullanan bir derleme aracı kullandığınızda çalışmaz.

```typescript
@Module({
  imports: [
    MikroOrmModule.forRoot(),
  ],
  ...
})
export class AppModule {}
```

Ardından, `EntityManager` proje genelinde enjekte edilebilir durumda olacaktır (başka bir yerde herhangi bir modül içe aktarılmadan).

```ts
import { MikroORM } from '@mikro-orm/core';
// EntityManager'ı sürücü paketinizden veya `@mikro-orm/knex`'den içe aktarın
import { EntityManager } from '@mikro-orm/mysql';

@Injectable()
export class MyService {
  constructor(
    private readonly orm: MikroORM,
    private readonly em: EntityManager,
  ) {}
}
```

> ℹ️ **Bilgi** `EntityManager`, `@mikro-orm/driver` paketinden içe aktarılmaktadır, burada sürücü `mysql`, `sqlite`, `postgres` veya kullandığınız sürücü ne olursa olsun. `@mikro-orm/knex`'i bağımlılık olarak kuruluysa, `EntityManager`'ı oradan da içe aktarabilirsiniz.

#### Depolar

MikroORM, depo tasarım desenini destekler. Her varlık için bir depo oluşturabiliriz. Depolar hakkındaki tam belgeleri [burada](https://mikro-orm.io/docs/repositories) bulabilirsiniz. Mevcut kapsamda hangi depoların kaydedileceğini belirlemek için `forFeature()` yöntemini kullanabiliriz. Örneğin, şu şekilde:

> ℹ️ **Bilgi** Ana varlıklarınızı `forFeature()` ile kaydetmemelisiniz, çünkü bunlar için depo yoktur. Öte yandan, ana varlıklarınızın `forRoot()` (veya genel olarak ORM yapılandırmasında) listesinin bir parçası olması gerekir.

```typescript
// photo.module.ts
@Module({
  imports: [MikroOrmModule.forFeature([Photo])],
  providers: [PhotoService],
  controllers: [PhotoController],
})
export class PhotoModule {}
```

ve

 bunu kök `AppModule`'a içe aktarın:

```typescript
// app.module.ts
@Module({
  imports: [MikroOrmModule.forRoot(...), PhotoModule],
})
export class AppModule {}
```

Bu şekilde, `@InjectRepository()` dekoratörünü kullanarak `PhotoService`'e `PhotoRepository`'yi enjekte edebiliriz:

```typescript
@Injectable()
export class PhotoService {
  constructor(
    @InjectRepository(Photo)
    private readonly photoRepository: EntityRepository<Photo>,
  ) {}
}
```

#### Özel depoları kullanma

Özel depoları kullanırken, `@InjectRepository()` dekoratörüne ihtiyaç duymadan, depolarımızı `getRepositoryToken()` metodunun yaptığı gibi adlandırarak ihtiyacı ortadan kaldırabiliriz:

```ts
export const getRepositoryToken = <T>(entity: EntityName<T>) =>
  `${Utils.className(entity)}Repository`;
```

Başka bir deyişle, depoyu varlığın adıyla aynı şekilde adlandırdığımız sürece, `Repository` soneki eklenmiş olarak, depo otomatik olarak Nest DI konteynerine kaydedilecektir.

```ts
// `**./author.entity.ts**`
@Entity()
export class Author {
  // `em.getRepository()` içinde çıkarım yapabilmek için
  [EntityRepositoryType]?: AuthorRepository;
}

// `**./author.repository.ts**`
@Repository(Author)
export class AuthorRepository extends EntityRepository<Author> {
  // özel metodlarınız...
}
```

Özel depo adı, `getRepositoryToken()`'ın döndüğüyle aynı olduğu için artık `@InjectRepository()` dekoratörüne ihtiyacımız yoktur:

```ts
@Injectable()
export class MyService {
  constructor(private readonly repo: AuthorRepository) {}
}
```

#### Varlıkları Otomatik Yükleme

> ℹ️ **Bilgi** `autoLoadEntities` seçeneği v4.1.0 sürümüne eklendi.

Bağlantı seçeneklerinin varlık dizisine varlıkları manuel olarak eklemek zahmetli olabilir. Ayrıca, varlıklara kök modülden başvurmak uygulama alanı sınırlarını bozar ve diğer uygulama bölümlerine uygulama ayrıntılarını sızdırır. Bu sorunu çözmek için statik glob yolları kullanılabilir.

Ancak, glob yolları webpack tarafından desteklenmez, bu nedenle uygulamanızı bir monorepo içinde oluşturuyorsanız bunları kullanamazsınız. Bu sorunu çözmek için alternatif bir çözüm sunulmuştur. Varlıkları otomatik olarak yüklemek için, yapılandırma nesnesinin ( `forRoot()` yöntemine iletilen) `autoLoadEntities` özelliğini `true` olarak ayarlayın, aşağıda gösterildiği gibi:

```ts
@Module({
  imports: [
    MikroOrmModule.forRoot({
      ...
      autoLoadEntities: true,
    }),
  ],
})
export class AppModule {}
```

Bu seçenek belirtildiğinde, `forFeature()` yöntemi aracılığıyla kaydedilen her varlık otomatik olarak yapılandırma nesnesinin varlıklar dizisine eklenir.

> ℹ️ **Bilgi** `forFeature()` yöntemi aracılığıyla kaydedilmeyen, ancak yalnızca varlıktan (ilişki yoluyla) başvurulan varlıklar, `autoLoadEntities` ayarı yoluyla dahil edilmez.

> ℹ️ **Bilgi** `autoLoadEntities` kullanmak, MikroORM CLI üzerinde hiçbir etkisi yoktur - bunun için hala tüm varlıkların bulunduğu CLI yapılandırmasına ihtiyacımız vardır. Öte yandan, burada globları kullanabiliriz, çünkü CLI webpack üzerinden gitmeyecek.

#### Serileştirme

> ⚠️ **Not** MikroORM, her varlık ilişkisini daha iyi tür güvenliği sağlamak amacıyla bir `Reference<T>` veya `Collection<T>` nesnesine sarar. Bu, [Nest'in yerleşik serileştiricisinin](/docs/techniques/serialization) sarılmış ilişkilere kör olmasına neden olur. Başka bir deyişle, HTTP veya WebSocket işleyicilerinizden MikroORM varlıklarını döndürüyorsanız, tüm ilişkileri SERİLEŞTİRİLMEZ.

Neyse ki, MikroORM, `ClassSerializerInterceptor` yerine kullanılabilecek bir [serileştirme API](https://mikro-orm.io/docs/serializing) sağlar.

```typescript
@Entity()
export class Book {
  @Property({ hidden: true }) // class-transformer'ın `@Exclude`'nin eşdeğeri
  hiddenField = Date.now();

  @Property({ persist: false }) // class-transformer'ın `@Expose()`'ına benzer. Sadece bellekte var olacak ve seri hale getirilecek.
  count?: number;

  @ManyToOne({
    serializer: (value) => value.name,
    serializedName: 'authorName',
  }) // class-transformer'ın `@Transform()`'un eşdeğeri
  author: Author;
}
```

#### Kuyruklarda İstek Kapsamlı İşleyiciler

> ℹ️ **Bilgi** `@UseRequestContext()` dekoratörü v4.1.0 sürümüne eklendi

[Mikro-ORM belgelerinde](https://mikro-orm.io/docs/identity-map) belirtildiği gibi, her istek için temiz bir duruma ihtiyacımız var. Bu, middleware aracılığıyla kaydedilen `RequestContext` yardımcısı sayesinde otomatik olarak ele alınır.

Ancak middleware'ler yalnızca düzenli HTTP isteği işleyicileri için yürütülür, peki ya böyle bir işlemin dışında istek kapsamlı bir yönteme ihtiyacımız olursa? Bu, kuyruk işleyicileri veya zamanlanmış görevlerin bir örneğidir.

`@UseRequestContext()` dekoratörünü kullanabiliriz. İlk olarak mevcut bağlam için `MikroORM` örneğini enjekte etmeniz gerektiğini ister, ardından sizin için bağlam oluşturulacaktır. Dekoratör, yönteminiz için yeni bir istek bağlamını kaydeder ve içinde yürütür.

```ts
@Injectable()
export class MyService {
  constructor(private readonly orm: MikroORM) {}

  @UseRequestContext()
  async doSomething() {
    // bu, ayrı bir bağlam içinde yürütülecek
  }
}
```

#### İstek bağlamı için `AsyncLocalStorage` Kullanımı

Varsayılan olarak, `RequestContext` yardımcısı içinde `domain` API'si kullanılır. `@mikro-orm/core@4.0.3` sürümünden itibaren, güncel bir node sürümündeyseniz yeni `AsyncLocalStorage`'ı da kullanabilirsiniz:

```typescript
// yeni (global) depolama örneği oluştur
const storage = new AsyncLocalStorage<EntityManager>();

@Module({
  imports: [
    MikroOrmModule.forRoot({
      // ...
      registerRequestContext: false, // otomatik middleware'i devre dışı bırak
      context: () => storage.getStore(), // kendi AsyncLocalStorage örneğimizi kullan
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}

// istek bağlamı middleware'ini kaydet
const app = await NestFactory.create(AppModule, { ... });
const orm = app.get(MikroORM);

app.use((req, res, next) => {
  storage.run(orm.em.fork(true, true), next);
});
```

#### Test

`@mikro-orm/nestjs` paketi, depoyu sahteleştirmek için kullanılmak üzere bir hazırlık belirteci döndüren `getRepositoryToken()` işlevini açığa çıkarır.

```typescript
@Module({
  providers: [
    PhotoService,
    {
      provide: getRepositoryToken(Photo),
      useValue: mockedRepository,
    },
  ],
})
export class PhotoModule {}
```

#### Örnek

NestJS ile MikroORM kullanımının gerçek dünya örneğini [burada](https://github.com/mikro-orm/nestjs-realworld-example-app) bulabilirsiniz.