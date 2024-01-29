### Dosya Yükleme

Dosya yükleme işlemlerini yönetmek için, Nest, Express için [multer](https://github.com/expressjs/multer) ara yazılım paketine dayanan yerleşik bir modül sağlar. Multer, genellikle HTTP `POST` isteği aracılığıyla dosya yükleme amacıyla kullanılan `multipart/form-data` biçiminde gönderilen verileri işler. Bu modül tamamen yapılandırılabilir ve davranışını uygulama gereksinimlerinize uyacak şekilde ayarlayabilirsiniz.

> warning **Uyarı** Multer, desteklenmeyen `multipart/form-data` biçimindeki verileri işleyemez. Ayrıca, bu paketin `FastifyAdapter` ile uyumlu olmadığını unutun.

Daha iyi tür güvenliği için, Multer yazım paketini şu şekilde kurmamız gerekiyor:

```shell
$ npm i -D @types/multer
```

Bu paket kurulduktan sonra, `Express.Multer.File` türünü kullanabiliriz (bu türü şu şekilde içe aktarabilirsiniz: `import {{ '{' }} Express {{ '}' }} from 'express'`).

#### Temel örnek

Bir dosya yüklemek için, basitçe `FileInterceptor()` aracısını yönlendirme işleyicisine bağlayın ve `@UploadedFile()` dekoratörünü kullanarak `request` üzerinden `file`'ı çıkarın.

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
uploadFile(@UploadedFile() file: Express.Multer.File) {
  console.log(file);
}
@@switch
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
@Bind(UploadedFile())
uploadFile(file) {
  console.log(file);
}
```

> info **Hint** `FileInterceptor()` dekoratörü, `@nestjs/platform-express` paketinden dışa aktarılır. `@UploadedFile()` dekoratörü, `@nestjs/common` paketinden dışa aktarılır.

`FileInterceptor()` dekoratörü iki argüman alır:

- `fieldName`: HTML formundaki dosyayı tutan alanın adını sağlayan bir dizedir.
- `options`: isteğe bağlı `MulterOptions` türünde bir nesnedir. Bu, multer oluşturucusu tarafından kullanılan aynı nesnedir (daha fazla detay [burada](https://github.com/expressjs/multer#multeropts)).

> warning **Uyarı** `FileInterceptor()` üçüncü taraf bulut sağlayıcılarıyla, örneğin Google Firebase veya diğerleriyle uyumlu olmayabilir.

#### Dosya doğrulama

Çoğu zaman, gelen dosya meta verilerini, dosya boyutu veya dosya mime tipi gibi doğrulamak yararlı olabilir. Bunun için kendi [Pipe](https://docs.nestjs.com/pipes) sınıfınızı oluşturabilir ve `UploadedFile` dekoratörü ile işaretlenmiş parametreye bağlayabilirsiniz. Aşağıdaki örnek, temel bir dosya boyutu doğrulayıcı boru hattının nasıl uygulanacağını göstermektedir:

```typescript
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class FileSizeValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    // "value", dosyanın özelliklerini ve meta verilerini içeren bir nesnedir
    const oneKb = 1000;
    return value.size < oneKb;
  }
}
```

Nest, yaygın kullanım durumlarını ele almak ve yeni kullanım durumlarını kolaylaştırmak/standartlaştırmak için dahili bir boru sağlar. Bu boru `ParseFilePipe` olarak adlandırılır ve aşağıdaki gibi kullanabilirsiniz:

```typescript
@Post('file')
uploadFileAndPassValidation(
  @Body() body: SampleDto,
  @UploadedFile(
    new ParseFilePipe({
      validators: [
        // ... Burada dosya doğrulayıcı örnekleri kümesi
      ]
    })
  )
  file: Express.Multer.File,
) {
  return {
    body,
    file: file.buffer.toString(),
  };
}
```

Görüldüğü gibi, bu boruya çalışacak olan bir dizi dosya doğrulayıcısını belirtmek gereklidir. Bir doğrulayıcı sınıfın arayüzünü inceleyeceğiz, ancak bu borunun iki ek **isteğe bağlı** seçeneği de vardır:

<table>
  <tr>
    <td><code>errorHttpStatusCode</code></td>
    <td><b>Herhangi bir</b> doğrulayıcı başarısız olursa fırlatılacak HTTP durum kodu. Varsayılan değer <code>400</code> (BAD REQUEST)</td>
  </tr>
  <tr>
    <td><code>exceptionFactory</code></td>
    <td>Hata iletisi alanını alır ve bir hata döndüren bir fabrikadır.</td>
  </tr>
</table>

Şimdi, `FileValidator` arayüzüne geri dönelim. Bu arayüzü, istemci tarafından sağlanan seçeneklere göre dosyayı doğrular ve bunun için sınıfınızı sağlamanız gerekir. Nest'in projenizde kullanabileceğiniz iki dahili `FileValidator` uygulaması bulunmaktadır:

- `MaxFileSizeValidator` - Belirtilen değerden (bayt cinsinden) daha küçük olan bir dosyanın boyutunu kontrol eder
- `FileTypeValidator` - Belirtilen değerle eşleşen bir dosyanın mime tipini kontrol eder. 

> warning **Uyarı** Dosya türünü doğrulamak için, [FileTypeValidator](https://github.com/nestjs/nest/blob/master/packages/common/pipes/file/file-type.validator.ts) sınıfı, multer tarafından algılanan türü kullanır. Multer, varsayılan olarak, dosya türünü kullanıcının cihazındaki dosya uzantısından türeter. Ancak, gerçek dosya içeriğini kontrol etmez. Dosyalar, isteğe bağlı uzantılara yeniden adlandırılabilir, bu nedenle uygulamanız daha güvenli bir çözüm gerektiriyorsa, dosyanın [sihirli sayı](https://www.ibm.com/support/pages/what-magic-number) gibi gerçek içeriğini kontrol etmek gibi özel bir uygulama kullanmayı düşünmelisiniz.

Bu doğrulayıcıları yukarıda belirtilen `FileParsePipe` ile nasıl birleştireceğimizi anlamak için, yukarıda sunulan son örneğin değiştirilmiş bir parçasını kullanacağız:

```typescript
@UploadedFile(
  new ParseFilePipe({
    validators: [
      new MaxFileSizeValidator({ maxSize: 1000 }),
      new FileTypeValidator({ fileType: 'image/jpeg' }),
    ],
  }),
)
file: Express.Multer.File,
```
> info **Hint** Eğer doğrulayıcı sayısı büyükse veya seçenekleri dosyayı karıştırıyorsa, bu diziyi bir ayrı dosyada tanımlayabilir ve buraya `fileValidators` gibi bir isimli sabit olarak içe aktarabilirsiniz.

Son olarak, özel doğrulayıcıları elle başlatma yerine, doğrudan seçeneklerini ileterek doğrulayıcılarınızı birleştirebileceğiniz özel `ParseFilePipeBuilder` sınıfını kullanabilirsiniz:

```typescript
@UploadedFile(
  new ParseFilePipeBuilder()
    .addFileTypeValidator({
      fileType: 'jpeg',
    })
    .addMaxSizeValidator({
      maxSize: 1000
    })
    .build({
      errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY
    }),
)
file: Express.Multer.File,
```

#### Dosya Dizisi

Bir dosya dizisi yüklemek için (tek bir alan adı ile belirlenen), `FilesInterceptor()` dekoratörünü kullanın (dikkat dekoratör adındaki **Files** kelimesinin çoğul olmasına). Bu dekoratör üç argüman alır:

- `fieldName`: yukarıda açıklandığı gibi
- `maxCount`: kabul edilecek maksimum dosya sayısını tanımlayan isteğe bağlı bir sayı
- `options`: yukarıda açıklandığı gibi isteğe bağlı `MulterOptions` nesnesi

`FilesInterceptor()` kullanılırken, dosyaları `@UploadedFiles()` dekoratörü ile `request`'ten çıkarın.

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FilesInterceptor('files'))
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
@@switch
@Post('upload')
@UseInterceptors(FilesInterceptor('files'))
@Bind(UploadedFiles())
uploadFile(files) {
  console.log(files);
}
```

