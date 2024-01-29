### Passport (kimlik doğrulama)

[Passport](https://github.com/jaredhanson/passport), topluluk tarafından iyi bilinen ve birçok üretim uygulamasında başarıyla kullanılan en popüler node.js kimlik doğrulama kütüphanesidir. Bu kütüphaneyi **Nest** uygulamasına entegre etmek, `@nestjs/passport` modülünü kullanmak oldukça basittir. Yüksek seviyede, Passport şu adımları gerçekleştirir:

- Bir kullanıcıyı "kimlik bilgileri"ni (örneğin kullanıcı adı/parola, JSON Web Token ([JWT](https://jwt.io/)), veya bir Kimlik Sağlayıcıdan gelen kimlik belirtici) doğrulayarak kimliğini saptar.
- Kimlik doğrulama durumunu yönetir (taşınabilir bir belirteç, örneğin bir JWT, veya bir [Express oturumu](https://github.com/expressjs/session) oluşturarak).
- Kimlik doğrulanan kullanıcıyla ilgili bilgileri, bu bilgilerin daha sonra rota işleyicilerinde kullanılabilmesi için `Request` nesnesine ekler.

Passport'un [stratejiler](http://www.passportjs.org/) adlı zengin bir ekosistemi vardır ve çeşitli kimlik doğrulama mekanizmalarını uygular. Basit bir konsept olmasına rağmen, seçebileceğiniz Passport stratejileri seti büyük ve birçok çeşidi içerir. Passport, bu çeşitli adımları standart bir model içine soyutlar ve `@nestjs/passport` modülü, bu modeli Nest tarafından tanıdık yapılar içine sarmalar ve standartlaştırır.

Bu bölümde, bu güçlü ve esnek modülleri kullanarak bir RESTful API sunucusu için tamamen işlevsel bir kimlik doğrulama çözümü uygulayacağız. Burada açıklanan kavramları kullanarak herhangi bir Passport stratejisini uygulayabilir ve kimlik doğrulama şemanızı özelleştirebilirsiniz. Bu tam örnek için adımları takip ederek bu bölümdeki işlemleri gerçekleştirebilirsiniz.

#### Kimlik doğrulama gereksinimleri

İhtiyaçlarımızı daha ayrıntılı bir şekilde ele alalım. Bu kullanım durumu için istemciler, kullanıcı adı ve parola ile kimlik doğrulaması yaparak başlayacaklar. Kimlik doğrulandıktan sonra, sunucu bir JWT çıkaracak ve bu JWT, sonraki isteklerde kimliği kanıtlamak için bir [yetki başlığında taşınabilen bir belirteç](https://tools.ietf.org/html/rfc6750) olarak gönderilebilecek. Ayrıca, geçerli bir JWT içeren isteklere sadece erişilebilen bir korumalı bir rota oluşturacağız.

İlk olarak, bir kullanıcıyı kimlik doğrulama gereksinimimizi karşılamak üzere kimlik doğrulamamız gerekiyor. Ardından bunu bir JWT çıkararak genişleteceğiz. Son olarak da, istekte geçerli bir JWT kontrol eden bir korumalı bir rota oluşturacağız.

İlk olarak gerekli paketleri kurmamız gerekiyor. Passport, ihtiyaçlarımızı karşılayan bir kullanıcı adı/parola kimlik doğrulama mekanizması uygulayan [passport-local](https://github.com/jaredhanson/passport-local) adlı bir strateji sağlar, bu da bu kısmımız için uygundur.

```bash
$ npm install --save @nestjs/passport passport passport-local
$ npm install --save-dev @types/passport-local
```

> uyarı **Dikkat** Herhangi bir Passport stratejisi seçseniz bile, her zaman `@nestjs/passport` ve `passport` paketlerine ihtiyacınız olacaktır. Ardından, belirli bir kimlik doğrulama stratejisini uygulayan (örneğin, `passport-jwt` veya `passport-local`) strateji özel paketi kurmanız gerekecektir. Ayrıca, Passport stratejisinin tür tanımlamalarını da yükleyebilirsiniz, yukarıda `@types/passport-local` ile gösterildiği gibi, bu da TypeScript kodu yazarken yardımcı olur.

#### Passport Stratejilerinin Uygulanması

Artık kimlik doğrulama özelliğini uygulamaya hazırız. **Herhangi** bir Passport stratejisi için kullanılan sürecin genel bir bakışla başlayacağız. Passport'u kendi başına küçük bir çerçeve gibi düşünmek yardımcı olabilir. Çerçevenin zarafeti, kimlik doğrulama sürecini birkaç temel adıma soyutlaması ve bu adımları uyguladığınız stratejiye bağlı olarak özelleştirmenizi sağlamasıdır. Bir çerçeve gibidir çünkü özelleştirme parametrelerini (düz JSON nesneleri olarak) sağlayarak ve Passport'un uygun zamanlarda çağırdığı geri çağrı işlevleri şeklinde özel kod ekleyerek yapılandırılır. `@nestjs/passport` modülü, bu çerçeveyi Nest tarzında bir pakete sarar, böylece bir Nest uygulamasına entegre etmek kolay olur. Aşağıda `@nestjs/passport`'u kullanacağız, ancak önce **vanilya Passport**'un nasıl çalıştığını düşünelim.

Vanilya Passport'ta bir stratejiyi yapılandırırken şu iki şeyi sağlarsınız:

1. O stratejiye özgü seçeneklerin bir kümesi. Örneğin, JWT stratejisinde, belirteçleri imzalamak için bir sır vermiş olabilirsiniz.
2. "Doğrulama geri çağrısı" olarak adlandırılan ve Passport'a kullanıcı depolama alanınıza (kullanıcı hesaplarını yönettiğiniz yer) nasıl etkileşimde bulunacağınızı söylediğiniz yer. Burada, bir kullanıcının var olup olmadığını (ve/veya yeni bir kullanıcı oluşturup oluşturmadığınızı) ve kimlik bilgilerinin geçerli olup olmadığını doğrularsınız. Passport kütüphanesi, bu geri çağrının doğrulama başarılıysa tam bir kullanıcı döndürmesini, başarısızsa (kullanıcı bulunamaz veya passport-local durumunda parola eşleşmezse) null döndürmesini bekler.

`@nestjs/passport` ile, Passport stratejisini `PassportStrategy` sınıfını genişleterek yapılandırırsınız. Strateji seçeneklerini (yukarıda madde 1) alt sınıfınızda `super()` yöntemini çağırarak, isteğe bağlı olarak bir seçenek nesnesini geçirerek belirtirsiniz. Doğrulama geri çağrısını (yukarıda madde 2) alt sınıfınızda bir `validate()` yöntemi uygulayarak belirtirsiniz.

`AuthModule` ve içinde bir `AuthService` oluşturarak başlayacağız:

```bash
$ nest g module auth
$ nest g service auth
```

`AuthService`'i uygularken, kullanıcı işlemlerini bir `UsersService` içinde kapsamlandırmak faydalı olacaktır, bu nedenle bu modülü ve hizmeti şimdi oluşturalım:

```bash
$ nest g module users
$ nest g service users
```

Aşağıda gösterildiği gibi, bu oluşturulan dosyaların içeriğini değiştirin. Örnek uygulamamız için, `UsersService` basitçe kullanıcıların bellekteki sabit bir listesini tutar ve bir kullanıcı adına göre bir tane almak için bir `findOne` yöntemine sahiptir. Gerçek bir uygulamada, burada kullanıcı modelinizi ve isteğe bağlı olarak kullanmak istediğiniz kütüphaneyi (örneğin, TypeORM, Sequelize, Mongoose, vb.) kullanarak kalıcı katmanınızı oluşturacaksınız.

```typescript
@@filename(users/users.service)
import { Injectable } from '@nestjs/common';

// Gerçek bir kullanıcı varlığını temsil eden bir sınıf/arayüz olmalıdır
export type User = any;

@Injectable()
export class UsersService {
  private readonly users = [
    {
      userId: 1,
      username: 'john',
      password: 'changeme',
    },
    {
      userId: 2,
      username: 'maria',
      password: 'guess',
    },
  ];

  async findOne(username: string): Promise<User | undefined> {
    return this.users.find(user => user.username === username);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  constructor() {
    this.users = [
      {
        userId: 1,
        username: 'john',
        password: 'changeme',
      },
      {
        userId: 2,
        username: 'maria',
        password: 'guess',
      },
    ];
  }

  async findOne(username) {
    return this.users.find(user => user.username === username);
  }
}
```

`UsersModule` içinde tek yapmamız gereken değişiklik, `UsersService`'i `@Module` dekoratörünün `exports` dizisine eklemektir, böylece bu modül dışında görünür olur (bunu yakında `AuthService`'

imizde kullanacağız).

```typescript
@@filename(users/users.module)
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
@@switch
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

`AuthService`'in görevi bir kullanıcıyı almak ve parolayı doğrulamaktır. Bu amaçla bir `validateUser()` yöntemi oluşturuyoruz. Aşağıdaki kodda, kullanıcı nesnesinden parola özelliğini kaldırmak için kullanışlı bir ES6 spread operatörü kullanıyoruz. Birazdan `Passport` lokal stratejimizden bu `validateUser()` yöntemini çağıracağız.

```typescript
@@filename(auth/auth.service)
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
@Dependencies(UsersService)
export class AuthService {
  constructor(usersService) {
    this.usersService = usersService;
  }

  async validateUser(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }
}
```

> Uyarı **Uyarı** Tabii ki gerçek bir uygulamada bir parolayı düz metin olarak saklamazsınız. Bunun yerine, [bcrypt](https://github.com/kelektiv/node.bcrypt.js#readme) gibi bir kütüphaneyi, tuzlu bir tek yönlü hash algoritması ile kullanırsınız. Bu yaklaşımla sadece hashlenmiş parolaları saklarsınız ve ardından depolanan parolayı **gelen** parolanın hashlenmiş bir sürümü ile karşılaştırırsınız, bu nedenle kullanıcı parolalarını düz metin olarak saklamaz veya ortaya çıkarmazsınız. Örnek uygulamamızı basit tutmak için bu mutlak talimatı ihlal ediyoruz ve düz metin kullanıyoruz. **Gerçek uygulamanızda bunu yapmayın!**

Şimdi, `AuthModule`'umuzu `UsersModule`'i içe aktaracak şekilde güncellememiz gerekiyor.

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
})
export class AuthModule {}
```

#### Passport Local Stratejisinin Uygulanması

Şimdi Passport'un **yerel kimlik doğrulama stratejisi**'ni uygulayabiliriz. `auth` klasöründe `local.strategy.ts` adında bir dosya oluşturun ve aşağıdaki kodu ekleyin:

```typescript
@@filename(auth/local.strategy)
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { AuthService } from './auth.service';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super();
  }

  async validate(username: string, password: string): Promise<any> {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
@@switch
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException, Dependencies } from '@nestjs/common';
import { AuthService } from './auth.service';

@Injectable()
@Dependencies(AuthService)
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(authService) {
    super();
    this.authService = authService;
  }

  async validate(username, password) {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

Bu, önceki açıklamada belirtilen tarife uyduk. Bizim passport-local kullanım durumumuzda, yapılandırma seçenekleri olmadığından, kurucumuz sadece `super()`'ı, bir seçenek nesnesi olmadan çağırır.

> Bilgi **İpucu** Passport stratejisinin davranışını özelleştirmek için `super()` çağrısına bir seçenek nesnesi iletebiliriz. Bu örnekte, passport-local stratejisi varsayılan olarak istek gövdesinde `username` ve `password` adlı özellikleri bekler. Farklı özellik adlarını belirtmek için bir seçenek nesnesi iletebilirsiniz, örneğin: `super({{ '{' }} usernameField: 'email' {{ '}' }})`. Daha fazla bilgi için [Passport belgelerine](http://www.passportjs.org/docs/configure/) başvurabilirsiniz.

Ayrıca `validate()` yöntemini uyguladık. Her strateji için, Passport ilgili stratejiye özgü parametre kümesi kullanarak `@nestjs/passport`'taki `validate()` yöntemiyle implemente edilmiş doğrulama işlevini (verify function) çağırır. Local-strateji için Passport, aşağıdaki imza ile bir `validate()` yöntemi bekler: `validate(username: string, password:string): any`.

Doğrulama işinin çoğu, `AuthService`'imizde (ve `UsersService`'imizin yardımıyla) gerçekleştirildiği için bu yöntem oldukça basittir. **Herhangi** bir Passport stratejisi için `validate()` yöntemi aynı kalıbı takip eder, sadece kimlik bilgilerinin nasıl temsil edildiği detaylarında değişiklik gösterir. Bir kullanıcı bulunur ve kimlik bilgileri geçerliyse, kullanıcı Passport'un görevlerini tamamlamak için döndürülür (örneğin, `Request` nesnesinde `user` özelliğini oluşturur) ve isteğin işleme hattı devam eder. Bulunamazsa, bir istisna fırlatır ve <a href="exception-filters">istisna katmanımızın</a> bununla başa çıkmasına izin verir.

Genellikle, her strateji için `validate()` yöntemindeki tek önemli fark, bir kullanıcının var olup olmadığını ve geçerli olup olmadığını **nasıl** belirlediğinizdir. Örneğin, bir JWT stratejisinde, gereksinimlere bağlı olarak, çözülmüş belirteçte taşınan `userId`'nin kullanıcı veritabanımızdaki bir kayıtla eşleşip eşleşmediğini veya iptal edilmiş belirteçlerin bir listesiyle eşleşip eşleşmediğini değerlendirebiliriz. Bu nedenle, alt sınıflandırma ve stratejiye özgü doğrulama işlevini uygulama deseni tutarlı, zarif ve genişletilebilirdir.



`AuthModule`'umuzu tanımladığımız Passport özelliklerini kullanacak şekilde yapılandırmamız gerekiyor. `auth.module.ts` dosyasını şu şekilde güncelleyin:

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { LocalStrategy } from './local.strategy';

@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { LocalStrategy } from './local.strategy';

@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
```

#### Yerleşik Passport Guard'ları

<a href="guards">Guard'lar</a> bölümü, Guard'ların temel işlevini açıklar: bir isteğin route işleyicisi tarafından işlenip işlenmeyeceğini belirlemek. Bu hala geçerlidir ve bu standart yeteneği yakında kullanacağız. Ancak `@nestjs/passport` modülünü kullanma bağlamında, ilk başta kafa karıştırıcı olabilecek küçük bir yenilik ekleyeceğiz, bu nedenle şimdi bunu tartışalım. Uygulamanızın yetkilendirme açısından iki durumda bulunabileceğini düşünün:

1. kullanıcı/client **giriş yapmamış** (yetkilendirilmemiş durumda)
2. kullanıcı/client **giriş yapmış** (yetkilendirilmiş durumda)

İlk durumda (kullanıcı giriş yapmamışsa), iki farklı işlevi gerçekleştirmemiz gerekiyor:

- Yetkilendirilmemiş bir kullanıcının erişebileceği rotaları sınırlamak (yani, kısıtlı rotalara erişimi reddetmek). Bu işlevi ele almak için, korumalı rotalara bir Guard yerleştirerek bu Guard'ı kullanacağız. Beklediğiniz gibi, bu Guard'da geçerli bir JWT'nin varlığını kontrol edeceğiz, bu yüzden JWT'leri başarıyla verdiğimizde bu Guard üzerinde daha sonra çalışacağız.

- Daha önce yetkilendirilmemiş bir kullanıcının giriş yapmaya çalıştığında **kimlik doğrulama adımını başlatmak**. Bu adım, bir geçerli kullanıcıya bir JWT **verme** adımıdır. Bir an için bunu düşünerek, yetkilendirme başlatmak için kullanıcı adı/parola kimlik bilgilerini `POST` etmemiz gerektiğini biliyoruz, bu nedenle bunu ele alacak bir `POST /auth/login` rotası kuracağız. Bu raises the question: bu rotada passport-local stratejisini tam olarak nasıl çağırırız?

Cevap basittir: başka, biraz farklı bir Guard türü kullanarak. `@nestjs/passport` modülü, bize bunu yapmak için yerleşik bir Guard sağlar. Bu Guard, Passport stratejisini çağırır ve yukarıda açıklanan adımları başlatır (kimlik bilgilerini alma, doğrulama fonksiyonunu çalıştırma, `user` özelliğini oluşturma, vb.).

Yukarıda sıralanan ikinci durum (giriş yapmış kullanıcı), sadece zaten tartıştığımız standart Guard türüne dayanır ve giriş yapmış kullanıcılar için korumalı rotalara erişimi etkinleştirir.

<app-banner-courses-auth></app-banner-courses-auth>

#### Giriş Rota

Strateji hazır olduğuna göre, şimdi basit bir `/auth/login` rotasını uygulayabilir ve passport-local akışını başlatmak için yerleşik Guard'ı uygulayabiliriz.

`app.controller.ts` dosyasını açın ve içeriğini aşağıdaki ile değiştirin:

```typescript
@@filename(app.controller)
import { Controller, Request, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller()
export class AppController {
  @UseGuards(AuthGuard('local'))
  @Post('auth/login')
  async login(@Request() req) {
    return req.user;
  }
}
@@switch
import { Controller, Bind, Request, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller()
export class AppController {
  @UseGuards(AuthGuard('local'))
  @Post('auth/login')
  @Bind(Request())
  async login(req) {
    return req.user;
  }
}
```

`@UseGuards(AuthGuard('local'))` kullanarak, passport-local stratejisini genişlettiğimizde `@nestjs/passport` tarafından **otomatik olarak sağlanan** bir `AuthGuard` kullanıyoruz. Bunun ayrıntılarına inelim. Passport local stratejimizin varsayılan adı `'local'` dir. Bu adı `@UseGuards()` dekoratöründe kullanarak, bu adı `passport-local` paketi tarafından sağlanan kodla ilişkilendiriyoruz. Bu, uygulamamızda birden fazla Passport stratejimize sahip olabiliriz (her biri strateji özel bir `AuthGuard` sağlayabilir) ve hangi stratejiyi çağıracağını disambiguate etmek için kullanılır. Şu ana kadar sadece bir tane strateji ekledik, ancak kısa bir süre içinde ikinci bir tane ekleyeceğiz, bu nedenle bu disambiguation'a ihtiyaç duyulacaktır.

Rota testini yapmak için şu anda rotamızın sadece kullanıcıyı döndürmesini sağlayacağız. Bu aynı zamanda Passport'un başka bir özelliğini göstermemize de izin verir: Passport, `validate()` yönteminden döndürdüğümüz değere dayanarak otomatik olarak bir `user` nesnesi oluşturur ve bunu `Request` nesnesine `req.user` olarak atar. Daha sonra bunu, bir JWT oluşturmak ve döndürmek için kodla değiştireceğiz.

Bu API rotaları olduğu için, bunları yaygın olarak kullanılan [cURL](https://curl.haxx.se/) kütüphanesi kullanarak test edebilirsiniz. `UsersService` içinde sert kodlanmış olan herhangi bir `user` nesnesi ile test yapabilirsiniz.

```bash
$ # /auth/login rotasına POST
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # sonuç -> {"userId":1,"username":"john"}
```

Bu çalışsa da, `AuthGuard()`'a doğrudan strateji adını geçmek, kod tabanında sihirli dizeleri (magic strings) tanıtır. Bunun yerine, aşağıda gösterildiği gibi kendi sınıfınızı oluşturmanızı öneririz:

```typescript
@@filename(auth/local-auth.guard)
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}
```

Şimdi, `/auth/login` rota işleyicisini güncelleyebilir ve yerine `LocalAuthGuard`'ı kullanabiliriz:

```typescript
@UseGuards(LocalAuthGuard)
@Post('auth/login')
async login(@Request() req) {
  return req.user;
}
```

#### JWT Fonksiyonelliği

Auth sistemi JWT kısmına geçmeye hazırız. İhtiyaçlarımızı gözden geçirelim ve iyileştirelim:

- Kullanıcıların kullanıcı adı/şifre ile kimlik doğrulamasına izin vererek, ardından korumalı API uç noktalarına yönelik sonraki çağrılarda kullanılmak üzere bir JWT dönmek. Bu gereksinimi karşılamak için yolda ilerliyoruz. Tamamlamak için bir JWT çıkaran kodu yazmamız gerekecek.
- Bir geçerli bir JWT'nin varlığına dayanarak korunan API rotalarını oluşturun

Bu gereksinimleri desteklemek için birkaç daha fazla paket kurmamız gerekecek:

```bash
$ npm install --save @nestjs/jwt passport-jwt
$ npm install --save-dev @types/passport-jwt
```

`@nestjs/jwt` paketi (daha fazlası için [buraya](https://github.com/nestjs/jwt) bakın), JWT manipülasyonu konusunda yardımcı olan bir yardımcı pakettir. `passport-jwt` paketi, JWT stratejisini uygulayan Passport paketidir ve `@types/passport-jwt` TypeScript tür tanımlamalarını sağlar.

`POST /auth/login` isteğinin nasıl işlendiğine daha yakından bakalım. Yerleşik `AuthGuard` kullanarak rotayı süsledik, bu da demektir ki:

1. Rota işleyici **yalnızca kullanıcı doğrulandıysa çağrılacaktır**
2. `req` parametresi, (passport-local kimlik doğrulama akışı sırasında Passport tarafından doldurulan) bir `user` özelliği içerecektir

Bu bilgiyle, şimdi sonunda gerçek bir JWT oluşturabilir ve bu rotada onu döndürebiliriz. Hizmetleri temiz ve modüler tutmak için JWT oluşturmayı `authService` içinde ele alacağız. `auth` klasöründeki `auth.service.ts` dosyasını açın ve aşağıda gösterildiği gibi `login()` yöntemini ekleyin ve `JwtService`'i içeri aktarın:

```typescript
@@filename(auth/auth.service)
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user: any) {
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}

@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Dependencies(UsersService, JwtService)
@Injectable()
export class AuthService {
  constructor(usersService, jwtService) {
    this.usersService = usersService;
    this.jwtService = jwtService;
  }

  async validateUser(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user) {
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

JWT oluşturmak için `@nestjs/jwt` kütüphanesini kullanıyoruz. Bu kütüphane, `user` nesnesinin özelliklerinin bir alt kümesinden JWT'mizi oluşturan bir `sign()` fonksiyonu sağlar, ardından bu JWT'yi tek bir `access_token` özelliği olan basit bir nesne olarak döndürürüz. Not: `userId` değerimizi tutmak için `sub` adında bir özellik adını seçiyoruz, JWT standartlarına uygun olması için. JwtService sağlayıcısını `AuthService`'e enjekte etmeyi unutmayın.

Şimdi `AuthModule`'u yeni bağımlılıkları içe aktarmak ve `JwtModule`'u yapılandırmak için güncellememiz gerekiyor.

İlk olarak, `auth` klasöründe `constants.ts` dosyasını oluşturun ve aşağıdaki kodu ekleyin:

```typescript
@@filename(auth/constants)
export const jwtConstants = {
  secret: 'BU DEĞERİ KULLANMAYIN. BUNUN YERİNE KOMPLEKS BİR SECRET OLUŞTURUN VE KOD DIŞINDA GÜVENLİ BİR YERDE SAKLAYIN.',
};
@@switch
export const jwtConstants = {
  secret: 'BU DEĞERİ KULLANMAYIN. BUNUN YERİNE KOMPLEKS BİR SECRET OLUŞTURUN VE KOD DIŞINDA GÜVENLİ BİR YERDE SAKLAYIN.',
};
```

Bu anahtarı JWT'nin imzalanması ve doğrulanması adımları arasında paylaşmak için kullanacağız.

> Uyarı **Uyarı** **Bu anahtarı genel olarak ifşa etmeyin**. Burada bunu net bir şekilde ne yaptığını göstermek için yaptık, ancak üretim sistemlerinde **bu anahtarı korumalısınız** ve uygun önlemleri almalısınız, örneğin bir secrets vault, environment variable veya configuration service kullanarak.

Şimdi, `auth.module.ts` dosyasını `JwtModule`'u içe aktarmak ve yeni bağımlılıkları yapılandırmak için aşağıdaki gibi güncelleyin:

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { Users

Module } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy],
  exports: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

`JwtModule`'u `register()` kullanarak yapılandırıyoruz ve bir yapılandırma nesnesi ile geçiyoruz. Nest `JwtModule` hakkında daha fazla bilgi için [buraya](https://github.com/nestjs/jwt/blob/master/README.md) bakın ve mevcut yapılandırma seçenekleri hakkında daha fazla bilgi için [buraya](https://github.com/auth0/node-jsonwebtoken#usage) bakın.

Şimdi `/auth/login` rotasını bir JWT döndürmek için güncelleyebiliriz.

```typescript
@@filename(app.controller)
import { Controller, Request, Post, UseGuards } from '@nestjs/common';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }
}
@@switch
import { Controller, Bind, Request, Post, UseGuards } from '@nestjs/common';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  @Bind(Request())
  async login(req) {
    return this.authService.login(req.user);
  }
}
```

Hadi cURL kullanarak rotalarımızı test edelim. `UsersService` içinde sert kodlanmış herhangi bir `user` nesnesi ile test yapabilirsiniz.

```bash
$ # /auth/login rotasına POST
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # sonuç -> {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
$ # Not: Yukarıdaki JWT kısaltılmıştır
```

#### Passport JWT'yi Uygulama

Şimdi son gereksinimimizi ele alabiliriz: bir istekte geçerli bir JWT'nin bulunmasını şart koşarak uç noktaları korumak. Passport burada da bize yardımcı olabilir. JSON Web Token'lar ile RESTful uç noktalarını güvence altına alan [passport-jwt](https://github.com/mikenicholson/passport-jwt) stratejisini sağlar. İlk olarak, `auth` klasöründe `jwt.strategy.ts` adında bir dosya oluşturun ve aşağıdaki kodu ekleyin:

```typescript
@@filename(auth/jwt.strategy)
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { jwtConstants } from './constants';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.secret,
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, username: payload.username };
  }
}
@@switch
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { jwtConstants } from './constants';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.secret,
    });
  }

  async validate(payload) {
    return { userId: payload.sub, username: payload.username };
  }
}
```

`JwtStrategy` ile tüm Passport stratejileri için önceki açıklamalarda belirtilen aynı tarife uydum. Bu strateji, başlatma gerektirdiği için bunu `super()` çağrısında bir seçenek nesnesi ile yaparız. Kullanılabilir seçenekler hakkında [buradan](https://github.com/mikenicholson/passport-jwt#configure-strategy) daha fazla bilgi alabilirsiniz. Bizim durumumuzda, bu seçenekler şunlardır:

- `jwtFromRequest`: JWT'nin `Request` üzerinden nasıl çıkarılacağını sağlar. API isteklerimizin Authorization başlığında bir taşıyıcı belirteceği standart yaklaşımı kullanacağız. Diğer seçenekler için [buraya](https://github.com/mikenicholson/passport-jwt#extracting-the-jwt-from-the-request) bakabilirsiniz.
- `ignoreExpiration`: açık olmak için, varsayılan `false` ayarını seçtik. Bu, bir JWT'nin sona erip ermediğini kontrol etme sorumluluğunu Passport modülüne bırakır. Bu, rotamıza sona ermiş bir JWT sağlanırsa, istek reddedilecek ve `401 Unauthorized` yanıtı gönderilecektir. Passport bunu otomatik olarak bizim için ele alır.
- `secretOrKey`: Token'ı imzalamak için simetrik bir sır sağlama hızlı seçeneğini kullanıyoruz. Diğer seçenekler, örneğin bir PEM kodlu genel anahtar, üretim uygulamaları için daha uygun olabilir (daha fazla bilgi için [buraya](https://github.com/mikenicholson/passport-jwt#configure-strategy) bakın). Her durumda, daha önce imzaladığımız ve geçerli bir kullanıcıya verdiğimiz bir JWT aldığımızı garanti ettiğimizden emin olabiliriz. 

`validate()` metoduna biraz değinelim. Passport jwt-stratejisi için, Passport önce JWT'nin imzasını doğrular ve JSON'ı çözer. Ardından `validate()` metodumuzu, çözülmüş JSON'ı tek parametre olarak geçirerek çağırır. JWT imzalama işlemi temel alındığında, **geçerli bir token aldığımızdan emin oluyoruz**.

Bu nedenle `validate()` geri çağrısına verdiğimiz yanıt son derece basittir: sadece `userId` ve `username` özelliklerini içeren bir nesne döndürürüz. Tekrar hatırlatmak gerekirse, Passport, `validate()` metodumuzun dönüş değerine dayanarak bir `user` nesnesi oluşturacak ve bunu `Request` nesnesinde bir özellik olarak ekleyecektir.

Ayrıca bu yaklaşımın, işlem sürecine diğer iş mantığı eklemek için ('kancalar' aslında) alan bıraktığını belirtmek önemlidir. Örneğin, `validate()` metodumuzda bir veritabanı araması yapabilir ve kullanıcı hakkında daha fazla bilgi çıkarabiliriz, böylece `Request`'imizde daha zengin bir `user` nesnesi kullanılabilir hale gelir. Bu aynı zamanda daha fazla token doğrulaması yapabileceğimiz ve token iptalini gerçekleştirebileceğimiz yerdir. Burada örnek kodumuzda uyguladığımız model, her API çağrısının hemen geçerli bir JWT'nin varlığına dayanarak yetkilendirildiği hızlı, "stateless JWT" modelidir ve talep edenin (`userId` ve `username`) hakkında küçük bir bilgi sağlanır.

Yeni `JwtStrategy`'i `AuthModule`'da bir sağlayıcı olarak ekleyin:

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { JwtStrategy } from './jwt.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy, Jwt

Strategy],
  exports: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { JwtStrategy } from './jwt.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

JWT'yi imzalarken kullandığımız aynı sırrı içe aktararak, Passport tarafından gerçekleştirilen **doğrulama** aşaması ile AuthService tarafından gerçekleştirilen **imza** aşamasının ortak bir sırrı kullandığından emin oluruz.

Son olarak, yerleşik `AuthGuard`'ı genişleten `JwtAuthGuard` sınıfını tanımlıyoruz:

```typescript
@@filename(auth/jwt-auth.guard)
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

