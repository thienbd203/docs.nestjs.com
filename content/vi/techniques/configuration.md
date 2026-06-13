### Configuration

Các ứng dụng thường chạy trong các **môi trường** khác nhau. Tùy thuộc vào môi trường, các cài đặt cấu hình khác nhau nên được sử dụng. Ví dụ, môi trường local thường dựa vào các thông tin xác thực database cụ thể, chỉ hợp lệ cho instance DB local. Môi trường production sẽ sử dụng một bộ thông tin xác thực DB riêng biệt. Vì các biến cấu hình thay đổi, best practice là [lưu trữ các biến cấu hình](https://12factor.net/config) trong môi trường.

Các biến môi trường được định nghĩa bên ngoài có thể thấy bên trong Node.js thông qua global `process.env`. Chúng ta có thể cố gắng giải quyết vấn đề của nhiều môi trường bằng cách đặt các biến môi trường riêng biệt trong mỗi môi trường. Điều này có thể trở nên khó xử lý nhanh chóng, đặc biệt là trong các môi trường phát triển và kiểm thử nơi các giá trị này cần được mock và/hoặc thay đổi một cách dễ dàng.

Trong các ứng dụng Node.js, việc sử dụng các file `.env` là phổ biến, chứa các cặp key-value trong đó mỗi key đại diện cho một giá trị cụ thể, để đại diện cho mỗi môi trường. Chạy ứng dụng trong các môi trường khác nhau sau đó chỉ là việc thay thế file `.env` phù hợp.

Một cách tiếp cận tốt để sử dụng kỹ thuật này trong Nest là tạo một `ConfigModule` expose một `ConfigService` tải file `.env` phù hợp. Mặc dù bạn có thể chọn viết một module như vậy, vì sự tiện lợi, Nest cung cấp package `@nestjs/config` out-of-the-box. Chúng tôi sẽ đề cập đến package này trong chương hiện tại.

#### Cài đặt

Để bắt đầu sử dụng nó, trước tiên chúng ta cài đặt dependency cần thiết.

```bash
$ npm i --save @nestjs/config
```

> info **Gợi ý** Package `@nestjs/config` sử dụng nội bộ [dotenv](https://github.com/motdotla/dotenv).

> warning **Lưu ý** `@nestjs/config` yêu cầu TypeScript 4.1 hoặc mới hơn.

#### Bắt đầu

Sau khi quá trình cài đặt hoàn tất, chúng ta có thể nhập `ConfigModule`. Thông thường, chúng ta sẽ nhập nó vào `AppModule` gốc và kiểm soát hành vi của nó bằng phương thức tĩnh `.forRoot()`. Trong bước này, các cặp key/value biến môi trường được phân tích cú pháp và giải quyết. Sau này, chúng ta sẽ thấy một số tùy chọn để truy cập class `ConfigService` của `ConfigModule` trong các module tính năng khác của chúng ta.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot()],
})
export class AppModule {}
```

Đoạn mã trên sẽ tải và phân tích cú pháp một file `.env` từ vị trí mặc định (thư mục gốc của dự án), hợp nhất các cặp key/value từ file `.env` với các biến môi trường được gán cho `process.env`, và lưu trữ kết quả trong một cấu trúc riêng tư mà bạn có thể truy cập thông qua `ConfigService`. Phương thức `forRoot()` đăng ký provider `ConfigService`, cung cấp phương thức `get()` để đọc các biến cấu hình đã được phân tích cú pháp/hợp nhất. Vì `@nestjs/config` dựa trên [dotenv](https://github.com/motdotla/dotenv), nó sử dụng các quy tắc của package đó để giải quyết xung đột trong tên biến môi trường. Khi một key tồn tại cả trong môi trường runtime như một biến môi trường (ví dụ, thông qua các exports shell OS như `export DATABASE_USER=test`) và trong một file `.env`, biến môi trường runtime sẽ được ưu tiên.

Một file `.env` mẫu trông giống như sau:

```json
DATABASE_USER=test
DATABASE_PASSWORD=test
```

Nếu bạn cần một số biến env có sẵn ngay cả trước khi `ConfigModule` được tải và ứng dụng Nest được bootstrap (ví dụ, để chuyển cấu hình microservice sang phương thức `NestFactory#createMicroservice`), bạn có thể sử dụng tùy chọn `--env-file` của Nest CLI. Tùy chọn này cho phép bạn chỉ định đường dẫn đến file `.env` nên được tải trước khi ứng dụng bắt đầu. Hỗ trợ flag `--env-file` được giới thiệu trong Node v20, xem [tài liệu](https://nodejs.org/dist/v20.18.1/docs/api/cli.html#--env-fileconfig) để biết thêm chi tiết.

