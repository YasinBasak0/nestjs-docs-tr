### Yetkilendirme

**Yetkilendirme**, bir kullanıcının ne yapabilir olduğunu belirleyen süreci ifade eder. Örneğin, bir yönetici kullanıcı, gönderileri oluşturabilir, düzenleyebilir ve silebilir. Bir yönetici olmayan kullanıcı ise yalnızca gönderileri okuma iznine sahiptir.

Yetkilendirme, kimlik doğrulamadan ayrık ve bağımsızdır. Ancak, yetkilendirme bir kimlik doğrulama mekanizmasını gerektirir.

Yetkilendirme işlemi ele almak için birçok farklı yaklaşım ve strateji vardır. Herhangi bir proje için seçilen yaklaşım, özel uygulama gereksinimlerine bağlıdır. Bu bölüm, çeşitli gereksinimlere uyarlanabilen yetkilendirme için birkaç yaklaşımı sunar.

#### Temel RBAC uygulaması

Rol tabanlı erişim kontrolü (**RBAC**), roller ve ayrıcalıklar etrafında tanımlanan bir politika nötr erişim kontrol mekanizmasıdır. Bu bölümde, çok temel bir RBAC mekanizmasını Nest [koruyucular](/docs/guards) kullanarak nasıl uygulayacağımızı göstereceğiz.

İlk olarak, sistemi temsil eden rolleri içeren bir `Role` enum oluşturalım:

```typescript
@@filename(role.enum)
export enum Role {
  User = 'user',
  Admin = 'admin',
}
```

> info **İpucu** Daha karmaşık sistemlerde rolleri bir veritabanında saklayabilir veya dış kimlik doğrulama sağlayıcısından çekebilirsiniz.

Bunu hazırladıktan sonra, `@Roles()` dekoratörünü oluşturabiliriz. Bu dekoratör, belirli kaynaklara erişim için hangi rollerin gerektiğini belirtmeyi sağlar.

```typescript
@@filename(roles.decorator)
import { SetMetadata } from '@nestjs/common';
import { Role } from '../enums/role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
@@switch
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles) => SetMetadata(ROLES_KEY, roles);
```

Artık özel bir `@Roles()` dekoratörümüz olduğuna göre, bunu herhangi bir rota işleyiciye eklemek için kullanabiliriz.

```typescript
@@filename(cats.controller)
@Post()
@Roles(Role.Admin)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles(Role.Admin)
@Bind(Body())
create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

Son olarak, `RolesGuard` sınıfını oluşturuyoruz. Bu sınıf, mevcut kullanıcıya atanan rolleri, işlenen mevcut rotanın gerektirdiği gerçek rollerle karşılaştıracaktır. Rota'nın rol(ler)ine (özel meta veri) erişmek için çerçeve tarafından kutudan çıkan `Reflector` yardımcı sınıfını kullanacağız.

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
@Dependencies(Reflector)
export class RolesGuard {
  constructor(reflector) {
    this.reflector = reflector;
  }

  canActivate(context) {
    const requiredRoles = this.reflector.getAllAndOverride(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles.includes(role));
  }
}
```

> info **İpucu** `Reflector`'u bir bağlam duyarlı bir şekilde kullanmak için Execution context bölümündeki [Yansıma ve meta veri](/docs/fundamentals/execution-context#reflection-and-metadata) bölümüne başvurun.

> warning **Uyarı** Bu örnek, sadece rota işleyici düzeyinde rollerin varlığını kontrol ettiğimiz için "**temel**" olarak adlandırılm

ıştır. Gerçek dünya uygulamalarında, birkaç işlem içeren uç noktalara/işleyicilere sahip olabilirsiniz. Her biri belirli bir izin kümesini gerektiren bu durumda, izinleri belirli eylemlerle ilişkilendiren merkezi bir yer olmadığı için rolleri kontrol etmek için bir mekanizma sağlamanız gerekecektir, bu da bakım açısından biraz daha zor hale getirecektir.