#### Korumalı Rota ve JWT Stratejisi Muhafızlarını Uygulama

Şimdi korumalı rotamızı ve ilgili muhafızını uygulayabiliriz.

`app.controller.ts` dosyasını açın ve aşağıdaki gibi güncelleyin:

```typescript
@@filename(app.controller)
import { Controller, Get, Request, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from './auth/jwt-auth.guard';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }

  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
@@switch
import { Controller, Dependencies, Bind, Get, Request, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from './auth/jwt-auth.guard';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Dependencies(AuthService)
@Controller()
export class AppController {
  constructor(authService) {
    this.authService = authService;
  }

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  @Bind(Request())
  async login(req) {
    return this.authService.login(req.user);
  }

  @UseGuards(JwtAuthGuard)
  @Get('profile')
  @Bind(Request())
  getProfile(req) {
    return req.user;
  }
}
```

Bir kez daha, `@nestjs/passport` modülünün bize otomatik olarak sağladığı `AuthGuard`'ı uyguluyoruz ve bu, passport-jwt modülünü yapılandırdığımızda otomatik olarak oluşturulan stratejiyi referans aldığı için default adı olan `jwt` ile ilişkilendirilir. `GET /profile` rotamıza erişildiğinde, Muhafız otomatik olarak passport-jwt özel yapılandırmalı stratejimizi çağırır, JWT'yi doğrular ve `user` özelliğini `Request` nesnesine atar.

