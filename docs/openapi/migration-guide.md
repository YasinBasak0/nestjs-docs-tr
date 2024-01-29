### Geçiş Kılavuzu

Eğer şu anda `@nestjs/swagger@3.*` kullanıyorsanız, sürüm 4.0'daki aşağıdaki kırılma/API değişikliklerine dikkat edin.

#### Kırılma Değişiklikleri

Aşağıdaki dekoratörler değiştirilmiş/yeniden adlandırılmıştır:

- `@ApiModelProperty` artık `@ApiProperty`
- `@ApiModelPropertyOptional` artık `@ApiPropertyOptional`
- `@ApiResponseModelProperty` artık `@ApiResponseProperty`
- `@ApiImplicitQuery` artık `@ApiQuery`
- `@ApiImplicitParam` artık `@ApiParam`
- `@ApiImplicitBody` artık `@ApiBody`
- `@ApiImplicitHeader` artık `@ApiHeader`
- `@ApiOperation({{ '{' }} title: 'test' {{ '}' }})` artık `@ApiOperation({{ '{' }} summary: 'test' {{ '}' }})`
- `@ApiUseTags` artık `@ApiTags`

`DocumentBuilder` kırılma değişiklikleri (güncellenmiş method imzaları):

- `addTag`
- `addBearerAuth`
- `addOAuth2`
- `setContactEmail` artık `setContact`
- `setHost` kaldırıldı
- `setSchemes` kaldırıldı (`addServer`'ı kullanın, örneğin, `addServer('http://')`)

#### Yeni Methodlar

Aşağıdaki methodlar eklenmiştir:

- `addServer`
- `addApiKey`
- `addBasicAuth`
- `addSecurity`
- `addSecurityRequirements`