### Belgelendirme

**Compodoc**, Angular uygulamaları için bir belgelendirme aracıdır. Nest ve Angular benzer proje ve kod yapılarına sahip oldukları için, **Compodoc**, Nest uygulamaları ile de çalışır.

#### Kurulum

Mevcut bir Nest projesinde Compodoc'u kurmak çok basittir. İlk olarak, aşağıdaki komutu OS terminalinizde kullanarak dev-bağımlılığını ekleyin:

```bash
$ npm i -D @compodoc/compodoc
```

#### Oluşturma

Proje belgelerini aşağıdaki komutu kullanarak oluşturun (`npx` desteği için npm 6 gerekir). Daha fazla seçenek için [resmi belgelere](https://compodoc.app/guides/usage.html) bakın.

```bash
$ npx @compodoc/compodoc -p tsconfig.json -s
```

Tarayıcınızı açın ve [http://localhost:8080](http://localhost:8080) adresine gidin. Başlangıçta bir Nest CLI projesini görmelisiniz:

<figure><img src="/assets/documentation-compodoc-1.jpg" /></figure>
<figure><img src="/assets/documentation-compodoc-2.jpg" /></figure>

#### Katkıda Bulunun

Compodoc projesine [buradan](https://github.com/compodoc/compodoc) katılabilir ve katkıda bulunabilirsiniz.