Uygulamanın çalıştığından emin olun ve `cURL` kullanarak rotaları test edin.

```bash
$ # GET /profile
$ curl http://localhost:3000/profile
$ # result -> {"statusCode":401,"message":"Unauthorized"}

$ # POST /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm... }

$ # GET /profile using access_token returned from previous step as bearer code
$ curl http://localhost:3000/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
$ # result -> {"userId":1,"username":"john"}
```

`AuthModule` içinde JWT'yi `60 saniye` süreyle kullanabilecek şekilde yapılandırdık. Bu muhtemelen çok kısa bir süre olduğundan, token'ın sona erme ayrıntıları ile başa çıkmak bu makalenin kapsamının ötesindedir. Ancak, JWT'lerin ve passport-jwt stratejisinin önemli bir özelliğini göstermek için bunu seçtik. Oturum açtıktan sonra `GET /profile` isteğinde bulunmadan önce 60 saniye beklerseniz, `401 Unauthorized` yanıtı alırsınız. Bu, Passport'un otomatik olarak JWT'nin sona erme zamanını kontrol etmesi nedeniyle uygulamanızda bunu yapma zahmetinden kurtarılırsınız.

JWT kimlik doğrulama uygulamamızı tamamladık. JavaScript istemcileri (Angular/React/Vue gibi), ve diğer JavaScript uygulamaları artık API Sunucumuzla güvenli bir şekilde kimlik doğrulayabilir ve iletişim kurabilirler.

