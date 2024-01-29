### Router modülü

> info **İpucu** Bu bölüm, yalnızca HTTP tabanlı uygulamalar için geçerlidir.

HTTP uygulamasında (örneğin, REST API), bir işleme (handler) ait rota yolu, denetleyici için belirtilen (isteğe bağlı) ön ek ( `@Controller` dekoratörü içinde),
ve yöntemin dekoratöründe belirtilen yolun (örneğin, `@Get('users')`) birleştirilmesiyle belirlenir. Bu konuda daha fazla bilgi edinebilirsiniz [bu bölümde](/docs/controllers#routing). Ayrıca,
uygulamanızda kayıtlı tüm rotalar için [global bir ön ek](/docs/faq/global-prefix) tanımlayabilir veya [sürümleme](/docs/techniques/versioning) etkinleştirebilirsiniz.

Ayrıca, bir modül düzeyinde bir ön ek tanımlamanın (ve bu modül içinde kayıtlı tüm denetleyiciler için) yararlı olabileceği durumlar vardır.
Örneğin, "Dashboard" adlı uygulama bölümü tarafından kullanılan birkaç farklı uç noktayı açıklayan bir REST uygulamasını hayal edin.
Bu durumda, her denetleyici içinde `/dashboard` ön ekini tekrarlamak yerine, aşağıdaki gibi bir yardımcı `RouterModule` modülü kullanabilirsiniz:

```typescript
@Module({
  imports: [
    DashboardModule,
    RouterModule.register([
      {
        path: 'dashboard',
        module: DashboardModule,
      },
    ]),
  ],
})
export class AppModule {}
```

> info **İpucu** `RouterModule` sınıfı, `@nestjs/core` paketinden içe aktarılmıştır.

Ayrıca, hiyerarşik yapılar tanımlayabilirsiniz. Bu, her modülün `children` (çocuk) modüllere sahip olabileceği anlamına gelir.
Çocuk modüller, üst modülün ön ekini devralır. Aşağıdaki örnekte, `AdminModule`'u `DashboardModule` ve `MetricsModule`'un üst modülü olarak kaydedeceğiz.

```typescript
@Module({
  imports: [
    AdminModule,
    DashboardModule,
    MetricsModule,
    RouterModule.register([
      {
        path: 'admin',
        module: AdminModule,
        children: [
          {
            path: 'dashboard',
            module: DashboardModule,
          },
          {
            path: 'metrics',
            module: MetricsModule,
          },
        ],
      },
    ])
  ],
});
```

> info **İpucu** Bu özellik, çok dikkatlice kullanılmalıdır, aşırı kullanım, kodun zamanla bakımını zorlaştırabilir.

Yukarıdaki örnekte, `DashboardModule` içine kaydedilen her denetleyiciye ek bir `/admin/dashboard` ön eki eklenecektir (çünkü modül, üstten alta - yani, üst modülden çocuk modüle doğru - rekürsif olarak yolları birleştirir).
Benzer şekilde, `MetricsModule` içinde tanımlanan her denetleyiciye ek bir modül düzeyi ön ek olan `/admin/metrics` eklenecektir.