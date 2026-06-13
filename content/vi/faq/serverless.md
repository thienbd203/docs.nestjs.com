### Serverless

Serverless computing là một execution model cloud computing trong đó cloud provider allocates machine resources on-demand, lo việc các servers thay mặt cho khách hàng của họ. Khi một app không được sử dụng, không có computing resources được allocated đến app. Pricing dựa trên actual amount of resources được tiêu thụ bởi một application (source).

Với một **serverless architecture**, bạn tập trung thuần túy vào các individual functions trong application code của bạn. Các services như AWS Lambda, Google Cloud Functions, và Microsoft Azure Functions lo việc tất cả physical hardware, virtual machine operating system, và web server software management.

> info **Gợi ý** Chương này không bao gồm pros và cons của serverless functions cũng không đi sâu vào các đặc điểm của bất kỳ cloud providers nào.

#### Cold start

Một cold start là lần đầu tiên code của bạn đã được executed trong một thời gian. Tùy thuộc vào cloud provider bạn sử dụng, nó có thể span nhiều different operations, từ downloading code và bootstrapping runtime đến cuối cùng chạy code của bạn.

Quá trình này thêm **significant latency** tùy thuộc vào một số factors, language, số lượng packages application của bạn yêu cầu, v.v.

Cold start là quan trọng và mặc dù có những things beyond our control, vẫn có rất nhiều things chúng ta có thể làm ở phía mình để làm cho nó càng ngắn càng tốt.

Mặc dù bạn có thể nghĩ về Nest như một fully-fledged framework được thiết kế để được sử dụng trong các complex, enterprise applications,
nó cũng **suitable cho much "simpler" applications** (hoặc scripts). Ví dụ, với sử dụng tính năng Standalone applications, bạn có thể tận dụng DI system của Nest trong các workers đơn giản, CRON jobs, CLIs, hoặc serverless functions.

#### Benchmarks

Để hiểu rõ hơn cost của việc sử dụng Nest hoặc các libraries well-known khác (như `express`) trong context của serverless functions, hãy so sánh thời gian Node runtime cần để chạy các scripts sau:

```typescript
// #1 Express
import * as express from 'express';

async function bootstrap() {
  const app = express();
  app.get('/', (req, res) => res.send('Hello world!'));
  await new Promise<void>((resolve) => app.listen(3000, resolve));
}
bootstrap();

// #2 Nest (with @nestjs/platform-express)
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { logger: ['error'] });
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();

// #3 Nest as a Standalone application (no HTTP server)
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AppService } from './app.service';

async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule, {
    logger: ['error'],
  });
  console.log(app.get(AppService).getHello());
}
bootstrap();

// #4 Raw Node.js script
async function bootstrap() {
  console.log('Hello world!');
}
bootstrap();
```

Đối với tất cả các scripts này, chúng tôi sử dụng compiler `tsc` (TypeScript) và vì vậy code vẫn remain unbundled (`webpack` không được sử dụng).

||                                      |                   |
|| ------------------------------------ | ----------------- |
|| Express                              | 0.0079s (7.9ms)   |
|| Nest với `@nestjs/platform-express` | 0.1974s (197.4ms) |
|| Nest (standalone application)        | 0.1117s (111.7ms) |
|| Raw Node.js script                   | 0.0071s (7.1ms)   |

> info **Ghi chú** Máy: MacBook Pro Mid 2014, 2.5 GHz Quad-Core Intel Core i7, 16 GB 1600 MHz DDR3, SSD.

Bây giờ, hãy lặp lại tất cả benchmarks nhưng lần này, sử dụng `webpack` (nếu bạn có Nest CLI được cài đặt, bạn có thể chạy `nest build --webpack`) để bundle application của bạn vào một single executable JavaScript file.
Tuy nhiên, thay vì sử dụng cấu hình `webpack` mặc định mà Nest CLI ships với, chúng tôi sẽ đảm bảo bundle tất cả dependencies (`node_modules`) cùng nhau, như sau:

```javascript
module.exports = (options, webpack) => {
  const lazyImports = [
    '@nestjs/microservices/microservices-module',
    '@nestjs/websockets/socket-module',
  ];

  return {
    ...options,
    externals: [],
    plugins: [
      ...options.plugins,
      new webpack.IgnorePlugin({
        checkResource(resource) {
          if (lazyImports.includes(resource)) {
            try {
              require.resolve(resource);
            } catch (err) {
              return true;
            }
          }
          return false;
        },
      }),
    ],
  };
};
```

> info **Gợi ý** Để hướng dẫn Nest CLI sử dụng cấu hình này, tạo một file `webpack.config.js` mới trong root directory của project của bạn.

Với cấu hình này, chúng tôi nhận được kết quả sau:

||                                      |                  |
|| ------------------------------------ | ---------------- |
|| Express                              | 0.0068s (6.8ms)  |
|| Nest với `@nestjs/platform-express` | 0.0815s (81.5ms) |
|| Nest (standalone application)        | 0.0319s (31.9ms) |
|| Raw Node.js script                   | 0.0066s (6.6ms)  |

> info **Ghi chú** Máy: MacBook Pro Mid 2014, 2.5 GHz Quad-Core Intel Core i7, 16 GB 1600 MHz DDR3, SSD.

> info **Gợi ý** Bạn có thể optimize nó thậm chí hơn nữa bằng cách áp dụng additional code minification & optimization techniques (sử dụng `webpack` plugins, v.v.).

Như bạn có thể thấy, cách bạn compile (và liệu bạn bundle code của bạn) là quan trọng và có significant impact trên overall startup time. Với `webpack`, bạn có thể lấy bootstrap time của một standalone Nest application (starter project với một module, controller, và service) xuống ~32ms trung bình, và xuống ~81.5ms cho một HTTP thường xuyên, express-based NestJS app.

Đối với các Nest applications phức tạp hơn, ví dụ, với 10 resources (được tạo ra qua `$ nest g resource` schematic = 10 modules, 10 controllers, 10 services, 20 DTO classes, 50 HTTP endpoints + `AppModule`), overall startup trên MacBook Pro Mid 2014, 2.5 GHz Quad-Core Intel Core i7, 16 GB 1600 MHz DDR3, SSD là khoảng 0.1298s (129.8ms). Chạy một monolithic application như một serverless function thường không có quá nhiều ý nghĩa anyway, vì vậy hãy nghĩ về benchmark này nhiều hơn như một example về cách bootstrap time có thể tăng tiềm năng khi application của bạn phát triển.

#### Runtime optimizations

Cho đến nay chúng tôi đã bao gồm compile-time optimizations. Những điều này không liên quan đến cách bạn định nghĩa providers và load Nest modules trong application của bạn, và điều đó đóng một vai trò thiết yếu khi application của bạn trở nên lớn hơn.

Ví dụ, hãy tưởng tượng có một database connection được định nghĩa như một asynchronous provider. Async providers được thiết kế để delay application start cho đến khi một hoặc nhiều asynchronous tasks được hoàn thành.
Điều đó có nghĩa là, nếu serverless function của bạn trung bình yêu cầu 2s để kết nối đến database (trên bootstrap), endpoint của bạn sẽ cần ít nhất hai extra seconds (vì nó phải đợi cho đến khi connection được thiết lập) để gửi một response trở lại (khi nó là một cold start và application của bạn chưa chạy).

Như bạn có thể thấy, cách bạn cấu trúc providers của bạn là hơi khác nhau trong một **serverless environment** nơi bootstrap time là quan trọng.
Một example tốt khác là nếu bạn sử dụng Redis cho caching, nhưng chỉ trong một số scenarios nhất định. Có lẽ, trong trường hợp này, bạn không nên định nghĩa một Redis connection như một async provider, vì nó sẽ làm chậm bootstrap time, ngay cả khi nó không được yêu cầu cho function invocation cụ thể này.

Ngoài ra, đôi khi bạn có thể lazy load toàn bộ modules, sử dụng class `LazyModuleLoader`, như được mô tả trong chương này. Caching là một example tuyệt vời ở đây cũng như.
Hãy tưởng tượng rằng application của bạn có, hãy nói, `CacheModule` mà nội bộ kết nối đến Redis và cũng, exports `CacheService` để tương tác với Redis storage. Nếu bạn không cần nó cho tất cả các potential function invocations,
bạn có thể chỉ load nó on-demand, lazily. Cách này bạn sẽ có một startup time nhanh hơn (khi một cold start xảy ra) cho tất cả các invocations không yêu cầu caching.

