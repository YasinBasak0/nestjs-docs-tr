### Güvenlik

Belirli bir işlem için hangi güvenlik mekanizmalarının kullanılması gerektiğini tanımlamak için `@ApiSecurity()` dekoratörünü kullanın.

```typescript
@ApiSecurity('basic')
@Controller('cats')
export class CatsController {}
```

Uygulamanızı çalıştırmadan önce, güvenlik tanımınızı `DocumentBuilder` kullanarak temel belgenize eklemeyi unutmayın:

```typescript
const options = new DocumentBuilder().addSecurity('basic', {
  type: 'http',
  scheme: 'basic',
});
```

Bazı en popüler kimlik doğrulama teknikleri (örneğin, `basic` ve `bearer`) yerleşik olduğu için yukarıda gösterildiği gibi güvenlik mekanizmalarını manuel olarak tanımlamanıza gerek yoktur.

#### Temel kimlik doğrulama

Temel kimlik doğrulamayı etkinleştirmek için `@ApiBasicAuth()` kullanın.

```typescript
@ApiBasicAuth()
@Controller('cats')
export class CatsController {}
```

Uygulamanızı çalıştırmadan önce, güvenlik tanımınızı `DocumentBuilder` kullanarak temel belgenize eklemeyi unutmayın:

```typescript
const options = new DocumentBuilder().addBasicAuth();
```

#### Taşıyıcı kimlik doğrulama

Taşıyıcı kimlik doğrulamayı etkinleştirmek için `@ApiBearerAuth()` kullanın.

```typescript
@ApiBearerAuth()
@Controller('cats')
export class CatsController {}
```

Uygulamanızı çalıştırmadan önce, güvenlik tanımınızı `DocumentBuilder` kullanarak temel belgenize eklemeyi unutmayın:

```typescript
const options = new DocumentBuilder().addBearerAuth();
```

#### OAuth2 kimlik doğrulama

OAuth2'yi etkinleştirmek için `@ApiOAuth2()` kullanın.

```typescript
@ApiOAuth2(['pets:write'])
@Controller('cats')
export class CatsController {}
```

Uygulamanızı çalıştırmadan önce, güvenlik tanımınızı `DocumentBuilder` kullanarak temel belgenize eklemeyi unutmayın:

```typescript
const options = new DocumentBuilder().addOAuth2();
```

#### Çerez kimlik doğrulama

Çerez kimlik doğrulamayı etkinleştirmek için `@ApiCookieAuth()` kullanın.

```typescript
@ApiCookieAuth()
@Controller('cats')
export class CatsController {}
```

Uygulamanızı çalıştırmadan önce, güvenlik tanımınızı `DocumentBuilder` kullanarak temel belgenize eklemeyi unutmayın:

```typescript
const options = new DocumentBuilder().addCookieAuth('optional-session-id');
```
