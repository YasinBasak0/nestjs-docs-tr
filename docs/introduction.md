---
sidebar_position: 10
---

### Giriş

Nest (NestJS), etkili ve ölçeklenebilir [Node.js](https://nodejs.org/) sunucu tarafı uygulamaları geliştirmek için bir çerçevedir. İlerleyici JavaScript kullanır, [TypeScript](http://www.typescriptlang.org/) ile oluşturulmuş ve tam olarak desteklenir (ancak geliştiricilere saf JavaScript'te kodlama olanağı sağlar) ve Nesne Yönelimli Programlama (OOP), Fonksiyonel Programlama (FP) ve Fonksiyonel Reaktif Programlama (FRP) öğelerini birleştirir.

Arka planda, Nest, [Express](https://expressjs.com/) (varsayılan) gibi güçlü HTTP Sunucu çerçevelerinden yararlanır ve isteğe bağlı olarak [Fastify](https://github.com/fastify/fastify) kullanacak şekilde yapılandırılabilir!

Nest, bu yaygın Node.js çerçeveleri (Express/Fastify) üzerinde bir soyutlama seviyesi sağlar, ancak geliştiriciye bunların API'lerini doğrudan sunar. Bu, geliştiricilere temel platform için mevcut olan çeşitli üçüncü taraf modülleri kullanma özgürlüğü tanır.

#### Felsefe

Son yıllarda, Node.js sayesinde JavaScript, webin hem ön hem de arka uygulamaları için "lingua franca" haline gelmiştir. Bu durum, [Angular](https://angular.io/), [React](https://github.com/facebook/react) ve [Vue](https://github.com/vuejs/vue) gibi harika projelerin ortaya çıkmasına yol açmıştır. Bu projeler, geliştirici üretkenliğini artırır ve hızlı, test edilebilir ve genişletilebilir ön uç uygulamalarının oluşturulmasını mümkün kılar. Ancak, Node (ve sunucu tarafı JavaScript) için birçok harika kütüphane, yardımcı ve araç bulunmasına rağmen, bunlar genellikle **Mimari** sorununu etkili bir şekilde çözmez.

Nest, geliştiricilere ve takımlara son derece test edilebilir, ölçeklenebilir, gevşek bağlı ve kolay bakım yapılabilen uygulamalar oluşturmalarına izin veren hazır bir uygulama mimarisi sunar. Bu mimari, Angular'dan ağır şekilde ilham alınmıştır.

#### Kurulum

Başlamak için projeyi ya [Nest CLI](/docs/cli/overview) ile iskeletleştirebilir veya bir başlangıç projesi klonlayabilirsiniz (her ikisi de aynı sonucu üretir).

Nest CLI ile projeyi iskeletleştirmek için aşağıdaki komutları çalıştırın. Bu, yeni bir proje dizini oluşturacak ve dizini başlangıçtaki temel Nest dosyaları ve destekleyici modüllerle doldurarak projeniz için geleneksel bir temel yapı oluşturacaktır. **Nest CLI** ile yeni bir proje oluşturmak, ilk kez kullanıcılar için önerilir. Bu yaklaşımı [İlk Adımlar](first-steps) bölümünde sürdüreceğiz.

```bash
$ npm i -g @nestjs/cli
$ nest new proje-adı
```

> info **İpucu** Daha katı bir özellik setine sahip yeni bir TypeScript projesi oluşturmak için `nest new` komutuna `--strict` bayrağını geçirin.

#### Alternatifler

Alternatif olarak, TypeScript başlangıç projesini **Git** ile kurmak için:

```bash
$ git clone https://github.com/nestjs/typescript-starter.git proje
$ cd proje
$ npm install
$ npm run start
```

> info **İpucu** Eğer git geçmişi olmadan depoyu klonlamak istiyorsanız, [degit](https://github.com/Rich-Harris/degit) kullanabilirsiniz.

Tarayıcınızı açın ve [`http://localhost:3000/`](http://localhost:3000/) adresine gidin.

Starter projesinin JavaScript sürümünü kurmak için yukarıdaki komut dizisinde `javascript-starter.git`i kullanabilirsiniz.

Ayrıca, çekirdek ve destek dosyalarını **npm** (veya **yarn**) ile manuel olarak yükleyerek sıfırdan yeni bir proje oluşturabilirsiniz. Bu durumda, tabii ki, proje temel dosyalarını kendiniz oluşturmakla sorumlu olacaksınız.

```bash
$ npm i --save @nestjs/core @nestjs/common rxjs reflect-metadata
```