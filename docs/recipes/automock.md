### Automock

`Automock`, birim testleri için tasarlanmış güçlü, bağımsız bir kütüphanedir. Dahili olarak TypeScript Reflection API'yi kullanarak, sınıfların harici bağımlılıklarını otomatik olarak taklit ederek test sürecini basitleştirir. `Automock`, test geliştirmeyi optimize etmenizi sağlar ve güçlü, verimli birim testleri yazmaya odaklanmanıza yardımcı olur.

> info **info** `Automock`, üçüncü taraf bir pakettir ve NestJS çekirdek ekibi tarafından yönetilmez.
> Lütfen kütüphane ile ilgili bulduğunuz herhangi bir sorunu [ilgili depoda](https://github.com/automock/automock) bildirin.

#### Giriş

Bağımlılık Enjeksiyonu (DI) konteyneri, Nest modül sisteminin temel bir öğesidir ve hem uygulama çalışma zamanında hem de test aşamalarında önemlidir. Birim testlerde, özellikle belirli bileşenlerin davranışlarını izole etmek ve değerlendirmek için sahte bağımlılıklar gereklidir. Ancak, bu sahte nesnelerin manuel olarak yapılandırılması ve yönetilmesi karmaşık olabilir ve hatalara duyarlı olabilir.

`Automock`, bu soruna basit bir çözüm sunar. Gerçek Nest DI konteyneriyle etkileşimde bulunmak yerine, `Automock`, bağımlılıkların otomatik olarak sahte yapılır olduğu sanal bir konteyner tanıtır. Bu yaklaşım, DI konteynerindeki her bir sağlayıcıyı sahte uygulamalarla değiştirmenin manuel görevini atlar. `Automock` ile, tüm bağımlılıklar için sahte nesnelerin oluşturulması otomatikleştirilmiş olup, birim testi kurulum sürecini basitleştirir.

#### Kurulum

`Automock`, hem Jest hem de Sinon'u destekler. Test çerçevenize uygun olan paketi yükleyin. Ayrıca, `@automock/adapters.nestjs`'i (Automock'un diğer adaptörleriyle uyumlu olduğu için) yüklemeniz gerekecek.

Jest için:

```bash
$ npm i -D @automock/jest @automock/adapters.nestjs
```

Sinon için:

```bash
$ npm i -D @automock/sinon @automock/adapters.nestjs
```

#### Örnek

Burada verilen örnek, `Automock`'un Jest ile nasıl entegre edileceğini göstermektedir. Ancak, aynı prensipler ve işlevsellik Sinon için de geçerlidir.

Aşağıdaki `CatService` sınıfını ele alalım. Bu sınıf, kedileri almak için bir `Database` sınıfına bağımlıdır. `Database` sınıfını test etmek için `CatsService` sınıfını izole etmek üzere `Database` sınıfını sahte yapacağız.

```typescript
@Injectable()
export class Database {
  getCats(): Promise<Cat[]> { ... }
}

@Injectable()
class CatsService {
  constructor(private database: Database) {}

  async getAllCats(): Promise<Cat[]> {
    return this.database.getCats();
  }
}
```

`CatsService` sınıfı için bir birim testi kurmak için aşağıdaki gibi yapabiliriz.

`TestBed`'i `@automock/jest` paketinden kullanarak test ortamımızı oluşturacağız.

```typescript
import { TestBed } from '@automock/jest';

describe('Cats Service Unit Test', () => {
  let catsService: CatsService;
  let database: jest.Mocked<Database>;

  beforeAll(() => {
    const { unit, unitRef } = TestBed.create(CatsService).compile();

    catsService = unit;
    database = unitRef.get(Database);
  });

  it('should retrieve cats from the database', async () => {
    const mockCats: Cat[] = [{ id: 1, name: 'Catty' }, { id: 2, name: 'Mitzy' }];
    database.getCats.mockResolvedValue(mockCats);

    const cats = await catsService.getAllCats();

    expect(database.getCats).toHaveBeenCalled();
    expect(cats).toEqual(mockCats);
  });
});
```

Test kurulumunda:

1. `TestBed.create(CatsService).compile()` kullanarak `CatsService` için bir test ortamı oluşturuyoruz.
2. `unit` ile `CatsService`'in gerçek örneğini ve `unitRef.get(Database)` ile `Database`'in sahte örneğini alıyoruz.
3. `Database` sınıfının `getCats` metodunu önceden belirlenmiş kediler listesini döndürecek şekilde sahte yapıyoruz.
4. Ardından `CatsService`'in `getAllCats` metodunu çağırıyor ve onun `Database` sınıfıyla doğru etkileşimde bulunduğunu ve beklenen kedileri döndüğünü doğruluyoruz.

**Logger Eklemek**

Örneğimizi genişletelim ve `Logger` arabirimini ekleyerek `CatsService` sınıfına entegre edelim.

```typescript
@Injectable()
class Logger {
  log(message: string): void { ... }
}

@Injectable()
class CatsService {
  constructor(private database: Database, private logger: Logger) {}

  async getAllCats(): Promise<Cat[]> {
    this.logger.log('Fetching all cats..');
    return this.database.getCats();
  }
}
```

Şimdi, testinizi kurarken `Logger` bağımlılığını da sahte yapmanız gerekecektir:

```typescript
beforeAll(() => {
  let logger: jest.Mocked<Logger>;
  const { unit, unitRef } = TestBed.create(CatsService).compile();

  catsService = unit;
  database = unitRef.get(Database);
  logger = unitRef.get(Logger);
});

it('should log a message and retrieve cats from the database', async () => {
  const mockCats: Cat[] = [{ id: 1, name: 'Catty' }, { id: 2, name

: 'Mitzy' }];
  database.getCats.mockResolvedValue(mockCats);

  const cats = await catsService.getAllCats();

  expect(logger.log).toHaveBeenCalledWith('Fetching all cats..');
  expect(database.getCats).toHaveBeenCalled();
  expect(cats).toEqual(mockCats);
});
```

**Mock Uygulaması İçin `.mock().using()` Kullanmak**

`Automock`, `.mock().using()` metod zinciri kullanarak mock uygulamalarını belirtmek için daha deklaratif bir yol sağlar. Bu, `TestBed`'i kurarken mock davranışını doğrudan tanımlamanıza olanak tanır.

Test kurulumunu bu yaklaşımı kullanacak şekilde değiştirebilirsiniz:

```typescript
beforeAll(() => {
  const mockCats: Cat[] = [{ id: 1, name: 'Catty' }, { id: 2, name: 'Mitzy' }];

  const { unit, unitRef } = TestBed.create(CatsService)
    .mock(Database)
    .using({ getCats: async () => mockCats })
    .compile();

  catsService = unit;
  database = unitRef.get(Database);
});
```

Bu yaklaşımda, `TestBed`'i kurarken test gövdesinde `getCats` metodunu manuel olarak sahte yapma ihtiyacını ortadan kaldırdık. Bunun yerine, `.mock().using()` kullanarak test kurulumunu doğrudan mock davranışını tanımlayarak gerçekleştirdik.

#### Bağımlılık Referansları ve Örnek Erişimi

`TestBed` kullanırken, `compile()` yöntemi, iki önemli özellik içeren bir nesne döndürür: `unit` ve `unitRef`.
Bu özellikler, sırasıyla test edilen sınıfın örneğine ve bağımlılıklarına erişim sağlar.

`unit` - Unit özelliği, test edilen sınıfın gerçek örneğini temsil eder. Örneğin, `CatsService` sınıfının bir örneğine karşılık gelir.
Bu, sınıf ile doğrudan etkileşimde bulunmanıza ve test senaryoları sırasında metodlarını çağırmanıza olanak tanır.

`unitRef` - UnitRef özelliği, test edilen sınıfın bağımlılıklarına bir referans olarak hizmet eder. Örneğimizde, `CatsService`
tarafından kullanılan `Logger` bağımlılığına atıfta bulunur. `unitRef`'i kullanarak, bağımlılık için otomatik olarak oluşturulan
sahte nesneyi alabilirsiniz. Bu, sahte metodlarını tanımlamanıza, davranışları belirlemenize ve sahte nesne üzerinde metod
çağrılarını doğrulamanıza olanak tanır.

#### Farklı Sağlayıcılarla Çalışma

Sağlayıcılar, Nest'in en önemli unsurlarından biridir. Birçok varsayılan Nest sınıfını, hizmetleri, depoları, fabrikaları, yardımcıları
ve benzerlerini sağlayıcılar olarak düşünebilirsiniz. Bir sağlayıcının temel görevi, bir `Injectable` bağımlılığı formunda alınan bir yapıya sahip olmaktır.

Aşağıdaki `CatsService`'i düşünün, aşağıdaki `Logger` arabiriminden bir örnektir:

```typescript
export interface Logger {
  log(message: string): void;
}

@Injectable()
export class CatsService {
  constructor(private logger: Logger) {}
}
```

TypeScript'in Reflection API'si henüz arabirim yansımasını desteklememektedir. Nest, bu sorunu string/symbol tabanlı
enjeksiyon belirteçleri ile çözer (bkz. [Özel Sağlayıcılar](https://docs.nestjs.com/fundamentals/custom-providers)):

```typescript
export const MyLoggerProvider = {
  provide: 'LOGGER_TOKEN',
  useValue: { ... },
}

@Injectable()
export class CatsService {
  constructor(@Inject('LOGGER_TOKEN') readonly logger: Logger) {}
}
```

Automock, bu uygulamayı takip eder ve `unitRef.get()` yönteminde gerçek sınıfı sağlamak yerine, bir string tabanlı (veya sembol tabanlı) belirteç sağlamanıza olanak tanır:

```typescript
const { unit, unitRef } = TestBed.create(CatsService).compile();

let loggerMock: jest.Mocked<Logger> = unitRef.get('LOGGER_TOKEN');
```

#### Daha Fazla Bilgi

Daha fazla bilgi için [Automock GitHub deposunu](https://github.com/automock/automock) veya [Automock web sitesini](https://automock.dev) ziyaret edebilirsiniz.