#### Muhafızları Genişletme

Çoğu durumda, sağlanan bir `AuthGuard` sınıfını kullanmak yeterlidir. Ancak, varsayılan hata işleme veya kimlik doğrulama mantığını basitçe genişletmek istediğiniz durumlar olabilir. Bu durumda, yerleşik sınıfı genişletebilir ve alt sınıf içinde yöntemleri geçersiz kılabilirsiniz.

```typescript
import {
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    // Buraya özel kimlik doğrulama mantığınızı ekleyin
    // örneğin, super.logIn(request) çağırarak bir oturum oluşturun.
    return super.canActivate(context);
  }

  handleRequest(err, user, info) {
    // "info" veya "err" argümanlarına dayanarak istisna fırlatabilirsiniz
    if (err || !user) {
      throw err || new UnauthorizedException();
    }
    return user;
  }
}
```

Varsayılan hata işleme ve kimlik doğrulama mantığı genişletmenin yanı sıra, kimliği doğrulamaya bir dizi strateji üzerinden geçmeye izin verebiliriz. Başarılı olan ilk strateji, yönlendirme veya hata, zinciri duraklatacaktır. Kimlik doğrulama başarısızlıkları, her bir stratejiden geçerek devam edecek ve sonunda tüm stratejiler başarısız olursa sonuç başarısız olacaktır.

