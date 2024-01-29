#### CI/CD Entegrasyonu

> info **İpucu** Bu bölüm, Nest Devtools'un Nest çerçevesiyle entegrasyonunu kapsar. Devtools uygulamasını arıyorsanız, lütfen [Devtools](https://devtools.nestjs.com) web sitesini ziyaret edin.

CI/CD entegrasyonu, **[Enterprise](/docs/settings)** planına sahip kullanıcılar için kullanılabilir.

CI/CD entegrasyonunun size nasıl ve neden yardımcı olabileceğini öğrenmek için bu videoyu izleyebilirsiniz:

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/r5RXcBrnEQ8"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### Grafikleri Yayınlama

İlk olarak, uygulama başlatma dosyasını (`main.ts`) `GraphPublisher` sınıfını kullanacak şekilde yapılandıralım (`@nestjs/devtools-integration` tarafından ihraç edilen - daha fazla ayrıntı için önceki bölüme bakın):

```typescript
async function bootstrap() {
  const shouldPublishGraph = process.env.PUBLISH_GRAPH === "true";

  const app = await NestFactory.create(AppModule, {
    snapshot: true,
    preview: shouldPublishGraph,
  });

  if (shouldPublishGraph) {
    await app.init();

    const publishOptions = { ... } // NOT: bu seçenek nesnesi, kullandığınız CI/CD sağlayıcısına bağlı olarak değişecektir
    const graphPublisher = new GraphPublisher(app);
    await graphPublisher.publish(publishOptions);

    await app.close();
  } else {
    await app.listen(3000);
  }
}
```

Görüldüğü gibi, burada `GraphPublisher`'ı kullanarak serileştirilmiş grafiklerimizi merkezi kayıta yayınlıyoruz. `PUBLISH_GRAPH`, grafik yayınlanıp yayınlanmamasını kontrol etmemizi sağlayan özel bir ortam değişkenidir (CI/CD iş akışı) veya değildir (normal uygulama başlatma). Ayrıca, burada `preview` özelliğini `true` olarak ayarlıyoruz. Bu bayrağı etkinleştirildiğinde, uygulamamız öncizgi modunda başlatılacaktır - bu temelde uygulamamızın kontrolörlerinin, geliştiricilerinin ve sağlayıcılarının tümünün yapıcıları (ve yaşam döngü kancaları) çalıştırılmayacaktır. Not - bu zorunlu değildir, ancak uygulamamızı CI/CD boru hattında çalıştırırken gerçekten veritabanına bağlanmak gibi şeyleri yapmamız gerekmeyeceğinden işleri daha basit hale getirir.

`publishOptions` nesnesi, kullandığınız CI/CD sağlayıcısına bağlı olarak değişecektir. Aşağıda en popüler CI/CD sağlayıcıları için talimatları sağlayacağız, sonraki bölümlerde.

Graf başarıyla yayınlandığında, iş akışı görünümünüzde şu çıktıyı görmelisiniz:

<figure><img src="/assets/devtools/graph-published-terminal.png" /></figure>

Graf her yayınlandığında, ilgili projenin sayfasında yeni bir giriş görmeliyiz:

<figure><img src="/assets/devtools/project.png" /></figure>

#### Raporlar

Devtools, her derleme için bir rapor oluşturur **Ancak** zaten merkezi kayıtta karşılık gelen bir öncü saklanmışsa. Örneğin, bir grafin zaten yayınlandığı `master` şubesine karşı bir PR oluşturursanız - o zaman uygulama farkları algılayabilir ve bir rapor oluşturabilir. Aksi takdirde, rapor oluşturulmaz.

Raporları görmek için, ilgili projenin sayfasına gidin (organizasyonlara bakın).

<figure><img src="/assets/devtools/report.png" /></figure>

Bu, kod incelemeleri sırasında fark edilmemiş değişiklikleri belirlemede özellikle yardımcıdır. Örneğin, birisi bir **derin iç içe sağlayıcının kapsamını** değiştirmişse. Bu değişiklik inceleyici için hemen açık olmayabilir, ancak Devtools ile bu tür değişiklikleri kolayca tespit edebilir ve bunların kasıtlı olduğundan emin olabiliriz. Veya belirli bir uca bir korumayı kaldırırsak, raporda etkilenen olarak görünecektir. Şimdi bu rota için entegrasyon veya e2e testlerimiz yoksa, korunmadığını fark etmeyebiliriz ve fark ettiğimizde, çok geç olabilir.

