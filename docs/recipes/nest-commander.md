### Nest Commander

[Standalone uygulama](/docs/application-context) dokümantasyonlarına ek olarak, tipik bir Nest uygulamasına benzer bir yapıda komut satırı uygulamaları yazmak için [nest-commander](https://jmcdo29.github.io/nest-commander) paketi bulunmaktadır.

> info **info** `nest-commander` üçüncü taraf bir pakettir ve tamamen NestJS çekirdek ekibi tarafından yönetilmez. Lütfen, kütüphane ile ilgili bulduğunuz herhangi bir sorunu [uygun depoya](https://github.com/jmcdo29/nest-commander/issues/new/choose) bildirin.

#### Kurulum

Herhangi bir paket gibi, kullanmadan önce bunu yüklemeniz gerekmektedir.

```bash
$ npm i nest-commander
```

#### Bir Komut Dosyası

`nest-commander`, sınıflar için `@Command()` dekoratörü ve bu sınıfların yöntemleri için `@Option()` dekoratörü aracılığıyla [dekoratörler](https://www.typescriptlang.org/docs/handbook/decorators.html) kullanarak yeni komut satırı uygulamaları yazmayı kolaylaştırır. Her komut dosyasının `CommandRunner` soyut sınıfını uygulaması ve bir `@Command()` dekoratörü ile süslenmiş olması gerekmektedir.

Her komut, Nest tarafından bir `@Injectable()` olarak görülür, bu nedenle normal Bağımlılık Enjeksiyonunuz beklediğiniz gibi çalışır. Tek dikkat edilmesi gereken şey, her komutun bir `run` yöntemine sahip olmasıdır. Bu yöntem, `Promise<void>` döndürmeli ve `string[], Record<string, any>` parametrelerini almalıdır. `run` komutu, seçenek bayrakları ile eşleşmeyen parametreleri dizi olarak alacak şekilde tasarlanmıştır, eğer gerçekten birden çok parametre ile çalışmak istiyorsanız. Seçenekler için, `Record<string, any>`, bu özelliklerin adları, `@Option()` dekoratörlerine verilen `name` özelliğiyle eşleşir, değerleri ise seçeneği işleyenin dönüş değeriyle eşleşir. Daha iyi bir tip güvenliği istiyorsanız, seçenekleriniz için bir arayüz oluşturabilirsiniz.

#### Komutu Çalıştırma

NestJS uygulamasında bir sunucu oluşturmak ve `listen` ile çalıştırmak için `NestFactory`'yi kullanabiliyorsak, `nest-commander` paketi de sunucunuzu çalıştırmak için kullanımı kolay bir API sunar. `CommandFactory`'i içe aktarın ve `static` yöntemi olan `run`'ı kullanın ve uygulamanızın kök modülünü geçirin. Bu muhtemelen aşağıdaki gibi görünecektir:

```ts
import { CommandFactory } from 'nest-commander';
import { AppModule } from './app.module';

async function bootstrap() {
  await CommandFactory.run(AppModule);
}

bootstrap();
```

Varsayılan olarak, `CommandFactory` kullanıldığında Nest'in günlük tutucusu devre dışı bırakılmıştır. Bununla birlikte, `run` işlevine ikinci bir argüman olarak sağlayabilirsiniz. Burada ya özel bir NestJS günlük tutucusu ya da tutmak istediğiniz günlük seviyelerinin bir dizisi verilebilir - yalnızca Nest'in hata günlüklerini yazdırmak istiyorsanız burada `['error']` sağlamak yararlı olabilir.

```ts
import { CommandFactory } from 'nest-commander';
import { AppModule } from './app.module';
import { LogService } './log.service';

async function bootstrap() {
  await CommandFactory.run(AppModule, new LogService());

  // veya, yalnızca Nest'in uyarılarını ve hatalarını yazdırmak istiyorsanız
  await CommandFactory.run(AppModule, ['warn', 'error']);
}

bootstrap();
```

Ve bu kadar. `CommandFactory` arka planda sizin için `NestFactory`'yi çağırmayla ve gerekli olduğunda `app.close()`'ı çağırmayla ilgilenecek, bu nedenle orada bellek sızıntıları hakkında endişelenmenize gerek yok. Biraz hata işleme eklemeniz gerekiyorsa, her zaman `try/catch`'i `run` komutunu sarmalayabilir veya `bootstrap()` çağrısına bir `.catch()` yöntemi ekleyebilirsiniz.

#### Test

Eğer süper harika bir komut satırı betiği yazıyorsanız, bunu süper kolay bir şekilde test edemezseniz ne işe yarar değil mi? Neyse ki, `nest-commander`'ın NestJS ekosistemiyle mükemmel bir uyum sağlayan bazı yardımcı programları vardır. Test modunda komutu oluşturmak için `CommandFactory` yerine `CommandTestFactory`'yi kullanabilir ve metadata'nızı çok benzer bir şekilde `@nestjs/testing`'den gelen `Test.createTestingModule` gibi geçirebilirsiniz. Aslında, bu paketi içsel olarak kullanır. Yine de testte DI parçalarınızı doğrudan testte değiştirmek için `compile()`'ı çağırmadan önce `overrideProvider` yöntemlerini zincirleyebilirsiniz.

#### Hepsi bir arada

Aşağıdaki sınıf, CLI komutu alabilen veya doğrudan çağrılabilen bir CLI komutunu temsil eder, `-n`, `-s` ve `-b` (ve uzun bayraklarıyla birlikte) tüm bu seçenekleri destekler ve her seçenek için özel ayrıştırıcılarla birlikte gelir. `--help` bayrağı da, genellikle commander ile uyumlu olarak, desteklenir.

```ts
import { Command, CommandRunner, Option } from 'nest-commander';
import { LogService } from './log.service';

interface BasicCommandOptions {
  string?: string;
  boolean?: boolean;
  number?: number;
}

@Command({ name: 'basic', description: 'Parametre ayrıştırma' })
export class BasicCommand extends CommandRunner {
  constructor(private readonly logService: LogService) {
    super()
  }

  async run(
    passedParam: string[],
    options?: BasicCommandOptions,
  ): Promise<void> {
    if (options?.boolean !== undefined && options?.boolean !== null) {
      this.runWithBoolean(passedParam, options.boolean);
    } else if (options?.number) {
      this.runWithNumber(passedParam, options.number);
    } else if (options?.string) {
      this.runWithString(passedParam, options.string);
    } else {
      this.runWithNone(passedParam);
    }
  }

  @Option({
    flags: '-n, --number [number]',
    description: 'Temel bir sayı ayrıştırıcı',
  })
  parseNumber(val: string): number {
    return Number(val);
  }

  @Option({
    flags: '-s, --string [string]',
    description: 'Bir dize dönüş',
  })
  parseString(val: string): string {
    return val;
  }

  @Option({
    flags: '-b, --boolean [boolean]',
    description: 'Bir boolean ayrıştırıcı',
  })
  parseBoolean(val: string): boolean {
    return JSON.parse(val);
  }

  runWithString(param: string[], option: string): void {
    this.logService.log({ param, string: option });
  }

  runWithNumber(param: string[], option: number): void {
    this.logService.log({ param, number: option });
  }

  runWithBoolean(param: string[], option: boolean): void {
    this.logService.log({ param, boolean: option });
  }

  runWithNone(param: string[]): void {
    this.logService.log({ param });
  }
}
```

Komut sınıfının bir modüle eklenmesini sağlayın

```ts
@Module({
  providers: [LogService, BasicCommand],
})
export class AppModule {}
```

Ve şimdi CLI'yi main.ts dosyanızda çalıştırabilmek için aşağıdaki adımları yapabilirsiniz

```ts
async function bootstrap() {
  await CommandFactory.run(AppModule);
}

bootstrap();
```

Ve işte bu kadar, bir komut satırı uygulamanız var.

#### Daha Fazla Bilgi

Daha fazla bilgi, örnekler ve API belgeleri için [nest-commander belge sitesini](https://jmcdo29.github.io/nest-commander) ziyaret edin.