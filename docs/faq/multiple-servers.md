### HTTPS

HTTPS protokolünü kullanan bir uygulama oluşturmak için `NestFactory` sınıfının `create()` yöntemine iletilen seçenek nesnesinde `httpsOptions` özelliğini ayarlayın:

```typescript
const httpsOptions = {
  key: fs.readFileSync('./secrets/private-key.pem'),
  cert: fs.readFileSync('./secrets/public-certificate.pem'),
};
const app = await NestFactory.create(AppModule, {
  httpsOptions,
});
await app.listen(3000);
```

Eğer `FastifyAdapter` kullanıyorsanız, uygulamayı aşağıdaki gibi oluşturun:

```typescript
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter({ https: httpsOptions }),
);
```

#### Eşzamanlı çoklu sunucular

Aşağıdaki örnek, aynı anda birden fazla portta (örneğin, HTTPS olmayan bir port ve bir HTTPS portu) dinleyen bir Nest uygulamasını nasıl başlatacağınızı gösterir.

```typescript
const httpsOptions = {
  key: fs.readFileSync('./secrets/private-key.pem'),
  cert: fs.readFileSync('./secrets/public-certificate.pem'),
};

const server = express();
const app = await NestFactory.create(
  AppModule,
  new ExpressAdapter(server),
);
await app.init();

const httpServer = http.createServer(server).listen(3000);
const httpsServer = https.createServer(httpsOptions, server).listen(443);
```

Çünkü kendimiz `http.createServer` / `https.createServer` çağırdık, NestJS onları `app.close` / sonlandırma sinyali çağrıldığında kapatmaz. Bunun için kendimiz yapmalıyız:

```typescript
@Injectable()
export class ShutdownObserver implements OnApplicationShutdown {
  private httpServers: http.Server[] = [];

  public addHttpServer(server: http.Server): void {
    this.httpServers.push(server);
  }

  public async onApplicationShutdown(): Promise<void> {
    await Promise.all(
      this.httpServers.map((server) =>
        new Promise((resolve, reject) => {
          server.close((error) => {
            if (error) {
              reject(error);
            } else {
              resolve(null);
            }
          });
        })
      ),
    );
  }
}

const shutdownObserver = app.get(ShutdownObserver);
shutdownObserver.addHttpServer(httpServer);
shutdownObserver.addHttpServer(httpsServer);
```

> info **İpucu** `ExpressAdapter`, `@nestjs/platform-express` paketinden içe aktarılır. `http` ve `https` paketleri, yerel Node.js paketleridir.

> **Uyarı** Bu tarif, [GraphQL Abonelikleri](/docs/graphql/subscriptions) ile çalışmaz.