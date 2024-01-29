---
sidebar_position: 9
---

### Interceptorlar

Bir interceptor, `@Injectable()` dekoratörü ile işaretlenmiş ve `NestInterceptor` arabirimini uygulamış bir sınıftır.

<figure><img src="/assets/Interceptors_1.png" /></figure>

Interceptor'lar, [Aspect Oriented Programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming) (AOP) tekniğinden ilham alınarak tasarlanmış bir dizi faydalı özelliğe sahiptir. Bu özellikler şunları mümkün kılar:

- metodun çalıştırılması öncesi / sonrasına ekstra mantık bağlamak
- bir fonksiyonun döndüğü sonucu dönüştürmek
- bir fonksiyondan fırlatılan istisnayı dönüştürmek
- temel fonksiyon davranışını genişletmek
- belirli koşullara bağlı olarak bir fonksiyonu tamamen geçersiz kılmak (örneğin, önbellekleme amaçlı)

#### Temeller

Her interceptor, `intercept()` metodunu uygular; bu metod iki argüman alır. İlk argüman, `ExecutionContext` örneğidir (tam olarak [guards](/docs/guards) için olduğu gibi). `ExecutionContext`, `ArgumentsHost` sınıfından türetilir. Bu konu hakkında daha fazla bilgi için [exception filters](https://docs.nestjs.com/exception-filters#arguments-host) bölümüne başvurabilirsiniz.

#### Execution context

`ArgumentsHost`'u genişleterek, `ExecutionContext` ayrıca mevcut yürütme süreci hakkında ek ayrıntılar sağlayan bir dizi yeni yardımcı metod ekler. Bu detaylar, daha genel interceptor'lar oluşturmak için faydalı olabilir. `ExecutionContext` hakkında daha fazla bilgi için [buraya](/docs/fundamentals/execution-context) bakın.

#### Call handler

İkinci argüman, bir `CallHandler`'dır. `CallHandler` arabirimi, interceptor'ınızın bir noktasında route handler metodunu çağırmak için kullanabileceğiniz `handle()` metodunu uygular. `intercept()` metodunuzun içinde `handle()` metodunu çağırmazsanız, route handler metodu hiç çalıştırılmaz.

Bu yaklaşım, `intercept()` metodunun etkili bir şekilde istek/yanıt akışını **sararak** çalıştığı anlamına gelir. Sonuç olarak, `intercept()` metodunuzda `handle()`'ı çağırmadan önce ve sonra özel mantık uygulayabilirsiniz. `handle()` metodunu çağırmazsanız, route handler metodu hiç çalıştırılmaz. `handle()` çağrıldıktan sonra (ve `Observable`'ı döndürdükten sonra), `create()` handler tetiklenir. Ve yanıt akışı `Observable` aracılığıyla alındığında, akış üzerinde ek işlemler gerçekleştirilebilir ve çağıran tarafına nihai bir sonuç döndürülebilir.

<app-banner-devtools></app-banner-devtools>

#### Aspect interception

İlk olarak, bir interceptor'ı kullanarak kullanıcı etkileşimini (örneğin, kullanıcı çağrılarını depolama, olayları asenkron olarak gönderme veya bir zaman damgası hesaplama) kaydetmek için bir kullanım durumuna bakacağız. Aşağıda basit bir `LoggingInterceptor` örneği gösterilmiştir:

```typescript
@@filename(logging.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor {
  intercept(context, next) {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```

> info **İpucu** `NestInterceptor<T, R>` türünde generic bir arabirimdir. Burada `T`, bir `Observable<T>`'nin türünü (yanıt akışını destekleme) gösterir ve `R`, `Observable<R>` ile sarmalanmış değerin türünü gösterir.

> warning **Notice** Interceptor'lar, controller'lar, sağlayıcılar, guard'lar vb. gibi, bağımlılıkları kendi `constructor`'ları aracılığıyla **enjekte edebilir**.