```typescript
if (request.method === RequestMethod[RequestMethod.GET]) {
  const { CacheModule } = await import('./cache.module');
  const moduleRef = await this.lazyModuleLoader.load(() => CacheModule);

  const { CacheService } = await import('./cache.service');
  const cacheService = moduleRef.get(CacheService);

  return cacheService.get(ENDPOINT_KEY);
}
```

Một example tuyệt vời khác là một webhook hoặc worker, mà tùy thuộc vào một số conditions cụ thể (ví dụ, input arguments), có thể thực hiện các operations khác nhau.
Trong trường hợp như vậy, bạn có thể chỉ định một condition bên trong route handler của bạn mà lazily loads một module thích hợp cho function invocation cụ thể, và chỉ load mỗi module khác lazily.

```typescript
if (workerType === WorkerType.A) {
  const { WorkerAModule } = await import('./worker-a.module');
  const moduleRef = await this.lazyModuleLoader.load(() => WorkerAModule);
  // ...
} else if (workerType === WorkerType.B) {
  const { WorkerBModule } = await import('./worker-b.module');
  const moduleRef = await this.lazyModuleLoader.load(() => WorkerBModule);
  // ...
}
```

#### Example integration

Cách entry file của application của bạn (thường là file `main.ts`) được cho là trông như thế nào **phụ thuộc vào một số factors** và vì vậy **không có single template** mà chỉ hoạt động cho mọi scenario.
Ví dụ, file initialization được yêu cầu để spin up serverless function của bạn varies bởi cloud providers (AWS, Azure, GCP, v.v.).
Ngoài ra, tùy thuộc vào liệu bạn muốn chạy một HTTP application điển hình với nhiều routes/endpoints hay chỉ cung cấp một single route (hoặc thực hiện một phần cụ thể của code),
code của application của bạn sẽ trông khác nhau (ví dụ, cho approach endpoint-per-function bạn có thể sử dụng `NestFactory.createApplicationContext` thay vì booting HTTP server, setting up middleware, v.v.).

Chỉ cho mục đích illustration, chúng tôi sẽ integrate Nest (sử dụng `@nestjs/platform-express` và vì vậy spinning up toàn bộ, fully functional HTTP router)
với framework Serverless (trong trường hợp này, targeting AWS Lambda). Như chúng tôi đã đề cập trước đó, code của bạn sẽ khác tùy thuộc vào cloud provider bạn chọn, và nhiều factors khác.

Đầu tiên, hãy cài đặt các packages được yêu cầu:

```bash
$ npm i @codegenie/serverless-express aws-lambda
$ npm i -D @types/aws-lambda serverless-offline
```

> info **Gợi ý** Để speed up development cycles, chúng tôi cài đặt plugin `serverless-offline` mà emulates AWS λ và API Gateway.

Khi quá trình cài đặt hoàn tất, hãy tạo file `serverless.yml` để cấu hình framework Serverless:

```yaml
service: serverless-example

plugins:
  - serverless-offline

provider:
  name: aws
  runtime: nodejs14.x

functions:
  main:
    handler: dist/main.handler
    events:
      - http:
          method: ANY
          path: /
      - http:
          method: ANY
          path: '{proxy+}'
```

> info **Gợi ý** Để tìm hiểu thêm về framework Serverless, truy cập official documentation.

Với điều này ở chỗ, bây giờ chúng ta có thể điều hướng đến file `main.ts` và cập nhật bootstrap code của chúng tôi với boilerplate được yêu cầu:

```typescript
import { NestFactory } from '@nestjs/core';
import serverlessExpress from '@codegenie/serverless-express';
import { Callback, Context, Handler } from 'aws-lambda';
import { AppModule } from './app.module';

let server: Handler;

async function bootstrap(): Promise<Handler> {
  const app = await NestFactory.create(AppModule);
  await app.init();

  const expressApp = app.getHttpAdapter().getInstance();
  return serverlessExpress({ app: expressApp });
}

export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  server = server ?? (await bootstrap());
  return server(event, context, callback);
};
```