> info **Hint** `FilesInterceptor()` dekoratörü, `@nestjs/platform-express` paketinden dışa aktarılır. `@UploadedFiles()` dekoratörü, `@nestjs/common` paketinden dışa aktarılır.

#### Birden Çok Dosya

Birden çok dosya yüklemek için (hepsi farklı alan adı anahtarlarıyla), `FileFieldsInterceptor()` dekoratörünü kullanın. Bu dekoratör iki argüman alır:

- `uploadedFields`: her biri bir `name` özelliğini, yukarıda açıklandığı gibi bir alan adını belirleyen bir dize değeri içeren bir nesneler dizisi
- `options`: yukarıda açıklandığı gibi isteğe bağlı `MulterOptions` nesnesi

`FileFieldsInterceptor()` kullanılırken, dosyaları `@UploadedFiles()` dekoratörü ile `request`'ten çıkarın.

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FileFieldsInterceptor([
  { name: 'avatar', maxCount: 1 },
  { name: 'background', maxCount: 1 },
]))
uploadFile(@UploadedFiles() files: { avatar?: Express.Multer.File[], background?: Express.Multer.File[] }) {
  console.log(files);
}
@@switch
@Post('upload')
@Bind(UploadedFiles())
@UseInterceptors(FileFieldsInterceptor([
  { name: 'avatar', maxCount: 1 },
  { name: 'background', maxCount: 1 },
]))
uploadFile(files) {
  console.log(files);
}
```

#### Herhangi bir dosya

Herhangi bir alan adı anahtarları ile tüm alanları yüklemek için, `AnyFilesInterceptor()` dekoratörünü kullanın. Bu dekoratör, yukarıda açıklandığı gibi isteğe bağlı bir `options` nesnesini kabul edebilir.

`AnyFilesInterceptor()` kullanılırken, dosyaları `@UploadedFiles()` dekoratörü ile `request`'ten çıkarın.

```typescript
@@filename()
@Post('upload')
@UseInterceptors(AnyFilesInterceptor())
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
@@switch
@Post('upload')
@Bind(UploadedFiles())
@UseInterceptors(AnyFilesInterceptor())
uploadFile(files) {
  console.log(files);
}
```

#### Dosya Yok

`multipart/form-data`'yı kabul etmek ancak herhangi bir dosyanın yüklenmesine izin vermemek için, `NoFilesInterceptor`'ı kullanın. Bu, çoklu veriyi istek gövdesi üzerinde özellikler olarak ayarlar. İsteğe dosya eklenirse, `BadRequestException` hatası alırsınız.

```typescript
@Post('upload')
@UseInterceptors(NoFilesInterceptor())
handleMultiPartData(@Body() body) {
  console.log(body)
}
```

#### Varsayılan Seçenekler

Yukarıda açıklandığı gibi dosya interceptor'larında multer seçeneklerini belirleyebilirsiniz. Varsayılan seçenekleri belirlemek için, `MulterModule`'u içe aktardığınızda desteklenen seçenekleri geçerek statik `register()` metodunu çağırabilirsiniz. [Burada](https://github.com/expressjs/multer#multeropts) listelenen tüm seçenekleri kullanabilirsiniz.

```typescript
MulterModule.register({
  dest: './upload',
});
```

> info **Hint** `MulterModule` sınıfı, `@nestjs/platform-express` paketinden dışa aktarılır.

#### Async Konfigürasyon

`MulterModule` seçeneklerini statik olarak değil de asenkron olarak ayarlamak istediğinizde, `registerAsync()` metodunu kullanın. Çoğu dinamik modül gibi, Nest asenkron konfigürasyonla başa çıkmanız için birkaç teknik sağlar.

Bir teknik, bir fabrika fonksiyonu kullanmaktır:

```typescript
MulterModule.registerAsync({
  useFactory: () => ({
    dest: './upload',
  }),
});
```

Diğer [fabrika sağlayıcıları](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) gibi, fabrika fonksiyonumuz `async` olabilir ve `inject` aracılığıyla bağımlılıkları enjekte edebilir.

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    dest: configService.get<string>('MULTER_DEST'),
  }),
  inject: [ConfigService],
});
```

Alternatif olarak, `MulterModule`'u bir fabrika yerine bir sınıf kullanarak yapılandırabilirsiniz, aşağıda gösterildiği gibi:

```typescript
MulterModule.registerAsync({
  useClass: MulterConfigService,
});
```

Yukarıdaki yapının içinde, `MulterConfigService` sınıfının `MulterOptionsFactory` arabirimini uygulaması gerektiğine dikkat edin. `MulterModule`, sağlanan sınıfın örneği üzerinde `createMulterOptions()` metodunu çağırır.

```typescript
@Injectable()
class MulterConfigService implements MulterOptionsFactory {
  createMulterOptions(): MulterModuleOptions {
    return {
      dest: './upload',
    };
  }
}
```

`MulterModule` içinde özel bir kopya oluşturmak yerine mevcut bir seçenek sağlayıcısını yeniden kullanmak istiyorsanız, `useExisting` sözdizimini kullanın.

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/29-file-upload) bulunmaktadır.

