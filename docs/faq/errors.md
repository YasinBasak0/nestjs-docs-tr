### Ortak Hatalar

NestJS geliştirmeleriniz sırasında, çeşitli hatalarla karşılaşabilirsiniz ve bu hataları çözmeyi öğrenirken farklı senaryolarla karşılaşabilirsiniz.

#### "Cannot resolve dependency" Hatası

> info **Öneri** [NestJS Devtools](/docs/devtools/overview#investigating-the-cannot-resolve-dependency-error) sayfasını inceleyerek "Cannot resolve dependency" hatasını çözmekte yardımcı olabilirsiniz.

Muhtemelen karşılaştığınız en yaygın hata, Nest'in bir sağlayıcının bağımlılıklarını çözemediği hakkındadır. Hata mesajı genellikle şu şekildedir:

```bash
Nest can't resolve dependencies of the <provider> (?). Please make sure that the argument <unknown_token> at index [<index>] is available in the <module> context.

Potential solutions:
- Is <module> a valid NestJS module?
- If <unknown_token> is a provider, is it part of the current <module>?
- If <unknown_token> is exported from a separate @Module, is that module imported within <module>?
  @Module({
    imports: [ /* the Module containing <unknown_token> */ ]
  })
```

En yaygın sorun, `<provider>`'ı modülün `providers` dizisine eklememenizdir. Lütfen sağlayıcının gerçekten `providers` dizisinde olduğundan ve [standart NestJS sağlayıcı uygulamalarını](/docs/fundamentals/custom-providers#di-fundamentals) takip ettiğinden emin olun.

Yaygın olan birkaç şey var. Bir tanesi bir sağlayıcıyı bir `imports` dizisine koymaktır. Bu durumda, hata sağlayıcının adını `<module>`'un olması gereken yerde içerir.

Bu hatayla karşılaşırsanız, hata mesajındaki modülü inceleyin ve `providers`'larına bakın. `providers` dizisindeki her sağlayıcı için, modülün tüm bağımlılıklara erişim sağladığından emin olun. Çoğu zaman, `providers`, "Özellik Modülü" ve "Kök Modül" olarak iki kez sahip olduğu için Nest, sağlayıcıyı iki kez örneklemeye çalışacaktır. `<provider>`'ı çoğaltılan modülün "Kök Modülü"'nün `imports` dizisine eklenmesi daha olasıdır.

Yukarıda belirtilen `<unknown_token>` string ise `dependency` ise, muhtemelen döngüsel bir dosya içe aktarması vardır. Bu, aşağıdaki [döngüsel bağımlılık](/docs/faq/common-errors#circular-dependency-error) ile karıştırmadan farklıdır çünkü sağlayıcılar sırasıyla birbirine bağlı olmak yerine, iki dosyanın birbirini içe aktarması anlamına gelir. Yaygın bir durum, bir modül dosyasının bir belirteç bildirme ve bir sağlayıcı içe aktarma ve sağlayıcının, modül dosyasından belirteci almak için modül dosyasından içe aktarma olabilir. Barrel dosyaları kullanıyorsanız, barrel içe aktarmalarınızın da bu döngüsel içe aktarmalara neden olmamasına dikkat edin.

Yukarıda belirtilen `<unknown_token>` string ise `Object` ise, muhtemelen bir sağlayıcı belirtisi olmadan bir tür/schnitt/interface kullanıyorsunuz demektir. Bunun için, sınıf referansını doğru bir şekilde içe aktardığınızdan veya `@Inject()` dekoratörü ile özel bir belirteç kullanıyor olduğunuzdan emin olun. [Özel sağlayıcılar sayfasını](/docs/fundamentals/custom-providers) okuyun.

Ayrıca, sağlayıcıyı kendisine enjekte etmediğinizden emin olun, çünkü NestJS'de kendine enjeksiyonlar izin verilmez. Bu durumda, `<unknown_token>` muhtemelen `<provider>` ile aynı olacaktır.

**Monorepo kurulumu** kullanıyorsanız, yukarıdaki hatayla aynı hatayla karşılaşabilirsiniz, ancak `<unknown_token>` adında çekirdek sağlayıcı olan `ModuleRef` için:

```bash
Nest can't resolve dependencies of the <provider> (?).
Please make sure that the argument ModuleRef at index [<index>] is available in the <module> context.
...
```

Bu muhtemelen projenizin `@nestjs/core` paketinin iki Node modülünü yüklemeye çalışması durumunda ortaya çıkar, örneğin:

```text
.
├── package.json
├── apps
│   └── api
│       └── node_modules
│           └── @nestjs/bull
│               └── node_modules
│                   └── @nestjs/core
└── node_modules
    ├── (diğer paketler)
    └── @nestjs/core
```

Çözümler:

- **Yarn** Workspaces için, paketi `@nestjs/core`'yi kaldırmak için [nohoist özelliğini](https://classic.yarnpkg.com/blog/2018/02/15/nohoist) kullanın.
- **pnpm** Workspaces için, `@nestjs/core`'yi diğer modülde peerDependencies olarak ayarlayın ve modülün içe aktarıldığı app package.json'da `"dependenciesMeta": "{{ '{' }}"` other-module

- `"name": "{{ '{' }}"injected": true{{ '}}' }}`'yi ayarlayın. bkz: [dependenciesmetainjected](https://pnpm.io/package_json#dependenciesmetainjected)

#### "Circular dependency" Hatası

Ara sıra uygulamanızda [döngüsel bağımlılıklardan](https://docs.nestjs.com/fundamentals/circular-dependency) kaçınılmaz olabilir. Bu durumu çözmek için bazı adımlar atmanız gerekecek. Döngüsel bağımlılıklardan kaynaklanan hatalar şu şekildedir:

```bash
Nest cannot create the <module> instance.
The module at index [<index>] of the <module> "imports" array is undefined.

Potential causes:
- A circular dependency between modules. Use forwardRef() to avoid it. Read more: https://docs.nestjs.com/fundamentals/circular-dependency
- The module at index [<index>] is of type "undefined". Check your import statements and the type of the module.

Scope [<module_import_chain>]
# example chain AppModule -> FooModule
```

Döngüsel bağımlılıklar, hem sağlayıcıların birbirine bağlı olmasından kaynaklanabilir hem de TypeScript dosyalarının sabitler için birbirine bağlı olmasından kaynaklanabilir, örneğin bir modül dosyasından sabitleri dışa aktarma ve bir servis dosyasında bunları içe aktarma. İkinci durumda, sabitleriniz için ayrı bir dosya oluşturmanız önerilir. İlk durumda ise, lütfen döngüsel bağımlılıklar konusundaki kılavuzu takip edin ve hem modüllerin hem de sağlayıcıların `forwardRef` ile işaretlendiğinden emin olun.

#### Bağımlılık Hatalarını Hata Ayıklama

Nest 8.1.0 sürümü ile birlikte, bağımlılıkların çözümü sırasında Nest'in tüm bağımlılıkları çözümleme sürecinde ekstra günlük bilgileri almak için `NEST_DEBUG` ortam değişkenini doğru sonuçlanan bir dizeye ayarlayabilirsiniz.

<figure><img src="/assets/injector_logs.png" /></figure>

Yukarıdaki resimde, sarı renkteki dize enjekte edilen bağımlılığın ana sınıfını, mavi renkteki dize enjekte edilen bağımlılığın adını veya enjeksiyon belirtecesini ve mor renkteki dize bağımlılığın arandığı modülü temsil eder. Bu kullanarak genellikle neyin olduğunu ve neden bağımlılık enjeksiyonu sorunları yaşadığınızı geri izleyebilirsiniz.

#### "File change detected" Sonsuz Döngü

Windows kullanıcıları, TypeScript sürümü 4.9 ve üstünü kullandığında bu sorunla karşılaşabilir. Bu sorunla karşılaşıyorsanız, uygulamanızı izleme modunda çalıştırmaya çalıştığınızda, örneğin `npm run start:dev`, ve aşağıdaki log mesajlarının sonsuz bir döngüsünü görüyorsanız:

```bash
XX:XX:XX AM - File change detected. Starting incremental compilation...
XX:XX:XX AM - Found 0 errors. Watching for file changes.
```

Uygulamanızı izleme modunda başlatmak için NestJS CLI'yi kullandığınızda, `tsc --watch`'i çağırarak yapılır ve TypeScript'in 4.9 sürümünden itibaren dosya değişikliklerini algılama için yeni bir strateji kullanılır, bu muhtemelen bu sorunun nedenidir.
Bu sorunu düzeltmek için, tsconfig.json dosyanıza, `"compilerOptions"` seçeneğinden sonra aşağıdaki gibi bir ayar eklemeniz gerekiyor:

```bash
  "watchOptions": {
    "watchFile": "fixedPollingInterval"
  }
```

Bu, TypeScript'e dosya değişikliklerini kontrol etmek için dosya sistem olayları yerine anketleme yöntemini kullanmasını söyler (yeni varsayılan yöntem), bu da bazı makinelerde sorunlara neden olabilir.
"watchFile" seçeneği hakkında daha fazla bilgiyi [TypeScript belgelerinde](https://www.typescriptlang.org/tsconfig#watch-watchDirectory) bulabilirsiniz.