### Platform Bağımsızlığı

Nest, platform bağımsız bir çerçevedir. Bu, farklı türdeki uygulamalarda kullanılabilen **yeniden kullanılabilir mantıksal parçalar** geliştirebileceğiniz anlamına gelir. Örneğin, çoğu bileşen, Express ve Fastify gibi farklı temel HTTP sunucu çerçeveleri üzerinde değişiklik yapmadan yeniden kullanılabilir, hatta farklı _türlerde_ uygulamalar arasında (örneğin, HTTP sunucu çerçeveleri, farklı taşıma katmanlarına sahip Mikroservisler ve Web Soketleri) kullanılabilir.

#### Bir kez oluştur, her yerde kullan

Belgelerin **Genel Bakış** bölümü, temel olarak HTTP sunucu çerçeveleri kullanarak kodlama tekniklerini gösterir (örneğin, REST API sağlayan uygulamalar veya MVC tarzında sunucu taraflı oluşturulan bir uygulama). Ancak, tüm bu yapı taşları farklı taşıma katmanları üzerinde (örneğin, [mikroservisler](/docs/microservices/basics) veya [web soketleri](/docs/websockets/gateways)) kullanılabilir.

Ayrıca, Nest, özel bir [GraphQL](/docs/graphql/quick-start) modülü ile birlikte gelir. GraphQL'i REST API sağlamakla birlikte API katmanı olarak da kullanabilirsiniz.

Ek olarak, [uygulama bağlamı](/docs/application-context) özelliği, Nest'in üzerine her türlü Node.js uygulamasını oluşturmanıza yardımcı olur - bunlar arasında CRON görevleri ve CLI uygulamaları gibi şeyler de bulunabilir.

Nest, uygulamalarınıza daha yüksek bir modülerlik ve yeniden kullanılabilirlik düzeyi getirmeyi amaçlayan tam teşekküllü bir Node.js platformu olma hedefini taşır. Bir kez oluştur, her yerde kullan!