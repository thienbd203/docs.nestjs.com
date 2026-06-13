### Hot Reload

Tác động cao nhất đến quá trình bootstrap của ứng dụng của bạn là **biên dịch TypeScript**. May mắn thay, với [webpack](https://github.com/webpack/webpack) HMR (Hot-Module Replacement), chúng ta không cần biên dịch lại toàn bộ dự án mỗi khi một thay đổi xảy ra. Điều này giảm đáng kể lượng thời gian cần thiết để khởi tạo ứng dụng của bạn, và làm cho phát triển lặp đi lặp lại dễ dàng hơn nhiều.

> warning **Cảnh báo** Lưu ý rằng `webpack` sẽ không tự động sao chép các tài sản của bạn (ví dụ file `graphql`) đến thư mục `dist`. Tương tự, `webpack` không tương thích với các đường dẫn tĩnh glob (ví dụ, thuộc tính `entities` trong `TypeOrmModule`).

### Với CLI

Nếu bạn đang sử dụng [Nest CLI](https://docs.nestjs.com/cli/overview), quá trình cấu hình khá đơn giản. CLI bọc `webpack`, cho phép sử dụng `HotModuleReplacementPlugin`.

#### Cài đặt

Trước tiên cài đặt các gói cần thiết:

```bash
$ npm i --save-dev webpack-node-externals run-script-webpack-plugin webpack
```

> info **Gợi ý** Nếu bạn sử dụng **Yarn Berry** (không phải Yarn cổ điển), cài đặt gói `webpack-pnp-externals` thay vì `webpack-node-externals`.

#### Cấu hình

Khi cài đặt hoàn tất, tạo một file `webpack-hmr.config.js` trong thư mục gốc của ứng dụng của bạn.

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

> info **Gợi ý** Với **Yarn Berry** (không phải Yarn cổ điển), thay vì sử dụng `nodeExternals` trong thuộc tính cấu hình `externals`, sử dụng `WebpackPnpExternals` từ gói `webpack-pnp-externals`: `WebpackPnpExternals({ exclude: ['webpack/hot/poll?100'] })`.

Hàm này nhận đối tượng gốc chứa cấu hình webpack mặc định như đối số đầu tiên, và tham chiếu đến gói `webpack` cơ bản được sử dụng bởi Nest CLI như thứ hai. Ngoài ra, nó trả về một cấu hình webpack đã sửa đổi với các plugin `HotModuleReplacementPlugin`, `WatchIgnorePlugin`, và `RunScriptWebpackPlugin`.

#### Hot-Module Replacement

Để bật **HMR**, mở file entry ứng dụng (`main.ts`) và thêm các hướng dẫn liên quan đến webpack sau:

```typescript
declare const module: any;

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);

  if (module.hot) {
    module.hot.accept();
    module.hot.dispose(() => app.close());
  }
}
bootstrap();
```

Để đơn giản hóa quá trình thực thi, thêm một script vào file `package.json` của bạn.

```json
"start:dev": "nest build --webpack --webpackPath webpack-hmr.config.js --watch"
```

Bây giờ chỉ cần mở dòng lệnh của bạn và chạy lệnh sau:

```bash
$ npm run start:dev
```

### Không có CLI

Nếu bạn không sử dụng [Nest CLI](https://docs.nestjs.com/cli/overview), cấu hình sẽ phức tạp hơn một chút (sẽ yêu cầu nhiều bước thủ công hơn).

#### Cài đặt

Trước tiên cài đặt các gói cần thiết:

```bash
$ npm i --save-dev webpack webpack-cli webpack-node-externals ts-loader run-script-webpack-plugin
```

> info **Gợi ý** Nếu bạn sử dụng **Yarn Berry** (không phải Yarn cổ điển), cài đặt gói `webpack-pnp-externals` thay vì `webpack-node-externals`.

#### Cấu hình

Khi cài đặt hoàn tất, tạo một file `webpack.config.js` trong thư mục gốc của ứng dụng của bạn.

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
  plugins: [new webpack.HotModuleReplacementPlugin(), new RunScriptWebpackPlugin({ name: 'server.js', autoRestart: false })],
  output: {
    path: path.join(__dirname, 'dist'),
    filename: 'server.js',
  },
};
```

> info **Gợi ý** Với **Yarn Berry** (không phải Yarn cổ điển), thay vì sử dụng `nodeExternals` trong thuộc tính cấu hình `externals`, sử dụng `WebpackPnpExternals` từ gói `webpack-pnp-externals`: `WebpackPnpExternals({ exclude: ['webpack/hot/poll?100'] })`.

Cấu hình này nói với webpack một vài điều thiết yếu về ứng dụng của bạn: vị trí của file entry, thư mục nào nên được sử dụng để giữ các file **được biên dịch**, và loại loader nào chúng ta muốn sử dụng để biên dịch các file nguồn. Nói chung, bạn nên có thể sử dụng file này như là, ngay cả khi bạn không hiểu đầy đủ tất cả các tùy chọn.

#### Hot-Module Replacement

Để bật **HMR**, mở file entry ứng dụng (`main.ts`) và thêm các hướng dẫn liên quan đến webpack sau:

```typescript
declare const module: any;

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);

  if (module.hot) {
    module.hot.accept();
    module.hot.dispose(() => app.close());
  }
}
bootstrap();
```

Để đơn giản hóa quá trình thực thi, thêm một script vào file `package.json` của bạn.

```json
"start:dev": "webpack --config webpack.config.js --watch"
```

Bây giờ chỉ cần mở dòng lệnh của bạn và chạy lệnh sau:

```bash
$ npm run start:dev
```

#### Ví dụ

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/08-webpack).