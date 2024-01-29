### CLI Komut Referansı

#### nest new

Yeni bir (standart modlu) Nest projesi oluşturur.

```bash
$ nest new <name> [options]
$ nest n <name> [options]
```

##### Açıklama

Yeni bir Nest projesi oluşturur ve başlatır. Paket yöneticisi için sorma işlemi yapar.

- `<name>` adındaki bir klasör oluşturur
- Klasörü yapılandırma dosyalarıyla doldurur
- Kaynak kodu (`/src`) ve uçtan uca testler (`/test`) için alt klasörler oluşturur
- Alt klasörleri, uygulama bileşenleri ve testler için varsayılan dosyalarla doldurur

##### Argümanlar

| Argüman  | Açıklama                  |
| -------- | --------------------------- |
| `<name>` | Yeni proje adı              |

##### Seçenekler

| Seçenek                                | Açıklama                                                                                                                                                                                              |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--dry-run`                           | Yapılacak değişiklikleri raporlar, ancak dosya sistemini değiştirmez.<br/> Takma ad: `-d`                                                                                                             |
| `--skip-git`                          | Git deposu başlatma işlemini atlar.<br/> Takma ad: `-g`                                                                                                                                                 |
| `--skip-install`                      | Paket kurulumunu atlar.<br/> Takma ad: `-s`                                                                                                                                                          |
| `--package-manager [package-manager]` | Paket yöneticisini belirtir. `npm`, `yarn` veya `pnpm` kullanabilirsiniz. Paket yöneticisi global olarak kurulmalıdır.<br/> Takma ad: `-p`                                                                                  |
| `--language [language]`               | Programlama dilini belirtir (`TS` veya `JS`).<br/> Takma ad: `-l`                                                                                                                                        |
| `--collection [collectionName]`       | Şematik koleksiyonunu belirtir. Şematik içeren kurulu npm paketi paket adını kullanın.<br/> Takma ad: `-c`                                                                                      |
| `--strict`                            | Projenizi aşağıdaki TypeScript derleyici bayraklarıyla başlatır: `strictNullChecks`, `noImplicitAny`, `strictBindCallApply`, `forceConsistentCasingInFileNames`, `noFallthroughCasesInSwitch` |

#### nest generate

Bir şematik temel alınarak dosyalar oluşturur ve/veya değiştirir.

```bash
$ nest generate <schematic> <name> [options]
$ nest g <schematic> <name> [options]
```

##### Argümanlar

| Argüman      | Açıklama                                                                                              |
| ------------- | -------------------------------------------------------------------------------------------------------- |
| `<schematic>` | Oluşturulacak olan `schematic` veya `collection:schematic`. Kullanılabilir şematikler için aşağıdaki tabloya bakın. |
| `<name>`      | Oluşturulan bileşenin adı                                                                             |

##### Şematikler

| Ad            | Takma Ad | Açıklama                                                                                                           |
| ------------- | ----- | ----------------------------------------------------------------------------------------------------------------------|
| `app`         |       | Standart bir yapıya sahipse monorepo içinde yeni bir uygulama oluşturur (standart yapıdaysa monorepo'ya dönüştürülür).  |
| `library`     | `lib` | Standart bir yapıya sahipse monorepo içinde yeni bir kütüphane oluşturur (standart yapıdaysa monorepo'ya dönüştürülür). |
| `class`       | `cl`  | Yeni bir sınıf oluşturur.                                                                                            |
| `controller`  | `co`  | Bir denetleyici bildirimi oluşturur.                                                                               |
| `decorator`   | `d`   | Özel bir dekoratör oluşturur.                                                                                     |
| `filter`      | `f`   | Bir filtre bildirimi oluşturur.                                                                                    |
| `gateway`     | `ga`  | Bir ağ geçidi bildirimi oluşturur.                                                                               |
| `guard`       | `gu`  | Bir bekçi bildirimi oluşturur.                                                                                   |
| `interface`   | `itf` | Bir arayüz oluşturur.                                                                                            |
| `interceptor` | `itc` | Bir interceptor bildirimi oluşturur

.                                                                             |
| `middleware`  | `mi`  | Bir middleware bildirimi oluşturur.                                                                              |
| `module`      | `mo`  | Bir modül bildirimi oluşturur.                                                                                  |
| `pipe`        | `pi`  | Bir pipe bildirimi oluşturur.                                                                                   |
| `provider`    | `pr`  | Bir sağlayıcı bildirimi oluşturur.                                                                               |
| `resolver`    | `r`   | Bir çözücü bildirimi oluşturur.                                                                                 |
| `resource`    | `res` | Yeni bir CRUD kaynağı oluşturur. Daha fazla ayrıntı için [CRUD (resource) oluşturucusuna](/docs/recipes/crud-generator) bakın. (TS sadece)|
| `service`     | `s`   | Bir hizmet bildirimi oluşturur.                                                                                 |

##### Seçenekler

| Seçenek                          | Açıklama                                                                                                     |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `--dry-run`                     | Yapılacak değişiklikleri raporlar, ancak dosya sistemini değiştirmez.<br/> Takma ad: `-d`                        |
| `--project [project]`           | Bileşenin eklenmesi gereken proje.<br/> Takma ad: `-p`                                                       |
| `--flat`                        | Eleman için bir klasör oluşturma.                                                                       |
| `--collection [collectionName]` | Şematik koleksiyonunu belirtir. Kurulu npm paketinin paket adını kullanın.<br/> Takma ad: `-c` |
| `--spec`                        | Spec dosyalarının oluşturulmasını zorunlu kılar (varsayılan)                                                                         |
| `--no-spec`                     | Spec dosyalarının oluşturulmasını devre dışı bırakır                                                                                   |

#### nest build

Bir uygulamayı veya çalışma alanını bir çıkış klasörüne derler.

Ayrıca, `build` komutu şu sorumluluklardan sorumludur:

- yolları eşleme (path alias'leri kullanılıyorsa) via `tsconfig-paths`
- DTO'ları OpenAPI dekoratörleriyle işaretleme (eğer `@nestjs/swagger` CLI eklentisi etkinse)
- DTO'ları GraphQL dekoratörleriyle işaretleme (eğer `@nestjs/graphql` CLI eklentisi etkinse)

```bash
$ nest build <name> [options]
```

##### Argümanlar

| Argüman  | Açıklama                       |
| -------- | ------------------------------- |
| `<name>` | Derlenmek istenen proje adı.    |

##### Seçenekler

| Seçenek             | Açıklama                                                                                                                                                                                |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--path [path]`    | `tsconfig` dosyasının yolu. <br/>Takma adı `-p`                                                                                                                                       |
| `--config [path]`  | `nest-cli` yapılandırma dosyasının yolu. <br/>Takma adı `-c`                                                                                                                          |
| `--watch`          | İzleme modunda çalış (canlı yeniden yükleme).<br /> Derleme için `tsc` kullanıyorsanız, uygulamayı yeniden başlatmak için `rs` yazabilirsiniz (`manualRestart` seçeneği `true` olarak ayarlandığında). <br/>Takma adı `-w` |
| `--builder [name]` | Derleme için kullanılacak yapılandırıcıyı belirt ( `tsc`, `swc`, veya `webpack`). <br/>Takma adı `-b`                                                                                                   |
| `--webpack`        | Derleme için webpack kullan (deprecated: bunun yerine `--builder webpack` kullanın).                                                                                                                 |
| `--webpackPath`    | Webpack yapılandırmasının yolu.                                                                                                                                                             |
| `--tsc`            | Derleme için zorla `tsc` kullanımı.                                                                                                                                                           |

