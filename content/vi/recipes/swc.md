### SWC

[SWC](https://swc.rs/) (Speedy Web Compiler) là một nền tảng dựa trên Rust có thể mở rộng có thể được sử dụng cho cả biên dịch và bundling.
Sử dụng SWC với Nest CLI là một cách tuyệt vời và đơn giản để tăng đáng kể tốc độ quá trình phát triển của bạn.

> info **Gợi ý** SWC nhanh hơn khoảng **x20 lần** so với trình biên dịch TypeScript mặc định.

#### Cài đặt

Để bắt đầu, trước tiên cài đặt một vài gói:

```bash
$ npm i --save-dev @swc/cli @swc/core
```

#### Bắt đầu

Khi quá trình cài đặt hoàn tất, bạn có thể sử dụng builder `swc` với Nest CLI, như sau:

```bash
$ nest start -b swc
# HOẶC nest start --builder swc
```

> info **Gợi ý** Nếu repository của bạn là một monorepo, hãy xem [phần này](/recipes/swc#monorepo).

Thay vì truyền cờ `-b` bạn cũng có thể chỉ đặt thuộc tính `compilerOptions.builder` thành `"swc"` trong file `nest-cli.json` của bạn, như sau:

```json
{
  "compilerOptions": {
    "builder": "swc"
  }
}
```

Để tùy chỉnh hành vi của builder, bạn có thể truyền một đối tượng chứa hai thuộc tính, `type` (`"swc"`) và `options`, như sau:

```json
{
  "compilerOptions": {
    "builder": {
      "type": "swc",
      "options": {
        "swcrcPath": "infrastructure/.swcrc",
      }
    }
  }
}
```

Ví dụ, để làm cho swc biên dịch các file `.jsx` và `.tsx`, hãy làm:

```json
{
  "compilerOptions": {
    "builder": {
      "type": "swc",
      "options": { "extensions": [".ts", ".tsx", ".js", ".jsx"] }
    },
  }
}

```

Để chạy ứng dụng ở chế độ watch, sử dụng lệnh sau:

```bash
$ nest start -b swc -w
# HOẶC nest start --builder swc --watch
```

#### Kiểm tra kiểu

SWC không thực hiện bất kỳ kiểm tra kiểu nào (ngược lại với trình biên dịch TypeScript mặc định), vì vậy để bật nó, bạn cần sử dụng cờ `--type-check`:

```bash
$ nest start -b swc --type-check
```

Lệnh này sẽ hướng dẫn Nest CLI chạy `tsc` ở chế độ `noEmit` cùng với SWC, sẽ thực hiện kiểm tra kiểu bất đồng bộ. Một lần nữa, thay vì truyền cờ `--type-check` bạn cũng có thể chỉ đặt thuộc tính `compilerOptions.typeCheck` thành `true` trong file `nest-cli.json` của bạn, như sau:

```json
{
  "compilerOptions": {
    "builder": "swc",
    "typeCheck": true
  }
}
```

#### CLI Plugins (SWC)

Cờ `--type-check` sẽ tự động thực thi **CLI Plugins NestJS** và tạo một file metadata được serialized sau đó có thể được tải bởi ứng dụng tại runtime.

#### Cấu hình SWC

Builder SWC được cấu hình trước để khớp với các yêu cầu của các ứng dụng NestJS. Tuy nhiên, bạn có thể tùy chỉnh cấu hình bằng cách tạo một file `.swcrc` trong thư mục gốc và tinh chỉnh các tùy chọn theo ý muốn.

```json
{
  "$schema": "https://swc.rs/schema.json",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

#### Monorepo

Nếu repository của bạn là một monorepo, thì thay vì sử dụng builder `swc` bạn phải cấu hình `webpack` để sử dụng `swc-loader`.

Trước tiên, hãy cài đặt gói cần thiết:

```bash
$ npm i --save-dev swc-loader
```

Khi cài đặt hoàn tất, tạo một file `webpack.config.js` trong thư mục gốc của ứng dụng của bạn với nội dung sau:

```js
const swcDefaultConfig = require('@nestjs/cli/lib/compiler/defaults/swc-defaults').swcDefaultsFactory().swcOptions;

module.exports = {
  module: {
    rules: [
      {
        test: /\.ts$/,
        exclude: /node_modules/,
        use: {
          loader: 'swc-loader',
          options: swcDefaultConfig,
        },
      },
    ],
  },
};
```

#### Monorepo và CLI plugins

Bây giờ nếu bạn sử dụng CLI plugins, `swc-loader` sẽ không tải chúng tự động. Thay vào đó, bạn phải tạo một file riêng sẽ tải chúng thủ công. Để làm điều đó,
khai báo một file `generate-metadata.ts` gần file `main.ts` với nội dung sau:

```ts
import { PluginMetadataGenerator } from '@nestjs/cli/lib/compiler/plugins/plugin-metadata-generator';
import { ReadonlyVisitor } from '@nestjs/swagger/dist/plugin';

const generator = new PluginMetadataGenerator();
generator.generate({
  visitors: [new ReadonlyVisitor({ introspectComments: true, pathToSource: __dirname })],
  outputDir: __dirname,
  watch: true,
  tsconfigPath: 'apps/<name>/tsconfig.app.json',
});
```

> info **Gợi ý** Trong ví dụ này chúng ta sử dụng plugin `@nestjs/swagger`, nhưng bạn có thể sử dụng bất kỳ plugin nào bạn chọn.

Phương thức `generate()` chấp nhận các tùy chọn sau:

||                    |                                                                                                |
|| ------------------ | ---------------------------------------------------------------------------------------------- |
|| `watch`            | Có watch dự án cho các thay đổi không.                                                      |
|| `tsconfigPath`     | Đường dẫn đến file `tsconfig.json`. Liên quan đến thư mục làm việc hiện tại (`process.cwd()`). |
|| `outputDir`        | Đường dẫn đến thư mục nơi file metadata sẽ được lưu.                                   |
|| `visitors`         | Một mảng các visitors sẽ được sử dụng để tạo metadata.                                   |
|| `filename`         | Tên của file metadata. Mặc định là `metadata.ts`.                                      |
|| `printDiagnostics` | Có in chẩn đoán ra console không. Mặc định là `true`.                               |

Cuối cùng, bạn có thể chạy script `generate-metadata` trong một cửa sổ terminal riêng biệt với lệnh sau:

```bash
$ npx ts-node src/generate-metadata.ts
# HOẶC npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

#### Các cạm bẫy phổ biến

Nếu bạn sử dụng TypeORM/MikroORM hoặc bất kỳ ORM nào khác trong ứng dụng của bạn, bạn có thể gặp các vấn đề import vòng tròn. SWC không xử lý **circular imports** tốt, vì vậy bạn nên sử dụng workaround sau:

```typescript
@Entity()
export class User {
  @OneToOne(() => Profile, (profile) => profile.user)
  profile: Relation<Profile>; // <--- xem kiểu "Relation<>" ở đây thay vì chỉ "Profile"
}
```

> info **Gợi ý** Kiểu `Relation` được xuất từ gói `typeorm`.

Làm điều này ngăn loại của thuộc tính được lưu trong mã đã biên dịch lại trong metadata thuộc tính, ngăn các vấn đề phụ thuộc vòng tròn.

Nếu ORM của bạn không cung cấp workaround tương tự, bạn có thể định nghĩa kiểu wrapper của chính mình:

```typescript
/**
 * Kiểu wrapper được sử dụng để tránh vấn đề phụ thuộc vòng tròn module ESM
 * gây ra bởi reflection metadata lưu loại của thuộc tính.
 */
export type WrapperType<T> = T; // WrapperType === Relation
```

Đối với tất cả [các injection phụ thuộc vòng tròn](/fundamentals/circular-dependency) trong dự án của bạn, bạn cũng sẽ cần sử dụng kiểu wrapper tùy chỉnh được mô tả ở trên:

```typescript
@Injectable()
export class UsersService {
  constructor(
    @Inject(forwardRef(() => ProfileService))
    private readonly profileService: WrapperType<ProfileService>,
  ) {};
}
```

### Jest + SWC

Để sử dụng SWC với Jest, bạn cần cài đặt các gói sau:

```bash
$ npm i --save-dev jest @swc/core @swc/jest
```

Khi cài đặt hoàn tất, cập nhật file `package.json`/`jest.config.js` (tùy thuộc vào cấu hình của bạn) với nội dung sau:

```json
{
  "jest": {
    "transform": {
      "^.+\\.(t|j)s?$": ["@swc/jest"]
    }
  }
}
```

Ngoài ra bạn sẽ cần thêm các thuộc tính `transform` sau vào file `.swcrc` của bạn: `legacyDecorator`, `decoratorMetadata`:

```json
{
  "$schema": "https://swc.rs/schema.json",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "transform": {
      "legacyDecorator": true,
      "decoratorMetadata": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

Nếu bạn sử dụng NestJS CLI Plugins trong dự án của bạn, bạn sẽ phải chạy `PluginMetadataGenerator` thủ công. Điều hướng đến [phần này](/recipes/swc#monorepo-and-cli-plugins) để biết thêm.

### Vitest

[Vitest](https://vitest.dev/) là một trình chạy kiểm tra nhanh và nhẹ được thiết kế để làm việc với Vite. Nó cung cấp một giải pháp kiểm tra hiện đại, nhanh và dễ sử dụng có thể được tích hợp với các dự án NestJS.

#### Cài đặt

Để bắt đầu, trước tiên cài đặt các gói cần thiết:

```bash
$ npm i --save-dev vitest unplugin-swc @swc/core @vitest/coverage-v8
```

#### Cấu hình

Tạo một file `vitest.config.ts` trong thư mục gốc của ứng dụng của bạn với nội dung sau:

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    root: './',
  },
  plugins: [
    // Điều này được yêu cầu để xây dựng các file kiểm tra với SWC
    swc.vite({
      // Đặt rõ kiểu module để tránh thừa kế giá trị này từ file cấu hình `.swcrc`
      module: { type: 'es6' },
    }),
  ],
  resolve: {
    alias: {
      // Đảm bảo Vitest giải quyết chính xác các alias đường dẫn TypeScript
      'src': resolve(__dirname, './src'),
    },
  },
});
```

File cấu hình này thiết lập môi trường Vitest, thư mục gốc, và plugin SWC. Bạn cũng nên tạo một file cấu hình
riêng cho các kiểm tra e2e, với một trường `include` bổ sung chỉ định nghĩa regex đường dẫn kiểm tra:

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    root: './',
  },
  plugins: [swc.vite()],
});
```

Ngoài ra, bạn có thể đặt các tùy chọn `alias` để hỗ trợ các đường dẫn TypeScript trong các kiểm tra của bạn:

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    alias: {
      '@src': './src',
      '@test': './test',
    },
    root: './',
  },
  resolve: {
    alias: {
      '@src': './src',
      '@test': './test',
    },
  },
  plugins: [swc.vite()],
});
```

