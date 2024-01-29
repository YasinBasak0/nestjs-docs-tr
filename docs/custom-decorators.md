---
sidebar_position: 4
---

### Özel Rota Dekoratörleri

Nest, **dekoratörler** adı verilen bir dil özelliği etrafında inşa edilmiştir. Dekoratörler, birçok yaygın olarak kullanılan programlama dilinde iyi bilinen bir kavramdır, ancak JavaScript dünyasında hala görece yeni bir konsepttir. Dekoratörlerin nasıl çalıştığını daha iyi anlamak için [bu makaleyi](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841) okumanızı öneririz. İşte basit bir tanım:

<blockquote class="external">
  ES2016 dekoratörü, bir işlevi döndüren ve hedefi, adı ve özellik tanımını argüman olarak alabilen bir ifadedir.
  Uygulamak için dekoratörü bir <code>@</code> karakteri ile öne ekleyerek dekore etmek istediğiniz şeyin en üstüne yerleştirirsiniz.
  Dekoratörler bir sınıf, bir yöntem veya bir özellik için tanımlanabilir.
</blockquote>

#### Param dekoratörleri

Nest, HTTP rota işleyicileri ile birlikte kullanabileceğiniz bir dizi kullanışlı **param dekoratörü** sağlar. Aşağıda, sağlanan dekoratörlerin ve temel Express (veya Fastify) nesnelerinin bir listesi bulunmaktadır

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td>
    </tr>
    <tr>
      <td><code>@Response(), @Res()</code></td>
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
      <td><code>@Param(param?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[param]</code></td>
    </tr>
    <tr>
      <td><code>@Body(param?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[param]</code></td>
    </tr>
    <tr>
      <td><code>@Query(param?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[param]</code></td>
    </tr>
    <tr>
      <td><code>@Headers(param?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[param]</code></td>
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

Ek olarak, kendi **özel dekoratörlerinizi** oluşturabilirsiniz. Bu neden kullanışlıdır?

Node.js dünyasında, genellikle özellikleri **request** nesnesine eklemek yaygın bir uygulamadır. Ardından, her rota işleyicisinde bunları aşağıdaki gibi kullanarak manuel olarak çıkarırsınız:

```typescript
const user = req.user;
```

Kodunuzu daha okunabilir ve şeffaf hale getirmek için `@User()` adında bir dekoratör oluşturabilir ve bunu tüm denetleyicilerinizde yeniden kullanabilirsiniz.

```typescript
@@filename(user.decorator)
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

Daha sonra, gereksinimlerinize uygun yerlerde bunu basitçe kullanabilirsiniz.

```typescript
@@filename()
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}
@@switch
@Get()
@Bind(User())
async findOne(user) {
  console.log(user);
}
```

#### Veri Geçişi

Dekoratörünüzün davranışı bazı koşullara bağlıysa, `data` parametresini kullanarak dekoratörün fabrika işlevine bir argüman iletebilirsiniz. Bu için bir kullanım örneği, örneğin <a href="techniques/authentication#implementing-passport-strategies">kimlik doğrulama katmanı</a> istekleri doğrular ve bir kullanıcı varlığını istek nesnesine ekler. Authenticated bir istek için kullanıcı varlığı şu şekilde görünebilir:

```json
{
  "id": 101,
  "firstName": "Alan",
  "lastName": "Turing",
  "email": "alan@email.com",
  "roles": ["admin"]
}
```

Şimdi, bir özelliğin adını key olarak alan ve ilişkili değeri döndüren bir dekoratör tanımlayalım (eğer varsa veya varlık oluşturulmamışsa `undefined` değerini döndürür).

```typescript
@@filename(user.decorator)
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
@@switch
import { createParamDecorator } from '@nestjs/common';

export const User = createParamDecorator((data, ctx) => {
  const request = ctx.switchToHttp().getRequest();
  const user = request.user;

  return data ? user && user[data] : user;
});
```

Daha sonra, kontrolcüde `@User()` dekoratörü aracılığıyla belirli bir özelliğe nasıl erişeceğinizi gösteren bir örnek:

```typescript
@@filename()
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
@@switch
@Get()
@Bind(User('firstName'))
async findOne(firstName) {
  console.log(`Hello ${firstName}`);
}
```

Bu dekoratörü farklı anahtarlarla kullanarak farklı özelliklere erişmek için kullanabilirsiniz. Eğer `user` nesnesi derin veya karmaşıksa, bu, daha kolay ve okunabilir istek işleyici uygulamaları için kullanışlı olabilir.

> info **Hint** TypeScript kullanıcıları için, `createParamDecorator<T>()` generic bir tip içerir. Bu, açıkça tip güvenliğini zorlamak için kullanılabilir, örneğin `createParamDecorator<string>((data, ctx) => ...)`. Alternatif olarak, fabrika işlevinde bir parametre tipi belirleyebilirsiniz, örneğin `createParamDecorator((data: string, ctx) => ...)`. İkisini de atlarsanız, `data` için tip, `any` olacaktır.

#### Pipe'larla Çalışma

Nest, özel param dekoratörlerini yerleşik olanlarla (`@Body()`, `@Param()` ve `@Query()`) aynı şekilde işler. Bu, boruların özel olarak işaretlenmiş parametreler için de (bizim örneklerimizde `user` argümanı) yürütüldüğü anlamına gelir. Dahası, boruyu doğrudan özel dekoratöre uygulayabilirsiniz:

```typescript
@@filename()
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {
  console.log(user);
}
@@switch
@Get()
@Bind(User(new ValidationPipe({ validateCustomDecorators: true })))
async findOne(user) {
  console.log(user);
}
```

> info **Hint** `validateCustomDecorators` seçeneğinin `true` olarak ayarlanmış olması gerekir. `ValidationPipe`, özel dekoratörlerle işaretlenmiş argümanları varsayılan olarak doğrulamaz.

#### Dekoratör Kompozisyonu

Nest, birden çok dekoratörü birleştirmek için yardımcı bir yöntem sunar. Örneğin, yetkilendirme ile ilgili tüm dekoratörleri tek bir dekoratör içine koymak istiyorsanız, aşağıdaki yapıyı kullanabilirsiniz:

```typescript
@@filename(auth.decorator)
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
@@switch
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

Daha sonra bu özel `@Auth()` dekoratörünü şu şekilde kullanabilirsiniz:

```typescript
@Get('users')
@Auth('admin')
findAllUsers() {}
```

Bu, dört dekoratörü tek bir bildirimle uygulamanın etkisine sahiptir.

> warning **Uyarı** `@nestjs/swagger` paketinde bulunan `@ApiHideProperty()` dekoratörü birleştirilemez (composable) ve `applyDecorators` fonksiyonu ile düzgün çalışmaz.