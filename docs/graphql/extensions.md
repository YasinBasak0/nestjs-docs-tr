### Uzantılar

> uyarı **Uyarı** Bu bölüm sadece kod tabanlı yaklaşıma uygundur.

Uzantılar, tiplerin konfigürasyonunda keyfi veriler tanımlamanıza izin veren **gelişmiş, düşük seviyeli bir özelliktir**. Belirli alanlara özel metadata eklemek, daha karmaşık, genel çözümler oluşturmanıza olanak tanır. Örneğin, uzantılarla belirli alanlara erişim sağlamak için gereken alan düzeyinde rolleri tanımlayabilirsiniz. Bu tür roller, çağrıcının belirli bir alanı almak için yeterli izne sahip olup olmadığını belirlemek için çalışma zamanında yansıtılabilir.

#### Özel metadata eklemek

Bir alan için özel metadata eklemek için, `@nestjs/graphql` paketinden ihraç edilen `@Extensions()` dekoratörünü kullanın.

```typescript
@Field()
@Extensions({ role: Role.ADMIN })
password: string;
```

Yukarıdaki örnekte, `role` metadata özelliğine `Role.ADMIN` değerini atadık. `Role`, sistemimizde bulunan tüm kullanıcı rollerini gruplayan basit bir TypeScript enum'dir.

Not olarak, alanlara metadata eklemenin yanı sıra, `@Extensions()` dekoratörünü sınıf düzeyinde ve metod düzeyinde (örneğin, sorgu işleyicisinde) kullanabilirsiniz.

#### Özel metadata kullanma

Özel metadata'yı kullanan mantık, ihtiyaca göre karmaşık olabilir. Örneğin, bir metod çağrısı başına olayları depolayan/kaydeden basit bir interceptor oluşturabilir veya bir [alan ara yazılımı](/docs/graphql/field-middleware) oluşturabilirsiniz ki bu, bir alanı almak için gereken rolleri, çağrıcının izinleriyle eşleştirir (alan düzeyinde izin sistemi).

Örnek olması açısından, kullanıcının rolünü (burada sabitlenmiş) bir hedef alanına erişmek için gereken rolle karşılaştıran bir `checkRoleMiddleware` tanımlayalım:

```typescript
export const checkRoleMiddleware: FieldMiddleware = async (
  ctx: MiddlewareContext,
  next: NextFn,
) => {
  const { info } = ctx;
  const { extensions } = info.parentType.getFields()[info.fieldName];

  /**
   * Gerçek bir uygulamada, "userRole" değişkeni
   * çağrıcının (kullanıcının) rolünü temsil etmelidir (örneğin, "ctx.user.role").
   */
  const userRole = Role.USER;
  if (userRole === extensions.role) {
    // veya sadece "return null" diyerek yok sayabilirsiniz
    throw new ForbiddenException(
      `Kullanıcının "${info.fieldName}" alanına erişim izinleri yok.`,
    );
  }
  return next();
};
```

Bu konumda, `password` alanı için bir ara yazılım kaydedebiliriz, aşağıdaki gibi:

```typescript
@Field({ middleware: [checkRoleMiddleware] })
@Extensions({ role: Role.ADMIN })
password: string;
```