Çünkü `handle()` bir RxJS `Observable` döndürdüğü için, akışı manipüle etmek için kullanabileceğimiz geniş bir operatör seçeneğimiz var. Yukarıdaki örnekte `tap()` operatörünü kullandık; bu operatör, gözlemlenebilir akışın zarif veya istisnai bir şekilde sona erdiğinde anonim günlüğe yazma fonksiyonumuzu çağırır, ancak aksi takdirde yanıt döngüsüyle etkileşmez.

#### Interceptor'ları Bağlama

Interceptor'ı kurmak için, `@nestjs/common` paketinden içe aktarılan `@UseInterceptors()` dekoratörünü kullanıyoruz. [Pipes](/docs/pipes) ve [guards](/docs/guards) gibi, interceptor'lar controller düzeyinde, metod düzeyinde veya global düzeyde olabilir.

```typescript
@@filename(cats.controller)
@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```

> info **İpucu** `@UseInterceptors()` dekoratörü, `@nestjs/common` paketinden içe aktarılır.

Yukarıdaki yapıyı kullanarak, `CatsController` içinde tanımlanan her route handler, `LoggingInterceptor`'ı kullanacaktır. Birisi `GET /cats` endpoint'ini çağırdığında, standart çıktınızda aşağıdaki çıktıyı göreceksiniz:

```typescript
Before...
After... 1ms
```

Unutmayın ki, `LoggingInterceptor` türünü (bir örneğin yerine) geçtik, bu, örnekleme sorumluluğunu çerçeveye bırakır ve bağımlılık enjekte etmeyi mümkün kılar. Pipes, guards ve istisna filtreleri gibi, yerinde bir örnek de geçebiliriz:

```typescript
@@filename(cats.controller)
@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```

Yukarıda verilen yapı, bu controller tarafından bildirilen her handler'a interceptor'ı ekler. Eğer interceptor'ın kapsamını yalnızca tek bir metoda sınırlamak istersek, dekoratörü **metod düzeyinde** uygularız.

Global bir interceptor kurmak için, Nest uygulama örneğinin `useGlobalInterceptors()` yöntemini kullanırız:

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

Global interceptor'lar, tüm uygulama genelinde, her controller ve her route handler için kullanılır. Bağımlılık enjeksiyonu açısından, global interceptor'lar (yukarıdaki örnekte olduğu gibi) herhangi bir modül dışından kaydedildiğinde, bu, herhangi bir modül bağlamı içinde yapılmadığından bağımlılıkları enjekte edemez. Bu sorunu çözmek için, doğrudan herhangi bir modülden bir interceptor'ı aşağıdaki yapıyı kullanarak kurabilirsiniz:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

> info **İpucu** Interceptor için bağımlılık enjekte etmek için bu yaklaşımı kullanırken, bu yapının uygulandığı modülün önemli olmadığına dikkat edin, çünkü interceptor aslında globaldir. Nerede yapılmalı? Yukarıdaki örnekte olduğu gibi, interceptor'ın tanımlandığı modülü seçin. Ayrıca, `useClass` özel sağlayıcı kaydıyla başa çıkmak için tek yol değildir. Daha fazla bilgi için [buraya](/docs/fundamentals/custom-providers) bakın.


#### Yanıt Eşlemesi

Zaten `handle()`'ın bir `Observable` döndürdüğünü biliyoruz. Akış, route handler tarafından **döndürülen** değeri içerir ve bu nedenle RxJS'in `map()` operatörünü kullanarak kolayca değiştirebiliriz.

> warning **Uyarı** Yanıt eşleme özelliği, kütüphane özgü yanıt stratejisi ile (doğrudan `@Res()` nesnesini kullanmak yasaktır) çalışmaz.

Hadi `TransformInterceptor`'ı oluşturalım. Bu interceptor, her yanıtı göstermek için değeri bir nesnenin `data` özelliğine atayan basit bir şekilde her yanıtı değiştirecektir.

