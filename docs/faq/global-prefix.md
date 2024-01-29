### Global Prefix

Her HTTP uygulamasına kaydedilmiş **her yol** için bir önek belirlemek için, `INestApplication` örneğinin `setGlobalPrefix()` metodunu kullanın.

```typescript
const app = await NestFactory.create(AppModule);
app.setGlobalPrefix('v1');
```

Aşağıdaki yapıyı kullanarak global önekten rotaları hariç tutabilirsiniz:

```typescript
app.setGlobalPrefix('v1', {
  exclude: [{ path: 'health', method: RequestMethod.GET }],
});
```

Alternatif olarak, rotayı bir dize olarak belirleyebilirsiniz (bu, her istek yöntemine uygulanır):

```typescript
app.setGlobalPrefix('v1', { exclude: ['cats'] });
```

> info **İpucu** `path` özelliği, wildcard parametreleri [path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters) paketi kullanarak destekler. Not: bu, wildcard yıldızları `*` kabul etmez. Bunun yerine, parametreleri kullanmalısınız (örneğin, `(.*)`, `:splat*`).