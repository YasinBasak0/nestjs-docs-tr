### Döngüsel Bağımlılık

Döngüsel bağımlılık, iki sınıfın birbirine bağlı olduğu durumlarda ortaya çıkar. Örneğin, A sınıfı B sınıfına ihtiyaç duyar ve B sınıfı da A sınıfına ihtiyaç duyar. Döngüsel bağımlılıklar, Nest arasında modüller ve sağlayıcılar arasında ortaya çıkabilir.

Mümkünse döngüsel bağımlılıklardan kaçınılmalıdır, ancak her zaman bu durumu önleyemeyebilirsiniz. Bu tür durumlarda, Nest, döngüsel bağımlılıkları çözmek için iki yöntem sunar. Bu bölümde, **ileri referans** kullanma tekniğini bir yöntem olarak açıklıyoruz ve başka bir yöntem olarak DI konteynerinden bir sağlayıcı örneği almak için **ModuleRef** sınıfını kullanmayı anlatıyoruz.

> uyarı **Uyarı** Döngüsel bir bağımlılık, ithalatları gruplamak için "barrel files" / index.ts dosyalarını kullanırken de oluşabilir. Modül/sağlayıcı sınıfları ile ilgili dosyalar arasında, yani `cats/cats.controller` dosyasının `cats/cats.service` dosyasını içe aktarmak için `cats` kullanmamalısınız. Daha fazla bilgi için [bu GitHub konusuna](https://github.com/nestjs/nest/issues/1181#issuecomment-430197191) bakın.

#### İleri referans

**İleri referans**, Nest'in henüz tanımlanmamış sınıflara referans yapmasına olanak tanır ve `forwardRef()` yardımcı işlevini kullanır. Örneğin, `CatsService` ve `CommonService` birbirine bağlıysa, ilişkinin her iki tarafı da döngüsel bağımlılığı çözmek için `@Inject()` ve `forwardRef()` yardımcı işlevini kullanabilir. Aksi takdirde Nest, gerekli tüm metadata mevcut olmadığı için bunları örneklemeyecektir. İşte bir örnek:

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    @Inject(forwardRef(() => CommonService))
    private commonService: CommonService,
  ) {}
}
```

> info **İpucu** `forwardRef()` işlevi, `@nestjs/common` paketinden içe aktarılır.

Bu, ilişkinin bir tarafını kapsar. Şimdi `CommonService` ile aynısını yapalım:

```typescript
@@filename(common.service)
@Injectable()
export class CommonService {
  constructor(
    @Inject(forwardRef(() => CatsService))
    private catsService: CatsService,
  ) {}
}
```

> uyarı **Uyarı** Oluşturma sırası belirsizdir. Kodunuzun hangi kurucunun önce çağrıldığına bağlı olmadığından emin olun. Döngüsel bağımlılıkların `Scope.REQUEST` ile sağlayıcılara bağlı olması, tanımsız bağımlılıklara yol açabilir. Daha fazla bilgi [burada](https://github.com/nestjs/nest/issues/5778) bulunabilir.

#### ModuleRef sınıfı alternatifi

`forwardRef()` kullanmanın alternatiflerinden biri, kodunuzu yeniden düzenlemek ve (aksi takdirde) döngüsel ilişkinin bir tarafında bir sağlayıcıyı almak için `ModuleRef` sınıfını kullanmaktır. `ModuleRef` yardımcı sınıfı hakkında daha fazla bilgiyi [buradan](/docs/fundamentals/module-reference) edinebilirsiniz.

#### Modül ileri referans

Modüller arasında döngüsel bağımlılıkları çözmek için, modüller birlikteliğinin her iki tarafında da `forwardRef()` yardımcı işlevini kullanın. Örneğin:

```typescript
@@filename(common.module)
@Module({
  imports: [forwardRef(() => CatsModule)],
})
export class CommonModule {}
```

Bu, ilişkinin bir tarafını kapsar. Şimdi aynısını `CatsModule` ile yapalım:

```typescript
@@filename(cats.module)
@Module({
  imports: [forwardRef(() => CommonModule)],
})
export class CatsModule {}
```