```typescript
export class JwtAuthGuard extends AuthGuard(['strategy_jwt_1', 'strategy_jwt_2', '...']) { ... }
```

#### Kimlik Doğrulamayı Global Olarak Etkinleştirme

Eğer çoğu endpoint'iniz varsayılan olarak korunmalıysa, kimlik doğrulama muhafızını [global muhafız](/docs/guards#binding-guards) olarak kaydedebilir ve her denetleyicinin üstüne `@UseGuards()` dekoratörünü kullanmak yerine yalnızca hangi rotaların genel olduğunu işaretleyebilirsiniz.

Önce, `JwtAuthGuard`'ı aşağıdaki şekilde (herhangi bir modülde) global bir muhafız olarak kaydedin:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

Bu işlem yapıldığında, Nest otomatik olarak `JwtAuthGuard`'ı tüm uç noktalara bağlar.

Şimdi rotaları genel olarak tanımlamak için bir mekanizma sağlamamız gerekiyor. Bunun için, `SetMetadata` dekoratör fabrika işlevini kullanarak özel bir dekoratör oluşturabiliriz.

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

Yukarıdaki dosyada, iki sabit ihraç ettik. Birincisi `IS_PUBLIC_KEY` adında metadata anahtarı, diğeri ise yeni dekoratörümüzü `Public` (isteğe bağlı olarak `SkipAuth` veya `AllowAnon` olarak adlandırabilirsiniz, projenize uygun olanı seçin) adında ihraç ettik.

