### Alan Ara Yazılımı (Field Middleware)

> ⚠️ **Uyarı** Bu bölüm yalnızca kod ilk yaklaşımına uygulanır.

Alan Aracı, bir alan çözümlenmeden önce veya sonra herhangi bir kodu çalıştırmanıza olanak tanır. Bir alan aracı işlevi, bir alanın sonucunu dönüştürmek, bir alanın argümanlarını doğrulamak veya hatta alan düzeyinde rolleri kontrol etmek (örneğin, bir aracı işlevi için erişim gerekiyorsa hedef alanına erişim için) için kullanılabilir.

Bir alana birden çok aracı işlevi bağlayabilirsiniz. Bu durumda, bunlar önceki aracı işlevi bir sonrakini çağırmaya karar verdiğinde zincir boyunca ardışık olarak çağrılacaktır. `middleware` dizisindeki araç işlevlerinin sırası önemlidir. İlk çözücü en dış katman, bu nedenle ilk ve sonuncu olarak yürütülecektir (bu, `graphql-middleware` paketindeki gibi). İkinci çözücü ikinci dış katman, bu nedenle ikinci ve ikinci sıradaki olarak yürütülecektir.

#### Başlangıç

Hemen bir alan değeri istemciye gönderilmeden önce günlüğe kaydeden basit bir araç oluşturarak başlayalım:

```typescript
import { FieldMiddleware, MiddlewareContext, NextFn } from '@nestjs/graphql';

const loggerMiddleware: FieldMiddleware = async (
  ctx: MiddlewareContext,
  next: NextFn,
) => {
  const value = await next();
  console.log(value);
  return value;
};
```

> ℹ️ **İpucu** `MiddlewareContext`, genellikle GraphQL çözücü işlevi tarafından normalde alınan argümanlardan oluşan bir nesnedir (`{{ '{' }} source, args, context, info {{ '}' }}`), `NextFn` ise bir sonraki aracı yığınında (bu alana bağlı) veya gerçek alan çözücüsünde çalıştırmanıza olanak tanıyan bir işlevdir.

> ⚠️ **Uyarı** Alan aracı işlevleri, bağımlılıkları enjekte edemez veya Nest'in DI konteynerine erişemez, çünkü çok hafif olmaları ve potansiyel olarak zaman alıcı işlemler gerçekleştirmemeleri (örneğin, veritabanından veri almak gibi) gerektiği için tasarlanmışlardır. Harici hizmetleri çağırmak/veri kaynağından sorgu yapmak istiyorsanız, bunu bir kök sorgu/mutasyon işleyicisine bağlı bir koruma/ara katmanında yapmalısınız ve bu işlemi alan aracısından (spesifik olarak, `MiddlewareContext` nesnesinden) erişebileceğiniz bir `context` nesnesine atamalısınız.

Yukarıdaki örnekte, alan aracında `next()` işlevini (gerçek alan çözücüsünü çalıştırır ve bir alan değeri döndürür) çalıştırırız ve sonra bu değeri terminalimize kaydederiz. Ayrıca, araç işlevinden dönen değer, önceki değeri tamamen geçersiz kılar ve değişiklik yapmak istemiyorsak sadece orijinal değeri döndürmeliyiz.

Bunu yerine getirmek için, aracımızı doğrudan `@Field()` dekoratöründe kaydedebiliriz, örneğin:

```typescript
@ObjectType()
export class Recipe {
  @Field({ middleware: [loggerMiddleware] })
  title: string;
}
```

Artık `Recipe` nesne türünün `title` alanını talep ettiğimizde, orijinal alan değeri konsola kaydedilecek.

> ℹ️ **İpucu** [Uzantılar](/docs/graphql/extensions) özelliğini kullanarak bir alan düzeyinde izin sistemi nasıl uygulayabileceğinizi öğrenmek için [bu bölüme](/docs/graphql/extensions#using-custom-metadata) göz atın.

> ⚠️ **Uyarı** Alan aracı yalnızca `ObjectType` sınıflarına uygulanabilir. Daha fazla ayrıntı için [bu soruna](https://github.com/nestjs/graphql/issues/2446) göz atın.

Yukarıda bahsedildiği gibi, alan aracı işlevi içinde alanın değerini kontrol edebiliriz. Gösterim amaçlı, bir tarifin başlığını büyük harf yapalım (varsa):

```typescript
const value = await next();
return value?.toUpperCase();
```

Bu durumda, her başlık talep edildiğinde otomatik olarak büyük harfe dönüştürülecektir.

Benzer şekilde, bir alan çözücüsüne bağlı bir alan aracısını ( `@ResolveField()` dekoratörü ile işaretlenmiş bir yöntem) şu şekilde bağlayabilirsiniz:

```typescript
@ResolveField(() => String, { middleware: [loggerMiddleware] })
title() {
  return 'Placeholder';
}
```

> ⚠️ **Uyarı** Eğer iyileştiriciler, alan çözücü seviyesinde etkinleştirilmişse ([daha fazlasını oku](/docs/graphql/other-features#execute-enhancers-at-the-field-resolver-level)), alan aracı işlevleri, **yönteme bağlı** (ancak sorgu veya mutasyon işleyicileri için kaydedilen kök düzey iy

ileştiricilerden sonra) başka hiçbir interceptor, koruyucu vb. tarafından çalıştırılmadan önce çalıştırılır. 

#### Global Alan Aracı

Belirli bir alana doğrudan bağlamak yerine, bir veya birden çok aracı işlevini küresel olarak kaydedebilirsiniz. Bu durumda, bunlar otomatik olarak tüm nesne türlerinizin tüm alanlarına bağlanacaktır.

```typescript
GraphQLModule.forRoot({
  autoSchemaFile: 'schema.gql',
  buildSchemaOptions: {
    fieldMiddleware: [loggerMiddleware],
  },
}),
```

> ℹ️ **İpucu** Küresel olarak kaydedilen alan aracı işlevleri, yerel olarak kaydedilenlerden önce çalıştırılır (doğrudan belirli alanlara bağlananlar).