### Genel Bakış

[Nest CLI](https://github.com/nestjs/nest-cli), Nest uygulamalarınızı başlatmanıza, geliştirmenize ve sürdürmenize yardımcı olan bir komut satırı arayüzü aracıdır. Projeyi şablona dökme, geliştirme modunda sunma ve uygulamayı üretim dağıtımı için derleme ve paketleme gibi çeşitli yollarla yardımcı olur. İyi yapılandırılmış uygulamaları teşvik etmek için en iyi uygulama mimari kalıplarını içerir.

#### Kurulum

**Not**: Bu kılavuzda, Nest CLI dahil paketleri yüklemek için [npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm) kullanımını açıklıyoruz. Başka paket yöneticilerini tercih edebilirsiniz. npm ile, `nest` CLI ikili dosyasının konumunu OS komut satırının nasıl çözeceğini yönetmek için birkaç seçeneğiniz vardır. Burada, `nest` ikilisini `-g` seçeneği kullanarak global olarak yüklemeyi açıklıyoruz. Bu, bir ölçüde kolaylık sağlar ve belgeler boyunca varsaydığımız yaklaşımdır. Herhangi bir `npm` paketini global olarak yüklemenin, kullanıcının doğru sürümü çalıştığından emin olma sorumluluğunu bıraktığını ve her projenin aynı CLI sürümünü çalıştırdığı anlamına geldiğini unutmayın. Mantıklı bir alternatif, Nest CLI'nin **yönetilen bir sürümünü** çalıştırmayı sağlamak için, `npm` cli içine yerleştirilmiş [npx](https://github.com/npm/cli/blob/latest/docs/lib/content/commands/npx.md) programını (veya diğer paket yöneticilerinde benzer özellikleri) kullanmaktır. Daha fazla bilgi için [npx belgelerine](https://github.com/npm/cli/blob/latest/docs/lib/content/commands/npx.md) veya DevOps destek personelinize başvurmanızı öneririz.

CLI'yi şu komutla global olarak yükleyin (`-g` seçeneği hakkında detaylar için yukarıdaki **Not** bölümüne bakın):

```bash
$ npm install -g @nestjs/cli
```

> Bilgi **İpucu** Alternatif olarak, `npx @nestjs/cli@latest` komutunu global olarak CLI'yi yüklemeden kullanabilirsiniz.

#### Temel Çalışma Akışı

Yüklendikten sonra, CLI komutlarını doğrudan OS komut satırından `nest` yürütülebilir dosyası aracılığıyla çağırabilirsiniz. Kullanılabilir `nest` komutlarını görmek için şu komutu girin:

```bash
$ nest --help
```

Bir komut hakkında yardım almak için aşağıdaki yapısı kullanın. Aşağıdaki örnekte olduğu gibi `new`, `add` vb. gibi herhangi bir komutu, `generate` örneğinde görüldüğü gibi yerine koyarak o komut hakkında detaylı yardım alabilirsiniz:

```bash
$ nest generate --help
```

Yeni bir temel Nest projesi oluşturmak, derleme modunda çalıştırmak ve çalıştırmak için, yeni proje oluşturulacak dizinin üst dizinine gidin ve aşağıdaki komutları çalıştırın:

```bash
$ nest new my-nest-project
$ cd my-nest-project
$ npm run start:dev
```

Tarayıcınızda [http://localhost:3000](http://localhost:3000) adresini açarak yeni uygulamanın çalıştığını görebilirsiniz. Uygulama, herhangi bir kaynak dosyasını değiştirdiğinizde otomatik olarak derlenir ve yeniden yüklenir.

> Bilgi **İpucu** Daha hızlı derlemeler için [SWC derleyicisini](/docs/recipes/swc) kullanmanızı öneririz (varsayılan TypeScript derleyicisine göre 10 kat daha performanslı).

### Proje Yapısı

`nest new` komutunu çalıştırdığınızda, Nest yeni bir klasör oluşturarak ve başlangıç dosyalarını ekleyerek bir temel uygulama yapısı oluşturur. Bu varsayılan yapıda çalışmaya devam edebilir ve bu belgeler boyunca açıklanan şekilde yeni bileşenler ekleyebilirsiniz. `nest new` tarafından oluşturulan proje yapısına **standart mod** adını veriyoruz. Nest aynı zamanda, **monorepo modu** adı verilen ve birden çok projeyi ve kütüphaneyi yönetmek için kullanılan başka bir yapıyı da destekler.

**Yapı** sürecinin nasıl çalıştığına dair birkaç belirli düşünce dışında (temelde, monorepo modu zaman zaman monorepo tarzı proje yapılarından kaynaklanabilen derleme karmaşıklıklarını basitleştirir), ve yerleşik [kütüphane](/docs/cli/libraries) desteği dışında, Nest özellikleri ve bu belgeler standart ve monorepo modu proje yapılarına eşit şekilde uygulanır. Aslında, istediğiniz zaman standart moddan monorepo moduna geçebilirsiniz, bu nedenle Nest hakkında daha fazla bilgi edinirken bu kararı güvenle erteleyebilirsiniz.

Birden çok projeyi yönetmek için her iki modu da kullanabilirsiniz. İşte farklarının hızlı bir özeti:

| Özellik                                               | Standart Mod                                                      | Monorepo Modu                                              |
| ----------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------- |
| Birden çok proje                                      | Ayrı dosya sistem yapıları                                       | Tek dosya sistem yapısı                                    |
| `node_modules` ve `package.json`                      | Ayrı örnekler                                                      | Monorepo genelinde paylaşılan                               |
| Varsayılan derleyici                                   | `tsc`                                                              | webpack                                                    |
| Derleyici ayarları                                     | Ayrı ayrı belirtilir                                               | Monorepo için varsayılanlar, projeye göre geçersiz kılınabilir   |
| `.eslintrc.js`, `.prettierrc` gibi yapılandırma dosyaları | Ayrı ayrı belirtilir                                               | Monorepo genelinde paylaşılan                               |
| `nest build` ve `nest start` komutları                | Hedef varsayılanı otomatik olarak bağlamdaki (tek) projeye yönlendirilir | Hedef varsayılanı monorepo içindeki **varsayılan proje**'ye yönlendirilir |
| Kütüphaneler                                           | Genellikle npm paketleme aracılığıyla manuel olarak yönetilir      | Yol yönetimi ve paketleme dahil, yerleşik destek          |

Hangi modun sizin için daha uygun olduğuna karar vermenize yardımcı olacak daha ayrıntılı bilgi için [Workspaces](/docs/cli/monorepo) ve [Libraries](/docs/cli/libraries) bölümlerini okuyun.

<app-banner-courses></app-banner-courses>

#### CLI Komut Sözdizimi

Tüm `nest` komutları aynı formatta izlenir:

```bash
nest commandOrAlias requiredArg [optionalArg] [options]
```

Örneğin:

```bash
$ nest new my-nest-project --dry-run
```

Burada `new` _komutVeyaAlias_'tir. `new` komutunun bir `n` kısaltması vardır. `my-nest-project` _zorunluArg_'dir. Eğer _zorunluArg_ komut satırına verilmezse, `nest` kullanıcıdan bunu girmesini ister. Ayrıca, `--dry-run`'ın kısaltılmış formu `-d`'dir. Bu bilgilerle birlikte, aşağıdaki komut yukarıdakiyle aynıdır:

```bash
$ nest n my-nest-project -d
```

Çoğu komut ve bazı seçeneklerin kısaltmaları vardır. Bu seçenekleri ve kısaltmaları görmek ve yukarıdaki yapılar hakkındaki anlayışınızı doğrulamak için `nest new --help` komutunu çalıştırın.

#### Komut Genel Bakışı

Herhangi bir komut için `nest <komut> --help` komutunu çalıştırarak komuta özgü seçenekleri görebilirsiniz.

Her komut için ayrıntılı açıklamalar için [kullanım](/docs/cli/usages) bölümüne bakın.

| Komut      | Kısaltma | Açıklama                                                                                            |
| ---------- | -------- | --------------------------------------------------------------------------------------------------- |
| `new`      | `n`      | Çalıştırılacak tüm başlangıç dosyalarına sahip yeni bir _standart mod_ uygulaması oluşturur.          |
| `generate` | `g`      | Bir şemaya dayalı olarak dosyaları oluşturur ve/veya değiştirir.                                     |
| `build`    |          | Bir uygulamayı veya workspaces'ı bir çıkış klasörüne derler.                                        |
| `start`    |          | Bir uygulamayı (veya bir workspaces içindeki varsayılan projeyi) derler ve çalıştırır.              |
| `add`      |          | Bir **nest kütüphanesi** olarak paketlenmiş bir kütüphaneyi içe aktarır, yükleme şematiğini çalıştırır. |
| `info`     | `i`      | Yüklü nest paketleri ve diğer yararlı sistem bilgileri hakkında bilgi gösterir.                      |

#### Gereksinimler

Nest CLI, [uluslararasılaştırma desteği](https://nodejs.org/api/intl.html) (ICU) içeren bir Node.js ikilisi gerektirir, örneğin [Node.js proje sayfasından](https://nodejs.org/en/download) resmi ikililer. ICU ile ilgili hatalarla karşılaşırsanız, ikilinizin bu gereksinimi karşıladığını kontrol edin.

```bash
node -p process.versions.icu
```

Eğer komut `undefined` yazdırıyorsa, Node.js ikilinizde uluslararasılaştırma desteği bulunmamaktadır.