Artık özel bir `@Public()` dekoratörümüz olduğuna göre, bunu herhangi bir yöntemi aşağıdaki gibi süslemek için kullanabiliriz:

```typescript
@Public()
@Get()
findAll() {
  return [];
}
```

Son olarak, `JwtAuthGuard`'ı, `"isPublic"` metadata bulunduğunda `true` döndürecek şekilde yapılandırmamız gerekiyor. Bunun için `Reflector` sınıfını kullanacağız (daha fazla bilgi için [buraya](/docs/guards#putting-it-all-together) bakın).

```typescript
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      return true;
    }
    return super.canActivate(context);
  }
}
```

#### İstek Kapsamlı Stratejiler

Passport API'si, stratejileri kütüphanenin global örneğine kaydetmeye dayanır. Bu nedenle stratejiler, isteğe bağlı seçeneklere veya her istek için dinamik olarak oluşturulan bir örneğe sahip olacak şekilde tasarlanmamıştır (daha fazla bilgi için [request-scoped](/docs/fundamentals/injection-scopes) sağlayıcılarına bakın). Stratejiyi request-scoped olarak yapılandırdığınızda, Nest onu hiçbir zaman oluşturmaz, çünkü belirli bir rotaya bağlı değildir. Herhangi bir "request-scoped" stratejisinin isteğe bağlı olarak yürütülmesi gerektiğini fiziksel olarak belirlemenin bir yolu yoktur.

