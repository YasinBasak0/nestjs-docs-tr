### Serileştirme

Serileştirme, nesnelerin bir ağ yanıtında döndürülmeden önce gerçekleşen bir süreçtir. Bu, verileri istemciye döndürmek için dönüştürme ve temizleme kurallarını sağlamak için uygun bir yerdir. Örneğin, şifreler gibi hassas verilerin yanıta dahil edilmemesi her zaman gereklidir. Veya, bir varlığın özelliklerinin yalnızca bir alt kümesini gönderme gibi belirli özellikler ek dönüşümlere ihtiyaç duyabilir. Bu dönüşümleri manuel olarak yapmak sıkıcı ve hata eğilimli olabilir ve tüm durumların kapsandığından emin olamayabilirsiniz.

#### Genel Bakış

Nest, bu işlemlerin düz bir şekilde gerçekleştirilebilmesini sağlamak için dahili bir yetenek sağlar. `ClassSerializerInterceptor` interceptor, nesneleri dönüştürmek için güçlü [class-transformer](https://github.com/typestack/class-transformer) paketini kullanarak nesneleri dönüştürmenin deklaratif ve genişletilebilir bir yolunu sağlar. Temel işlemi, bir yöntem işleyicisinin döndürdüğü değeri almak ve [class-transformer](https://github.com/typestack/class-transformer) paketinin `instanceToPlain()` işlevini uygulamaktır. Bu şekilde, bir varlık/DTO sınıfındaki `class-transformer` dekoratörleri tarafından ifade edilen kuralları uygulayabilir.

> info **İpucu** Serileştirme, [StreamableFile](https://docs.nestjs.com/techniques/streaming-files#streamable-file-class) yanıtlarına uygulanmaz.

#### Özellikleri Hariç Tutma

Varsayalım ki, bir kullanıcı varlığından `password` özelliğini otomatik olarak hariç tutmak istiyoruz. Varlığı aşağıdaki gibi işaretleyebiliriz:

```typescript
import { Exclude } from 'class-transformer';

export class UserEntity {
  id: number;
  firstName: string;
  lastName: string;

  @Exclude()
  password: string;

  constructor(partial: Partial<UserEntity>) {
    Object.assign(this, partial);
  }
}
```

Şimdi, bu sınıfın bir örneğini döndüren bir yöntem işleyicisine sahip bir denetleyiciyi düşünün.

```typescript
@UseInterceptors(ClassSerializerInterceptor)
@Get()
findOne(): UserEntity {
  return new UserEntity({
    id: 1,
    firstName: 'Kamil',
    lastName: 'Mysliwiec',
    password: 'password',
  });
}
```

> **Uyarı** Bir sınıf örneği döndürmeliyiz. Örneğin, `{{ '{' }} user: new UserEntity() {{ '}' }}` gibi düz bir JavaScript nesnesi döndürürseniz, nesne düzgün bir şekilde serileştirilmeyecektir.

> info **İpucu** `ClassSerializerInterceptor`, `@nestjs/common` tarafından içe aktarılır.

Bu uç nokta talep edildiğinde, istemci aşağıdaki yanıtı alır:

```json
{
  "id": 1,
  "firstName": "Kamil",
  "lastName": "Mysliwiec"
}
```

Dikkat edilmesi gereken önemli bir nokta, interceptor'ün uygulanabileceği bir uygulama genelinde (burada açıklandığı [şekilde](https://docs.nestjs.com/interceptors#binding-interceptors)) uygulanabilmektedir. İnterceptor ve varlık sınıfı bildirimi kombinasyonu, `password` özelliğini herhangi bir `UserEntity` döndüren herhangi bir yöntemin kaldıracağından emin olacaktır. Bu, bu iş kuralının merkezi bir şekilde uygulanmasına olanak tanır.

#### Özellikleri Ortaya Çıkarma

Özelliklere takma adlar sağlamak veya bir özellik değerini hesaplamak için (analog bir **getter** işlevi gibi) `@Expose()` dekoratörünü kullanabilirsiniz.

```typescript
@Expose()
get fullName(): string {
  return `${this.firstName} ${this.lastName}`;
}
```

#### Dönüştürme

`@Transform()` dekoratörünü kullanarak ek veri dönüşümleri gerçekleştirebilirsiniz. Örneğin, aşağıdaki yapı, `RoleEntity`'nin sadece adını değil, tüm nesneyi döndürmez, sadece `RoleEntity`'nin `name` özelliğini döndürür.

```typescript
@Transform(({ value }) => value.name)
role: RoleEntity;
```

#### Seçenekleri Geçme

Dönüşüm işlevlerinin varsayılan davranışını değiştirmek isteyebilirsiniz. Varsayılan ayarları geçersiz kılmak için, `@SerializeOptions()` dekoratörü ile bir `options` nesnesini geçirin.

```typescript
@SerializeOptions({
  excludePrefixes: ['_'],
})
@Get()
findOne(): UserEntity {
  return new UserEntity();
}
```

> info **İpucu** `@SerializeOptions()` dekoratörü, `@nestjs/common` tarafından içe aktarılır.

`@SerializeOptions()` aracılığıyla geçirilen seçenekler, temel `instanceToPlain()` işlevinin ikinci argümanı olarak iletilir. Bu örnekte, `_` öneki ile başlayan tüm özellikleri otomatik olarak hariç tutuyoruz.

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/21-serializer) bulunmaktadır.

#### WebSockets ve Microservices

Bu bölüm, HTTP tarzı uygulamaları (örneğin, Express veya Fastify) kullanan örnekler gösterse de,

 `ClassSerializerInterceptor`, kullanılan taşıma yönteminden bağımsız olarak WebSockets ve Microservices için aynı şekilde çalışır.

#### Daha Fazla Bilgi

`class-transformer` paketi tarafından sağlanan mevcut dekoratörler ve seçenekler hakkında daha fazla bilgi için [buraya](https://github.com/typestack/class-transformer) bakın.