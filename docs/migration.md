---
sidebar_position: 12
---

### Geçiş Kılavuzu

Bu makale, Nest sürüm 9'dan sürüm 10'a geçiş için bir dizi kılavuz sağlar.
Sürüm 10'da eklediğimiz yeni özellikler hakkında daha fazla bilgi edinmek için [bu makaleyi](https://trilon.io/blog/nestjs-10-is-now-available) inceleyebilirsiniz.
Çoğu kullanıcıyı etkilememesi gereken çok küçük bozukluklar oldu - bunların tam listesini [burada](https://github.com/nestjs/nest/releases/tag/v10.0.0) bulabilirsiniz.

#### Paketleri Güncelleme

Paketlerinizi manuel olarak güncelleyebilirsiniz, ancak [ncu (npm check updates)](https://npmjs.com/package/npm-check-updates) kullanımını öneririz.

#### Cache Modülü

`CacheModule` artık `@nestjs/common` paketinden kaldırıldı ve artık bağımsız bir paket olarak kullanılabilir - `@nestjs/cache-manager`. Bu değişiklik, `@nestjs/common` paketinde gereksiz bağımlılıkları önlemek için yapılmıştır. `@nestjs/cache-manager` paketi hakkında daha fazla bilgi edinebilirsiniz [burada](https://docs.nestjs.com/techniques/caching).

#### Depreke Edilmişler

Tüm depreke edilmiş yöntemler ve modüller kaldırıldı.

#### CLI Eklentileri ve TypeScript >= 4.8

NestJS CLI Eklentileri (`@nestjs/swagger` ve `@nestjs/graphql` paketleri için kullanılabilir) artık TypeScript'in >= v4.8 sürümünü gerektirecektir, bu nedenle TypeScript'in eski sürümleri artık desteklenmeyecek. Bu değişikliğin nedeni, [TypeScript v4.8'in Soyut Sözdizimi Ağacında (AST) bir dizi kırıcı değişiklik tanıtmasıdır](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-8.html#decorators-are-placed-on-modifiers-on-typescripts-syntax-trees), ki bu da OpenAPI ve GraphQL şemalarını otomatik olarak oluşturduğumuz AST'yi etkiler.

#### Node.js v12 Desteğinin Düşürülmesi

NestJS 10'dan itibaren Node.js v12'yi desteklemiyoruz, çünkü [v12 30 Nisan 2022'de EOL'a girdi](https://twitter.com/nodejs/status/1524081123579596800). Bu, NestJS 10'un Node.js v16 veya daha yüksek sürümünü gerektirdiği anlamına gelir. Bu karar, TypeScript yapılandırmamızda hedefi nihayetinde `ES2021` olarak ayarlamamıza izin vermesi ve geçmişte yaptığımız gibi polyfill'leri göndermememiz içindi.

Artık, her resmi NestJS paketi varsayılan olarak `ES2021` hedefine derlenecek ve bu, daha küçük bir kütüphane boyutuna ve bazen (hafifçe) daha iyi performansa yol açmalıdır.

Ayrıca, en son LTS sürümünü kullanmanızı kesinlikle öneririz.