Benzer şekilde, **büyük bir kod tabanında** çalışıyorsak ve bir modülü genel yapmak için değiştiriyorsak, grafa kaç kenarın eklendiğini görebiliriz ve bu - çoğu durumda - yanlış bir şey yaptığımızın bir işaretidir.

#### Derleme Önizleme

Her yayınlanan grafik için zaman içinde geri gidebilir ve nasıl göründüğünü önceden görebiliriz **Önizleme** düğmesine tıklayarak. Dahası, rapor oluşturulmuşsa, grafikteki farkları vurgulanmış olarak görmeliyiz:

- Yeşil düğmeler eklenen öğeleri temsil eder
- Açık beyaz düğmeler güncellenmiş öğeleri temsil eder
- Kırmızı düğmeler silinmiş öğeleri temsil eder

Aşağıdaki ekran görüntüsünde gösterildiği gibi:

<figure><img src="/assets/devtools/nodes-selection.png" /></figure>

Zamanda geri gitme yeteneği, mevcut grafiği öncekilerle karşılaştırarak sorunu araştırmanıza ve sorun gidermenize olanak tanır. Her bir çekme isteğinin (veya hatta her bir taahhütün) kayıtta karşılık gelen bir öncüsü olduğuna bağlı olarak, kolayca geçmişe gidip nelerin değiştiğini görebilirsiniz. Devtools'ü, Nest'in uygulama grafinizi nasıl oluşturduğunu anlayan bir Git gibi düşünün ve bunu **görselleştirmek** için yeteneğine sahip olarak.

#### Entegrasyonlar: GitHub Actions

İlk olarak, projemizdeki `.github/workflows` dizininde yeni bir GitHub iş akışı oluşturmaya başlayalım ve bu iş akışına, örneğin, `publish-graph.yml` adını verelim. Bu dosyanın içinde aşağıdaki tanımı kullanalım:

```yaml
name: Devtools

on:
  push:
    branches:
      - master
  pull_request:
    branches:
      - '*'

jobs:
  publish:
    if: github.actor!= 'dependabot[bot]'
    name: Publish graph
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '16'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Setup Environment (PR)
        if: ${{ github.event_name == 'pull_request' }}
        shell: bash
        run: |
          echo "COMMIT_SHA=${{ github.event.pull_request.head.sha }}" >>${GITHUB_ENV}
      - name: Setup Environment (Push)
        if: ${{ github.event_name == 'push' }}
        shell: bash
        run: |
          echo "COMMIT_SHA=${GITHUB_SHA}" >> ${GITHUB_ENV}
      - name: Publish
        run: PUBLISH_GRAPH=true npm run start
        env:
          DEVTOOLS_API_KEY: CHANGE_THIS_TO_YOUR_API_KEY
          REPOSITORY_NAME: ${{ github.event.repository.name }}
          BRANCH_NAME: ${{ github.head_ref || github.ref_name }}
          TARGET_SHA: ${{ github.event.pull_request.base.sha }}
```

İdeali, `DEVTOOLS_API_KEY` ortam değişkeninin GitHub Secrets'ten alınması gerekmektedir, daha fazlasını [buradan](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) okuyabilirsiniz.

Bu iş akışı, `master` şubesine hedeflenen her bir pull request için veya doğrudan `master` şubesine yapılan herhangi bir taahhüt için çalışacaktır. Bu yapıyı projenizin ihtiyaçlarına göre düzenleyebilirsiniz. Burada esas olan, `GraphPublisher` sınıfımız için gerekli ortam değişkenlerini (çalıştırmak için) sağlamaktır.

Ancak, bu iş akışını kullanmaya başlamadan önce güncellenmesi gereken bir değişken var - `DEVTOOLS_API_KEY`. Projemiz için bu **sayfada** özel bir API anahtarı oluşturabiliriz.

Son olarak, `main.ts` dosyasına tekrar gidip daha önce boş bıraktığımız `publishOptions` nesnesini güncelleyelim.

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.REPOSITORY_NAME,
  owner: process.env.GITHUB_REPOSITORY_OWNER,
  sha: process.env.COMMIT_SHA,
  target: process.env.TARGET_SHA,
  trigger: process.env.GITHUB_BASE_REF ? 'pull' : 'push',
  branch: process.env.BRANCH_NAME,
};
```

En iyi geliştirici deneyimi için, projeniz için "GitHub uygulamasını entegre et" düğmesine tıklayarak **GitHub uygulamasını** entegre ettiğinizden emin olun (aşağıdaki ekran görüntüsüne bakın). Not - bu zorunlu değildir.

<figure><img src="/assets/devtools/integrate-github-app.png" /></figure>

Bu entegrasyon ile pull request'inizde önizleme/rapor oluşturma sürecinin durumunu doğrudan görebileceksiniz:

<figure><img src="/assets/devtools/actions-preview.png" /></figure>

#### Entegrasyonlar: GitLab Pipelines

İlk olarak, projemizin kök dizininde yeni bir GitLab CI yapılandırma dosyası oluşturmaya başlayalım ve buna örneğin, `.gitlab-ci.yml` adını verelim. Bu dosyanın içinde aşağıdaki tanımı kullanalım:

```yaml
image: node:16