#### nest start

Bir uygulamayı derler ve çalıştırır (veya çalışma alanındaki varsayılan proje).

```bash
$ nest start <name> [options]
```

##### Argümanlar

| Argüman  | Açıklama                       |
| -------- | ------------------------------- |
| `<name>` | Çalıştırılmak istenen proje adı.|

##### Seçenekler

| Seçenek                  | Açıklama                                                                                                          |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `--path [path]`         | `tsconfig` dosyasının yolu. <br/>Takma adı `-p`                                                                    |
| `--config [path]`       | `nest-cli` yapılandırma dosyasının yolu. <br/>Takma adı `-c`                                                        |
| `--watch`               | İzleme modunda çalış (canlı yeniden yükleme). <br/>Takma adı `-w`                                                  |
| `--builder [name]`      | Derleme için kullanılacak yapılandırıcıyı belirt (`tsc`, `swc`, veya `webpack`). <br/>Takma adı `-b`           |
| `--preserveWatchOutput` | Ekranı temizleme yerine eski konsol çıkışını izleme modunda tutun. (`tsc` izleme modu için sadece)               |
| `--watchAssets`         | İzleme modunda çalış (canlı yeniden yükleme), TS olmayan dosyaları (varlıklar) izle. Daha fazla ayrıntı için [Assets](/docs/cli/workspaces#assets)'e bakın. |
| `--debug [hostport]`    | Hata ayıklama modunda çalış ( --inspect bayrağı ile). <br/>Takma adı `-d`                                        |
| `--webpack`             | Derleme için webpack kullan. (deprecated: bunun yerine `--builder webpack` kullanın)                              |
| `--webpackPath`         | Webpack yapılandırmasının yolu.                                                                                    |
| `--tsc`                 | Derleme için zorla `tsc` kullanımı.                                                                                |
| `--exec [binary]`       | Çalıştırılacak ikili (varsayılan: `node`). <br/>Takma adı `-e`                                                    |
| `-- [key=value]`        | `process.argv` ile başvurulabilen komut satırı argümanları.                                                      |

#### nest add

Bir **nest library** olarak paketlenmiş bir kütüphaneyi içeri alır ve yükleme şematiğini çalıştırır.

```bash
$ nest add <name> [options]
```

##### Argümanlar

| Argüman  | Açıklama                            |
| -------- | ---------------------------------- |
| `<name>` | İçeri alınacak kütüphanenin adı.    |

#### nest info

Yüklü nest paketleri ve diğer faydalı sistem bilgileri hakkında bilgi görüntüler. Örneğin:

```bash
$ nest info
```

```bash
 _   _             _      ___  _____  _____  _     _____
| \ | |           | |    |_  |/  ___|/  __ \| |   |_   _|
|  \| |  ___  ___ | |_     | |\ `--. | /  \/| |     | |
| . ` | / _ \/ __|| __|    | | `--. \| |    | |     | |
| |\  ||  __/\__ \| |_ /\__/ //\__/ /| \__/\| |_____| |_
\_| \_/ \___||___/ \__|\____/ \____/  \____/\_____/\___/

[Sistem Bilgisi]
İşletim Sistemi Sürümü : macOS High Sierra
NodeJS Sürümü : v16.18.0
[Nest Bilgisi]
microservices versiyonu : 10.0.0
websockets versiyonu : 10.0.0
testing versiyonu : 10.0.0
common versiyonu : 10.0.0
core versiyonu : 10.0.0
```