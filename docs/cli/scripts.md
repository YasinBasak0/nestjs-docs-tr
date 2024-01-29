### Nest CLI ve Betikler

Bu bölüm, `nest` komutunun derleyiciler ve betiklerle nasıl etkileşimde bulunduğuna dair ek bilgiler sunar ve Geliştirme ve İşletme (DevOps) personelinin geliştirme ortamını yönetmelerine yardımcı olur.

Bir Nest uygulaması, çalıştırılmadan önce JavaScript'e derlenmesi gereken **standart** bir TypeScript uygulamasıdır. Derleme adımını gerçekleştirmek için çeşitli yöntemler vardır ve geliştiriciler/ekipler, kendileri için en iyi çalışan bir yöntemi seçme özgürlüğüne sahiptir. Bu bağlamda, Nest aşağıdakileri yapmayı amaçlayan bir dizi araç seti sunar:

- Mantıklı varsayılanlarla "çalışan" bir standart derleme/çalıştırma süreci, komut satırında kullanılabilir.
- Derleme/çalıştırma sürecinin **açık** olmasını sağlamak, böylece geliştiriciler, bunları doğal özellikler ve seçenekler kullanarak özelleştirebilirler.
- Tamamen standart bir TypeScript/Node.js çerçevesi olmaya devam etmek, böylece tüm derleme/dağıtım/çalıştırma boru hattını geliştirme ekibinin kullanmayı seçtiği harici araçlarla yönetebilmesini sağlamak.

Bu hedef, `nest` komutu, yerel olarak yüklü TypeScript derleyicisi ve `package.json` betiklerinin bir kombinasyonu ile başarılır. Bu teknolojilerin bir arada nasıl çalıştığını aşağıda açıklıyoruz. Bu, derleme/çalıştırma sürecinin her adımında neler olduğunu ve gerektiğinde bu davranışı nasıl özelleştirebileceğinizi anlamanıza yardımcı olmalıdır.

#### `nest` İkili Dosyası

`nest` komutu, bir işletim sistemi seviyesi ikili dosyadır (yani, işletim sistemi komut satırından çalışır). Bu komut aslında aşağıda açıklanan 3 ayrı alanı kapsar. Bir projenin çatısını örneklediğinizde (`nest build` ve `nest start` komutları), bunu `package.json` betikleri aracılığıyla yapmanızı öneririz.

#### Derleme

