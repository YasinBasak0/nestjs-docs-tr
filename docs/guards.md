---
sidebar_position: 8
---

### Guardlar

Bir guard, `@Injectable()` dekoratörü ile işaretlenmiş ve `CanActivate` arabirimini uygulayan bir sınıftır.

<figure><img src="/assets/Guards_1.png" /></figure>

Guard'lar **tek bir sorumluluğa** sahiptir. Belirli koşullara (izinler, roller, ACL'ler vb.) bağlı olarak, belirli bir isteğin yönlendirme işleyicisi tarafından işlenip işlenmeyeceğini belirler. Bu genellikle **yetkilendirme** olarak adlandırılır. Yetkilendirme (genellikle işbirliği yaptığı **kimlik doğrulama** ile birlikte) genellikle geleneksel Express uygulamalarında [middleware](/docs/middleware) ile ele alınmıştır. Kimlik doğrulama gibi şeyler, token doğrulama ve `request` nesnesine özellikler eklemek gibi şeyler, belirli bir yönlendirme bağlamı (ve metadata) ile güçlü bir şekilde bağlı değildir. 

Ancak, doğası gereği middleware "aptal" dır. `next()` fonksiyonunu çağırdıktan sonra hangi işleyicinin yürütüleceğini bilmez. Diğer yandan, **Guard'lar**, `ExecutionContext` örneğine erişime sahiptir ve bu nedenle tam olarak neyin yürütüleceğini bilir. Guard'lar, tam olarak doğru noktada işleme mantık eklemenize ve bunu deklaratif bir şekilde yapmanıza izin vermek üzere tasarlanmıştır, aynı şekilde istisna filtreleri, borular ve interceptorlar gibi. Bu, kodunuzu DRY (Don't Repeat Yourself) ve deklaratif tutmanıza yardımcı olur.

> info **İpucu** Guard'lar, tüm middleware'lerden **sonra**, ancak herhangi bir interceptor veya pipe'den **önce** çalıştırılır.

#### Yetkilendirme Guard'ı

Bahsedildiği gibi, **yetkilendirme**, belirli rotaların, çağrıyı yapanın (genellikle belirli bir kimliği doğrulanmış kullanıcı) yeterli izinlere sahip olduğunda kullanılacak harika bir durumdur. Şu anda oluşturacağımız `AuthGuard`, kimliği doğrulanmış bir kullanıcıyı varsaymaktadır (ve bu nedenle bir token'ın istek başlıklarına ekli olduğu varsayılmaktadır). Bu, isteğin devam edip edemeyeceğini belirlemek için çıkarılacak ve doğrulanacak olan token'ı kullanır.

```typescript
@@filename(auth.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class AuthGuard {
  async canActivate(context) {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

> info **İpucu** Uygulamanızda bir kimlik doğrulama mekanizması nasıl uygulanacağına dair gerçek bir örnek arıyorsanız, [bu bölüme](/docs/security/authentication) göz atın. Benzer şekilde, daha karmaşık bir yetkilendirme örneği için [bu sayfaya](/docs/security/authorization) bakın.

`validateRequest()` fonksiyonu içindeki mantık, ihtiyaca bağlı olarak basit veya karmaşık olabilir. Bu örneğin ana noktası, guard'ların istek/yanıt döngüsü içine nasıl uyduğunu göstermektir.

Her guard, `canActivate()` fonksiyonunu uygulamalıdır. Bu fonksiyon, mevcut isteğin izin verilip verilmediğini belirten bir boolean değeri döndürmelidir. İşlemi kontrol etmek için Nest, dönüş değerini kullanır:

- Eğer `true` dönerse, istek işlenecek.
- Eğer `false` dönerse, Nest isteği reddedecektir.

#### İşlem bağlamı

`canActivate()` fonksiyonu, tek bir argüman olan `ExecutionContext` örneğini alır. `ExecutionContext`, `ArgumentsHost`'dan miras alır. `ArgumentsHost`'u daha önce istisna filtreleri bölümünde görmüştük. Yukarıdaki örnekte, yine önce kullandığımız `ArgumentsHost` üzerinde tanımlanan aynı yardımcı yöntemleri kullanmak için yapılmıştır, böylece `Request` nesnesine bir referans elde ederiz. Bu konuyla ilgili daha fazla bilgi için [İşlem bağlamı](/docs/fundamentals/execution-context) bölümüne bakabilirsiniz.

`ArgumentsHost`'u genişleterek, `ExecutionContext` ayrıca mevcut yürütme süreci hakkında ek detaylar sağlayan birkaç yeni yardımcı yöntem ekler. Bu detaylar, daha geniş bir set içinde çalışabilen daha genel geçerli guard'lar oluşturmak için kullanışlı olabilir. `ExecutionContext` hakkında daha fazla bilgi için [buraya](/docs/fundamentals/execution-context) bakın.

#### Rol tabanlı kimlik doğrulama

Sadece belirli bir role sahip kullanıcılara erişim izni veren daha işlevsel bir guard oluşturalım. Temel bir guard şablonu ile başlayacağız ve gelecek bölümlerde onu geliştireceğiz. Şu an için tüm isteklere izin verir:

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ):

 boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class RolesGuard {
  canActivate(context) {
    return true;
  }
}
```

#### Guardları Bağlama

Borular ve istisna filtreleri gibi, guard'lar da **controller kapsamlı** (controller-scoped), method kapsamlı veya global kapsamlı olabilir. Aşağıda, `@UseGuards()` dekoratörünü kullanarak bir controller kapsamlı guard kuruyoruz. Bu dekoratör, tek bir argüman veya virgülle ayrılmış bir argüman listesi alabilir. Bu, tek bir bildirimle uygun guard setini kolayca uygulamanıza olanak tanır.

```typescript
@@filename()
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

> info **İpucu** `@UseGuards()` dekoratörü, `@nestjs/common` paketinden içe aktarılır.

Yukarıda, `RolesGuard` sınıfını (bir örnek yerine) geçtik, instantiation (örnekleme) sorumluluğunu çerçeveye bıraktık ve bağımlılık enjeksiyonunu etkinleştirdik. Borular ve istisna filtreleri gibi, yerinde bir örnek de geçebiliriz:

```typescript
@@filename()
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

Yukarıdaki yapı, bu controller tarafından bildirilen her işleyiciye guard'ı ekler. Eğer guard'ın yalnızca belirli bir metoda uygulanmasını istiyorsak, `@UseGuards()` dekoratörünü **metod düzeyinde** kullanırız.

Global bir guard kurmak için, Nest uygulama örneğinin `useGlobalGuards()` metodunu kullanın:

```typescript
@@filename()
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

> warning **Dikkat** Hibrit uygulamalar için `useGlobalGuards()` metodu varsayılan olarak gateway ve mikro servisleri için guard'lar kurmaz (bu davranışı nasıl değiştireceğinizle ilgili bilgi için [Hibrit Uygulama](/docs/faq/hybrid-application) bölümüne bakın). "Standart" (hibrit olmayan) mikroservis uygulamaları için, `useGlobalGuards()` guard'ları global olarak bağlar.

Global guard'lar, tüm uygulama genelinde, her controller ve her route handler için kullanılır. Bağımlılık enjeksiyonu açısından, (yukarıdaki örnekte olduğu gibi) herhangi bir modül dışında kaydedilen global guard'lar, herhangi bir modül bağlamı içinde yapılmadığı için bağımlılıkları enjekte edemez. Bu sorunu çözmek için guard'ı doğrudan herhangi bir modülden şu şekilde kurabilirsiniz:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

> info **İpucu** Guard için bağımlılık enjeksiyonu yapmak için bu yaklaşımı kullanırken, bu yapının uygulandığı modülden bağımsız olarak guard'ın, aslında, global olduğunu unutmayın. Bu nerede yapılmalıdır? Guard'ın (`RolesGuard` yukarıdaki örnekte olduğu gibi) tanımlandığı modülü seçin. Ayrıca, `useClass`, özel sağlayıcı kaydıyla başa çıkmanın tek yolu değildir. [Buradan](/docs/fundamentals/custom-providers) daha fazla bilgi edinin.

#### Handler'a Özel Roller Belirleme

`RolesGuard` şu anda çalışıyor, ancak henüz çok akıllı değil. En önemli guard özelliğinden - [execution context](/docs/fundamentals/execution-context)den - henüz yararlanmıyoruz. Henüz rollerden veya her işleyici için hangi rollerin izin verildiğinden haberdar değil. Örneğin, `CatsController`'ın farklı rotalar için farklı izin şemaları olabilir. Bazıları sadece bir yönetici kullanıcısı için kullanılabilirken, diğerleri herkese açık olabilir. Rollerin rotalara nasıl eşleştirileceğini esnek ve yeniden kullanılabilir bir şekilde nasıl yapabiliriz?

İşte burada **özel metadata** devreye giriyor (daha fazla bilgi için [buraya](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata) bakın). Nest, rotaların işleyicilerine özel **metadata** eklemek için ya `Reflector#createDecorator` static metodunu kullanarak oluşturulan dekoratörler veya dahili `@SetMetadata()` dekoratörünü sağlar.

Örneğin, `Reflector#createDecorator` metodunu kullanarak `@Roles()` dekoratörü oluşturarak metadata'yı işleyiciye ekleyen bir dekoratör oluşturalım. `Reflector`, çerçeve tarafından kutudan sağlanan ve `@nestjs/core` paketinden açığa çıkan bir sınıftır.

```ts
@@filename(roles.decorator)
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

Buradaki `Roles` dekoratörü, `string[]` türünde tek bir argüman alan bir fonksiyondur.

Şimdi, bu dekoratörü kullanmak için, sadece işleyiciye anotasyon ekleriz:

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

Burada, `create()` metoduna `Roles` dekoratör metadata'sını ekledik ve bu rotaya sadece `admin` rolüne sahip kullanıcıların erişmesine izin verilmesi gerektiğini belirttik.

Bunun yerine, `Reflector#createDecorator` metodunu kullanmak yerine, dahili `@SetMetadata()` dekoratörünü kullanabilirdik. Daha fazla bilgi için [buraya](/docs/fundamentals/execution-context#low-level-approach) bakın.

#### Tümünü Birleştirme

Şimdi bunu `RolesGuard` ile birleştirelim. Şu anda, tüm durumlarda sadece `true` döndürerek her isteğin devam etmesine izin veriyor. Geri dönen değeri, **mevcut kullanıcıya atanmış olan rolleri** ile işlenmekte olan mevcut rotanın gerektirdiği rolleri karşılaştırarak koşula bağlamak istiyoruz. Rota tarafından gereken rol(ler)e (özel metadata) erişmek için `Reflector` yardımcı sınıfını tekrar kullanacağız, şöyle:

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
@Dependencies(Reflector)
export class RolesGuard {
  constructor(reflector) {
    this.reflector = reflector;
  }

  canActivate(context) {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

> info **Hint** Node.js dünyasında, yetkilendirilmiş kullanıcıyı `request` nesnesine eklemek yaygın bir uygulamadır. Bu nedenle, yukarıdaki örnek kodumuzda, `request.user`'ın kullanıcı örneğini ve izin verilen rolleri içerdiğini varsayıyoruz. Uygulamanızda, bu ilişkiyi kendi özel **authentication guard**'ınızda (veya middleware) yapacaksınız. Bu konuda daha fazla bilgi için [bu bölüme](/docs/security/authentication) bakın.

> warning **Warning** `matchRoles()` fonksiyonu içindeki mantık, ihtiyaca uygun olarak basit veya karmaşık olabilir. Bu örneğin temel amacı, guard'ların istek/yanıt döngüsüne nasıl uyduğunu göstermektir.

`RolesGuard` tarafından yetkisi olmayan bir kullanıcı bir uç noktayı istediğinde, Nest otomatik olarak aşağıdaki yanıtı döndürür:

```typescript
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

Unutmayın ki bir guard `false` döndüğünde, çerçeve bir `ForbiddenException` fırlatır. Farklı bir hata yanıtı döndürmek istiyorsanız, kendi belirli istisnayı fırlatmalısınız. Örneğin:

```typescript
throw new UnauthorizedException();
```

Guard tarafından fırlatılan herhangi bir istisna, [exceptions layer](/docs/exception-filters) (global exceptions filter ve geçerli bağlam için uygulanan tüm exceptions filter'lar) tarafından işlenir.

> info **Hint** Eğer yetkilendirme nasıl uygulanacağı konusunda gerçek bir örnek arıyorsanız, [bu bölüme](/docs/security/authorization) bakın.