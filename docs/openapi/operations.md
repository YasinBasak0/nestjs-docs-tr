### Operasyonlar

OpenAPI terimleriyle, yollar (paths), API'nizin ortaya çıkardığı kaynaklardır ve operasyonlar, bu yolları manipüle etmek için kullanılan HTTP metotlarıdır, örneğin, `GET`, `POST` veya `DELETE`.

#### Etiketler (Tags)

Bir denetleyiciyi belirli bir etiketle ilişkilendirmek için `@ApiTags(...tags)` dekoratörünü kullanın.

```typescript
@ApiTags('cats')
@Controller('cats')
export class CatsController {}
```

#### Başlıklar (Headers)

İstek parçası olarak beklenen özel başlıkları tanımlamak için `@ApiHeader()` kullanın.

```typescript
@ApiHeader({
  name: 'X-MyHeader',
  description: 'Özel başlık',
})
@Controller('cats')
export class CatsController {}
```

#### Yanıtlar (Responses)

Özel bir HTTP yanıtını tanımlamak için `@ApiResponse()` dekoratörünü kullanın.

```typescript
@Post()
@ApiResponse({ status: 201, description: 'Kayıt başarıyla oluşturuldu.'})
@ApiResponse({ status: 403, description: 'Yasak.'})
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Nest, `@ApiResponse` dekoratöründen miras alan bir dizi kısa **API yanıtı** dekoratörü sağlar:

- `@ApiOkResponse()`
- `@ApiCreatedResponse()`
- `@ApiAcceptedResponse()`
- `@ApiNoContentResponse()`
- `@ApiMovedPermanentlyResponse()`
- `@ApiFoundResponse()`
- `@ApiBadRequestResponse()`
- `@ApiUnauthorizedResponse()`
- `@ApiNotFoundResponse()`
- `@ApiForbiddenResponse()`
- `@ApiMethodNotAllowedResponse()`
- `@ApiNotAcceptableResponse()`
- `@ApiRequestTimeoutResponse()`
- `@ApiConflictResponse()`
- `@ApiPreconditionFailedResponse()`
- `@ApiTooManyRequestsResponse()`
- `@ApiGoneResponse()`
- `@ApiPayloadTooLargeResponse()`
- `@ApiUnsupportedMediaTypeResponse()`
- `@ApiUnprocessableEntityResponse()`
- `@ApiInternalServerErrorResponse()`
- `@ApiNotImplementedResponse()`
- `@ApiBadGatewayResponse()`
- `@ApiServiceUnavailableResponse()`
- `@ApiGatewayTimeoutResponse()`
- `@ApiDefaultResponse()`

```typescript
@Post()
@ApiCreatedResponse({ description: 'Kayıt başarıyla oluşturuldu.'})
@ApiForbiddenResponse({ description: 'Yasak.'})
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Bir isteğin dönüş modelini belirtmek için bir sınıf oluşturmamız ve tüm özellikleri `@ApiProperty()` dekoratörü ile işaretlememiz gerekmektedir.

```typescript
export class Cat {
  @ApiProperty()
  id: number;

  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

Sonra, `Cat` modeli, yanıt dekoratörünün `type` özelliği ile birleştirilerek kullanılabilir.

```typescript
@ApiTags('cats')
@Controller('cats')
export class CatsController {
  @Post()
  @ApiCreatedResponse({
    description: 'Kayıt başarıyla oluşturuldu.',
    type: Cat,
  })
  async create(@Body() createCatDto: CreateCatDto): Promise<Cat> {
    return this.catsService.create(createCatDto);
  }
}
```

Tarayıcıyı açalım ve oluşturulan `Cat` modelini doğrulayalım:

<figure><img src="/assets/swagger-response-type.png" /></figure>

#### Dosya Yükleme

Dosya yükleme özelliğini belirli bir yöntem için `@ApiBody` dekoratörü ile birlikte `@ApiConsumes()` kullanarak etkinleştirebilirsiniz. İşte [Dosya Yükleme](/docs/techniques/file-upload) tekniğini kullanarak tam bir örnek:

```typescript
@UseInterceptors(FileInterceptor('file'))
@ApiConsumes('multipart/form-data')
@ApiBody({
  description: 'Kedi listesi',
  type: FileUploadDto,
})
uploadFile(@UploadedFile() file) {}
```

`FileUploadDto` aşağıdaki gibi tanımlanır:

```typescript
class FileUploadDto {
  @ApiProperty({ type: 'string', format: 'binary' })
  file: any;
}
```

Birden çok dosyanın yüklenmesiyle ilgileniyorsanız, `FilesUploadDto`'yu aşağıdaki gibi tanımlayabilirsiniz:

```typescript
class FilesUploadDto {
  @ApiProperty({ type: 'array', items: { type: 'string', format: 'binary' } })
  files: any[];
}
```

#### Uzantılar

Bir uzantıyı bir isteğe eklemek için `@ApiExtension()` dekoratörünü kullanın. Uzantı adı `x-` ile öne eklenmelidir.

```typescript
@ApiExtension('x-foo', { hello: 'world' })
```

#### Gelişmiş: Generic `ApiResponse`

[Raw Tanımlar](/docs/openapi/types-and-parameters#raw-definitions) sağlama yeteneği ile Swagger UI için Generic şema tanımlayabiliriz. Aşağıdaki DTO'ya sahip olduğumuzu varsayalım:

```ts
export class PaginatedDto<TData> {
  @ApiProperty()
  total: number;

