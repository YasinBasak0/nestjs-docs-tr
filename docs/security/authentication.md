### Kimlik Doğrulama

Kimlik doğrulama, çoğu uygulamanın **temel** bir parçasıdır. Kimlik doğrulamayı yönetmek için birçok farklı yaklaşım ve strateji bulunmaktadır. Herhangi bir projede alınan yaklaşım, özel uygulama gereksinimlerine bağlıdır. Bu bölüm, çeşitli gereksinimlere uygun şekilde adapte edilebilecek kimlik doğrulama yaklaşımlarını sunmaktadır.

Önce gereksinimlerimizi detaylandıralım. Bu kullanım durumu için istemciler, bir kullanıcı adı ve şifre ile kimlik doğrulaması yaparak başlayacaklar. Kimlik doğrulandıktan sonra, sunucu, kimliği kanıtlamak için bir sonraki isteklerde bir [taşıyıcı belirteci (bearer token)](https://tools.ietf.org/html/rfc6750) olarak gönderilebilecek bir JWT çıkaracaktır. Ayrıca, geçerli bir JWT içeren isteklere yalnızca erişilebilen bir korumalı bir rota oluşturacağız.

İlk gereksinimle başlayalım: bir kullanıcının kimlik doğrulaması. Ardından bunu bir JWT çıkararak genişleteceğiz. Son olarak, istekte geçerli bir JWT'nin kontrol edildiği bir korumalı bir rota oluşturacağız.

#### Bir kimlik doğrulama modülü oluşturma

Önce bir `AuthModule` ve içinde bir `AuthService` ve bir `AuthController` oluşturarak başlayacağız. `AuthService`'i kimlik doğrulama mantığını uygulamak ve `AuthController`'ı kimlik doğrulama uç noktalarını açmak için kullanacağız.

```bash
$ nest g module auth
$ nest g controller auth
$ nest g service auth
```

`AuthService`'i uygularken, kullanıcı işlemlerini kapsamak için bir `UsersService`'i kullanışlı bulacağımızı göreceğiz, bu nedenle bu modülü ve servisi şimdi oluşturalım:

```bash
$ nest g module users
$ nest g service users
```

Aşağıdaki gibi bu oluşturulan dosyaların içeriğini değiştirin. Örnek uygulamamız için `UsersService` sadece bellekte sabit bir kullanıcı listesini korur ve bir kullanıcıyı kullanıcı adına göre almak için bir yöntem içerir. Gerçek bir uygulamada burada kullanıcı modelinizi ve tercih ettiğiniz kütüphaneyi kullanarak kalıcılık katmanınızı oluşturacaksınız (örneğin, TypeORM, Sequelize, Mongoose, vb.).

```typescript
@@filename(users/users.service)
import { Injectable } from '@nestjs/common';

// Gerçek bir kullanıcı varlığını temsil eden bir sınıf/arayüz olmalı
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

`UsersModule` içinde tek gerekli değişiklik, `@Module` dekoratörünün `exports` dizisine `UsersService`'i eklemektir, böylece bu modül dışında görünür olur (yakında `AuthService`'de kullanacağız).

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

#### "Giriş Yap" uç noktasını uygulama

`AuthService`'in görevi bir kullanıcıyı almak ve şifreyi doğrulamaktır. Bu amacı için bir `signIn()` yöntemi oluşturuyoruz. Aşağıdaki kodda, kullanıcı nesnesinden şifre özelliğini çıkarmak için kullanışlı bir ES6 yayılım operatörü kullanıyoruz. Bu, kullanıcı nesnelerini döndürürken şifre gibi hassas alanları (parolalar veya diğer güvenlik anahtarları gibi) açıklamak istemezsiniz.

```typescript
@@filename(auth/auth.service)
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}

  async signIn(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const { password, ...result } = user;
    // TODO: Burada bir JWT üret

 ve bunu kullanıcı nesnesi yerine döndür
    return result;
  }
}
@@switch
import { Injectable, Dependencies, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
@Dependencies(UsersService)
export class AuthService {
  constructor(usersService) {
    this.usersService = usersService;
  }

  async signIn(username: string, pass: string) {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const { password, ...result } = user;
    // TODO: Burada bir JWT üret ve bunu kullanıcı nesnesi yerine döndür
    return result;
  }
}
```

> Uyarı **Uyarı** Elbette gerçek bir uygulamada bir şifreyi düz metinde saklamazsınız. Bunun yerine [bcrypt](https://github.com/kelektiv/node.bcrypt.js#readme) gibi bir kütüphaneyi, tuzlu bir tek yönlü karma algoritmasıyla kullanırsınız. Bu yaklaşım ile sadece karma şifreleri saklarsınız ve daha sonra saklanan şifreyi **gelen** şifrenin karma bir sürümü ile karşılaştırırsınız, böylece kullanıcı şifrelerini düz metinde saklamaz veya ortaya çıkarmazsınız. Örnek uygulamamızı basit tutmak için bu mutlak talimati ihlal ediyor ve düz metin kullanıyoruz. **Gerçek uygulamanızda bunu yapmayın!**

Şimdi, `AuthModule`'umuzu `UsersModule`'i içe aktaracak şekilde güncelliyoruz.

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}
```

Bunu yaparak, `AuthController`'ı açalım ve ona bir `signIn()` yöntemi ekleyelim. Bu yöntem, bir kullanıcıyı kimlik doğrulamak için istemci tarafından çağrılacaktır. İstek gövdesinde kullanıcı adı ve şifreyi alacak ve kullanıcı kimlik doğrulandıysa bir JWT belirteci döndürecektir.

```typescript
@@filename(auth/auth.controller)
import { Body, Controller, Post, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }
}
```

> info **İpucu** İdeali olarak, `Record<string, any>` türü yerine isteğin gövdesinin şeklini tanımlayan bir DTO sınıfını kullanmalıyız. Daha fazla bilgi için [doğrulama](/docs/techniques/validation) bölümüne bakın.

<app-banner-courses-auth></app-banner-courses-auth>

#### JWT Belirteci

Kimlik doğrulama sistemimizin JWT kısmına geçmeye hazırız. Gereksinimlerimizi gözden geçirelim ve rafine edelim:

- Kullanıcılara kullanıcı adı/şifre ile kimlik doğrulama olanağı tanıyarak, ardışık çağrılarda kullanılmak üzere bir JWT döndürme. Bu gereksinimi karşılamak için iyi bir yoldayız. Bunun tamamlanması için bir JWT çıkaran kodu yazmamız gerekecek.
- Geçerli bir JWT'nin varlığına dayanarak korumalı API uç noktalarına yapılan çağrıları koruyan API rotaları oluşturun.

JWT gereksinimlerimizi desteklemek için bir ek paket yüklememiz gerekecek:

```bash
$ npm install --save @nestjs/jwt
```

> info **İpucu** `@nestjs/jwt` paketi (daha fazla bilgi için [buraya](https://github.com/nestjs/jwt) bakın), JWT manipülasyonu konusunda yardımcı olan yardımcı bir pakettir. Bu, JWT belirteçlerini oluşturma ve doğrulama işlemlerine yardımcı olur.

Servislerimizi temiz bir şekilde modüler hale getirmek için JWT oluşturmayı `authService`'de ele alacağız. `auth.service.ts` dosyasını `auth` klasöründe açın, `JwtService`'i enjekte edin ve `signIn` yöntemini aşağıda gösterildiği gibi bir JWT belirteci oluşturacak şekilde güncelleyin:

```typescript
@@filename(auth/auth.service)
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async signIn(
    username: string,
    pass: string,
  ): Promise<{ access_token: string }> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { sub: user.userId, username: user.username };
    return {
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
@@switch
import { Injectable, Dependencies, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Dependencies(UsersService, JwtService)
@Injectable()
export class AuthService {
  constructor(usersService, jwtService) {
    this.usersService = usersService;
    this.jwtService = jwtService;
  }

  async signIn(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
```

JWT oluşturmak için `@nestjs/jwt` kütüphanesini kullanıyoruz. Bu kütüphane, `user` nesnesinin özelliklerinin bir alt kümesinden JWT'mizi oluşturan `signAsync()` fonksiyonunu sağlar, ardından bunu basit bir nesne olarak `access_token` adlı tek bir özellikle döndürür. Not: JWT standartlarıyla tutarlı olması için `userId` değerimizi içeren `sub` adlı bir özellik adı seçiyoruz. `AuthService`'e `JwtService` sağlayıcısını enjekte etmeyi unutmayın.

Şimdi `AuthModule`'u yeni bağımlılıkları içe aktaracak ve `JwtModule`'u yapılandıracak şekilde güncellememiz gerekiyor.

İlk olarak, `auth` klasöründe `constants.ts` dosyasını oluşturun ve aşağıdaki kodu ekleyin:

```typescript
@@filename(auth/constants)
export const jwtConstants = {
  secret: 'BU DEĞERİ KULLANMAYIN. BUNUN YERİNE KARMAŞIK BİR SIRAYI OLUŞTURUN VE KAYNAK KODUN DIŞINDA GÜVENLİ BİR ŞEKİLDE SAKLAYIN.',
};
@@switch
export const jwtConstants = {
  secret: 'BU DEĞERİ KULLANMAYIN. BUNUN YERİNE KARMAŞIK BİR SIRAYI OLUŞTURUN VE KAYNAK KODUN DIŞINDA GÜVENLİ BİR ŞEKİLDE SAKLAYIN.',
};
```

Bu, JWT imzalama ve doğrulama adımları arasında anahtarımızı paylaşmak için kullanılacak.

> Uyarı **Uyarı** Bu anahtarı herkese açık bir şekilde ifşa etmeyin. Burada kodun ne yaptığını açıkça anlamak için yayınladık, ancak üretim sistemlerinde **bu anahtarı korumanız gerekiyor** ve bunu bir güvenlik kasası, ortam değişkeni veya yapılandırma servisi gibi uygun önlemlerle korumalısınız.

Şimdi, `auth` klasöründeki `auth.module.ts` dosyasını açın ve onu şu şekilde güncelleyin:

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { JwtModule } from '@nestjs/jwt';
import { AuthController } from './auth.controller';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { JwtModule } from '@nestjs/jwt';
import { AuthController } from './auth.controller';
import { jwtConstants } from './

constants';

@Module({
  imports: [
    UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

> İpucu **İpucu** `JwtModule`'u global olarak kaydediyoruz ki işlerimiz kolay olsun. Bu, `JwtModule`'u uygulamamızın başka bir yerinde içe aktarmamıza gerek olmadığı anlamına gelir.

`JwtModule`'u `register()` kullanarak yapılandırıyoruz ve bir yapılandırma nesnesini ileterek. Daha fazla bilgi için [buraya](https://github.com/nestjs/jwt/blob/master/README.md) ve [buraya](https://github.com/auth0/node-jsonwebtoken#usage) bakın.

Hadi şimdi cURL kullanarak rotalarımızı test edelim. `UsersService`'de sert kodlu olan `user` nesnelerinden herhangi biri ile test yapabilirsiniz.

```bash
$ # /auth/login'e POST yap
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
$ # Not: Yukarıdaki JWT kırpılmıştır
```

#### Kimlik doğrulama koruyucusu uygulama

Şimdi son gereksinimimize gelebiliriz: bir istekte geçerli bir JWT'nin bulunmasını zorunlu kılarak uç noktaları korumak. Bunun için rotalarımızı korumak için kullanabileceğimiz bir `AuthGuard` oluşturarak yapacağız.

```typescript
@@filename(auth/auth.guard)
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { jwtConstants } from './constants';
import { Request } from 'express';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      const payload = await this.jwtService.verifyAsync(
        token,
        {
          secret: jwtConstants.secret
        }
      );
      // 💡 We're assigning the payload to the request object here
      // so that we can access it in our route handlers
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

Artık korunan rotamızı uygulayabilir ve `AuthGuard`'ı kullanarak korumak için kaydedebiliriz.

`auth.controller.ts` dosyasını açın ve aşağıdaki gibi güncelleyin:

```typescript
@@filename(auth/auth.controller)
import {
  Body,
  Controller,
  Get,
  HttpCode,
  HttpStatus,
  Post,
  Request,
  UseGuards
} from '@nestjs/common';
import { AuthGuard } from './auth.guard';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }

  @UseGuards(AuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

Yeni oluşturduğumuz `AuthGuard`'ı, `GET /profile` rotasına uyguluyoruz böylece bu rota koruma altına alınmış olacak.

Uygulamanın çalıştığından emin olun ve `cURL` kullanarak rotaları test edin.

```bash
$ # GET /profile
$ curl http://localhost:3000/auth/profile
{"statusCode":401,"message":"Unauthorized"}

$ # POST /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."}

$ # GET /profile using access_token returned from previous step as bearer code
$ curl http://localhost:3000/auth/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
{"sub":1,"username":"john","iat":...,"exp":...}
```

`AuthModule`'da JWT'nin süresini `60 saniye` olarak ayarladık. Bu çok kısa bir süre, ve belirteç süresi ve yenileme detaylarıyla uğraşmak bu makalenin kapsamının ötesindedir. Ancak, JWT'lerin önemli bir özelliğini göstermek için bunu seçtik. Yetkilendirmenin ardından `GET /auth/profile` isteği yapmadan önce 60 saniye beklerseniz, `401 Unauthorized` yanıtını alırsınız. Bu, `@nestjs/jwt`'nin JWT'nin son kullanma zamanını otomatik olarak kontrol etmesi sayesinde uygulamanızda bunu yapma zahmetinden sizi kurtarır.

Şimdi JWT kimlik doğrulama uygulamamızı tamamladık. JavaScript istemcileri (Angular/React/Vue gibi), ve diğer JavaScript uygulamaları, artık API Sunucumuzla güvenli bir şekilde kimlik doğrulama yapabilir ve iletişim kurabilir.

#### Kimlik doğrulamayı genel olarak etkinleştirme

Eğer uç noktalarınızın büyük bir çoğunluğunun varsayılan olarak korunması gerekiyorsa, kimlik doğrulama koruyucusunu [global koruyucu](/docs/guards#binding-guards) olarak kaydedebilir ve her denetleyicinin üstüne `@UseGuards()` dekoratörünü kullanmak yerine hangi rotaların kamuya açık olması gerektiğini basitçe belirtebilirsiniz.

İlk olarak, `AuthGuard`'ı bir global koruyucu olarak kaydedin (örneğin `AuthModule` içinde herhangi bir modülde):

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: AuthGuard,
  },
],
```

Bu yapılandırmayla, Nest otomatik olarak `AuthGuard`'ı tüm uç noktalara bağlar.

Şimdi rotaları kamuya açık olarak işaretlemek için bir mekanizma sağlamamız gerekiyor. Bunun için `SetMetadata` dekoratör fabrika fonksiyonunu kullanarak özel bir dekoratör oluşturabiliriz.

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

Yukarıdaki dosyada iki sabit ihraç ettik. Birincisi `IS_PUBLIC_KEY` adındaki meta veri anahtarı, diğeri ise yeni dekoratörümüz olan `Public`'ı (isteğe bağlı olarak `SkipAuth` veya `AllowAnon` gibi adlandırabilirsiniz, projenize uyanı kullanabilirsiniz).

Artık özel bir `@Public()` dekoratörümüz olduğuna göre, herhangi bir yöntemi aşağıdaki gibi süsleyebiliriz:

```typescript
@Public()
@Get()
findAll() {
  return [];
}
```

Son olarak, `AuthGuard`'ın `"isPublic"` meta verisi bulunduğunda `true` döndürmesi gerekiyor. Bunun için `Reflector` sınıfını kullanacağız (daha fazlasını [buradan](/docs/guards#putting-it-all-together) okuyabilirsiniz).

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService, private reflector: Reflector) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      // 💡 Bu koşulu gör
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      const payload = await this.jwtService.verifyAsync(token, {
        secret: jwtConstants.secret,
      });
      // 💡 Burada payload'ı istek nesnesine atıyoruz
      // böylece rotalarımızdaki işleyicilerde erişebiliriz
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

#### Passport entegrasyonu

[Passport](https://github.com/jaredhanson/passport), topluluk tarafından iyi bilinen ve birçok üretim uygulamasında başarıyla kullanılan en popüler node.js kimlik doğrulama kütüphanesidir. Bu kütüphaneyi **Nest** uygulamanızla entegre etmek oldukça basittir ve `@nestjs/passport` modülünü kullanarak bunu yapabilirsiniz.

Passport'u NestJS ile nasıl entegre edebileceğinizi öğrenmek için bu [bölüme](/docs/recipes/passport) göz atın.

#### Örnek

Bu bölümdeki kodun tamamlanmış bir sürümünü [buradan](https://github.com/nestjs/nest/tree/master/sample/19-auth-jwt) bulabilirsiniz.