### Alias đường dẫn

Khác với Jest, Vitest không tự động giải quyết các alias đường dẫn TypeScript như `src/`. Điều này có thể dẫn đến các lỗi giải quyết phụ thuộc trong quá trình kiểm tra. Để giải quyết vấn đề này, thêm cấu hình `resolve.alias` sau trong file `vitest.config.ts` của bạn:

```ts
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      'src': resolve(__dirname, './src'),
    },
  },
});
```
Điều này đảm bảo rằng Vitest giải quyết đúng các imports module, ngăn các lỗi liên quan đến các phụ thuộc bị thiếu.

#### Cập nhật imports trong các kiểm tra E2E

Thay đổi bất kỳ imports kiểm tra E2E sử dụng `import * as request from 'supertest'` thành `import request from 'supertest'`. Điều này là cần thiết vì Vitest, khi được bundle với Vite, mong đợi một import mặc định cho supertest. Sử dụng một import namespace có thể gây ra vấn đề trong thiết lập cụ thể này.

Cuối cùng, cập nhật các scripts kiểm tra trong file package.json của bạn thành như sau:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:cov": "vitest run --coverage",
    "test:debug": "vitest --inspect-brk --inspect --logHeapUsage --threads=false",
    "test:e2e": "vitest run --config ./vitest.config.e2e.ts"
  }
}
```


Các script này cấu hình Vitest để chạy các kiểm tra, watch các thay đổi, tạo báo cáo bao phủ mã, và gỡ lỗi. Script test:e2e cụ thể để chạy các kiểm tra E2E với một file cấu hình tùy chỉnh.

Với thiết lập này, bây giờ bạn có thể tận hưởng lợi ích của việc sử dụng Vitest trong dự án NestJS của bạn, bao gồm thực thi kiểm tra nhanh hơn và trải nghiệm kiểm tra hiện đại hơn.

> info **Gợi ý** Bạn có thể kiểm tra một ví dụ hoạt động trong [repository](https://github.com/TrilonIO/nest-vitest) này