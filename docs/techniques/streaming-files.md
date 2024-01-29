### Dosya Akışı

> info **Not** Bu bölüm, dosyaları **HTTP uygulamanızdan** akış halinde gönderme yöntemini göstermektedir. Aşağıda sunulan örnekler, GraphQL veya Mikroservis uygulamalarına uygulanmaz.

REST API'nizden istemciye bir dosya göndermek istediğiniz durumlar olabilir. Bunun için genellikle Nest'te aşağıdakileri yapardınız:

```ts
@Controller('file')
export class FileController {
  @Get()
  getFile(@Res() res: Response) {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    file.pipe(res);
  }
}
```

Ancak bu şekilde, kontrolör sonrası interceptor mantığını kaybedersiniz. Bununla başa çıkmak için `StreamableFile` örneği döndürebilir ve altyapı olarak, çerçeve boruların işini halletecektir.

#### Streamable File Sınıfı

`StreamableFile`, geri döndürülecek akışı tutan bir sınıftır. Yeni bir `StreamableFile` oluşturmak için, `StreamableFile` yapıcısına bir `Buffer` veya bir `Stream` geçirebilirsiniz.

> info **Tavsiye** `StreamableFile` sınıfı, `@nestjs/common` tarafından içe aktarılabilir.

#### Çoklu platform desteği

Fastify, varsayılan olarak `stream.pipe(res)` çağırmadan dosyaları göndermeyi destekleyebilir, bu nedenle `StreamableFile` sınıfını hiç kullanmanıza gerek yoktur. Ancak, Nest, her iki platform türünde de `StreamableFile` kullanımını destekler, bu nedenle Express ve Fastify arasında uyumluluk konusunda endişe etmenize gerek yoktur.

#### Örnek

`package.json`'ı bir JSON yerine dosya olarak döndürmenin basit bir örneğini aşağıda bulabilirsiniz, ancak bu fikir doğal olarak resimler, belgeler ve diğer herhangi bir dosya türüne genişletilebilir.

```ts
import { Controller, Get, StreamableFile } from '@nestjs/common';
import { createReadStream } from 'fs';
import { join } from 'path';

@Controller('file')
export class FileController {
  @Get()
  getFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }
}
```

Varsayılan içerik türü `application/octet-stream`'dir; yanıtı özelleştirmeniz gerekirse, `res.set` yöntemini veya [`@Header()`](/docs/controllers#headers) dekoratörünü kullanabilirsiniz. Aşağıdaki gibi:

```ts
import { Controller, Get, StreamableFile, Res } from '@nestjs/common';
import { createReadStream } from 'fs';
import { join } from 'path';
import type { Response } from 'express';

@Controller('file')
export class FileController {
  @Get()
  getFile(@Res({ passthrough: true }) res: Response): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    res.set({
      'Content-Type': 'application/json',
      'Content-Disposition': 'attachment; filename="package.json"',
    });
    return new StreamableFile(file);
  }

  // Veya daha kısa:
  @Get()
  @Header('Content-Type', 'application/json')
  @Header('Content-Disposition', 'attachment; filename="package.json"')
  getStaticFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }  
}
```