```bash
$ nest start --env-file .env
```

#### Đường dẫn file env tùy chỉnh

Theo mặc định, package tìm file `.env` trong thư mục gốc của ứng dụng. Để chỉ định một đường dẫn khác cho file `.env`, đặt thuộc tính `envFilePath` của một đối tượng tùy chọn (tùy chọn) mà bạn truyền cho `forRoot()`, như sau:

```typescript
ConfigModule.forRoot({
  envFilePath: '.development.env',
});
```

Bạn cũng có thể chỉ định nhiều đường dẫn cho các file `.env` như sau:

```typescript
ConfigModule.forRoot({
  envFilePath: ['.env.development.local', '.env.development'],
});
```

Nếu một biến được tìm thấy trong nhiều file, file đầu tiên sẽ được ưu tiên.

#### Tắt tải biến môi trường

Nếu bạn không muốn tải file `.env`, thay vào đó chỉ muốn truy cập các biến môi trường từ môi trường runtime (như với các exports shell OS như `export DATABASE_USER=test`), đặt thuộc tính `ignoreEnvFile` của đối tượng tùy chọn thành `true`, như sau:

```typescript
ConfigModule.forRoot({
  ignoreEnvFile: true,
});
```

#### Sử dụng module toàn cục

Khi bạn muốn sử dụng `ConfigModule` trong các module khác, bạn sẽ cần nhập nó (như tiêu chuẩn với bất kỳ module Nest nào). Ngoài ra, khai báo nó là [global module](https://docs.nestjs.com/modules#global-modules) bằng cách đặt thuộc tính `isGlobal` của đối tượng tùy chọn thành `true`, như hiển thị bên dưới. Trong trường hợp đó, bạn sẽ không cần nhập `ConfigModule` trong các module khác sau khi nó đã được tải trong module gốc (ví dụ, `AppModule`).

```typescript
ConfigModule.forRoot({
  isGlobal: true,
});
```

#### File cấu hình tùy chỉnh

Đối với các dự án phức tạp hơn, bạn có thể sử dụng các file cấu hình tùy chỉnh để trả về các đối tượng cấu hình lồng nhau. Điều này cho phép bạn nhóm các cài đặt cấu hình liên quan theo chức năng (ví dụ, các cài đặt liên quan đến database), và lưu trữ các cài đặt liên quan trong các file riêng biệt để giúp quản lý chúng độc lập.

Một file cấu hình tùy chỉnh xuất một factory function trả về một đối tượng cấu hình. Đối tượng cấu hình có thể là bất kỳ đối tượng JavaScript thuần lồng nhau tùy ý. Đối tượng `process.env` sẽ chứa các cặp key/value biến môi trường được giải quyết hoàn toàn (với file `.env` và các biến được định nghĩa bên ngoài được giải quyết và hợp nhất như mô tả <a href="techniques/configuration#getting-started">ở trên</a>). Vì bạn kiểm soát đối tượng cấu hình trả về, bạn có thể thêm bất kỳ logic cần thiết để ép kiểu các giá trị thành kiểu phù hợp, đặt giá trị mặc định, v.v. Ví dụ:

```typescript
@@filename(config/configuration)
export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT, 10) || 5432
  }
});
```

Chúng ta tải file này bằng thuộc tính `load` của đối tượng tùy chọn mà chúng ta truyền cho phương thức `ConfigModule.forRoot()`:

```typescript
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration],
    }),
  ],
})
export class AppModule {}
```

> info **Thông báo** Giá trị được gán cho thuộc tính `load` là một mảng, cho phép bạn tải nhiều file cấu hình (ví dụ: `load: [databaseConfig, authConfig]`)

Với các file cấu hình tùy chỉnh, chúng ta cũng có thể quản lý các file tùy chỉnh như các file YAML. Dưới đây là một ví dụ về cấu hình sử dụng định dạng YAML:

```yaml
http:
  host: 'localhost'
  port: 8080

db:
  postgres:
    url: 'localhost'
    port: 5432
    database: 'yaml-db'

  sqlite:
    database: 'sqlite.db'
```

Để đọc và phân tích cú pháp các file YAML, chúng ta có thể tận dụng package `js-yaml`.

```bash
$ npm i js-yaml
$ npm i -D @types/js-yaml
```

Sau khi package được cài đặt, chúng ta sử dụng hàm `yaml#load` để tải file YAML mà chúng ta vừa tạo ở trên.

```typescript
@@filename(config/configuration)
import { readFileSync } from 'node:fs';
import { join } from 'node:path';
import * as yaml from 'js-yaml';

const YAML_CONFIG_FILENAME = 'config.yaml';

export default () => {
  return yaml.load(
    readFileSync(join(__dirname, YAML_CONFIG_FILENAME), 'utf8'),
  ) as Record<string, any>;
};
```

> warning **Lưu ý** Nest CLI không tự động di chuyển các "asset" (file non-TS) của bạn vào thư mục `dist` trong quá trình build. Để đảm bảo rằng các file YAML của bạn được sao chép, bạn phải chỉ định điều này trong đối tượng `compilerOptions#assets` trong file `nest-cli.json`. Ví dụ, nếu thư mục `config` ở cùng cấp với thư mục `src`, thêm `compilerOptions#assets` với giá trị `"assets": [{{ '{' }}"include": "../config/*.yaml", "outDir": "./dist/config"{{ '}' }}]`. Đọc thêm [tại đây](/cli/monorepo#assets).

Chỉ một lưu ý nhanh - các file cấu hình không được tự động xác thực, ngay cả khi bạn đang sử dụng tùy chọn `validationSchema` trong `ConfigModule` của NestJS. Nếu bạn cần xác thực hoặc muốn áp dụng bất kỳ chuyển đổi nào, bạn sẽ phải xử lý điều đó trong factory function nơi bạn có kiểm soát hoàn toàn đối tượng cấu hình. Điều này cho phép bạn implement bất kỳ logic xác thực tùy chỉnh nào theo cần.

Ví dụ, nếu bạn muốn đảm bảo rằng port nằm trong một phạm vi nhất định, bạn có thể thêm bước xác thực vào factory function:

```typescript
@@filename(config/configuration)
export default () => {
  const config = yaml.load(
    readFileSync(join(__dirname, YAML_CONFIG_FILENAME), 'utf8'),
  ) as Record<string, any>;

  if (config.http.port < 1024 || config.http.port > 49151) {
    throw new Error('HTTP port must be between 1024 and 49151');
  }

  return config;
};
```

Bây giờ, nếu port nằm ngoài phạm vi đã chỉ định, ứng dụng sẽ throw một lỗi trong quá trình khởi động.

<app-banner-devtools></app-banner-devtools>

#### Sử dụng `ConfigService`

Để truy cập các giá trị cấu hình từ `ConfigService` của chúng ta, trước tiên chúng ta cần inject `ConfigService`. Như với bất kỳ provider nào, chúng ta cần nhập module chứa nó - `ConfigModule` - vào module sẽ sử dụng nó (trừ khi bạn đặt thuộc tính `isGlobal` trong đối tượng tùy chọn được truyền cho phương thức `ConfigModule.forRoot()` thành `true`). Nhập nó vào một module tính năng như hiển thị bên dưới.

```typescript
@@filename(feature.module)
@Module({
  imports: [ConfigModule],
  // ...
})
```

Sau đó chúng ta có thể inject nó bằng cách sử dụng constructor injection tiêu chuẩn:

```typescript
constructor(private configService: ConfigService) {}
```

> info **Gợi ý** `ConfigService` được nhập từ package `@nestjs/config`.

Và sử dụng nó trong class của chúng ta:

```typescript
// get an environment variable
const dbUser = this.configService.get<string>('DATABASE_USER');

// get a custom configuration value
const dbHost = this.configService.get<string>('database.host');
```

Như hiển thị ở trên, sử dụng phương thức `configService.get()` để lấy một biến môi trường đơn giản bằng cách truyền tên biến. Bạn có thể thực hiện type hinting của TypeScript bằng cách truyền kiểu, như hiển thị ở trên (ví dụ, `get<string>(...)`). Phương thức `get()` cũng có thể duyệt qua một đối tượng cấu hình tùy chỉnh lồng nhau (được tạo thông qua <a href="techniques/configuration#custom-configuration-files">File cấu hình tùy chỉnh</a>), như hiển thị trong ví dụ thứ hai ở trên.

Bạn cũng có thể lấy toàn bộ đối tượng cấu hình tùy chỉnh lồng nhau bằng cách sử dụng một interface làm type hint:

```typescript
interface DatabaseConfig {
  host: string;
  port: number;
}

const dbConfig = this.configService.get<DatabaseConfig>('database');

// you can now use `dbConfig.port` and `dbConfig.host`
const port = dbConfig.port;
```

Phương thức `get()` cũng nhận một đối số thứ hai tùy chỉnh định nghĩa một giá trị mặc định, sẽ được trả về khi key không tồn tại, như hiển thị bên dưới:

```typescript
// use "localhost" when "database.host" is not defined
const dbHost = this.configService.get<string>('database.host', 'localhost');
```

`ConfigService` có hai generics (tham số kiểu) tùy chọn. Generic đầu tiên là để giúp ngăn chặn việc truy cập một thuộc tính cấu hình không tồn tại. Sử dụng nó như hiển thị bên dưới:

```typescript
interface EnvironmentVariables {
  PORT: number;
  TIMEOUT: string;
}

// somewhere in the code
constructor(private configService: ConfigService<EnvironmentVariables>) {
  const port = this.configService.get('PORT', { infer: true });

  // TypeScript Error: this is invalid as the URL property is not defined in EnvironmentVariables
  const url = this.configService.get('URL', { infer: true });
}
```

Với thuộc tính `infer` được đặt thành `true`, phương thức `ConfigService#get` sẽ tự động suy ra kiểu thuộc tính dựa trên interface, vì vậy ví dụ, `typeof port === "number"` (nếu bạn không sử dụng flag `strictNullChecks` của TypeScript) vì `PORT` có kiểu `number` trong interface `EnvironmentVariables`.

Ngoài ra, với tính năng `infer`, bạn có thể suy ra kiểu của thuộc tính của đối tượng cấu hình tùy chỉnh lồng nhau, ngay cả khi sử dụng ký hiệu dot, như sau:

```typescript
constructor(private configService: ConfigService<{ database: { host: string } }>) {
  const dbHost = this.configService.get('database.host', { infer: true })!;
  // typeof dbHost === "string"                                          |
  //                                                                     +--> non-null assertion operator
}
```

Generic thứ hai dựa vào generic đầu tiên, hoạt động như một type assertion để loại bỏ tất cả các kiểu `undefined` mà các phương thức của `ConfigService` có thể trả về khi `strictNullChecks` được bật. Ví dụ:

```typescript
// ...
constructor(private configService: ConfigService<{ PORT: number }, true>) {
  //                                                               ^^^^
  const port = this.configService.get('PORT', { infer: true });
  //    ^^^ The type of port will be 'number' thus you don't need TS type assertions anymore
}
```

> info **Gợi ý** Để đảm bảo rằng phương thức `ConfigService#get` truy xuất các giá trị chỉ từ các file cấu hình tùy chỉnh và bỏ qua các biến `process.env`, đặt tùy chọn `skipProcessEnv` thành `true` trong đối tượng tùy chọn của phương thức `forRoot()` của `ConfigModule`.

#### Namespaces cấu hình

`ConfigModule` cho phép bạn định nghĩa và tải nhiều file cấu hình tùy chỉnh, như hiển thị trong <a href="techniques/configuration#custom-configuration-files">File cấu hình tùy chỉnh</a> ở trên. Bạn có thể quản lý các phân cấp đối tượng cấu hình phức tạp với các đối tượng cấu hình lồng nhau như hiển thị trong phần đó. Ngoài ra, bạn có thể trả về một đối tượng cấu hình "namespaced" với hàm `registerAs()` như sau:

```typescript
@@filename(config/database.config)
export default registerAs('database', () => ({
  host: process.env.DATABASE_HOST,
  port: process.env.DATABASE_PORT || 5432
}));
```

Như với các file cấu hình tùy chỉnh, bên trong factory function `registerAs()` của bạn, đối tượng `process.env` sẽ chứa các cặp key/value biến môi trường được giải quyết hoàn toàn (với file `.env` và các biến được định nghĩa bên ngoài được giải quyết và hợp nhất như mô tả <a href="techniques/configuration#getting-started">ở trên</a>).

> info **Gợi ý** Hàm `registerAs` được xuất từ package `@nestjs/config`.

Tải một cấu hình namespaced với thuộc tính `load` của đối tượng tùy chọn của phương thức `forRoot()`, theo cùng cách bạn tải một file cấu hình tùy chỉnh:

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [databaseConfig],
    }),
  ],
})
export class AppModule {}
```

Bây giờ, để lấy giá trị `host` từ namespace `database`, sử dụng ký hiệu dot. Sử dụng `'database'` làm tiền tố cho tên thuộc tính, tương ứng với tên của namespace (được truyền làm đối số đầu tiên cho hàm `registerAs()`):

```typescript
const dbHost = this.configService.get<string>('database.host');
```

Một cách thay thế hợp lý là inject namespace `database` trực tiếp. Điều này cho phép chúng ta hưởng lợi từ strong typing:

```typescript
constructor(
  @Inject(databaseConfig.KEY)
  private dbConfig: ConfigType<typeof databaseConfig>,
) {}
```

> info **Gợi ý** `ConfigType` được xuất từ package `@nestjs/config`.

#### Cấu hình namespaced trong các module

Để sử dụng một cấu hình namespaced làm đối tượng cấu hình cho một module khác trong ứng dụng của bạn, bạn có thể tận dụng phương thức `.asProvider()` của đối tượng cấu hình. Phương thức này chuyển đổi cấu hình namespaced của bạn thành một provider, sau đó có thể được truyền cho `forRootAsync()` (hoặc bất kỳ phương thức tương đương nào) của module bạn muốn sử dụng.

Dưới đây là một ví dụ:

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [
    TypeOrmModule.forRootAsync(databaseConfig.asProvider()),
  ],
})
```