```typescript
@@filename(transform.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class TransformInterceptor {
  intercept(context, next) {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

> info **İpucu** Nest interceptor'ları, hem senkron hem de asenkron `intercept()` metodlarıyla çalışır. Gerekirse basitçe metodu `async`'e geçirebilirsiniz.

Yukarıdaki yapıyla, birisi `GET /cats` endpoint'ini çağırdığında, yanıt aşağıdaki gibi görünecektir (varsayalım ki route handler bir boş dizi `[]` döndürüyor):

```json
{
  "data": []
}
```

Interceptor'lar, uygulama genelinde ortaya çıkan gereksinimlere yönelik yeniden kullanılabilir çözümler oluşturmada büyük bir değere sahiptir. Örneğin, bir `null` değerinin her örneğini boş bir dize `''`'e dönüştürmemiz gerektiğini düşünün. Bu gereksinimi tek bir satır kod kullanarak gerçekleştirebilir ve interceptor'ı global olarak bağlayabiliriz, böylece her kaydedilmiş handler tarafından otomatik olarak kullanılacaktır.

```typescript
@@filename()
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor {
  intercept(context, next) {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
```

#### İstisna Eşleme

Başka bir ilginç kullanım durumu, atılan istisnaları geçersiz kılmak için RxJS'in `catchError()` operatöründen faydalanmaktır:

```typescript
@@filename(errors.interceptor)
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
@@switch
import { Injectable, BadGatewayException } from '@nestjs/common';
import { throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor {
  intercept(context, next) {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
```

#### Akışın Geçersiz Kılınması

Bazen işleyiciyi tamamen engellemek ve bunun yerine farklı bir değeri döndürmek isteyebileceğimiz birkaç neden vardır. Açık bir örnek, yanıt süresini iyileştirmek için önbellek uygulamaktır. Gerçek bir örnekte, TTL, önbellek geçersiz kılma, önbellek boyutu gibi diğer faktörleri düşünmek isteyeceğimizden, ancak bu tartışmanın kapsamının ötesindedir. Burada, ana konsepti gösteren temel bir örnek sunacağız.

```typescript
@@filename(cache.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { of } from 'rxjs';

@Injectable()
export class CacheInterceptor {
  intercept(context, next) {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

`CacheInterceptor`'ümüzde sabit bir `isCached` değişkeni ve sabit bir yanıt `[]` var. Dikkat edilmesi gereken temel nokta, burada RxJS `of()` operatörü tarafından oluşturulan yeni bir akışı döndürüyor olmamızdır, bu nedenle yol işleyicisi hiç **çağrılmayacak**. Bir uç noktayı çağıran birini arayan bir uç noktayı çağırdığında, `CacheInterceptor`'ı kullanan bir yanıt (sabit, boş bir dizi) hemen dönecektir. Genel bir çözüm oluşturmak için `Reflector`'dan yararlanabilir ve özel bir dekoratör oluşturabilirsiniz. `Reflector`, [guards](/docs/guards) bölümünde iyi bir şekilde açıklanmıştır.


#### Daha Fazla Operatör

RxJS operatörlerini kullanarak akışı manipüle etme olasılığı, bize birçok yetenek sunar. Başka yaygın bir kullanım durumunu düşünelim. Uç nokta isteklerinde **zaman aşımı** işlemek istediğinizi hayal edin. Uç noktanız belirli bir süre boyunca hiçbir şey döndürmezse, bir hata yanıtı ile sonlandırmak istersiniz. Aşağıdaki yapı bunu mümkün kılar:

```typescript
@@filename(timeout.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
@@switch
import { Injectable, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor {
  intercept(context, next) {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
```

5 saniye sonra istek işlemi iptal edilecektir. `RequestTimeoutException` fırlatmadan önce özel mantık eklemek de mümkündür (örneğin, kaynakları serbest bırakmak).