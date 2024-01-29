---
sidebar_position: 3
---

### Kontrolcüler

Kontrolcüler, gelen **isteği (request)** işleme ve istemciye **yanıt (response)** döndürme sorumluluğundadır.

<figure><img src="/assets/Controllers_1.png" /></figure>

Bir kontrolcünün amacı, uygulama için belirli istekleri alabilmektir. **Yönlendirme** mekanizması, hangi kontrolcünün hangi istekleri aldığını kontrol eder. Genellikle her kontrolcünün birden fazla rotası vardır ve farklı rotalar farklı işlemleri gerçekleştirebilir.

Temel bir kontrolcü oluşturmak için sınıflar ve **dekoratörler** kullanırız. Dekoratörler, sınıfları gerekli meta verilerle ilişkilendirir ve Nest'in bir yönlendirme haritası oluşturmasına (isteği ilgili kontrolcilere bağlama) olanak tanır.

> info **İpucu** Hızlı bir [doğrulama](https://docs.nestjs.com/techniques/validation) içeren CRUD kontrolcüsü oluşturmak için CLI'ın [CRUD üreteci](https://docs.nestjs.com/recipes/crud-generator#crud-generator)'sini kullanabilirsiniz: `nest g resource [isim]`.

#### Yönlendirme

Aşağıdaki örnekte, temel bir kontrolcüyü tanımlamak için **gereken** `@Controller()` dekoratörünü kullanacağız. İsteğe bağlı olarak bir yol önekini `cats` olarak belirtiyoruz. Bir `@Controller()` dekoratöründe yol öneki kullanmak, ilgili rotaları kolayca gruplandırmamıza ve tekrarlayan kodu en aza indirmemize olanak tanır. Örneğin, bir kedi varlığıyla etkileşimleri yöneten bir dizi rotayı `/cats` rotası altında gruplandırmayı seçebiliriz. Bu durumda `@Controller()` dekoratöründe yol öneki olarak `cats`'i belirterek, dosyadaki her rota için bu bölümü tekrar etmek zorunda kalmayız.

```typescript
@@filename(cats.controller)
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

> info **İpucu** CLI kullanarak bir kontrolcü oluşturmak için basitçe `$ nest g controller [isim]` komutunu çalıştırın.

`@Get()` HTTP istek yöntemi dekoratörü, `findAll()` methodundan önce kullanıldığında, Nest'e HTTP istekleri için belirli bir uç nokta için bir işleyici oluşturmasını söyler. Uç nokta, HTTP isteği yöntemi (bu durumda GET) ve yol özelliği ile ilgilidir. Peki rota yolu nedir? Bir işleyicinin rota yolu, kontrolcü için belirtilen (isteğe bağlı) önek ve dekoratörde belirtilen yöntemin yolunu birleştirerek belirlenir. Biz burada her rota için bir önek belirledik (`cats`) ve dekoratörde herhangi bir yol bilgisi eklememiş olduğumuzdan, Nest bu işleyiciye `GET /cats` isteklerini eşleştirir. Bahsedildiği gibi, yol, isteğe bağlı kontrolcü rota öneki **ve** isteğin yöntem dekoratöründe belirtilen herhangi bir yol dizesini içerir. Örneğin, `@Get('breed')` dekoratörü ile birleştirilmiş `cats` önekli bir yol, `GET /cats/breed` gibi istekler için bir rota eşlemesi üretir.

Yukarıdaki örneğimizde bu uç noktaya bir GET isteği yapıldığında, Nest isteği kullanıcı tanımlı `findAll()` metodumuza yönlendirir. Burada seçtiğimiz metodun adının tamamen keyfi olduğuna dikkat edin. Elbette bir rota bağlamak için bir yöntem belirtmemiz gerekiyor, ancak Nest seçilen metodun adına herhangi bir anlam katmaz.

Bu metod, bu durumda yalnızca bir dize olan ilişkili yanıt ile birlikte 200 durum kodunu döndürecektir. Bu neden olur? Açıklamak için, Nest'in yanıtları işleme koymak için iki **farklı** seçenek kullandığını belirtelim:

<table>
  <tr>
    <td>Standart (tavsiye edilen)</td>
    <td>
      Bu yerleşik yöntemi kullanarak, bir istek işleyicisi bir JavaScript nesnesi veya dizisi döndürdüğünde, bu otomatik olarak JSON'a seri hale getirilecektir.
      Ancak JavaScript ilkel türünü (örneğin, <code>string</code>, <code>number</code>, <code>boolean</code>) döndürdüğünde, Nest, değeri seri hale getirmeye çalışmadan sadece değeri gönderecektir. Bu, yanıt işlemini basitleştirir: değeri sadece döndürün ve Nest gerisini halleder.
      <br />
      <br /> Ayrıca, yanıtın <strong>durum kodu</strong> varsayılan olarak her zaman 200'dür, yalnızca POST
      istekleri için 201 kullanılır. Bu davranışı <code>@HttpCode(...)</code>
      dekoratörünü bir işleyici düzeyinde ekleyerek kolayca değiştirebiliriz (bkz. <a href='controllers#status-code'>Durum Kodları</a>).
    </td>
  </tr>
 

 <tr>
    <td>Kütüphane özel</td>
    <td>
      Kütüphane özel (örneğin, Express) <a href="https://expressjs.com/en/api.html#res" rel="nofollow" target="_blank">yanıt nesnesi</a>ni kullanabiliriz, bu da yöntem işleyicisi imzasında (<code>findAll(@Res() response)</code> gibi) <code>@Res()</code> dekoratörünü kullanarak enjekte edilebilir. Bu yaklaşımla, bu nesne tarafından sağlanan özgün yanıt işleme yöntemlerini kullanma yeteneğine sahip olursunuz. Örneğin, Express ile, <code>response.status(200).send()</code> gibi kod kullanarak yanıtlar oluşturabilirsiniz.
    </td>
  </tr>
</table>

> warning **Uyarı** Nest, işleyicinin `@Res()` veya `@Next()` kullanıp kullanmadığını algılar; bu, kütüphane özel seçeneği seçtiğinizi gösterir. Her iki yaklaşımın aynı anda kullanılması durumunda, Standart yaklaşım **otomatik olarak devre dışı bırakılır** ve bu tek bir rota için beklenenden farklı çalışır. Her iki yaklaşımı aynı anda kullanmak için (örneğin, yanıt nesnesini yalnızca çerezleri/başlıkları ayarlamak için enjekte etmek ancak geri kalanını çerçeveye bırakmak), `@Res({{ '{' }} passthrough: true {{ '}' }})` dekoratöründe `passthrough` seçeneğini `true` olarak ayarlamanız gerekir.

<app-banner-devtools></app-banner-devtools>

#### İstek Nesnesi

İşleyiciler genellikle istemci **isteği (request)** detaylarına erişim sağlamak ister. Nest, temel platformun ([varsayılan olarak Express](https://expressjs.com/en/api.html#req)) istek nesnesine erişim sağlar. Nest'e, bu nesneyi işleyicinin imzasına `@Req()` dekoratörünü ekleyerek enjekte etmesini söyleyerek istek nesnesine erişebiliriz.

```typescript
@@filename(cats.controller)
import { Controller, Get, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Bind, Get, Req } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  @Bind(Req())
  findAll(request) {
    return 'This action returns all cats';
  }
}
```

> info **İpucu** `express` yazım avantajlarından yararlanmak için (`request: Request` parametre örneğinde olduğu gibi), `@types/express` paketini kurun.

İstek nesnesi, HTTP isteğini temsil eder ve istek sorgu dizisi, parametreleri, HTTP başlıkları ve gövdesi için özelliklere sahiptir (daha fazlasını [buradan](https://expressjs.com/en/api.html#req) okuyun). Çoğu durumda, bu özelliklere manuel olarak ulaşmak gerekli değildir. Bunun yerine, `@Body()` veya `@Query()` gibi özel dekoratörleri kullanabiliriz, bunlar kutudan çıkar.

Aşağıda sağlanan dekoratörlerin ve temel platforma özgü nesnelerin düz listesi bulunmaktadır.

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td>
    </tr>
    <tr>
      <td><code>@Response(), @Res()</code><span class="table-code-asterisk">*</span></td>
      <td><code>res</code></td>
    </tr>
    <tr>
      <td><code>@Next()</code></td>
      <td><code>next</code></td>
    </tr>
    <tr>
      <td><code>@Session()</code></td>
      <td><code>req.session</code></td>
    </tr>
    <tr>
      <td><code>@Param(key?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[key]</code></td>
    </tr>
    <tr>
      <td><code>@Body(key?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[key]</code></td>
    </tr>
    <tr>
      <td><code>@Query(key?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[key]</code></td>
    </tr>
    <tr>
      <td><code>@Headers(name?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[name]</code></td>
    </tr>
    <tr>
      <td><code>@Ip()</code></td>
      <td><code>req.ip</code></td>
    </tr>
    <tr>
      <td><code>@HostParam()</code></td>
      <td><code>req.hosts</code></td>
    </tr>
  </tbody>
</table>

<sup>\* </sup>Altta yatan HTTP platformları boyunca yazım uyumluluğu için (örneğin, Express ve Fastify), Nest `@Res()` ve `@Response()` dekoratörlerini sağlar. `@Res()`, basitçe `@Response()` için bir takma adtır. Her ikisi de doğrudan altta yatan native platform `response` nesnesi arayüzünü açığa çıkarır. Bunları kullanırken, tamamen yararlanmak için altındaki kütüphane yazımını da içe aktarmalısınız (örneğin, `@types/express`). Bir yöntem işleyicisine `@Res()` veya `@Response()` enjekte ettiğinizde, Nest'i **Kütüphane özel mod** a sokarsınız ve yanıtı yönetme sorumluluğu size geçer. Bunu yaparken, `response` nesnesi üzerinde bir çağrı yapmalısınız (örneğin, `res.json(...)` veya `res.send(...)`), aksi takdirde HTTP sunucusu askıya alınır.

> info **İpucu** Kendi özel dekoratörlerinizi nasıl oluşturacağınızı öğrenmek için [bu](/docs/custom-decorators) bölümü ziyaret edin.

#### Kaynaklar

Daha önce, kediler kaynağını almak için bir uç nokta tanımlamıştık (**GET** rotası). Genellikle yeni kayıtlar oluşturan bir uç nokta sağlamak isteyeceğiz. Bunun için **POST** işleyicisini oluşturalım:

```typescript
@@filename(cats.controller)
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(): string {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create() {
    return 'This action adds a new cat';
  }

  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

Bu kadar basit. Nest, tüm standart HTTP metodları için dekoratörler sağlar: `@Get()`, `@Post()`, `@Put()`, `@Delete()`, `@Patch()`, `@Options()`, ve `@Head()`. Ayrıca, `@All()`, hepsini işleyen bir uç noktayı tanımlar.

#### Rota jokerleri

Desen tabanlı rotalar da desteklenir. Örneğin, yıldız (*) joker olarak kullanılır ve herhangi bir karakter kombinasyonunu eşleştirir.

```typescript
@Get('ab*cd')
findAll() {
  return 'This route uses a wildcard';
}
```

`'ab*cd'` rota yolu, `abcd`, `ab_cd`, `abecd` ve benzeri kombinasyonları eşleştirir. `?`, `+`, `*` ve `()` karakterleri bir rota yolunda kullanılabilir ve bunlar düzenli ifade karşılıklarının alt kümeleridir. Kısa çizgi (`-`) ve nokta (`.`), string tabanlı yollar tarafından kelime anlamında yorumlanır.

> warning **Uyarı** Rota ortasında joker, sadece express tarafından desteklenir.

#### Durum kodu

Yukarıda belirtildiği gibi, yanıt **durum kodu** her zaman varsayılan olarak **200**'dür, sadece POST istekleri için **201**'dir. Bu davranışı `@HttpCode(...)` dekoratörünü işleyici düzeyinde ekleyerek kolayca değiştirebiliriz.

```typescript
@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

> info **İpucu** `HttpCode`'u `@nestjs/common` paketinden içe aktarın.

Genellikle durum kodunuz statik değil, çeşitli faktörlere bağlıdır. Bu durumda, bir kütüphane özel **yanıt** nesnesi (`@Res()` ile enjekte edilir) (veya bir hata durumunda istisna fırlatma) kullanabilirsiniz.

#### Başlıklar

Özel bir yanıt başlığı belirtmek için ya `@Header()` dekoratörünü veya kütüphane özel bir yanıt nesnesini (ve doğrudan `res.header()` çağırmayı) kullanabilirsiniz.

```typescript
@Post()
@Header('Cache-Control', 'none')
create() {
  return 'This action adds a new cat';
}
```

> info **İpucu** `Header`'ı `@nestjs/common` paketinden içe aktarın.

#### Yönlendirme

Bir yanıtı belirli bir URL'ye yönlendirmek için ya `@Redirect()` dekoratörünü veya kütüphane özel bir yanıt nesnesini (ve doğrudan `res.redirect()` çağırmayı) kullanabilirsiniz.

`@Redirect()`, iki argüman alır, `url` ve `statusCode`, her ikisi de isteğe bağlıdır. `statusCode`'un varsayılan değeri, belirtilmemişse `302` (`Found`) dir.

```typescript
@Get()
@Redirect('https://nestjs.com', 301)
```

> info **İpucu** Bazen HTTP durum kodunu veya yönlendirme URL'sini dinamik olarak belirlemek isteyebilirsiniz. Bunun için `@nestjs/common`'dan gelen `HttpRedirectResponse` arayüzünü takip eden bir nesne döndürün.

Döndürülen değerler, `@Redirect()` dekoratörüne geçirilen argümanları geçersiz kılacaktır. Örneğin:

```typescript
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

#### Rota parametreleri

Statik yollara sahip rotalar, isteğin bir parçası olarak **dinamik veri** kabul etmeniz gerektiğinde çalışmaz (örneğin, `GET /cats/1` ile id'si `1` olan kedi bilgisini almak). Parametreli rotaları tanımlamak için, isteğin URL'sindeki bu konumda dinamik değeri yakalamak için rota parametre **token'ları** ekleyebiliriz. Aşağıdaki `@Get()` dekoratörü örneğinde rota parametresi token'ının kullanımını göstermektedir. Bu şekilde tanımlanan rota parametreleri, metot imzasına `@Param()` dekoratörünün eklenmesiyle erişilebilir.

> info **İpucu** Parametreli rotalar, herhangi bir statik yolun sonrasında belirtilmelidir. Bu, parametreli yolların statik yollar için hedeflenen trafiği engellemesini önler.

```typescript
@@filename()
@Get(':id')
findOne(@Param() params: any): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
@@switch
@Get(':id')
@Bind(Param())
findOne(params) {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

`@Param()`, bir metot parametresini (`params` yukarıdaki örnekte olduğu gibi) dekore etmek için kullanılır ve bu dekore edilmiş metot parametresinin içinde metotun gövdesi içinde **rota** parametrelerini kullanılabilir hale getirir. Yukarıdaki kod örneğinde görüldüğü gibi, `id` parametresine `params.id` referansını kullanarak erişebiliriz. Ayrıca, dekore ediciye belirli bir parametre token'ı iletebilir ve sonra metot gövdesinde doğrudan adıyla rota parametresine referans verebilirsiniz.

> info **İpucu** `Param`'ı `@nestjs/common` paketinden içe aktarın.

```typescript
@@filename()
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
@@switch
@Get(':id')
@Bind(Param('id'))
findOne(id) {
  return `This action returns a #${id} cat`;
}
```

#### Alt Alan Rotalama

`@Controller` dekoratörü, gelen isteklerin HTTP ana bilgisinin belirli bir değerle eşleşmesini gerektiren bir `host` seçeneğini alabilir.

```typescript
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin sayfası';
  }
}
```

> **Uyarı** **Fastify**, iç içe geçmiş yönlendiricilere destek eksikliği nedeniyle, alt alan rota kullanılırken varsayılan Express adaptörü yerine kullanılmalıdır.

Bir rota `path` gibi, `hosts` seçeneği konumundaki dinamik değeri ana bilgisinde yakalamak için token'ları kullanabilir. Aşağıdaki `@Controller()` dekoratörü örneğindeki ana bilgisi parametre token'ı, bu kullanımı gösterir. Bu şekilde tanımlanan ana bilgisi parametreleri, metot imzasına eklenen `@HostParam()` dekoratörü ile erişilebilir.

```typescript
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account: string) {
    return account;
  }
}
```

#### Kapsamlar

Farklı programlama dilinden gelen kişiler için Nest'te neredeyse her şeyin gelen istekler arasında paylaşıldığını öğrenmek beklenmedik olabilir. Veritabanı için bir bağlantı havuzumuz, global durumu olan singleton servislerimiz vb. var. Unutmayın ki Node.js, her isteğin ayrı bir iş parçacığı tarafından işlenildiği istek/yanıt Çoklu İş Parçacıklı Durumsuz Modelini takip etmez. Bu nedenle, uygulamalarımız için singleton örneklerini kullanmak tamamen **güvenlidir**.

Ancak, denetleyicinin istek tabanlı ömrü istenen davranış olabilir, örneğin GraphQL uygulamalarında istek başına önbelleğe alma, istek izleme veya çok kiracılılık. Kapsamları nasıl kontrol edeceğinizi öğrenmek için [buraya](/docs/fundamentals/injection-scopes) göz atın.

#### Asenkronluk

Modern JavaScript'i seviyoruz ve veri çıkarma işleminin çoğunlukla **asenkron** olduğunu biliyoruz. Bu nedenle, Nest `async` işlevleriyle iyi çalışır ve destekler.

> info **İpucu** `async / await` özelliği hakkında daha fazla bilgi edinin [buradan](https://kamilmysliwiec.com/typescript-2-1-introduction-async-await).

Her asenkron işlevin bir `Promise` döndürmesi gerekir. Bu, Nest'in kendi çözebileceği ertelemeli bir değeri döndürebileceğiniz anlamına gelir. Bunun bir örneğini görelim:

```typescript
@@filename(cats.controller)
@Get()
async findAll(): Promise<any[]> {
  return [];
}
@@switch
@Get()
async findAll() {
  return [];
}
```

Yukarıdaki kod tamamen geçerlidir. Dahası, Nest rota yönlendiricileri, RxJS [observable streams](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html) döndürme yeteneği ile daha da güçlüdür. Nest, kaynağın altındaki akışa otomatik olarak abone olacak ve akış tamamlandığında son emit edilen değeri alacaktır.

```typescript
@@filename(cats.controller)
@Get()
findAll(): Observable<any[]> {
  return of([]);
}
@@switch
@Get()
findAll() {
  return of([]);
}
```

Yukarıdaki iki yaklaşım da çalışır ve gereksinimlerinize uygun olanı kullanabilirsiniz.

#### İstek Verileri

Önceki POST rota işleyicisi örneğimiz, istemci parametrelerini kabul etmiyordu. Burada bunu düzeltmek için `@Body()` dekoratörünü ekleyerek düzeltebiliriz.

Ancak önce (TypeScript kullanıyorsanız), **DTO** (Data Transfer Object - Veri Transfer Objesi) şemasını belirlememiz gerekiyor. Bir DTO, verilerin ağ üzerinden nasıl gönderileceğini tanımlayan bir nesnedir. DTO şemasını **TypeScript** arabirimleri kullanarak veya basit sınıflar kullanarak belirleyebiliriz. İlginç bir şekilde, burada **sınıfları** kullanmayı öneriyoruz. Neden mi? Sınıflar, JavaScript ES6 standardının bir parçasıdır ve bu nedenle derlenmiş JavaScript'te gerçek varlıklar olarak korunurlar. Öte yandan, TypeScript arabirimleri transpile sırasında kaldırıldığından, Nest çalışma zamanında onlara referans sağlayamaz. Bu, **Pipes** gibi özelliklerin çalışma zamanında değişkenin metatipine erişim sağladığında ek olanaklar sunmasından dolayı önemlidir.

Hadi `CreateCatDto` sınıfını oluşturalım:

```typescript
@@filename(create-cat.dto)
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Bu sınıf yalnızca üç temel özelliğe sahiptir. Bundan sonra, yeni oluşturulan DTO'yu `CatsController` içinde kullanabiliriz:

```typescript
@@filename(cats.controller)
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
@@switch
@Post()
@Bind(Body())
async create(createCatDto) {
  return 'This action adds a new cat';
}
```

> info **İpucu** `ValidationPipe`'imiz, yöntem işleyicisinin alması gerekmeyen özellikleri filtreleyebilir. Bu durumda, kabul edilebilir özellikleri beyaz listeleyebilir ve beyaz listede bulunmayan her özellik otomatik olarak elde edilen nesneden çıkarılır. `CreateCatDto` örneğinde beyaz listemiz `name`, `age` ve `breed` özellikleridir. Daha fazla bilgi için [buraya](https://docs.nestjs.com/techniques/validation#stripping-properties) göz atın.

#### Hataların İşlenmesi

Hataları işlemenin (örneğin, istisnalarla çalışmanın) ayrı bir bölümü bulunmaktadır. [Buradan](/docs/exception-filters) bu konu hakkında daha fazla bilgi edinebilirsiniz.

#### Tam Kaynak Örneği

Aşağıda, temel bir denetleyici oluşturmak için mevcut dekoratörlerden birkaçını kullanan bir örnek bulunmaktadır. Bu denetleyici, dahili verilere erişmek ve bunları manipüle etmek için birkaç yöntem açıklar.

```typescript
@@filename(cats.controller)
import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
@@switch
import { Controller, Get, Query, Post, Body, Put, Param, Delete, Bind } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Body())
  create(createCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  @Bind(Query())
  findAll(query) {
    console.log(query);
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  @Bind(Param('id'))
  findOne(id) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  @Bind(Param('id'), Body())
  update(id, updateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  @Bind(Param('id'))
  remove(id) {
    return `This action removes a #${id} cat`;
  }
}
```

> info **İpucu** Nest CLI, tüm bu işlemleri yapmaktan kaçınmak ve geliştirici deneyimini çok daha basit hale getirmek için **tüm başlangıç kodunu otomatik olarak oluşturan** bir üreteç (şematik) sağlar. Bu özellik hakkında daha fazla bilgi için [buraya](/docs/recipes/crud-generator) göz atın.

#### Çalışmaya Başlama

Yukarıdaki tam olarak tanımlanan denetleyiciyle bile Nest, `CatsController` sınıfının varlığını bilmediği için bu sınıfın bir örneğini oluşturmaz.

Denetleyiciler her zaman bir modüle aittir, bu nedenle `@Module()` dekoratörü içinde `controllers` dizisini dahil ederiz. Henüz root `AppModule` dışında başka modüller tanımlamadığımızdan, `CatsController`'ı tanıtmak için bunu kullanacağız:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller

';

@Module({
  controllers: [CatsController],
})
export class AppModule {}
```