Ancak, strateji içinde request-scoped sağlayıcıları dinamik olarak çözmek için yollar vardır. Bunun için [module reference](/docs/fundamentals/module-ref) özelliğini kullanırız.

İlk olarak, `local.strategy.ts` dosyasını açın ve `ModuleRef`'i normal yoldan enjekte edin:

```typescript
constructor(private moduleRef: ModuleRef) {
  super({
    passReqToCallback: true,
  });
}
```

> info **İpucu** `ModuleRef` sınıfı, `@nestjs/core` paketinden içe aktarılır.

Yukarıdaki şekilde `passReqToCallback` yapılandırma özelliğini `true` olarak ayarlamayı unutmayın.

Sonraki adımda, isteği bağlam tabanlı bir kimlik oluşturmak için, request örneği yerine yeni bir tane oluşturmak yerine, mevcut bağlam kimliğini almak için `validate()` yöntemi içinde `ContextIdFactory` sınıfının `getByRequest()` yöntemini kullanın:

```typescript
async validate(
  request: Request,
  username: string,
  password: string,
) {
  const contextId = ContextIdFactory.getByRequest(request);
  // "AuthService" bir request-scoped sağlayıcıdır
  const authService = await this.moduleRef.resolve(AuthService, contextId);
  ...
}
```

