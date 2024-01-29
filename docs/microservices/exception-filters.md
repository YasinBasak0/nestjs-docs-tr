### İstisna Filtreleri

HTTP [istisna filtresi](/docs/exception-filters) katmanı ile karşılaştırıldığında, yalnızca `HttpException` fırlatmak yerine `RpcException` kullanmanız gerektiğidir.

```typescript
throw new RpcException('Geçersiz kimlik bilgileri.');
```

> info **Hint** `RpcException` sınıfı, `@nestjs/microservices` paketinden içe aktarılır.

Yukarıdaki örnekle, Nest fırlatılan istisnayı işleyecek ve aşağıdaki yapısıyla `error` nesnesini döndürecektir:

```json
{
  "status": "error",
  "message": "Geçersiz kimlik bilgileri."
}
```

#### Filtreler

Mikroservis istisna filtreleri, HTTP istisna filtreleri ile benzer şekilde davranır, ancak küçük bir fark vardır. `catch()` yöntemi bir `Observable` döndürmelidir.

```typescript
@@filename(rpc-exception.filter)
import { Catch, RpcExceptionFilter, ArgumentsHost } from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { RpcException } from '@nestjs/microservices';

@Catch(RpcException)
export class ExceptionFilter implements RpcExceptionFilter<RpcException> {
  catch(exception: RpcException, host: ArgumentsHost): Observable<any> {
    return throwError(() => exception.getError());
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { throwError } from 'rxjs';

@Catch(RpcException)
export class ExceptionFilter {
  catch(exception, host) {
    return throwError(() => exception.getError());
  }
}
```

> warning **Uyarı** [Hybrid uygulama](/docs/faq/hybrid-application) kullanıyorsanız, genel mikroservis istisna filtreleri varsayılan olarak etkin değildir.

Aşağıdaki örnek, manuel olarak oluşturulan yöntem kapsamlı bir filtre kullanmaktadır. HTTP tabanlı uygulamalarda olduğu gibi, denetleyici kapsamlı filtreleri de kullanabilirsiniz (örneğin, denetleyici sınıfını `@UseFilters()` dekoratörü ile önce ekleyin).

```typescript
@@filename()
@UseFilters(new ExceptionFilter())
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseFilters(new ExceptionFilter())
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```

#### Miras

Genellikle, uygulama gereksinimlerinizi karşılamak için özel istisna filtreleri oluşturursunuz. Ancak, belirli faktörlere dayalı olarak davranışı geçersiz kılmak istediğiniz durumlar olabilir.

İstisna işlemini temel filtreye devretmek için `BaseExceptionFilter`'ı genişletmeniz ve miras alınan `catch()` yöntemini çağırmanız gerekir.

```typescript
@@filename()
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseRpcExceptionFilter } from '@nestjs/microservices';

@Catch()
export class AllExceptionsFilter extends BaseRpcExceptionFilter {
  catch(exception: any, host: ArgumentsHost) {
    return super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseRpcExceptionFilter } from '@nestjs/microservices';

@Catch()
export class AllExceptionsFilter extends BaseRpcExceptionFilter {
  catch(exception, host) {
    return super.catch(exception, host);
  }
}
```

Yukarıdaki uygulama sadece yaklaşımı gösteren bir kabuk olduğu için, genişletilmiş istisna filtresinin uygulamanızın özel **iş mantığı** (örneğin, çeşitli koşulları ele alma) içermesi gerekir.