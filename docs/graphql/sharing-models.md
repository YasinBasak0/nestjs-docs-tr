### Modelleri Paylaşma

> warning **Uyarı** Bu bölüm yalnızca kod ilk yaklaşımına uygulanır.

Projenizin backend'i için Typescript kullanmanın en büyük avantajlarından biri, aynı modelleri bir Typescript tabanlı bir frontend uygulamasında, ortak bir Typescript paketi kullanarak yeniden kullanma yeteneğidir.

Ancak bir sorun var: kod ilk yaklaşımını kullanarak oluşturulan modeller, GraphQL ile ilgili dekoratörlerle ağır şekilde dekore edilmiştir. Bu dekoratörler, frontend tarafında anlamsızdır ve performansı olumsuz etkiler.

#### Model Shim Kullanma

Bu sorunu çözmek için, NestJS, orijinal dekoratörleri inert kod ile değiştirmenizi sağlayan bir "shim" sağlar ve bunu `webpack` (veya benzeri) konfigürasyonu kullanarak yapabilirsiniz.
Bu shim'i kullanmak için, `@nestjs/graphql` paketi ile shim arasında bir takma ad yapılandırın.

Örneğin, webpack için bunun çözüldüğü yol şöyle:

```typescript
resolve: { // bkz: https://webpack.js.org/configuration/resolve/
  alias: {
      "@nestjs/graphql": path.resolve(__dirname, "../node_modules/@nestjs/graphql/dist/extra/graphql-model-shim")
  }
}
```

> info **İpucu** [TypeORM](/docs/techniques/database) paketi, benzer bir shim'e sahiptir ve [burada](https://github.com/typeorm/typeorm/blob/master/extra/typeorm-model-shim.js) bulunabilir.