Bu örnekte, `request.user`'ın kullanıcı örneği ve izinlere ( `roles` özelliği altında) izin verildiğini varsaydık. Uygulamanızda, bu ilişkiyi kendi özel **kimlik doğrulama koruyucunuzda** yapacaksınız - daha fazla ayrıntı için [kimlik doğrulama](/docs/security/authentication) bölümüne bakın.

Bu örneğin çalıştığından emin olmak için `User` sınıfınızın aşağıdaki gibi görünmesini sağlayın:

```typescript
class User {
  // ...diğer özellikler
  roles: Role[];
}
```

Son olarak, `RolesGuard`'ı, örneğin denetleyici düzeyinde veya global olarak kaydedin:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: RolesGuard,
  },
],
```

Yetkisi olmayan bir kullanıcı bir uç noktayı istediğinde, Nest otomatik olarak aşağıdaki yanıtı döndürür:

```typescript
{
  "statusCode": 403,
  "message": "Yasak kaynak",
  "error": "Yasak"
}
```

> info **İpucu** Farklı bir hata yanıtı döndürmek istiyorsanız, boolean bir değer yerine kendi belirli istisnayı fırlatmalısınız.

#### Talep Tabanlı Yetkilendirme

Bir kimlik oluşturulduğunda, bu kimliğe güvenilen bir tarafından verilen bir veya daha fazla talep atanabilir (claim). Bir talep, konunun ne yapabileceğini temsil eden bir ad-değer çiftidir, konunun ne olduğunu değil.

Nest'te Talep Tabanlı Yetkilendirme uygulamak için, yukarıda [RBAC](/docs/security/authorization#basic-rbac-implementation) bölümünde gösterdiğimiz adımları aynen takip edebilirsiniz, tek önemli fark şudur: belirli rolleri kontrol etmek yerine **izinleri** karşılaştırmalısınız. Her kullanıcının atanmış bir dizi izni olacaktır. Benzer şekilde, her bir kaynak/uç nokta, bunlara erişim için gereken izinleri tanımlar (örneğin, özel bir `@RequirePermissions()` dekoratörü aracılığıyla).

```typescript
@@filename(cats.controller)
@Post()
@RequirePermissions(Permission.CREATE_CAT)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@RequirePermissions(Permission.CREATE_CAT)
@Bind(Body())
create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **İpucu** Yukarıdaki örnekte `Permission` (RBAC bölümünde gösterdiğimiz `Role` gibi) sistemde bulunan tüm izinleri içeren bir TypeScript enum'dur.

#### CASL Entegrasyonu