  @ApiProperty()
  limit: number;

  @ApiProperty()
  offset: number;

  results: TData[];
}
```

`results`'u işaretlemiyoruz, çünkü daha sonra bunun için raw tanım sağlayacağız. Şimdi başka bir DTO tanımlayalım ve ona, örneğin, `CatDto` adını verelim:

```ts
export class CatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

Bunu yerine getirdikten sonra, bir `PaginatedDto<CatDto>` yanıtını şu şekilde tanımlayabiliriz:

```ts
@ApiOkResponse({
  schema: {
    allOf: [
      { $ref: getSchemaPath(PaginatedDto) },
      {
        properties: {
          results: {
            type: 'array',
            items: { $ref: getSchemaPath(CatDto) },
          },
        },
      },
    ],
  },
})
async findAll(): Promise<PaginatedDto<CatDto>> {}
```

Bu örnekte, yanıtın `PaginatedDto` ve `results` özelliğinin `Array<CatDto>` tipinde olacağını belirtiyoruz.

- `getSchemaPath()` fonksiyonu, bir model için OpenAPI Şema yolunu OpenAPI Spec Dosyası içinden döndürür.
- `allOf`, OAS 3'ün çeşitli Miras ile ilgili kullanım durumlarını kapsayan bir kavram sağlar.

Son olarak, `PaginatedDto` doğrudan hiçbir denetleyici tarafından referans edilmediğinden, `SwaggerModule` henüz bir karşılık gelen model tanımı oluşturamayacaktır. Bu durumda, bunu [Ek Model](/docs/openapi/types-and-parameters#extra-models) olarak eklemeliyiz. Örneğin, denetleyici düzeyinde `@ApiExtraModels()` dekoratörünü kullanabiliriz:

```ts
@Controller('cats')
@ApiExtraModels(PaginatedDto)
export class CatsController {}
```

Şimdi Swagger'ı çalıştırırsanız, bu özel uç nokta için oluşturulan `swagger.json`'ın aşağıdaki yanıtı tanımladığını görmelisiniz:

```json
"responses": {
  "200": {
    "description": "",
    "content": {
      "application/json": {
        "schema": {
          "allOf": [
            {
              "$ref": "#/components/schemas/PaginatedDto"
            },
            {
              "properties": {
                "results": {
                  "$ref": "#/components/schemas/CatDto"
                }
              }
            }
          ]
        }
      }
    }
  }
}
```

Bunu yeniden kullanılabilir hale getirmek için, `PaginatedDto` için özel bir dekoratör oluşturabiliriz:

```ts
export const ApiPaginatedResponse = <TModel extends Type<any>>(
  model: TModel,
) => {
  return applyDecorators(
    ApiExtraModels(PaginatedDto, model),
    ApiOkResponse({
      schema: {
        allOf: [
          { $ref: getSchemaPath(PaginatedDto) },
          {
            properties: {
              results: {
                type: 'array',
                items: { $ref: getSchemaPath(model) },
              },
            },
          },
        ],
      },
    }),
  );
};
```

> info **Hint** `Type<any>` arayüzü ve `applyDecorators` fonksiyonu, `@nestjs/common` paketinden içe aktarılır.

`SwaggerModule`'un model için bir tanım oluşturmasını sağlamak için, bunu önceki örnekte `PaginatedDto` ile yaptığımız gibi ek model olarak eklemeliyiz.

Bunu yerine getirdikten sonra, özel `@ApiPaginatedResponse()` dekoratörünü uç noktamızda kullanabiliriz:

```ts
@ApiPaginatedResponse(CatDto

)
async findAll(): Promise<PaginatedDto<CatDto>> {}
```

İstemci oluşturma araçları için, bu yaklaşım, `PaginatedResponse<TModel>`'in istemci için nasıl oluşturulduğu konusunda bir belirsizlik oluşturur. Aşağıdaki parça, yukarıdaki `GET /` uç noktası için istemci oluşturucu sonucunun bir örneğidir.

```typescript
// Angular
findAll(): Observable<{ total: number, limit: number, offset: number, results: CatDto[] }>
```

Görüldüğü gibi, burada **Dönüş Türü** belirsizdir. Bu sorunu çözmek için, `ApiPaginatedResponse` için şemaya bir `title` özelliği ekleyebiliriz:

```ts
export const ApiPaginatedResponse = <TModel extends Type<any>>(model: TModel) => {
  return applyDecorators(
    ApiOkResponse({
      schema: {
        title: `PaginatedResponseOf${model.name}`,
        allOf: [
          // ...
        ],
      },
    }),
  );
};
```

Şimdi, istemci oluşturucu aracının sonucu şu olacaktır:

```ts
// Angular
findAll(): Observable<PaginatedResponseOfCatDto>
```