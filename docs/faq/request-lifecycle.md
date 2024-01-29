### Request lifecycle

Nest uygulamaları, **istek yaşam döngüsü** olarak adlandırdığımız bir sıra içinde istekleri işler ve yanıtlar üretir. Middleware, borular, korumalar ve interceptor'ların kullanımıyla, özellikle global, denetleyici düzeyinde ve yol düzeyinde bileşenler devreye girdiğinde, istek yaşam döngüsü sırasında belirli bir kod parçasının nerede çalıştığını takip etmek zor olabilir. Genel olarak, bir istek middleware'den korumalara, ardından interceptor'lara, ardından borulara ve nihayet yanıt oluşturulurken tekrar interceptor'lara akar.

#### Middleware

Middleware, belirli bir sıra içinde yürütülür. İlk olarak, Nest, global olarak bağlanmış middleware'leri çalıştırır (örneğin, `app.use` ile bağlanmış middleware'leri) ve ardından [modülle bağlantılı middleware'leri](/docs/middleware) çalıştırır, bu da yollara bağlı olarak belirlenir. Middleware'ler, Express'teki gibi sırayla çalıştırılır. Farklı modüller arasında bağlanmış middleware'lerin durumunda, kök modüle bağlanan middleware önce çalışır, ardından modüllerin içe aktarıldığı sırayla çalışır.

#### Korumalar

Koruma yürütme işlemi global korumalarla başlar, ardından denetleyici korumalara ve nihayet yol korumalarına geçer. Middleware ile benzer şekilde, korumalar bağlandıkları sırayla çalışır. Örneğin:

```typescript
@UseGuards(Guard1, Guard2)
@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @UseGuards(Guard3)
  @Get()
  getCats(): Cats[] {
    return this.catsService.getCats();
  }
}
```

`Guard1`, `Guard2`'den önce, her ikisi de `Guard3`'den önce çalışacaktır.

> info **İpucu** Genel olarak bağlı olup olmadığımızı söylerken, fark, korumanın (veya başka bir bileşenin) nerede bağlandığıdır. Eğer `app.useGlobalGuard()` kullanıyorsanız veya bileşeni bir modül aracılığıyla sağlıyorsanız, genel olarak bağlıdır. Aksi takdirde, dekoratör bir denetleyici sınıfını takip ediyorsa bir denetleyiciye, bir yol tanımlamasını takip ediyorsa bir yola bağlıdır.

#### Interceptor'lar

Interceptor'lar, genellikle korumalarla aynı modeli takip eder, tek fark şudur: çünkü interceptor'lar [RxJS Observables](https://github.com/ReactiveX/rxjs) döndürür, observables, bir sırayla sona doğru çözülecektir. Bu nedenle gelen istekler, standart global, denetleyici, yol düzeyi çözümünü geçer, ancak isteği karşılayan denetleyici yöntem işleyicisinden döndükten sonra (örneğin, yanıttan sonra) çözülecek ve yol, denetleyiciye, global'e doğru çözülecektir. Ayrıca, borulardan, denetleyicilerden veya hizmetlerden fırlatılan hataların `catchError` operatörü içinde okunabileceği bir interceptor'da okunabilir.

#### Borular

Borular, genel olarak globalden denetleyiciye, ardından yola bağlı sırayla takip edilen bir sıra içinde çalışır, `@UsePipes()` parametreleri açısından ilk gelen ilk çıkar prensibine göre çalışır. Ancak, bir yol parametre düzeyinde birden fazla borunun çalıştığı bir durumda, bunlar son parametreden ilk parametreye doğru çalışır. Bu, yol düzeyi ve denetleyici düzeyi boruları için de geçerlidir. Örneğin, aşağıdaki denetleyiciye sahipsek:

```typescript
@UsePipes(GeneralValidationPipe)
@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @UsePipes(RouteSpecificPipe)
  @Patch(':id')
  updateCat(
    @Body() body: UpdateCatDTO,
    @Param() params: UpdateCatParams,
    @Query() query: UpdateCatQuery,
  ) {
    return this.catsService.updateCat(body, params, query);
  }
}
```

o zaman `GeneralValidationPipe`, `query` için çalışacak, ardından `params` ve ardından `body` nesneleri geçmeden önce `RouteSpecificPipe`'e geçecektir, bu da aynı sırayı takip eder. Herhangi bir parametre özgü borular varsa, bunlar (yine, en son parametreden ilk parametreye doğru) denetleyici ve yol düzeyi borularından sonra çalışacaktır.

#### Filtreler

Filtreler, genel olarak ilk çözülmeyen noktadan başlayarak global filtrelerden başlar. Yani, işlem, önce herhangi bir yol bağlı filtre ile başlar ve ardından denetleyici düzeyine, en sonunda ise global filtrelerine geçer. Not edilmesi gereken bir konu, filtreler arasında istisnaların iletilmemesidir; eğer bir yol düzeyi filtre istisnayı yakalarsa, bir denetleyici veya global düzey filtresi aynı istisnayı yakalayamaz. Bu tür bir etki elde etmenin tek yolu, filtreler arasında kalıtımı kullanmaktır.

> info **İpucu** Filtreler, istek işleme sürecinde herhangi bir yakalanmamış istisna oluştuğunda yalnızca çalıştırılır. `try

/catch` ile yakalanmış istisnalar gibi yakalanan istisnalar, İstisna Filtrelerini tetiklemez. Bir yakalanmamış istisna bulunduğunda, yaşam döngüsünün geri kalanı göz ardı edilir ve istek doğrudan filtreye geçer.

#### Özet

Genel olarak, istek yaşam döngüsü aşağıdaki gibi görünür:

1. Gelen istek
2. Middleware
   - 2.1. Genel olarak bağlı middleware'ler
   - 2.2. Modül olarak bağlı middleware'ler
3. Korumalar
   - 3.1 Genel korumalar
   - 3.2 Denetleyici korumaları
   - 3.3 Yol korumaları
4. Interceptor'lar (denetleyici öncesi)
   - 4.1. Genel interceptor'lar
   - 4.2. Denetleyici interceptor'ları
   - 4.3. Yol interceptor'ları
5. Borular
   - 5.1 Genel borular
   - 5.2 Denetleyici boruları
   - 5.3 Yol boruları
   - 5.4 Yol parametre boruları
6. Denetleyici (method işleyici)
7. Servis (varsa)
8. Interceptor'lar (sonraki istek)
   - 8.1 Yol interceptor'ları
   - 8.2 Denetleyici interceptor'ları
   - 8.3 Genel interceptor'lar
9. İstisna filtreleri
   - 9.1 Yol
   - 9.2 Denetleyici
   - 9.3 Genel
10. Sunucu yanıtı