Yukarıdaki örnekte, `resolve()` yöntemi, `AuthService` sağlayıcısının isteğe bağlı örneğini asenkron olarak döndürecektir (AuthService'nin bir request-scoped sağlayıcı olarak işaretlendiğini varsaydık).

#### Passport'ı Özelleştirme

Herhangi bir standart Passport özelleştirme seçeneği, aynı şekilde `register()` yöntemini kullanarak iletilir. Kullanılan stratejiye bağlı olarak mevcut seçenekler değişir. Örneğin:

```typescript
PassportModule.register({ session: true });
```

Ayrıca, stratejilere onları yapılandırmak için yapılandırma nesnesi geçebilirsiniz. Örneğin, yerel strateji için şu şekilde iletebilirsiniz:

```typescript
constructor(private authService: AuthService) {
  super({
    usernameField: 'email',
    passwordField: 'password',
  });
}
```

Özellik adlarını incelemek için resmi [Passport Web Sitesi'ne](http://www.passportjs.org/docs/oauth/) göz atın.

#### İsimlendirilmiş Stratejiler

Bir strateji uygularken, `PassportStrategy` fonksiyonuna ikinci bir argüman geçerek ona bir isim sağlayabilirsiniz. Bunu yapmazsanız, her stratejinin varsayılan bir adı olacaktır (örneğin, jwt-stratejisi için 'jwt'):

```typescript
export class JwtStrategy extends PassportStrategy(Strategy, 'myjwt')
```

Sonra bunu `@UseGuards(AuthGuard('myjwt'))` gibi bir dekoratör ile referans verebilirsiniz.

#### GraphQL

Bir [GraphQL](https://docs.nestjs.com/graphql/quick-start) ile AuthGuard kullanmak için, yerleşik AuthGuard sınıfını genişletin ve getRequest() yöntemini geçersiz kılın.

```typescript
@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  getRequest(context: ExecutionContext) {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req;
  }
}
```

GraphQL resolver'ınızda geçerli kimliği doğrulanmış kullanıcıyı almak için `@CurrentUser()` dekoratörünü tanımlayabilirsiniz:

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

export const CurrentUser = createParamDecorator(
  (data: unknown, context: ExecutionContext) => {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req.user;
  },
);
```

Yukarıdaki dekoratörü resolver'ınızda kullanmak için, lütfen onu sorgunuzun veya mutasyonunuzun parametresi olarak eklediğinizden emin olun:

```typescript
@Query(returns => User)
@UseGuards(GqlAuthGuard)
whoAmI(@CurrentUser() user: User) {
  return this.usersService.findById(user.id);
}
```