`@Module()` dekoratörü ile modül sınıfına metadata ekledik ve Nest artık hangi denetleyicilerin monte edilmesi gerektiğini kolayca yansıtabiliyor.

#### Kütüphane Özel Yaklaşım

Şu ana kadar Nest'in yanıt manipülasyonunu gerçekleştirmenin standart yolunu tartıştık. Yanıtı manipüle etmenin ikinci yolu, kütüphane özel bir [yanıt nesnesi](https://expressjs.com/en/api.html#res) kullanmaktır. Belirli bir yanıt nesnesini enjekte etmek için `@Res()` dekoratörünü kullanmamız gerekiyor. Farkları göstermek için `CatsController`'ı aşağıdaki gibi yeniden yazalım:

```typescript
@@filename()
import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Res() res: Response) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  findAll(@Res() res: Response) {
     res.status(HttpStatus.OK).json([]);
  }
}
@@switch
import { Controller, Get, Post, Bind, Res, Body, HttpStatus } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Res(), Body())
  create(res, createCatDto) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  @Bind(Res())
  findAll(res) {
     res.status(HttpStatus.OK).json([]);
  }
}
```

Bu yaklaşım çalışsa da ve yanıt nesnesine tam kontrol sağlayarak (başlıkların manipülasyonu, kütüphane özel özellikler vb.) bazı yönlere daha fazla esneklik sağlasa da, dikkatli kullanılmalıdır. Genel olarak, bu yaklaşım çok daha az açık ve bazı dezavantajlara sahiptir. Ana dezavantajı, kodunuzun platforma bağlı hale gelmesidir (temel kütüphaneler yanıt nesnesinde farklı API'lere sahip olabilir) ve test etmesi daha zor olmasıdır (yanıt nesnesini mocklamak zorunda kalabilirsiniz vb.).

Ayrıca yukarıdaki örnekte, Nest standardı yanıt işleme bağlı olan Nest özelliklerini kaybedersiniz, örneğin, Interceptörler ve `@HttpCode()` / `@Header()` dekoratörleri gibi. Bunu düzeltmek için, `passthrough` seçeneğini `true` olarak ayarlayabilirsiniz:

```typescript
@@filename()
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return [];
}
@@switch
@Get()
@Bind(Res({ passthrough: true }))
findAll(res) {
  res.status(HttpStatus.OK);
  return [];
}
```

Şimdi native yanıt nesnesiyle etkileşimde bulunabilirsiniz (örneğin, belirli koşullara bağlı olarak çerez veya başlıkları ayarlayabilirsiniz), ancak geri kalanını çerçeveye bırakabilirsiniz.