Để hiểu cách phương thức `.asProvider()` hoạt động, hãy xem xét giá trị trả về:

```typescript
// Return value of the .asProvider() method
{
  imports: [ConfigModule.forFeature(databaseConfig)],
  useFactory: (configuration: ConfigType<typeof databaseConfig>) => configuration,
  inject: [databaseConfig.KEY]
}
```

Cấu trúc này cho phép bạn tích hợp liền mạch các cấu hình namespaced vào các module của mình, đảm bảo rằng ứng dụng của bạn vẫn được tổ chức và module hóa, mà không cần viết code boilerplate, lặp lại.

#### Cache biến môi trường

Vì việc truy cập `process.env` có thể chậm, bạn có thể đặt thuộc tính `cache` của đối tượng tùy chọn được truyền cho `ConfigModule.forRoot()` để tăng hiệu suất của phương thức `ConfigService#get` khi đến các biến được lưu trữ trong `process.env`.

```typescript
ConfigModule.forRoot({
  cache: true,
});
```

#### Đăng ký một phần

Cho đến nay, chúng ta đã xử lý các file cấu hình trong module gốc của chúng ta (ví dụ, `AppModule`), với phương thức `forRoot()`. Có thể bạn có cấu trúc dự án phức tạp hơn, với các file cấu hình cụ thể tính năng nằm trong nhiều thư mục khác nhau. Thay vì tải tất cả các file này trong module gốc, package `@nestjs/config` cung cấp một tính năng gọi là **đăng ký một phần**, chỉ tham chiếu các file cấu hình được liên kết với mỗi module tính năng. Sử dụng phương thức tĩnh `forFeature()` trong một module tính năng để thực hiện đăng ký một phần này, như sau:

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [ConfigModule.forFeature(databaseConfig)],
})
export class DatabaseModule {}
```

> info **Cảnh báo** Trong một số trường hợp, bạn có thể cần truy cập các thuộc tính được tải thông qua đăng ký một phần bằng cách sử dụng hook `onModuleInit()`, thay vì trong một constructor. Điều này là do phương thức `forFeature()` được chạy trong quá trình khởi tạo module, và thứ tự khởi tạo module là không xác định. Nếu bạn truy cập các giá trị được tải theo cách này bởi một module khác, trong một constructor, module mà cấu hình phụ thuộc có thể chưa được khởi tạo. Phương thức `onModuleInit()` chỉ chạy sau khi tất cả các module mà nó phụ thuộc đã được khởi tạo, do đó kỹ thuật này là an toàn.

#### Xác thực schema

Đây là thực hành tiêu chuẩn để throw một exception trong quá trình khởi động ứng dụng nếu các biến môi trường bắt buộc chưa được cung cấp hoặc nếu chúng không đáp ứng các quy tắc xác thực nhất định. Package `@nestjs/config` cho phép hai cách khác nhau để làm điều này:

- [Joi](https://github.com/sideway/joi) validator tích hợp sẵn. Với Joi, bạn định nghĩa một schema đối tượng và xác thực các đối tượng JavaScript dựa trên nó.
- Một hàm `validate()` tùy chỉnh nhận các biến môi trường làm đầu vào.

Để sử dụng Joi, chúng ta phải cài đặt package Joi:

```bash
$ npm install --save joi
```

Bây giờ chúng ta có thể định nghĩa một schema xác thực Joi và truyền nó thông qua thuộc tính `validationSchema` của đối tượng tùy chọn của phương thức `forRoot()`, như hiển thị bên dưới:

```typescript
@@filename(app.module)
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().port().default(3000),
      }),
    }),
  ],
})
export class AppModule {}
```

Theo mặc định, tất cả các key schema được coi là tùy chọn. Ở đây, chúng ta đặt các giá trị mặc định cho `NODE_ENV` và `PORT` sẽ được sử dụng nếu chúng ta không cung cấp các biến này trong môi trường (file `.env` hoặc môi trường process). Ngoài ra, chúng ta có thể sử dụng phương thức xác thực `required()` để yêu cầu rằng một giá trị phải được định nghĩa trong môi trường (file `.env` hoặc môi trường process). Trong trường hợp này, bước xác thực sẽ throw một exception nếu chúng ta không cung cấp biến trong môi trường. Xem [phương thức xác thực Joi](https://joi.dev/api/?v=17.3.0#example) để biết thêm về cách xây dựng các schema xác thực.

Theo mặc định, các biến môi trường không xác định (các biến môi trường có key không có trong schema) được cho phép và không kích hoạt exception xác thực. Theo mặc định, tất cả các lỗi xác thực được báo cáo. Bạn có thể thay đổi các hành vi này bằng cách truyền một đối tượng tùy chọn thông qua key `validationOptions` của đối tượng tùy chọn `forRoot()`. Đối tượng tùy chọn này có thể chứa bất kỳ thuộc tính tùy chọn xác thực tiêu chuẩn nào được cung cấp bởi [tùy chọn xác thực Joi](https://joi.dev/api/?v=17.3.0#anyvalidatevalue-options). Ví dụ, để đảo ngược hai cài đặt ở trên, truyền các tùy chọn như sau:

```typescript
@@filename(app.module)
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().port().default(3000),
      }),
      validationOptions: {
        allowUnknown: false,
        abortEarly: true,
      },
    }),
  ],
})
export class AppModule {}
```

Package `@nestjs/config` sử dụng các cài đặt mặc định của:

- `allowUnknown`: kiểm soát có cho phép các key không xác định trong các biến môi trường hay không. Mặc định là `true`
- `abortEarly`: nếu true, dừng xác thực tại lỗi đầu tiên; nếu false, trả về tất cả các lỗi. Mặc định là `false`.

Lưu ý rằng một khi bạn quyết định truyền một đối tượng `validationOptions`, bất kỳ cài đặt nào bạn không truyền rõ ràng sẽ mặc định thành các mặc định tiêu chuẩn của `Joi` (không phải mặc định của `@nestjs/config`). Ví dụ, nếu bạn để `allowUnknowns` không được chỉ định trong đối tượng `validationOptions` tùy chỉnh của bạn, nó sẽ có giá trị mặc định của `Joi` là `false`. Do đó, có lẽ an toàn nhất khi chỉ định **cả hai** cài đặt này trong đối tượng tùy chỉnh của bạn.

> info **Gợi ý** Để tắt xác thực của các biến môi trường được định nghĩa trước, đặt thuộc tính `validatePredefined` thành `false` trong đối tượng tùy chọn của phương thức `forRoot()`. Các biến môi trường được định nghĩa trước là các biến process (biến `process.env`) được đặt trước khi module được nhập. Ví dụ, nếu bạn bắt đầu ứng dụng của bạn với `PORT=3000 node main.js`, thì `PORT` là một biến môi trường được định nghĩa trước.

#### Hàm validate tùy chỉnh

Ngoài ra, bạn có thể chỉ định một hàm `validate` **đồng bộ** nhận một đối tượng chứa các biến môi trường (từ file env và process) và trả về một đối tượng chứa các biến môi trường đã xác thực để bạn có thể chuyển đổi/mutate chúng nếu cần. Nếu hàm throw một lỗi, nó sẽ ngăn ứng dụng bootstrapping.

Trong ví dụ này, chúng ta sẽ tiếp tục với các package `class-transformer` và `class-validator`. Đầu tiên, chúng ta phải định nghĩa:

- một class với các ràng buộc xác thực,
- một hàm validate tận dụng các hàm `plainToInstance` và `validateSync`.

```typescript
@@filename(env.validation)
import { plainToInstance } from 'class-transformer';
import { IsEnum, IsNumber, Max, Min, validateSync } from 'class-validator';

