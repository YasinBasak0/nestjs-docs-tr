### HTTP Modülü

[Axios](https://github.com/axios/axios), geniş özelliklere sahip yaygın olarak kullanılan bir HTTP istemci paketidir. Nest, Axios'u sarmalar ve bunu yerleşik `HttpModule` aracılığıyla sunar. `HttpModule`, Axios tabanlı HTTP istekleri gerçekleştirmek için kullanılan `HttpService` sınıfını dışa aktarır. Kütüphane ayrıca HTTP yanıtlarını `Observable`'lara dönüştürür.

> info **İpucu** Ayrıca, [got](https://github.com/sindresorhus/got) veya [undici](https://github.com/nodejs/undici) gibi genel amaçlı Node.js HTTP istemci kütüphanelerini de doğrudan kullanabilirsiniz.

#### Kurulum

Kullanmaya başlamak için önce gerekli bağımlılıkları yükleriz.

```bash
$ npm i --save @nestjs/axios axios
```

#### Başlangıç

Kurulum işlemi tamamlandığında, `HttpService`'i kullanmak için önce `HttpModule`'u içe aktarırız.

```typescript
@Module({
  imports: [HttpModule],
  providers: [CatsService],
})
export class CatsModule {}
```

Daha sonra, normal constructor enjeksiyonu kullanarak `HttpService`'i enjekte ederiz.

> info **İpucu** `HttpModule` ve `HttpService`, `@nestjs/axios` paketinden içe aktarılır.

```typescript
@@filename()
@Injectable()
export class CatsService {
  constructor(private readonly httpService: HttpService) {}

  findAll(): Observable<AxiosResponse<Cat[]>> {
    return this.httpService.get('http://localhost:3000/cats');
  }
}
@@switch
@Injectable()
@Dependencies(HttpService)
export class CatsService {
  constructor(httpService) {
    this.httpService = httpService;
  }

  findAll() {
    return this.httpService.get('http://localhost:3000/cats');
  }
}
```

> info **İpucu** `AxiosResponse`, `axios` paketinden (`$ npm i axios`) dışa aktarılan bir arayüzdür.

Tüm `HttpService` metodları, bir `Observable` içinde bulunan bir `AxiosResponse` döndürür.

#### Konfigürasyon

[Axios](https://github.com/axios/axios), `HttpService`'in davranışını özelleştirmek için çeşitli seçeneklerle yapılandırılabilir. Bunlar hakkında daha fazla bilgiyi [buradan](https://github.com/axios/axios#request-config) okuyabilirsiniz. Altta yatan Axios örneğini yapılandırmak için, `HttpModule`'un `register()` metoduna içe aktarılırken opsiyonel bir seçenek nesnesi geçirin. Bu seçenek nesnesi, doğrudan altta yatan Axios yapıcısına iletilir.

```typescript
@Module({
  imports: [
    HttpModule.register({
      timeout: 5000,
      maxRedirects: 5,
    }),
  ],
  providers: [CatsService],
})
export class CatsModule {}
```

#### Asenkron konfigürasyon

Modül seçeneklerini statik olarak değil de asenkron olarak iletmek istediğinizde, `registerAsync()` metodunu kullanın. Çoğu dinamik modül gibi, Nest asenkron konfigürasyonla başa çıkmanız için birkaç teknik sağlar.

Bir teknik, bir fabrika fonksiyonu kullanmaktır:

```typescript
HttpModule.registerAsync({
  useFactory: () => ({
    timeout: 5000,
    maxRedirects: 5,
  }),
});
```

Diğer [fabrika sağlayıcıları](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) gibi, fabrika fonksiyonumuz [async](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) olabilir ve `inject` aracılığıyla bağımlılıkları enjekte edebilir.

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    timeout: configService.get('HTTP_TIMEOUT'),
    maxRedirects: configService.get('HTTP_MAX_REDIRECTS'),
  }),
  inject: [ConfigService],
});
```

Alternatif olarak, `HttpModule`'u bir fabrika yerine bir sınıf kullanarak yapılandırabilirsiniz, aşağıda gösterildiği gibi.

```typescript
HttpModule.registerAsync({
  useClass: HttpConfigService,
});
```

Yukarıdaki yapının içinde, `HttpConfigService` sınıfının `HttpModuleOptionsFactory` arayüzünü uygulaması gerektiğine dikkat edin. `HttpModule`, sağlanan sınıfın örneği üzerinde `createHttpOptions()` metodunu çağırır.

```typescript
@Injectable()
class HttpConfigService implements HttpModuleOptionsFactory {
  createHttpOptions(): HttpModuleOptions {
    return {
      timeout: 5000,
      maxRedirects: 5,
    };
  }
}
```

`HttpModule` içinde özel bir kopya oluşturmak yerine mevcut bir seçenek sağlayıcısını yeniden kullanmak istiyorsanız, `useExisting` sözdizimini kullanın.

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useExisting: HttpConfigService,
});
```

#### Axios'u doğrudan kullanma

Eğer `HttpModule.register`'ın seçeneklerinin size yetmediğini düşünüyorsanız veya sadece `@nestjs/axios` tarafından oluşturulan altta yatan Axios örneğine erişmek istiyorsanız, bunu şu şekilde `HttpService#axiosRef` üzerinden yapabilirsiniz:

```typescript
@Injectable()
export class CatsService {
  constructor(private readonly httpService: HttpService) {}

  findAll(): Promise<AxiosResponse<Cat[]>> {
    return this.httpService.axiosRef.get('http://localhost:3000/cats');
    //                      ^ AxiosInstance interface
  }
}
```

#### Tam örnek

`HttpService` metodlarının dönüş değeri bir `Observable` olduğundan, isteğin verilerini bir promise formunda almak için `rxjs` - `firstValueFrom` veya `lastValue

From`'i kullanabiliriz.

```typescript
import { catchError, firstValueFrom } from 'rxjs';

@Injectable()
export class CatsService {
  private readonly logger = new Logger(CatsService.name);
  constructor(private readonly httpService: HttpService) {}

  async findAll(): Promise<Cat[]> {
    const { data } = await firstValueFrom(
      this.httpService.get<Cat[]>('http://localhost:3000/cats').pipe(
        catchError((error: AxiosError) => {
          this.logger.error(error.response.data);
          throw 'An error happened!';
        }),
      ),
    );
    return data;
  }
}
```

> info **İpucu** Farkları öğrenmek için RxJS'in [`firstValueFrom`](https://rxjs.dev/api/index/function/firstValueFrom) ve [`lastValueFrom`](https://rxjs.dev/api/index/function/lastValueFrom) üzerine tıklayın.