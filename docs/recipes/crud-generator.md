### CRUD Oluşturucusu (Sadece TypeScript)

Bir projenin ömrü boyunca, yeni özellikler eklediğimizde genellikle uygulamamıza yeni kaynaklar eklememiz gerekebilir. Bu kaynaklar genellikle her seferinde yeni bir kaynağı tanımlarken tekrarlamamız gereken birden çok tekrarlı işlemi gerektirir.

#### Giriş

Gerçek dünya senaryosunu düşünelim, CRUD uç noktalarını açmamız gereken 2 varlık olduğunu varsayalım, diyelim ki **User (Kullanıcı)** ve **Product (Ürün)** varlıkları.
En iyi uygulama uygulamak için, her varlık için şu işlemleri gerçekleştirmemiz gerekecek:

- Kodu düzenli tutmak ve net sınırlar belirlemek (ilgili bileşenleri gruplama) için bir modül oluşturmak (`nest g mo`)
- CRUD rotalarını tanımlamak için bir denetleyici oluşturmak (`nest g co`) (veya GraphQL uygulamaları için sorguları/mutasyonları)
- İş mantığını uygulamak ve izole etmek için bir servis oluşturmak (`nest g s`)
- Kaynak veri şeklini temsil etmek için bir varlık sınıfı/ara yüzü oluşturmak
- Verinin ağ üzerinde nasıl gönderileceğini tanımlamak için Veri Transferi Nesneleri (veya GraphQL uygulamaları için girişler) oluşturmak

Bu birçok adım!

Bu tekrarlı süreci hızlandırmak için, [Nest CLI](/docs/cli/overview) bir jeneratör (şematik) sağlar, bu da tüm bu boilerplate kodunu otomatik olarak oluşturarak bize bu işlemleri yapmaktan kaçınmamıza ve geliştirici deneyimini çok daha basit hale getirmemize yardımcı olur.

> info **Not** Şematik, **HTTP** denetleyicileri, **Mikroservis** denetleyicileri, **GraphQL** çözücüleri (hem kod önce hem de şema önce), ve **WebSocket** Gateway'leri oluşturmayı destekler.

#### Yeni bir kaynak oluşturmak

Yeni bir kaynak oluşturmak için, projenizin kök dizininde şu komutu çalıştırın:

```shell
$ nest g resource
```

`nest g resource` komutu, yalnızca tüm NestJS yapı taşlarını (modül, servis, denetleyici sınıfları) değil, aynı zamanda bir varlık sınıfı, DTO sınıfları ve test (`.spec`) dosyalarını da otomatik olarak oluşturur.

Aşağıda, oluşturulan denetleyici dosyasını görebilirsiniz (REST API için):

```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(+id);
  }

  @Patch(':id')
  update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {
    return this.usersService.update(+id, updateUserDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(+id);
  }
}
```

Ayrıca, tüm CRUD uç noktaları için (REST API için rotalar, GraphQL için sorgular ve mutasyonlar, Mikroservisler ve WebSocket Gateway'leri için mesaj abonelikleri) otomatik olarak yer tutucular da oluşturur - hiçbir şey yapmadan.

> warning **Not** Oluşturulan servis sınıfları, belirli bir **ORM'ye (veya veri kaynağına)** bağlı değildir. Bu, jeneratörü herhangi bir projenin ihtiyaçlarına uygun hale getirmek için yeterince genel yapar. Varsayılan olarak, tüm yöntemler yer tutucular içerecek şekilde tasarlanmıştır ve proj

enize özgü veri kaynakları ile doldurmanıza olanak tanır.

Aynı şekilde, bir GraphQL uygulaması için çözücüler oluşturmak istiyorsanız, taşıma katmanı olarak `GraphQL (kod önce)` (veya `GraphQL (şema önce)`) seçeneğini seçin.

Bu durumda, NestJS, bir REST API denetleyicisi yerine bir çözücü sınıfı oluşturacaktır:

```shell
$ nest g resource users

> ? Hangi taşıma katmanını kullanıyorsunuz? GraphQL (kod önce)
> ? CRUD giriş noktaları oluşturmak ister misiniz? Evet
> CREATE src/users/users.module.ts (224 bytes)
> CREATE src/users/users.resolver.spec.ts (525 bytes)
> CREATE src/users/users.resolver.ts (1109 bytes)
> CREATE src/users/users.service.spec.ts (453 bytes)
> CREATE src/users/users.service.ts (625 bytes)
> CREATE src/users/dto/create-user.input.ts (195 bytes)
> CREATE src/users/dto/update-user.input.ts (281 bytes)
> CREATE src/users/entities/user.entity.ts (187 bytes)
> UPDATE src/app.module.ts (312 bytes)
```

> info **İpucu** Test dosyalarını oluşturmak istemiyorsanız, `--no-spec` bayrağını geçebilirsiniz, şu şekilde: `nest g resource users --no-spec`

Aşağıda, sadece tüm tekrarlı mutasyonları ve sorguların oluşturulmakla kalmayıp, her şeyin birbirine bağlı olduğunu görebiliriz. `UsersService`, `User` Varlığı ve DTO'larımızı kullanıyoruz.

```typescript
import { Resolver, Query, Mutation, Args, Int } from '@nestjs/graphql';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { CreateUserInput } from './dto/create-user.input';
import { UpdateUserInput } from './dto/update-user.input';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly usersService: UsersService) {}

  @Mutation(() => User)
  createUser(@Args('createUserInput') createUserInput: CreateUserInput) {
    return this.usersService.create(createUserInput);
  }

  @Query(() => [User], { name: 'users' })
  findAll() {
    return this.usersService.findAll();
  }

  @Query(() => User, { name: 'user' })
  findOne(@Args('id', { type: () => Int }) id: number) {
    return this.usersService.findOne(id);
  }

  @Mutation(() => User)
  updateUser(@Args('updateUserInput') updateUserInput: UpdateUserInput) {
    return this.usersService.update(updateUserInput.id, updateUserInput);
  }

  @Mutation(() => User)
  removeUser(@Args('id', { type: () => Int }) id: number) {
    return this.usersService.remove(id);
  }
}
```