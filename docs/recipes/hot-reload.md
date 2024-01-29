### Sıcak Yükleme (Hot Reload)

Uygulamanızın başlatma sürecindeki en yüksek etki, **TypeScript derlemesi** dir. Neyse ki, [webpack](https://github.com/webpack/webpack) HMR (Sıcak Modül Değiştirme) ile, her bir değişiklik olduğunda projeyi tamamen derlememize gerek kalmaz. Bu, uygulamanızı başlatmak için gereken zamanı önemli ölçüde azaltır ve iteratif geliştirmeyi çok daha kolay hale getirir.

> ⚠️ **Uyarı** `webpack`, varlıklarınızı (örneğin `graphql` dosyaları) `dist` klasörüne otomatik olarak kopyalamaz. Benzer şekilde, `webpack`, glob statik yollarıyla uyumlu değildir (örneğin, `TypeOrmModule`'daki `entities` özelliği).

### CLI ile

Eğer [Nest CLI](https://docs.nestjs.com/cli/overview) kullanıyorsanız, yapılandırma süreci oldukça basittir. CLI, `HotModuleReplacementPlugin`'i kullanmamıza izin veren `webpack`'i sarmalar.

#### Kurulum

İlk olarak gerekli paketleri kurun:

```bash
$ npm i --save-dev webpack-node-externals run-script-webpack-plugin webpack
```

> ℹ️ **İpucu** Eğer **Yarn Berry** (klasik Yarn değilse), `webpack-node-externals` yerine `webpack-pnp-externals` paketini kurun.

#### Yapılandırma

Kurulum tamamlandıktan sonra, uygulamanızın kök dizininde bir `webpack-hmr.config.js` dosyası oluşturun.

```typescript
const nodeExternals = require('webpack-node-externals');
const { RunScriptWebpackPlugin } = require('run-script-webpack-plugin');

module.exports = function (options, webpack) {
  return {
    ...options,
    entry: ['webpack/hot/poll?100', options.entry],
    externals: [
      nodeExternals({
        allowlist: ['webpack/hot/poll?100'],
      }),
    ],
    plugins: [
      ...options.plugins,
      new webpack.HotModuleReplacementPlugin(),
      new webpack.WatchIgnorePlugin({
        paths: [/\.js$/, /\.d\.ts$/],
      }),
      new RunScriptWebpackPlugin({ name: options.output.filename, autoRestart: false }),
    ],
  };
};
```

> ℹ️ **İpucu** **Yarn Berry** (klasik Yarn değilse), `externals` yapılandırma özelliğinde `nodeExternals`'ın yerine `webpack-pnp-externals` paketinden `WebpackPnpExternals`'ı kullanın: `WebpackPnpExternals({{ '{' }} exclude: ['webpack/hot/poll?100'] {{ '}' }})`.

Bu fonksiyon, ilk argüman olarak default webpack yapılandırmasını içeren orijinal nesneyi ve Nest CLI tarafından kullanılan temel `webpack` paketine referansı alır. Ayrıca, `HotModuleReplacementPlugin`, `WatchIgnorePlugin` ve `RunScriptWebpackPlugin` eklentileri ile değiştirilmiş bir webpack yapılandırması döndürür.

#### Sıcak Modül Değiştirme

**HMR'yi** etkinleştirmek için, uygulama giriş dosyasını (`main.ts`) açın ve aşağıdaki webpack ile ilgili talimatları ekleyin:

```typescript
declare const module: any;

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);

  if (module.hot) {
    module.hot.accept();
    module.hot.dispose(() => app.close());
  }
}
bootstrap();
```

Yürütme sürecini basitleştirmek için, `package.json` dosyanıza bir script ekleyin.

```json
"start:dev": "nest build --webpack --webpackPath webpack-hmr.config.js --watch"
```

Şimdi terminalinizi açın ve şu komutu çalıştırın:

```bash
$ npm run start:dev
```

### CLI Olmadan

Eğer [Nest CLI](https://docs.nestjs.com/cli/overview) kullanmıyorsanız, yapılandırma biraz daha karmaşık olacaktır (daha fazla manuel adım gerektire

cektir).

#### Kurulum

İlk olarak gerekli paketleri kurun:

```bash
$ npm i --save-dev webpack webpack-cli webpack-node-externals ts-loader run-script-webpack-plugin
```

> ℹ️ **İpucu** Eğer **Yarn Berry** (klasik Yarn değilse), `webpack-node-externals` yerine `webpack-pnp-externals` paketini kurun.

#### Yapılandırma

Kurulum tamamlandıktan sonra, uygulamanızın kök dizininde `webpack.config.js` adında bir dosya oluşturun.

```typescript
const webpack = require('webpack');
const path = require('path');
const nodeExternals = require('webpack-node-externals');
const { RunScriptWebpackPlugin } = require('run-script-webpack-plugin');

module.exports = {
  entry: ['webpack/hot/poll?100', './src/main.ts'],
  target: 'node',
  externals: [
    nodeExternals({
      allowlist: ['webpack/hot/poll?100'],
    }),
  ],
  module: {
    rules: [
      {
        test: /.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
    ],
  },
  mode: 'development',
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin(),
    new RunScriptWebpackPlugin({ name: 'server.js', autoRestart: false }),
  ],
  output: {
    path: path.join(__dirname, 'dist'),
    filename: 'server.js',
  },
};
```

> ℹ️ **İpucu** **Yarn Berry** (klasik Yarn değilse), `externals` yapılandırma özelliğinde `nodeExternals`'ın yerine `webpack-pnp-externals` paketinden `WebpackPnpExternals`'ı kullanın: `WebpackPnpExternals({{ '{' }} exclude: ['webpack/hot/poll?100'] {{ '}' }})`.

Bu yapılandırma, webpack'e uygulamanız hakkında birkaç temel bilgi verir: giriş dosyasının konumu, derlenmiş dosyaların hangi dizinde tutulması gerektiği ve kaynak dosyalarını derlemek için hangi yükleyiciyi kullanmak istediğimiz. Genellikle, bu dosyayı tam olarak anlamasanız bile, bu dosyayı olduğu gibi kullanabilirsiniz.

#### Sıcak Modül Değiştirme

**HMR'yi** etkinleştirmek için, uygulama giriş dosyasını (`main.ts`) açın ve aşağıdaki webpack ile ilgili talimatları ekleyin:

```typescript
declare const module: any;

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);

  if (module.hot) {
    module.hot.accept();
    module.hot.dispose(() => app.close());
  }
}
bootstrap();
```

Yürütme sürecini basitleştirmek için, `package.json` dosyanıza bir script ekleyin.

```json
"start:dev": "webpack --config webpack.config.js --watch"
```

Şimdi terminalinizi açın ve şu komutu çalıştırın:

```bash
$ npm run start:dev
```

#### Örnek

Çalışan bir örnek [burada](https://github.com/nestjs/nest/tree/master/sample/08-webpack) bulunabilir.