enum Environment {
  Development = "development",
  Production = "production",
  Test = "test",
  Provision = "provision",
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsNumber()
  @Min(0)
  @Max(65535)
  PORT: number;
}

export function validate(config: Record<string, unknown>) {
  const validatedConfig = plainToInstance(
    EnvironmentVariables,
    config,
    { enableImplicitConversion: true },
  );
  const errors = validateSync(validatedConfig, { skipMissingProperties: false });

  if (errors.length > 0) {
    throw new Error(errors.toString());
  }
  return validatedConfig;
}
```

Với điều này, sử dụng hàm `validate` làm tùy chọn cấu hình của `ConfigModule`, như sau:

```typescript
@@filename(app.module)
import { validate } from './env.validation';

@Module({
  imports: [
    ConfigModule.forRoot({
      validate,
    }),
  ],
})
export class AppModule {}
```

#### Hàm getter tùy chỉnh

`ConfigService` định nghĩa một phương thức generic `get()` để truy xuất giá trị cấu hình theo key. Chúng ta cũng có thể thêm các hàm `getter` để cho phép một phong cách code tự nhiên hơn một chút:

```typescript
@@filename()
@Injectable()
export class ApiConfigService {
  constructor(private configService: ConfigService) {}

  get isAuthEnabled(): boolean {
    return this.configService.get('AUTH_ENABLED') === 'true';
  }
}
@@switch
@Dependencies(ConfigService)
@Injectable()
export class ApiConfigService {
  constructor(configService) {
    this.configService = configService;
  }