> info **Gợi ý** Để tạo nhiều serverless functions và chia sẻ common modules giữa chúng, chúng tôi khuyến nghị sử dụng CLI Monorepo mode.

> warning **Cảnh báo** Nếu bạn sử dụng package `@nestjs/swagger`, có một số additional steps được yêu cầu để làm cho nó hoạt động đúng trong context của serverless function. Kiểm tra thread này để biết thêm thông tin.

Tiếp theo, mở file `tsconfig.json` và đảm bảo enable option `esModuleInterop` để làm cho package `@codegenie/serverless-express` load đúng.

```json
{
  "compilerOptions": {
    ...
    "esModuleInterop": true
  }
}
```

Bây giờ chúng ta có thể build application của chúng tôi (với `nest build` hoặc `tsc`) và sử dụng CLI `serverless` để start lambda function của chúng tôi locally:

```bash
$ npm run build
$ npx serverless offline
```

Khi application đang chạy, mở browser của bạn và điều hướng đến `http://localhost:3000/dev/[ANY_ROUTE]` (nơi `[ANY_ROUTE]` là bất kỳ endpoint nào được đăng ký trong application của bạn).

Trong các phần trên, chúng tôi đã chỉ ra rằng sử dụng `webpack` và bundling app của bạn có thể có significant impact trên overall bootstrap time.
Tuy nhiên, để làm cho nó hoạt động với example của chúng tôi, có một số additional configurations bạn phải thêm trong file `webpack.config.js` của bạn. Nhìn chung,
để đảm bảo function `handler` của chúng tôi sẽ được picked up, chúng tôi phải thay đổi property `output.libraryTarget` thành `commonjs2`.

```javascript
return {
  ...options,
  externals: [],
  output: {
    ...options.output,
    libraryTarget: 'commonjs2',
  },
  // ... the rest of the configuration
};
```

Với điều này ở chỗ, bây giờ bạn có thể sử dụng `$ nest build --webpack` để compile code của function của bạn (và sau đó `$ npx serverless offline` để test nó).

Nó cũng được khuyến nghị (nhưng **không được yêu cầu** vì nó sẽ làm chậm build process của bạn) để cài đặt package `terser-webpack-plugin` và override cấu hình của nó để giữ classnames nguyên vẹn khi minifying production build của bạn. Không làm như vậy có thể dẫn đến incorrect behavior khi sử dụng `class-validator` trong application của bạn.

```javascript
const TerserPlugin = require('terser-webpack-plugin');

return {
  ...options,
  externals: [],
  optimization: {
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          keep_classnames: true,
        },
      }),
    ],
  },
  output: {
    ...options.output,
    libraryTarget: 'commonjs2',
  },
  // ... the rest of the configuration
};
```

#### Using standalone application feature

Ngoài ra, nếu bạn muốn giữ function của bạn rất lightweight và bạn không cần bất kỳ HTTP-related features nào (routing, nhưng cũng guards, interceptors, pipes, v.v.),
bạn có thể chỉ sử dụng `NestFactory.createApplicationContext` (như được đề cập trước đó) thay vì chạy toàn bộ HTTP server (và `express` under the hood), như sau:

```typescript
@@filename(main)
import { HttpStatus } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { Callback, Context, Handler } from 'aws-lambda';
import { AppModule } from './app.module';
import { AppService } from './app.service';

export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  const appContext = await NestFactory.createApplicationContext(AppModule);
  const appService = appContext.get(AppService);

  return {
    body: appService.getHello(),
    statusCode: HttpStatus.OK,
  };
};
```

> info **Gợi ý** Hãy aware rằng `NestFactory.createApplicationContext` không wrap controller methods với enhancers (guard, interceptors, v.v.). Cho điều này, bạn phải sử dụng method `NestFactory.create`.

Bạn cũng có thể truyền object `event` xuống đến, hãy nói, provider `EventsService` mà có thể process nó và trả về một giá trị tương ứng (tùy thuộc vào input value và business logic của bạn).

```typescript
export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  const appContext = await NestFactory.createApplicationContext(AppModule);
  const eventsService = appContext.get(EventsService);
  return eventsService.process(event);
};
```