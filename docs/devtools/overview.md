### Genel Bakış

> info **İpucu** Bu bölüm, Nest framework ile Nest Devtools entegrasyonunu kapsamaktadır. Eğer Devtools uygulamasını arıyorsanız, lütfen [Devtools](https://devtools.nestjs.com) web sitesini ziyaret edin.

Yerel uygulamanızı hata ayıklamaya başlamak için, `main.ts` dosyasını açın ve uygulama seçenekleri nesnesinde `snapshot` özelliğini şu şekilde `true` olarak ayarladığınızdan emin olun:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    snapshot: true,
  });
  await app.listen(3000);
}
```

Bu, çerçevenin uygulamanızın grafiğini görselleştirmek için Nest Devtools'ın kullanmasına olanak tanıyan gerekli metaveriyi toplamasını sağlayacaktır.

Ardından, gerekli bağımlılığı yükleyelim:

```bash
$ npm i @nestjs/devtools-integration
```

> warning **Uyarı** Eğer uygulamanızda `@nestjs/graphql` paketini kullanıyorsanız, en son sürümü yüklediğinizden emin olun (`npm i @nestjs/graphql@11`).

Bu bağımlılığı yerine koyduktan sonra, `app.module.ts` dosyasını açın ve yüklediğimiz `DevtoolsModule`'u içe aktarın:

```typescript
@Module({
  imports: [
    DevtoolsModule.register({
      http: process.env.NODE_ENV !== 'production',
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

> warning **Uyarı** Burada `NODE_ENV` ortam değişkenini kontrol etmemizin nedeni, bu modülü asla üretimde kullanmamanız gerektiğidir!

`DevtoolsModule` içe aktarıldığında ve uygulamanız çalıştığında (`npm run start:dev`), [Devtools](https://devtools.nestjs.com) URL'sine gidip içgörülen grafiği görmelisiniz.

<figure><img src="/assets/devtools/modules-graph.png" /></figure>

> info **İpucu** Yukarıdaki ekran görüntüsünde gördüğünüz gibi, her modül `InternalCoreModule`'a bağlanır. `InternalCoreModule`, her zaman kök modüle içe aktarılan genel bir modüldür. Kök modüle global bir düğüm olarak kaydedildiğinden, Nest otomatik olarak tüm modüller arasında ve `InternalCoreModule` düğümü arasında kenarlar oluşturur. Şimdi, global modülleri grafiğinizden gizlemek istiyorsanız, yan kenar çubuğundaki "**Global modülleri gizle**" onay kutusunu kullanabilirsiniz.

Görüldüğü gibi, `DevtoolsModule` uygulamanızın ek bir HTTP sunucusu (8000 numaralı port üzerinde) açmasını sağlar ve Devtools uygulamasının uygulamanızı içgörülemesi için kullanır.

Her şeyin beklenildiği gibi çalıştığından emin olmak için grafik görünümünü "Sınıflar" olarak değiştirin. Aşağıdaki ekranı görmelisiniz:

<figure><img src="/assets/devtools/classes-graph.png" /></figure>

Bir belirli düğüme odaklanmak için dikdörtgen üzerine tıklayın ve grafik, pencereyi gösteren **"Odak"** düğmesi ile birlikte bir açılır pencere gösterecektir. Ayrıca, belirli bir düğümü bulmak için yan kenar çubuğundaki arama çubuğunu (yan kenar çubuğunda bulunur) kullanabilirsiniz.

> info **İpucu** **İncele** düğmesine tıklarsanız, uygulama sizi belirli bir düğümle seçili **/debug** sayfasına götürecektir.

<figure><img src="/assets/devtools/node-popup.png" /></figure>

> info **İpucu** Bir grafiği resim olarak dışa aktarmak için, grafiğin sağ köşesindeki **PNG olarak Dışa Aktar** düğmesine tıklayın.

Sol taraftaki yan kenar çubuğundaki form kontrollerini kullanarak, örneğin belirli bir uygulama alt ağacını görselleştirmek için kenarların yakınlığını kontrol edebilirsiniz:

<figure><img src="/assets/devtools/subtree-view.png" /></figure>

Bu özellik, ekibinizde **yeni geliştiriciler** olduğunda ve uygulamanızın nasıl yapılandırıldığını göstermek istediğinizde özellikle kullanışlı olabilir. Ayrıca, büyük bir uygulamayı daha küçük

 modüllere (örneğin, bireysel mikro servisler) bölerken işe yarayabilecek belirli bir modülü (örneğin, `TasksModule`) ve tüm bağımlılıklarını görselleştirmek için bu özelliği kullanabilirsiniz.

**Graph Explorer** özelliğini görmek için bu videoyu izleyebilirsiniz:

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/bW8V-ssfnvM"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### "Cannot resolve dependency" Hatasını İnceleme

> info **Not** Bu özellik, `@nestjs/core` sürümü `v9.3.10` ve üstü için desteklenmektedir.

Muhtemelen gördüğünüz en yaygın hata mesajlarından biri, Nest'in bir sağlayıcının bağımlılıklarını çözememesi ile ilgilidir. Nest Devtools kullanarak sorunu kolayca tanımlayabilir ve nasıl çözeceğinizi öğrenebilirsiniz.

İlk olarak, `main.ts` dosyasını açın ve `bootstrap()` çağrısını aşağıdaki gibi güncelleyin:

```typescript
bootstrap().catch((err) => {
  fs.writeFileSync('graph.json', PartialGraphHost.toString() ?? '');
  process.exit(1);
});
```

Ayrıca, `abortOnError`'u `false` olarak ayarladığınızdan emin olun:

```typescript
const app = await NestFactory.create(AppModule, {
  snapshot: true,
  abortOnError: false, // <--- BURASI
});
```

Artık uygulamanızın **"Cannot resolve dependency"** hatası nedeniyle başlatılamadığı her seferde, kök dizinde `graph.json` (bir kısmi grafik temsil eden) dosyasını bulacaksınız. Bu dosyayı Devtools'a sürükleyip bırakabilirsiniz (mevcut modu "Interactive"den "Preview"e değiştirmeyi unutmayın):

<figure><img src="/assets/devtools/drag-and-drop.png" /></figure>

Başarılı bir yükleme sonrasında, aşağıdaki grafik ve iletişim penceresini görmelisiniz:

<figure><img src="/assets/devtools/partial-graph-modules-view.png" /></figure>

Görüldüğü gibi, vurgulanan `TasksModule` üzerinde durmamız gereken modüldür. Ayrıca, iletişim penceresinde bu sorunu nasıl düzelteceğimize dair bazı talimatları görebilirsiniz.

"Classes" görünümüne geçersek, şunları göreceğiz:

<figure><img src="/assets/devtools/partial-graph-classes-view.png" /></figure>

Bu grafik, `TasksService`'e enjekte etmek istediğimiz `DiagnosticsService`'in, `TasksModule` modülü bağlamında bulunamadığını ve bu sorunu çözmek için muhtemelen `TasksModule` modülüne `DiagnosticsModule`'u içe aktarmamız gerektiğini gösterir!

#### Rotalar Gezgini

**Rotalar Gezgini** sayfasına gidildiğinde, kaydedilmiş tüm giriş noktalarını görmelisiniz:

<figure><img src="/assets/devtools/routes.png" /></figure>

> info **İpucu** Bu sayfa sadece HTTP rotalarını değil, aynı zamanda diğer giriş noktalarını (örneğin WebSocket'ler, gRPC, GraphQL çözümleyicileri vb.) de gösterir.

Giriş noktaları, ana denetleyicilerine göre gruplandırılmıştır. Ayrıca belirli bir giriş noktasını bulmak için arama çubuğunu kullanabilirsiniz.

Belirli bir giriş noktasına tıklarsanız, bir **akış grafiği** görüntülenir. Bu grafik, giriş noktasının (örneğin, bu rotaya bağlı koruyucular, interceptor'lar, borular vb.) yürütme akışını gösterir. Bu özellikle belirli bir rota için istek/yanıt döngüsünün nasıl göründüğünü anlamak veya belirli bir koruyucu/interceptor/boru nedeninin neden yürütülmediğini sorun giderme konusunda yararlıdır.

#### Kum Havuzu

JavaScript kodunu anında yürütmek ve uygulamanızla gerçek zamanlı etkileşimde bulunmak için **Kum Havuzu** sayfasına gidin:

<figure><img src="/assets/devtools/sandbox.png" /></figure>

Oyun alanı, API uç noktalarını **gerçek zamanlı** test etmek ve hızlı bir şekilde sorunları tanımlayıp düzeltmek için kullanılabilir; bu, örneğin, bir HTTP istemcisi kullanmadan. Ayrıca, kimlik doğrulama katmanını atlayabiliriz, bu nedenle artık giriş yapma veya test amaçları için özel bir kullanıcı hesabına ihtiyaç duymayız. Olaya dayalı uygulamalar için, oyun alanından doğrudan olayları tetikleyebilir ve uygulamanın onlara nasıl tepki verdiğini görebiliriz.

Gelişmeler, oyun alanının konsoluna düşen her şeyi düzenleyerek, neler olduğunu kolayca görmemize olanak tanır.

Kodu **anında** yürütün ve sonuçları hemen görün, uygulamayı yeniden oluşturmak ve sunucuyu yeniden başlatmak zorunda kalmadan.

<figure><img src="/assets/devtools/sandbox-table.png" /></figure>

> info **İpucu** Nesne dizisini güzel bir şekilde görüntülemek için `console.table()` (veya sadece `table()`) fonksiyonunu kullanın.

Bu özelliği görmek için bu videoyu izleyebilirsiniz: 

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/liSxEN_VXKM"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### Başlatma Performans Analizcisi

Tüm sınıf düğümlerinin (denetleyiciler, sağlayıcılar, geliştiriciler vb.) ve karşılık gelen oluşturma sürelerinin bir listesini görmek için **Başlatma Performansı** sayfasına gidin:

<figure><img src="/assets/devtools/bootstrap-performance.png" /></figure>

Bu sayfa, uygulamanızın başlatma sürecinin en yavaş kısımlarını tanımlamak istediğinizde (örneğin, özellikle, örneğin, serverless ortamlar için kritik olan uygulamanın başlatma süresini optimize etmek istediğinizde) oldukça kullanışlıdır.

#### Denetim

Otomatik olarak oluşturulan denetimi görmek - uygulamanızın serileştirilmiş grafiğinizi analiz ederken ortaya çıkan hatalar/uyarılar/ipuçları - için **Denetim** sayfasına gidin:

<figure><img src="/assets/devtools/audit.png" /></figure>

> info **İpucu** Yukarıdaki ekran görüntüsü mevcut denetim kurallarının tamamını göstermemektedir.

Bu sayfa, uygulamanızdaki potansiyel sorunları belirlemek istediğinizde işe yarar.

#### Önizleme statik dosyalar

Bir serileştirilmiş grafiği bir dosyaya kaydetmek için şu kodu kullanın:

```typescript
await app.listen(3000); // VEYA await app.init()
fs.writeFileSync('./graph.json', app.get(SerializedGraph).toString());
```

> info **İpucu** `SerializedGraph`, `@nestjs/core` paketinden ihraç edilmiştir.

Ardından bu dosyayı sürükleyip bırakabilir veya yükleyebilirsiniz:

<figure><img src="/assets/devtools/drag-and-drop.png" /></figure>

Bu, grafiğinizi başka biriyle (örneğin, iş arkadaşı) paylaşmak veya çevrimdışı olarak analiz etmek istediğinizde faydalıdır.