`nest build`, standart `tsc` derleyicisi veya `swc` derleyicisi (for [standart projeler](https://docs.nestjs.com/cli/overview#project-structure)) veya `ts-loader` kullanarak webpack paketleyicisi (for [monorepo](https://docs.nestjs.com/cli/overview#project-structure)) üzerine bir sarmalayıcıdır. Varsayılan olarak `tsconfig-paths`'i işlemek dışında başka derleme özellikleri veya adımları eklememektedir. Varoluş amacı, çoğu geliştiricinin, özellikle Nest'e yeni başlıyorsa, bazen karmaşık olabilen derleyici seçeneklerini (örneğin, `tsconfig.json` dosyası) ayarlamaya ihtiyaç duymamasıdır.

Daha fazla ayrıntı için [nest build](https://docs.nestjs.com/cli/usages#nest-build) belgelerine bakın.

#### Çalıştırma

`nest start` sadece projenin derlendiğinden emin olur (aynı `nest build` gibi) ve derlenmiş uygulamayı yürütmek için `node` komutunu taşınabilir ve kolay bir şekilde çağırır. Derlemeler gibi, bu süreci ihtiyacınıza göre özelleştirebilirsiniz, ya `nest start` komutu ve seçenekleri kullanarak ya da tamamen değiştirerek. Tüm süreç standart bir TypeScript uygulaması derleme ve yürütme boru hattıdır ve süreci buna göre yönetebilirsiniz.

Daha fazla ayrıntı için [nest start](https://docs.nestjs.com/cli/usages#nest-start) belgelerine bakın.

#### Oluşturma

`nest generate` komutları, adından da anlaşılacağı gibi, yeni Nest projeleri veya bunların içindeki bileşenleri oluşturur.

#### Paket Betikleri

`nest` komutlarını işletim sistemi komut düzeyinde çalıştırmak, `nest` ikilisinin global olarak yüklenmiş olmasını gerektirir. Bu, npm'in standart bir özelliğidir ve Nest'in doğrudan kontrolü dışındadır. Bunun bir sonucu olarak, global olarak yüklenmiş `nest` ikilisi, `package.json`'daki bir proje bağımlılığı olarak **yönetilmez**. Örneğin, iki farklı geliştirici, `nest` ikilisinin iki farklı sürümünü çalıştırabilir. Bu durum için standart çözüm, kullanılan araçları geliştirme bağımlılıkları olarak işlemek için paket betiklerini kullanmaktır.

`nest new` çalıştırdığınızda veya [typescript starter](https://github.com/nestjs/typescript-starter) deposunu klonladığınızda, Nest, yeni projenin `package.json` betiklerini `build` ve `start` gibi komutlarla doldurur. Ayrıca, temel derleyici araçlarını (`typescript` gibi) **geliştirme bağımlılıkları** olarak yükler.

Derleme ve yürütme betiklerini şu komutlarla çalıştırabilirsiniz:

```bash
$ npm run build
```

ve

```bash
$ npm run start
```

Bu komutlar, `nest build` veya

 `nest start`'ı **yerel olarak yüklenmiş** `nest` ikilisi kullanarak yürütmek için npm'nin betik çalıştırma yeteneklerini kullanır. Bu yerleşik paket betiklerini kullanarak, Nest CLI komutları üzerinde tam bağımlılık yönetimine sahip olursunuz\*. Bu, bu **tavsiye edilen** kullanımı takip ederek, organizasyonunuzdaki tüm üyelerin aynı komut sürümünü çalıştığından emin olmasını sağlar.

\*Bu, `build` ve `start` komutları için geçerlidir. `nest new` ve `nest generate` komutları derleme/çalıştırma boru hattının bir parçası olmadığından, bunlar farklı bir bağlamda çalışır ve yerleşik `package.json` betikleriyle gelmez.

Çoğu geliştirici/ekip için, Nest projelerini derlemek ve yürütmek için paket betiklerini kullanmak önerilir. Bu betiklerin davranışını (`--path`, `--webpack`, `--webpackPath`) ve/veya `tsc` veya webpack derleyici seçenek dosyalarını (örneğin, `tsconfig.json`) ihtiyaca göre tamamen özelleştirebilirsiniz. Ayrıca, TypeScript'i derlemek için tamamen özel bir derleme süreci çalıştırabilirsiniz (veya TypeScript'i `ts-node` ile doğrudan yürütmek için).

#### Geriye Dönük Uyumluluk

Nest uygulamaları saf TypeScript uygulamaları olduğu için, önceki sürümlerdeki Nest derleme/çalıştırma betikleri çalışmaya devam edecektir. Bunları yükseltmek zorunda değilsiniz. Yeni `nest build` ve `nest start` komutlarının avantajlarından yararlanabilirsiniz, hazır olduğunuzda veya önceki veya özelleştirilmiş betikleri çalıştırmaya devam edebilirsiniz.

#### Göç

Herhangi bir değişiklik yapmak zorunda değilseniz, önceki betikleri çalıştırmaya devam edebilirsiniz, ancak yeni CLI komutlarını kullanmayı tercih edebilirsiniz. Bu durumda, sadece `@nestjs/cli`'nin en son sürümünü global ve yerel olarak yükleyin:

```bash
$ npm install -g @nestjs/cli
$ cd  /some/project/root/folder
$ npm install -D @nestjs/cli
```

Daha sonra `package.json`'daki aşağıdaki betikleri ile değiştirebilirsiniz:

```typescript
"build": "nest build",
"start": "nest start",
"start:dev": "nest start --watch",
"start:debug": "nest start --debug --watch",
```