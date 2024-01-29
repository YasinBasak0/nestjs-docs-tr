### Prisma

[Prisma](https://www.prisma.io), Node.js ve TypeScript için [açık kaynak](https://github.com/prisma/prisma) bir ORM'dir. Düz SQL yazmak veya SQL sorgu oluşturucuları (örneğin [knex.js](https://knexjs.org/)) veya ORM'ler (örneğin [TypeORM](https://typeorm.io/) ve [Sequelize](https://sequelize.org/)) gibi başka bir veritabanı erişim aracı kullanmak yerine bir **alternatif** olarak kullanılır. Prisma şu anda PostgreSQL, MySQL, SQL Server, SQLite, MongoDB ve CockroachDB'yi ([Önizleme](https://www.prisma.io/docs/reference/database-reference/supported-databases)) desteklemektedir.

Prisma, düz JavaScript ile kullanılabileceği gibi, TypeScript'i benimser ve TypeScript ekosistemindeki diğer ORM'lerin sağladığı garanti düzeyinin ötesinde bir tür güvenliği sağlar. Prisma ve TypeORM'un tür güvenliği garantilerinin derinlemesine karşılaştırmasını [buradan](https://www.prisma.io/docs/concepts/more/comparisons/prisma-and-typeorm#type-safety) bulabilirsiniz.

> info **Not** Prisma'nın nasıl çalıştığına dair hızlı bir genel bakış almak istiyorsanız, [Hızlı Başlangıç](https://www.prisma.io/docs/getting-started/quickstart)'ı takip edebilir veya [belgelerdeki](https://www.prisma.io/docs/) [Giriş](https://www.prisma.io/docs/understand-prisma/introduction)'i okuyabilirsiniz. Ayrıca [`prisma-examples`](https://github.com/prisma/prisma-examples/) deposunda [REST](https://github.com/prisma/prisma-examples/tree/latest/typescript/rest-nestjs) ve [GraphQL](https://github.com/prisma/prisma-examples/tree/latest/typescript/graphql-nestjs) için hazır çalışan örnekler de bulunmaktadır.

#### Başlangıç

Bu reçete kapsamında, NestJS ve Prisma ile sıfırdan nasıl başlanacağınızı öğreneceksiniz. Bir örnek NestJS uygulaması oluşturacaksınız ve bu uygulama, bir veritabanında veri okuyup yazabilen bir REST API'ye sahip olacak.

Bu kılavuzun amacı için, bir [SQLite](https://sqlite.org/) veritabanı kullanarak bir veritabanı sunucusu kurma gerekliliğinden kaçınacaksınız. PostgreSQL veya MySQL kullanıyorsanız bile, bu kılavuzu takip edebilirsiniz - ilgili yerlerde bu veritabanlarını kullanmak için ek talimatlar alacaksınız.

> info **Not** Eğer zaten var olan bir projeniz varsa ve Prisma'ya geçmeyi düşünüyorsanız, [varolan bir projeye Prisma ekleme](https://www.prisma.io/docs/getting-started/setup-prisma/add-to-existing-project-typescript-postgres) rehberini takip edebilirsiniz. TypeORM'den geçiyorsanız, [TypeORM'den Prisma'ya Geçiş](https://www.prisma.io/docs/guides/migrate-to-prisma/migrate-from-typeorm) rehberini okuyabilirsiniz.

#### NestJS Projenizi Oluşturun

Başlamak için, NestJS CLI'yi yükleyin ve aşağıdaki komutlarla uygulama iskeletinizi oluşturun:

```bash
$ npm install -g @nestjs/cli
$ nest new hello-prisma
```

Bu komut tarafından oluşturulan proje dosyaları hakkında daha fazla bilgi için [İlk Adımlar](https://docs.nestjs.com/first-steps) sayfasına göz atın. Ayrıca, artık `npm start` komutunu kullanarak uygulamanızı başlatabilirsiniz. `http://localhost:3000/` adresinde çalışan REST API şu anda `src/app.controller.ts` dosyasında uygulanan tek bir rotayı sunmaktadır. Bu kılavuzun ilerleyen bölümlerinde, _kullanıcılar_ ve _gönderiler_ hakkında veri depolamak ve almak için ek rotalar uygulayacaksınız.

#### Prisma'yı Kurun

Prisma CLI'yi projenize geliştirme bağımlılığı olarak yükleyerek başlayın:

```bash
$ cd hello-prisma
$ npm install prisma --save-dev
```

Aşağıdaki adımlarda, [Prisma CLI](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-cli)'sini kullanacağız. İyi bir uygulama pratiği olarak, CLI'yi yerel olarak çağırmak için genellikle `npx` ile öneklemeniz önerilir:

```bash
$ npx prisma
```

<details>
<summary>Yarn kullanıyorsanız genişletin</summary>

Eğer Yarn kullanıyorsanız, Prisma CLI'yi aşağıdaki gibi yükleyebilirsiniz:

```bash
$ yarn add prisma --dev
```

Yüklendikten sonra, `yarn` ile önekleyerek çağırabilirsiniz:

```bash
$ yarn prisma
```

</details>

Şimdi, Prisma CLI'nin `init` komutunu kullanarak başlangıç Prisma kurulumunuzu oluşturun:

```bash
$ npx prisma init
```

Bu komut, aşağıdaki içeriğe sahip yeni bir `prisma` dizini oluşturur:

- `schema.prisma`: Veritabanı bağlantınızı belirtir ve veritabanı şemasını içerir
- `.env`: [dotenv](https://github.com/motdotla/dotenv) dosyası, genellikle veritabanı kimlik bilgilerinizi bir dizi ortam değişkeninde saklamak için kullanılır

#### Veritabanı Bağlantısını Ayarlayın

Veritabanı bağlantınızı `schema.prisma` dosyasındaki `datasource` bloğunda yapılandırırsınız. Varsayılan olarak `postgresql` olarak ayarlanmıştır, ancak bu kılavuzu SQLite veritabanıyla kullanıyorsanız `datasource` bloğunun `provider` alanını `sqlite` olarak ayarlamalısınız:

```groovy
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

Şimdi, `.env` dosyasını açın ve `DATABASE_URL` ortam değişkenini aşağıdaki gibi ayarlayın:

```bash
DATABASE_URL="file:./dev.db"
```

`DATABASE_URL` değişkeninin `.env` dosyasından alınmaması durumunda [ConfigModule](https://docs.nestjs.com/techniques/configuration) yapılandırılmış olduğundan emin olun.

SQLite veritabanları basit dosyalardır; SQLite veritabanını kullanmak için bir sunucuya ihtiyaç duyulmaz. Bu nedenle _host_ ve _port_ ile bir bağlantı URL'si yapılandırmak yerine, bunu bu durumda `dev.db` olarak adlandırılmış bir yerel dosyaya yönlendirebilirsiniz. Bu dosya bir sonraki adımda oluşturulacaktır.

<details>
<summary>PostgreSQL veya MySQL kullanıyorsanız genişletin</summary>

PostgreSQL ve MySQL ile, bağlantı URL'sini _veritabanı sunucusunu_ göstermesi için yapılandırmalısınız. Gerekli bağlantı URL formatı hakkında daha fazla bilgiyi [buradan](https://www.prisma.io/docs/reference/database-reference/connection-urls) öğrenebilirsiniz.

**PostgreSQL**

Eğer PostgreSQL kullanıyorsanız, `schema.prisma` ve `.env` dosyalarını aşağıdaki gibi ayarlamanız gerekir:

**`schema.prisma`**

```groovy
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

**`.env`**

```bash
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=SCHEMA"
```

Büyük harflerle yazılmış tüm yer tutucularını, veritabanı kimlik bilgilerinizle değiştirin. `SCHEMA` yer tutucusu için ne sağlayacağınız konusunda emin değilseniz, muhtemelen varsayılan değer olan `public` değeridir:

```bash
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=public"
```

PostgreSQL veritabanı nasıl kurulur öğrenmek istiyorsanız, [Heroku'da ücretsiz PostgreSQL veritabanı kurma](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1) rehberini takip edebilirsiniz.

**MySQL**

Eğer MySQL kullanıyorsanız, `schema.prisma` ve `.env` dosyalarını aşağıdaki gibi ayarlamanız gerekir:

**`schema.prisma`**

```groovy
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

**`.env`**

```bash
DATABASE_URL="mysql://USER:PASSWORD@HOST:PORT/DATABASE"
```

Büyük harflerle yazılmış tüm yer tutucularını, veritabanı kimlik bilgilerinizle değiştirin.

</details>

#### Prisma Migrate ile İki Veritabanı Tablosu Oluşturun

Bu bölümde, [Prisma Migrate](https://www.prisma.io/docs/concepts/components/prisma-migrate) kullanarak veritabanınızda iki yeni tablo oluşturacaksınız. Prisma Migrate, Prisma şemasındaki bildirimsel veri modeli tanımınıza göre SQL migrasyon dosyalarını oluşturur. Bu migrasyon dosyaları tamamen özelleştirilebilir, böylece altındaki veritabanının herhangi bir ek özelliğini yapılandırabilir veya tohumlama gibi ek komutları içerebilirsiniz.

`schema.prisma` dosyanıza şu iki modeli ekleyin:

```groovy
model User {
  id    Int     @default(autoincrement()) @id
  email String  @unique
  name  String?
  posts Post[]
}

model Post {
  id        Int      @default(autoincrement()) @id
  title     String
  content   String?
  published Boolean? @default(false)
  author    User?    @relation(fields: [authorId], references: [id])
  authorId  Int?
}
```

Prisma modelleriniz hazır olduğunda, SQL migrasyon dosyalarınızı oluşturabilir ve veritabanına karşı çalıştırabilirsiniz. Terminalinizde şu komutları çalıştırın:

```bash
$ npx prisma migrate dev --name init
```

Bu `prisma migrate dev` komutu SQL dosyalarını oluşturur ve bunları doğrudan veritabanına çalıştırır. Bu durumda, mevcut `prisma` dizininde aşağıdaki migrasyon dosyası oluşturuldu:

```bash
$ tree prisma
prisma
├── dev.db
├── migrations
│   └── 20201207100915_init
│       └── migration.sql
└── schema.prisma
```

<details>
<summary>Oluşturulan SQL ifadelerini görmek için genişlet</summary>

Aşağıdaki tablolar SQLite veritabanınızda oluşturuldu:

```sql
-- CreateTable
CREATE TABLE "User" (
    "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    "email" TEXT NOT NULL,
    "name" TEXT
);

-- CreateTable
CREATE TABLE "Post" (
    "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    "title" TEXT NOT NULL,
    "content" TEXT,
    "published" BOOLEAN DEFAULT false,
    "authorId" INTEGER,

    FOREIGN KEY ("authorId") REFERENCES "User"("id") ON DELETE SET NULL ON UPDATE CASCADE
);

-- CreateIndex
CREATE UNIQUE INDEX "User.email_unique" ON "User"("email");
```

</details>

#### Prisma Client Kurulumu ve Oluşturulması

Prisma Client, Prisma model tanımınızdan _üretilen_ ve tip güvenli bir veritabanı istemcisidir. Bu yaklaşım sayesinde, Prisma Client, modellerinize özel olarak uyarlanmış [CRUD](https://www.prisma.io/docs/concepts/components/prisma-client/crud) işlemlerini ortaya koyabilir.

Projenize Prisma Client'ı kurmak için terminalinizde şu komutu çalıştırın:

```bash
$ npm install @prisma/client
```

Kurulum sırasında Prisma, otomatik olarak sizin için `prisma generate` komutunu çağırır. Gelecekte Prisma modellerinizde _herhangi bir_ değişiklik yaptıktan sonra bu komutu çalıştırmanız gerekecektir.

> info **Not** `prisma generate` komutu, Prisma şemanızı okur ve oluşturulan Prisma Client kütüphanesini `node_modules/@prisma/client` içinde günceller.

NestJS hizmetlerinizde Prisma Client'ı kullanmak için önceki bölümlerde oluşturduğunuz hizmetlere soyutlama eklemek isteyeceksiniz. Başlamak için, `PrismaClient`'ı örneklemek ve veritabanınıza bağlanma işlemini yönetecek yeni bir `PrismaService` oluşturabilirsiniz.

`src` dizini içinde, `prisma.service.ts` adında yeni bir dosya oluşturun ve aşağıdaki kodu ekleyin:

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }
}
```

> info **Not** `onModuleInit` isteğe bağlıdır - eğer çıkartırsanız, Prisma, ilk veritabanı çağrısında bağlanacaktır.

Şimdi, Prisma Client ile veritabanı sorgularını göndermeye yetenekli hale geldiniz. Prisma Client ile sorgu oluşturma hakkında daha fazla bilgi edinmek istiyorsanız, [API belgelerine](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-client/crud) göz atabilirsiniz.

NestJS uygulamanızı kurarken, Prisma Client API'sini Prisma şemalarınız için veritabanı sorgularını yapacak bir hizmet içinde soyutlamak isteyeceksiniz. Başlamak için, `PrismaService`'i oluşturun ve `PrismaClient`'ı örneklemenin ve veritabanına bağlanmanın sorumluluğunu üstlenmesini sağlayın.

`src` dizini içinde, `user.service.ts` adında yeni bir dosya oluşturun ve aşağıdaki kodu ekleyin:

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { User, Prisma } from '@prisma/client';

@Injectable()
export class UserService {
  constructor(private prisma: PrismaService) {}

  async user(
    userWhereUniqueInput: Prisma.UserWhereUniqueInput,
  ): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: userWhereUniqueInput,
    });
  }

  async users(params: {
    skip?: number;
    take?: number;
    cursor?: Prisma.UserWhereUniqueInput;
    where?: Prisma.UserWhereInput;
    orderBy?: Prisma.UserOrderByWithRelationInput;
  }): Promise<User[]> {
    const { skip, take, cursor, where, orderBy } = params;
    return this.prisma.user.findMany({
      skip,
      take,
      cursor,
      where,
      orderBy,
    });
  }

  async createUser(data: Prisma.UserCreateInput): Promise<User> {
    return this.prisma.user.create({
      data,
    });
  }

  async updateUser(params: {
    where: Prisma.UserWhereUniqueInput;
    data: Prisma.UserUpdateInput;
  }): Promise<User> {
    const { where, data } = params;
    return this.prisma.user.update({
      data,
      where,
    });
  }

  async deleteUser(where: Prisma.UserWhereUniqueInput): Promise<User> {
    return this.prisma.user.delete({
      where,
    });
  }
}
```

Dikkat edin ki, Prisma Client'ın oluşturduğu tipleri kullanarak servisiniz tarafından sunulan yöntemlerin doğru şekilde tipize edildiğinden emin oluyorsunuz. Bu şekilde, modellerinizi yazmak ve ek arayüz veya DTO dosyaları oluşturmak için gerekli olan boilerplate işlemlerden kurtulursunuz.

Şimdi, aynısını `Post` modeli için yapın.

Hâlâ `src` dizini içinde, `post.service.ts` adında yeni bir dosya oluşturun ve aşağıdaki kodu ekleyin:

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { Post, Prisma } from '@prisma/client';

@Injectable()
export class PostService {
  constructor(private prisma: PrismaService) {}

  async post(
    postWhereUniqueInput: Prisma.PostWhereUniqueInput,
  ): Promise<Post | null> {
    return this.prisma.post.findUnique({
      where: postWhereUniqueInput,
    });
  }

  async posts(params: {
    skip?: number;
    take?: number;
    cursor?: Prisma.PostWhereUniqueInput;
    where?: Prisma.PostWhereInput;
    orderBy?: Prisma.PostOrderByWithRelationInput;
  }): Promise<Post[]> {
    const { skip, take, cursor, where, orderBy } = params;
    return this.prisma.post.findMany({
      skip,
      take,
      cursor,
      where,
      orderBy,
    });
  }

  async createPost(data: Prisma.PostCreateInput): Promise<Post> {
    return this.prisma.post.create({
      data,
    });
  }

  async updatePost(params: {
    where: Prisma.PostWhereUnique

Input;
    data: Prisma.PostUpdateInput;
  }): Promise<Post> {
    const { data, where } = params;
    return this.prisma.post.update({
      data,
      where,
    });
  }

  async deletePost(where: Prisma.PostWhereUniqueInput): Promise<Post> {
    return this.prisma.post.delete({
      where,
    });
  }
}
```

`UserService` ve `PostService`'iniz şu anda Prisma Client tarafından sunulan CRUD sorgularını saran metodları içeriyor. Gerçek bir uygulamada, servis aynı zamanda uygulamanıza iş mantığı eklemek için de yer olacaktır. Örneğin, `UserService` içinde kullanıcının şifresini güncelleme sorumluluğu olan bir `updatePassword` yöntemine sahip olabilirsiniz.

##### Ana uygulama denetleyicisinde REST API rotalarını uygula

Son olarak, önceki bölümlerde oluşturduğunuz hizmetleri uygulamanın farklı rotalarını uygulamak için kullanacaksınız. Bu rehberin amacıyla, rotalarınızı zaten mevcut olan `AppController` sınıfına yerleştireceksiniz.

`app.controller.ts` dosyasının içeriğini aşağıdaki kod ile değiştirin:

```typescript
import {
  Controller,
  Get,
  Param,
  Post,
  Body,
  Put,
  Delete,
} from '@nestjs/common';
import { UserService } from './user.service';
import { PostService } from './post.service';
import { User as UserModel, Post as PostModel } from '@prisma/client';

@Controller()
export class AppController {
  constructor(
    private readonly userService: UserService,
    private readonly postService: PostService,
  ) {}

  @Get('post/:id')
  async getPostById(@Param('id') id: string): Promise<PostModel> {
    return this.postService.post({ id: Number(id) });
  }

  @Get('feed')
  async getPublishedPosts(): Promise<PostModel[]> {
    return this.postService.posts({
      where: { published: true },
    });
  }

  @Get('filtered-posts/:searchString')
  async getFilteredPosts(
    @Param('searchString') searchString: string,
  ): Promise<PostModel[]> {
    return this.postService.posts({
      where: {
        OR: [
          {
            title: { contains: searchString },
          },
          {
            content: { contains: searchString },
          },
        ],
      },
    });
  }

  @Post('post')
  async createDraft(
    @Body() postData: { title: string; content?: string; authorEmail: string },
  ): Promise<PostModel> {
    const { title, content, authorEmail } = postData;
    return this.postService.createPost({
      title,
      content,
      author: {
        connect: { email: authorEmail },
      },
    });
  }

  @Post('user')
  async signupUser(
    @Body() userData: { name?: string; email: string },
  ): Promise<UserModel> {
    return this.userService.createUser(userData);
  }

  @Put('publish/:id')
  async publishPost(@Param('id') id: string): Promise<PostModel> {
    return this.postService.updatePost({
      where: { id: Number(id) },
      data: { published: true },
    });
  }

  @Delete('post/:id')
  async deletePost(@Param('id') id: string): Promise<PostModel> {
    return this.postService.deletePost({ id: Number(id) });
  }
}
```

Bu denetleyici aşağıdaki rotaları uygular:

###### `GET`

- `/post/:id`: `id`'ye göre tek bir gönderiyi getirir.
- `/feed`: Tüm _yayınlanmış_ gönderileri getirir.
- `/filtered-posts/:searchString`: `title` veya `content`'e göre gönderileri filtreler.

###### `POST`

- `/post`: Yeni bir gönderi oluşturur
  - Gövde:
    - `title: String` (zorunlu): Gönderinin başlığı
    - `content: String` (isteğe bağlı): Gönderinin içeriği
    - `authorEmail: String` (zorunlu): Gönderiyi oluşturan kullanıcının e-posta adresi
- `/user`: Yeni bir kullanıcı oluşturur
  - Gövde:
    - `email: String` (zorunlu): Kullanıcının e-posta adresi
    - `name: String` (isteğe bağlı): Kullanıcının adı

###### `PUT`

- `/publish/:id`: `id`'ye göre bir gönderiyi yayınlar

###### `DELETE`

- `/post/:id`: `id`'ye göre bir gönderiyi siler

#### Özet

Bu reçetede, Prisma'yı NestJS ile birlikte kullanarak bir REST API uygulaması nasıl geliştireceğinizi öğrendiniz. API'nin rotalarını uygulayan denetleyici, gelen isteklerin veri ihtiyaçlarını karşılamak için Prisma Client'ı kullanan bir `PrismaService`'i çağırıyor.

NestJS'yi Prisma ile kullanma konusunda daha fazla bilgi edinmek istiyorsanız, aşağıdaki kaynakları inceleyebilirsiniz:

- [NestJS & Prisma](https://www.prisma.io/nestjs)
- [REST ve GraphQL için Hazır Çalışan Örnek Projeler](https://github.com/prisma/prisma-examples/)
- [Üretim İçin Hazır Başlangıç Kiti](https://github.com/notiz-dev/nestjs-prisma-starter#instructions)
- [Video: NestJS ile Prisma'yı Kullanarak Veritabanlarına Erişim (5dk)](https://www.youtube.com/watch?v=UlVJ340UEuk&ab_channel=Prisma) - [Marc Stammerjohann](https://github.com/marcjulian) tarafından