stages:
  - build

cache:
  key:
    files:
      - package-lock.json
  paths:
    - node_modules/

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: always
    - if: $CI_COMMIT_BRANCH == "master" && $CI_PIPELINE_SOURCE == "push"
      when: always
    - when: never

install_dependencies:
  stage: build
  script:
    - npm ci

publish_graph:
  stage: build
  needs:
    - install_dependencies
  script: npm run start
  variables:
    PUBLISH_GRAPH: 'true'
    DEVTOOLS_API_KEY: 'CHANGE_THIS_TO_YOUR_API_KEY'
```

> Bilgi **İpucu** İdeali, `DEVTOOLS_API_KEY` ortam değişkeninin secrets'ten alınması gerekmektedir.

Bu iş akışı, `master` şubesine hedeflenen her bir pull request için veya doğrudan `master` şubesine yapılan herhangi bir taahhüt için çalışacaktır. Bu yapıyı projenizin ihtiyaçlarına göre düzenleyebilirsiniz. Burada esas olan, `GraphPublisher` sınıfımız için gerekli ortam değişkenlerini (çalıştırmak için) sağlamaktır.

Ancak, bu iş akışı tanımındaki bir değişkenin - `DEVTOOLS_API_KEY` - kullanılmadan önce güncellenmesi gerekiyor. Projemiz için bu **sayfada** özel bir API anahtarı oluşturabiliriz.

Son olarak, `main.ts` dosyasına tekrar gidip daha önce boş bıraktığımız `publishOptions` nesnesini güncelleyelim.

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.REPOSITORY_NAME,
  owner: process.env.GITHUB_REPOSITORY_OWNER,
  sha: process.env.COMMIT_SHA,
  target: process.env.TARGET_SHA,
  trigger: process.env.GITHUB_BASE_REF ? 'pull' : 'push',
  branch: process.env.BRANCH_NAME,
};
```

#### Diğer CI/CD Araçları

Nest Devtools CI/CD entegrasyonu, tercih ettiğiniz herhangi bir CI/CD aracıyla kullanılabilir (örneğin, [Bitbucket Pipelines](https://bitbucket.org/product/features/pipelines), [CircleCI](https://circleci.com/), vb.), bu yüzden burada açıkladıklarımızla sınırlı hissetmeyin.

Aşağıdaki `publishOptions` nesnesi konfigürasyonuna bakarak, belirli bir commit/build/PR için grafiği yayınlamak için hangi bilgilere ihtiyaç duyulduğunu anlayabilirsiniz.

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.CI_PROJECT_NAME,
  owner: process.env.CI_PROJECT_ROOT_NAMESPACE,
  sha: process.env.CI_COMMIT_SHA,
  target: process.env.CI_MERGE_REQUEST_DIFF_BASE_SHA,
  trigger: process.env.CI_MERGE_REQUEST_DIFF_BASE_SHA ? 'pull' : 'push',
  branch:
    process.env.CI_COMMIT_BRANCH ??
    process.env.CI_MERGE_REQUEST_SOURCE_BRANCH_NAME,
};
```

Bu bilgilerin çoğu, CI/CD yerleşik ortam değişkenleri aracılığıyla sağlanır (bkz. [CircleCI yerleşik ortam değişkeni listesi](https://circleci.com/docs/variables/#built-in-environment-variables) ve [Bitbucket değişkenleri](https://support.atlassian.com/bitbucket-cloud/docs/variables-and-secrets/)).

Grafiği yayınlamak için pipeline konfigürasyonu yaparken, aşağıdaki tetikleyicileri kullanmanızı öneririz:

- `push` etkinliği - yalnızca mevcut dal bir dağıtım ortamını temsil ediyorsa, örneğin `master`, `main`, `staging`, `production`, vb.
- `pull request` etkinliği - her zaman veya **hedef dal** bir dağıtım ortamını temsil ediyorsa (yukarıya bakınız)