  get isAuthEnabled() {
    return this.configService.get('AUTH_ENABLED') === 'true';
  }
}
```

Bây giờ chúng ta có thể sử dụng hàm getter như sau:

```typescript
@@filename(app.service)
@Injectable()
export class AppService {
  constructor(apiConfigService: ApiConfigService) {
    if (apiConfigService.isAuthEnabled) {
      // Authentication is enabled
    }
  }
}
@@switch
@Dependencies(ApiConfigService)
@Injectable()
export class AppService {
  constructor(apiConfigService) {
    if (apiConfigService.isAuthEnabled) {
      // Authentication is enabled
    }
  }
}
```

#### Hook biến môi trường đã tải

Nếu cấu hình module phụ thuộc vào các biến môi trường, và các biến này được tải từ file `.env`, bạn có thể sử dụng hook `ConfigModule.envVariablesLoaded` để đảm bảo rằng file đã được tải trước khi tương tác với đối tượng `process.env`, xem ví dụ sau:

```typescript
export async function getStorageModule() {
  await ConfigModule.envVariablesLoaded;
  return process.env.STORAGE === 'S3' ? S3StorageModule : DefaultStorageModule;
}
```

Cấu trúc này đảm bảo rằng sau khi Promise `ConfigModule.envVariablesLoaded` giải quyết, tất cả các biến cấu hình được tải lên.

#### Cấu hình module có điều kiện

Có thể có lúc bạn muốn tải một module có điều kiện và chỉ định điều kiện trong một biến env. May mắn thay, `@nestjs/config` cung cấp một `ConditionalModule` cho phép bạn làm đúng điều đó.

```typescript
@Module({
  imports: [
    ConfigModule.forRoot(),
    ConditionalModule.registerWhen(FooModule, 'USE_FOO'),
  ],
})
export class AppModule {}
```

Module trên sẽ chỉ tải `FooModule` nếu trong file `.env` không có giá trị `false` cho biến môi trường `USE_FOO`. Bạn cũng có thể truyền một điều kiện tùy chỉnh của chính mình, một hàm nhận tham chiếu `process.env` nên trả về một boolean để `ConditionalModule` xử lý:

```typescript
@Module({
  imports: [
    ConfigModule.forRoot(),
    ConditionalModule.registerWhen(
      FooBarModule,
      (env: NodeJS.ProcessEnv) => !!env['foo'] && !!env['bar'],
    ),
  ],
})
export class AppModule {}
```

Quan trọng là phải đảm bảo rằng khi sử dụng `ConditionalModule`, bạn cũng có `ConfigModule` được tải trong ứng dụng, để hook `ConfigModule.envVariablesLoaded` có thể được tham chiếu và sử dụng đúng cách. Nếu hook không được chuyển sang true trong vòng 5 giây, hoặc một timeout tính bằng mili giây, được đặt bởi người dùng trong tham số tùy chọn thứ ba của phương thức `registerWhen`, thì `ConditionalModule` sẽ throw một lỗi và Nest sẽ hủy bỏ việc bắt đầu ứng dụng.

#### Biến mở rộng

Package `@nestjs/config` hỗ trợ mở rộng biến môi trường. Với kỹ thuật này, bạn có thể tạo các biến môi trường lồng nhau, trong đó một biến được tham chiếu trong định nghĩa của một biến khác. Ví dụ:

```json
APP_URL=mywebsite.com
SUPPORT_EMAIL=support@${APP_URL}
```

Với cấu trúc này, biến `SUPPORT_EMAIL` giải quyết thành `'support@mywebsite.com'`. Lưu ý việc sử dụng cú pháp `${{ '{' }}...{{ '}' }}` để kích hoạt giải quyết giá trị của biến `APP_URL` trong định nghĩa của `SUPPORT_EMAIL`.

> info **Gợi ý** Đối với tính năng này, package `@nestjs/config` sử dụng nội bộ [dotenv-expand](https://github.com/motdotla/dotenv-expand).

Bật mở rộng biến môi trường bằng cách sử dụng thuộc tính `expandVariables` trong đối tượng tùy chọn được truyền cho phương thức `forRoot()` của `ConfigModule`, như hiển thị bên dưới:

```typescript
@@filename(app.module)
@Module({
  imports: [
    ConfigModule.forRoot({
      // ...
      expandVariables: true,
    }),
  ],
})
export class AppModule {}
```

#### Sử dụng trong `main.ts`

Trong khi cấu hình của chúng ta được lưu trữ trong một service, nó vẫn có thể được sử dụng trong file `main.ts`. Cách này, bạn có thể sử dụng nó để lưu trữ các biến như port ứng dụng hoặc host CORS.

Để truy cập nó, bạn phải sử dụng phương thức `app.get()`, theo sau là tham chiếu service:

```typescript
const configService = app.get(ConfigService);
```

Sau đó bạn có thể sử dụng nó như bình thường, bằng cách gọi phương thức `get` với key cấu hình:

```typescript
const port = configService.get('PORT');
```