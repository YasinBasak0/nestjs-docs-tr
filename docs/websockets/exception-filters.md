### İstisna Filtreleri

HTTP [exception filter](/docs/exception-filters) katmanı ile karşılaştırıldığında, tek fark karşılık gelen web soketleri katmanında `HttpException` yerine `WsException` kullanmanız gerektiğidir.

```typescript
throw new WsException('Geçersiz kimlik bilgileri.');
```

> info **İpucu** `WsException` sınıfı, `@nestjs/websockets` paketinden içe aktarılır.

Yukarıdaki örnekle, Nest hatayı işleyecek ve aşağıdaki yapısıyla `exception` iletimini yapacaktır:

```typescript
{
  status: 'error',
  message: 'Geçersiz kimlik bilgileri.'
}
```

#### Filtreler

Web soketleri istisna filtreleri, HTTP istisna filtreleriyle eşdeğer davranır. Aşağıdaki örnek, manuel olarak oluşturulmuş bir yöntem kapsamındaki filtre kullanır. HTTP tabanlı uygulamalarda olduğu gibi ayrıca ağ geçidi kapsamında filtreler de kullanabilirsiniz (örneğin, ağ geçidi sınıfını `@UseFilters()` dekoratörü ile önekleyin).

```typescript
@UseFilters(new WsExceptionFilter())
@SubscribeMessage('events')
onEvent(client, data: any): WsResponse<any> {
  const event = 'events';
  return { event, data };
}
```

#### Kalıtım

Genellikle, uygulama gereksinimlerinizi karşılamak için özel istisna filtreleri oluşturacaksınız. Ancak, belirli faktörlere dayanarak davranışı geçersiz kılmak istediğiniz durumlar olabilir.

İstisna işlemini temel filtreye devretmek için `BaseWsExceptionFilter`'ı genişletmeniz ve miras alınan `catch()` yöntemini çağırmanız gerekir.

```typescript
@@filename()
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseWsExceptionFilter } from '@nestjs/websockets';

@Catch()
export class AllExceptionsFilter extends BaseWsExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseWsExceptionFilter } from '@nestjs/websockets';

@Catch()
export class AllExceptionsFilter extends BaseWsExceptionFilter {
  catch(exception, host) {
    super.catch(exception, host);
  }
}
```

Yukarıdaki uygulama, yaklaşımı gösteren bir kabuk yalnızca shell'dir. Genişletilmiş istisna filtresinin uygulamanızın özelleştirilmiş **iş mantığını** içermesi gerekir (örneğin, çeşitli koşulları ele alma).