[CASL](https://casl.js.org/), bir istemcinin hangi kaynaklara erişmeye izinli olduğunu sınırlayan izomorfik bir yetkilendirme kütüphanesidir. Aşamalı olarak benimseme amacıyla tasarlanmıştır ve basit bir talep tabanlı ve tam özellikli konu ve öznitelik tabanlı yetkilendirme arasında kolayca ölçeklenebilir.

Başlamadan önce, önce `@casl/ability` paketini yükleyin:

```bash
$ npm i @casl/ability
```

> info **İpucu** Bu örnekte CASL'yi seçtik, ancak tercihlerinize ve projenizin ihtiyaçlarına bağlı olarak `accesscontrol` veya `acl` gibi başka bir kütüphaneyi de kullanabilirsiniz.

Kurulum tamamlandıktan sonra, CASL'nin mekaniğini göstermek için iki varlık sınıfı tanımlayacağız: `User` ve `Article`.

```typescript
class User {
  id: number;
  isAdmin: boolean;
}
```

`User` sınıfı, benzersiz bir kullanıcı tanımlayıcı olan `id` ve bir kullanıcının yönetici ayrıcalıklarına sahip olup olmadığını belirten `isAdmin` özelliklerinden oluşur.

```typescript
class Article {
  id: number;
  isPublished: boolean;
  authorId: number;
}
```

`Article` sınıfı, sırasıyla `id`, `isPublished` ve `authorId` özelliklerine sahiptir. `id`, benzersiz bir makale tanımlayıcıdır, `isPublished` bir makalenin zaten yayınlanıp yayınlanmadığını belirtir ve `authorId`, makaleyi yazan kullanıcının bir kimliğidir.

Şimdi bu örneğimiz için gereksinimlerimizi gözden geçirelim ve rafine edelim:

- Yöneticiler, tüm varlıkları (oluştur/okuma/güncelleme/silme) yönetebilir.
- Kullanıcılara her şeyi okuma izni vardır.
- Kullanıcılar makalelerini güncelleyebilir (`article.authorId === userId`).
- Zaten yayınlanmış makaleler silinemez (`article.isPublished === true`).

Bu bilinçle, kullanıcıların varlıklar üzerinde gerçekleştirebileceği tüm eylemleri temsil eden `Action` enum'unu oluşturabiliriz:

```typescript
export enum Action {
  Manage = 'manage',
  Create = 'create',
  Read = 'read',
  Update = 'update',
  Delete = 'delete',
}
```

> warning **Dikkat** `manage`, CASL'de "herhangi" eylemi temsil eden özel bir anahtar kelimedir.

CASL kütüphanesini kapsamak için, şimdi `CaslModule` ve `CaslAbilityFactory`'yi oluşturalım.

```bash
$ nest g module casl
$ nest g class casl/casl-ability.factory
```

Bunu yerine getirildiğinde, `CaslAbilityFactory` üzerinde `createForUser()` yöntemini tanımlayabiliriz. Bu yöntem, belirli bir kullanıcı için `Ability` nesnesini oluşturacaktır:

```typescript
type Subjects = InferSubjects<typeof Article | typeof User> | 'all';

export type AppAbility = Ability<[Action, Subjects]>;

@Injectable()
export class CaslAbilityFactory {
  createForUser(user: User) {
    const { can, cannot, build } = new AbilityBuilder<
      Ability<[Action, Subjects]>
    >(Ability as AbilityClass<AppAbility>);

    if (user.isAdmin) {
      can(Action.Manage, 'all'); // her şeye yazma ve okuma erişimi
    } else {
      can(Action.Read, 'all'); // her şeye sadece okuma erişimi
    }

    can(Action.Update, Article, { authorId: user.id });
    cannot(Action.Delete, Article, { isPublished: true });

    return build({
      // Ayrıntılar için https://casl.js.org/v6/en/guide/subject-type-detection#use-classes-as-subject-types adresine bakın
      detectSubjectType: (item) =>
        item.constructor as ExtractSubjectType<Subjects>,
    });
  }
}
```

> warning **Dikkat** `all`, CASL'de "herhangi konu"yu temsil eden özel bir anahtar kelimedir.

> info **İpucu** `Ability`, `AbilityBuilder`, `AbilityClass` ve

 `ExtractSubjectType` sınıfları, `@casl/ability` paketinden ihraç edilir.

> info **İpucu** `detectSubjectType` seçeneği, CASL'nin bir nesneden nasıl konu türü elde edeceğini anlamasına olanak tanır. Daha fazla bilgi için [CASL belgelerini](https://casl.js.org/v6/en/guide/subject-type-detection#use-classes-as-subject-types) inceleyin.

Yukarıdaki örnekte, `Ability` örneğini `AbilityBuilder` sınıfını kullanarak oluşturduk. Muhtemelen tahmin ettiğiniz gibi, `can` ve `cannot` aynı argümanları kabul eder, ancak farklı anlamları vardır, `can` belirtilen konuda bir eylemi yapmaya izin verir ve `cannot` yasaklar. Her ikisi de en fazla 4 argüman alabilir. Bu işlevler hakkında daha fazla bilgi edinmek için resmi [CASL belgelerini](https://casl.js.org/v6/en/guide/intro) ziyaret edin.

Son olarak, `CaslModule` modül tanımındaki `providers` ve `exports` dizilerine `CaslAbilityFactory`'yi eklediğinizden emin olun:

```typescript
import { Module } from '@nestjs/common';
import { CaslAbilityFactory } from './casl-ability.factory';

@Module({
  providers: [CaslAbilityFactory],
  exports: [CaslAbilityFactory],
})
export class CaslModule {}
```

Bunu yerine getirildiğinde, `CaslAbilityFactory`'yi, `CaslModule`'ün ana bağlamında içe aktarılmış olsa bile, standard konstruktor enjeksiyonu kullanarak herhangi bir sınıfa enjekte edebiliriz:

```typescript
constructor(private caslAbilityFactory: CaslAbilityFactory) {}
```

Ardından, bir sınıfta şu şekilde kullanabiliriz.

```typescript
const ability = this.caslAbilityFactory.createForUser(user);
if (ability.can(Action.Read, 'all')) {
  // "user" her şeyi okuma erişimine sahip
}
```

> info **İpucu** `Ability` sınıfı hakkında daha fazla bilgi için resmi [CASL belgelerini](https://casl.js.org/v6/en/guide/intro) inceleyin.

Örneğin, yönetici olmayan bir kullanıcımız olduğunu düşünelim. Bu durumda, kullanıcının makaleleri okuma izni olmalı, ancak yeni makaleler oluşturmak veya mevcut makaleleri kaldırmak yasak olmalıdır.

```typescript
const user = new User();
user.isAdmin = false;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Read, Article); // true
ability.can(Action.Delete, Article); // false
ability.can(Action.Create, Article); // false
```

> info **İpucu** Hem `Ability` hem de `AbilityBuilder` sınıfları, `can` ve `cannot` yöntemlerini sağlasa da, farklı amaçları vardır ve biraz farklı argümanları kabul ederler.

Ayrıca, gereksinimlerimizde belirttiğimiz gibi, kullanıcının makalelerini güncelleyebilmesi gerekir:

```typescript
const user = new User();
user.id = 1;

const article = new Article();
article.authorId = user.id;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Update, article); // true

article.authorId = 2;
ability.can(Action.Update, article); // false
```

Gördüğünüz gibi, `Ability` örneği, izinleri oldukça okunabilir bir şekilde kontrol etmemize olanak tanır. Benzer şekilde, `AbilityBuilder`, izinleri (ve çeşitli koşulları belirtmeyi) benzer bir şekilde tanımlamamıza olanak tanır. Daha fazla örnek bulmak için resmi belgelere başvurun.

#### İleri Düzey: Bir `PoliciesGuard` Uygulama

Bu bölümde, belirli **yetkilendirme politikalarını** karşılayıp karşılamadığını kontrol eden biraz daha karmaşık bir guard nasıl oluşturulacağını göstereceğiz. Bu politikaları yöntem düzeyinde yapılandırabilen bir mekanizma sağlama amacımız var (bunu sınıf düzeyinde yapılandırılan politikaları da dikkate alacak şekilde genişletebilirsiniz). Bu örnekte, sadece gösteri amaçlı olarak CASL paketini kullanacağız, ancak bu kütüphaneyi kullanmak zorunlu değildir. Ayrıca, önceki bölümde oluşturduğumuz `CaslAbilityFactory` sağlayıcısını kullanacağız.

İlk olarak, politika işleyicileri için arayüzleri tanımlayarak başlayalım:

```typescript
import { AppAbility } from '../casl/casl-ability.factory';

interface IPolicyHandler {
  handle(ability: AppAbility): boolean;
}

type PolicyHandlerCallback = (ability: AppAbility) => boolean;

export type PolicyHandler = IPolicyHandler | PolicyHandlerCallback;
```

Yukarıda belirtildiği gibi, bir politika işleyicisini tanımlamanın iki olası yolu vardır, bir nesne (IPolicyHandler arayüzünü uygulayan bir sınıf örneği) ve bir işlev (PolicyHandlerCallback türünü karşılayan).

Bunu yerine getirdikten sonra, `@CheckPolicies()` dekoratörünü oluşturalım. Bu dekoratör, belirli kaynaklara erişim için karşılanması gereken politikaları belirlemenizi sağlar.

```typescript
export const CHECK_POLICIES_KEY = 'check_policy';
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(CHECK_POLICIES_KEY, handlers);
```

Şimdi, bir route handler'a bağlı olan tüm politika işleyicilerini çıkarıp çalıştıran bir `PoliciesGuard` oluşturalım.

```typescript
@Injectable()
export class PoliciesGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private caslAbilityFactory: CaslAbilityFactory,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const policyHandlers =
      this.reflector.get<PolicyHandler[]>(
        CHECK_POLICIES_KEY,
        context.getHandler(),
      ) || [];

    const { user } = context.switchToHttp().getRequest();
    const ability = this.caslAbilityFactory.createForUser(user);

    return policyHandlers.every((handler) =>
      this.execPolicyHandler(handler, ability),
    );
  }

  private execPolicyHandler(handler: PolicyHandler, ability: AppAbility) {
    if (typeof handler === 'function') {
      return handler(ability);
    }
    return handler.handle(ability);
  }
}
```

> info **İpucu** Bu örnekte, `request.user`'ın kullanıcı örneğini içerdiğini varsaydık. Uygulamanızda, bu birleşik bağlantıyı özel **kimlik doğrulama guard**'ınızda yapacaksınız - daha fazla ayrıntı için [kimlik doğrulama](/docs/security/authentication) bölümüne bakın.

Bu örneği açıklayalım. `policyHandlers`, yöntem aracılığıyla `@CheckPolicies()` dekoratörü ile atanmış olan işleyicilerin bir dizisidir. Sonra, `CaslAbilityFactory#create` yöntemini kullanırız ki bu yöntem, belirli eylemleri gerçekleştirmek için kullanıcının yeterli izinlere sahip olup olmadığını doğrulamamıza olanak tanıyan `Ability` nesnesini oluşturur. Bu nesneyi, bir işlev veya `IPolicyHandler` arayüzünü uygulayan bir sınıfın örneği olan politika işleyicisine geçiriyoruz. Son olarak, `Array#every` yöntemini kullanarak her işleyicinin `true` değeri döndüğünden emin oluruz.

Son olarak, bu guard'ı test etmek için, herhangi bir route handler'a bağlayın ve içerideki politika işleyicisini (işlevsel yaklaşım için) aşağıdaki gibi kaydedin:

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies((ability: AppAbility) => ability.can(Action.Read, Article))
findAll() {
  return this.articlesService.findAll();
}
```

Alternatif olarak, `IPolicyHandler` arayüzünü uygulayan bir sınıfı tanımlayabiliriz:

```typescript
export class ReadArticlePolicyHandler implements IPolicyHandler {
  handle(ability: AppAbility) {
    return ability.can(Action.Read, Article);
  }
}
```

Ve aşağıdaki gibi kullanabiliriz:

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies(new ReadArticlePolicyHandler())
findAll() {
  return this.articlesService.findAll();
}
```

> warning **Dikkat** Politika işleyicisini `new` anahtar kelimesini kullanarak yerinde örneklememiz gerektiğinden, `ReadArticlePolicyHandler` sınıfı Bağımlılık Enjeksiyonunu kullanamaz. Bu durumu, `ModuleRef#get` yöntemi ile çözebilirsiniz (daha fazla bilgi için [buraya](/docs/fundamentals/module-ref) bakın). Temelde, `@CheckPolicies()` dekoratörü aracılığıyla işlevleri ve örnekleri kaydetmek yerine, bir `Type<IPolicyHandler>` geçmenize izin vermelisiniz. Ardından, guard'ınız içinde bir tür referansı kullanarak bir örneği alabilirsiniz: `moduleRef.get(YOUR_HANDLER_TYPE)` veya hatta `ModuleRef#create` yöntemini kullanarak dinamik olarak örneğini oluşturabilirsiniz.