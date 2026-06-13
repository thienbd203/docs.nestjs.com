### CLI Plugin

Hệ thống reflection metadata của [TypeScript](https://www.typescriptlang.org/docs/handbook/decorators.html) có một số hạn chế khiến việc xác định các thuộc tính của một class hoặc nhận biết xem một thuộc tính cụ thể là optional hay required trở nên bất khả thi. Tuy nhiên, một số giới hạn này có thể được giải quyết tại thời điểm biên dịch. Nest cung cấp một plugin nâng cao quy trình biên dịch TypeScript để giảm lượng mã boilerplate cần thiết.

> info **Gợi ý** Plugin này là **tùy chọn**. Nếu bạn muốn, bạn có thể khai báo tất cả decorators thủ công, hoặc chỉ các decorators cụ thể ở nơi bạn cần.

#### Tổng quan

Swagger plugin sẽ tự động:

- annotate tất cả các thuộc tính DTO với `@ApiProperty` trừ khi `@ApiHideProperty` được sử dụng
- đặt thuộc tính `required` tùy thuộc vào dấu hỏi (ví dụ: `name?: string` sẽ đặt `required: false`)
- đặt thuộc tính `type` hoặc `enum` tùy thuộc vào kiểu (hỗ trợ mảng cũng như)
- đặt thuộc tính `default` dựa trên giá trị mặc định được gán
- đặt một số quy tắc validation dựa trên decorators `class-validator` (nếu `classValidatorShim` được đặt thành `true`)
- thêm một response decorator đến mỗi endpoint với status và `type` thích hợp (response model)
- tạo descriptions cho các thuộc tính và endpoints dựa trên comments (nếu `introspectComments` được đặt thành `true`)
- tạo example values cho các thuộc tính dựa trên comments (nếu `introspectComments` được đặt thành `true`)

Vui lòng lưu ý rằng tên file của bạn **phải có** một trong các hậu tố sau: `['.dto.ts', '.entity.ts']` (ví dụ, `create-user.dto.ts`) để được phân tích bởi plugin.

Nếu bạn đang sử dụng một hậu tố khác, bạn có thể điều chỉnh hành vi của plugin bằng cách chỉ định tùy chọn `dtoFileNameSuffix` (xem dưới đây).

Trước đây, nếu bạn muốn cung cấp trải nghiệm tương tác với Swagger UI, bạn phải nhân đôi rất nhiều mã để cho package biết models/components của bạn nên được khai báo trong specification như thế nào. Ví dụ, bạn có thể định nghĩa một class `CreateUserDto` đơn giản như sau:

```typescript
export class CreateUserDto {
  @ApiProperty()
  email: string;

  @ApiProperty()
  password: string;

  @ApiProperty({ enum: RoleEnum, default: [], isArray: true })
  roles: RoleEnum[] = [];

  @ApiProperty({ required: false, default: true })
  isEnabled?: boolean = true;
}
```

Mặc dù không phải là vấn đề lớn với các dự án vừa và nhỏ, nó trở nên dài dòng và khó bảo trì khi bạn có một tập lớn các class.

Bằng cách [kích hoạt Swagger plugin](/openapi/cli-plugin#using-the-cli-plugin), định nghĩa class ở trên có thể được khai báo đơn giản:

```typescript
export class CreateUserDto {
  email: string;
  password: string;
  roles: RoleEnum[] = [];
  isEnabled?: boolean = true;
}
```

> info **Lưu ý** Swagger plugin sẽ suy ra các annotation @ApiProperty() từ các kiểu TypeScript và decorators class-validator. Điều này giúp mô tả rõ ràng API của bạn cho tài liệu Swagger UI được tạo. Tuy nhiên, validation tại runtime vẫn sẽ được xử lý bởi các decorators class-validator. Vì vậy, cần tiếp tục sử dụng các validators như `IsEmail()`, `IsNumber()`, v.v.

Do đó, nếu bạn dựa vào các annotation tự động để tạo tài liệu và vẫn muốn có validations tại runtime, thì các decorators class-validator vẫn cần thiết.

> info **Gợi ý** Khi sử dụng [mapped types utilities](https://docs.nestjs.com/openapi/mapped-types) (như `PartialType`) trong DTOs, hãy import chúng từ `@nestjs/swagger` thay vì `@nestjs/mapped-types` để plugin có thể lấy schema.

Plugin thêm các decorators thích hợp trên bay dựa trên **Abstract Syntax Tree**. Do đó bạn sẽ không phải vật lộn với các decorators `@ApiProperty` rải rác khắp mã.

> info **Gợi ý** Plugin sẽ tự động tạo bất kỳ swagger properties nào bị thiếu, nhưng nếu bạn cần ghi đè chúng, bạn chỉ cần đặt chúng một cách rõ ràng thông qua `@ApiProperty()`.

#### Comments introspection

Với tính năng comments introspection được kích hoạt, CLI plugin sẽ tạo descriptions và example values cho các thuộc tính dựa trên comments.

Ví dụ, given một thuộc tính `roles` ví dụ:

```typescript
/**
 * A list of user's roles
 * @example ['admin']
 */
@ApiProperty({
  description: `A list of user's roles`,
  example: ['admin'],
})
roles: RoleEnum[] = [];
```

Bạn phải nhân đôi cả description và example values. Với `introspectComments` được kích hoạt, CLI plugin có thể trích xuất các comments này và tự động cung cấp descriptions (và examples, nếu được định nghĩa) cho các thuộc tính. Bây giờ, thuộc tính ở trên có thể được khai báo đơn giản như sau:

```typescript
/**
 * A list of user's roles
 * @example ['admin']
 */
roles: RoleEnum[] = [];
```

Có các tùy chọn plugin `dtoKeyOfComment` và `controllerKeyOfComment` có sẵn để tùy chỉnh cách plugin gán giá trị cho các decorators `ApiProperty` và `ApiOperation`, tương ứng. Xem ví dụ dưới đây:

```typescript
export class SomeController {
  /**
   * Create some resource
   */
  @Post()
  create() {}
}
```

Điều này tương đương với lệnh sau:

```typescript
@ApiOperation({ summary: "Create some resource" })
```

> info **Gợi ý** Đối với models, cùng logic áp dụng nhưng được sử dụng với decorator `ApiProperty` thay vào đó.

Đối với controllers, bạn có thể cung cấp không chỉ summary mà còn description (remarks), tags (như `@deprecated`), và response examples, như sau:

```ts
/**
 * Create a new cat
 *
 * @remarks This operation allows you to create a new cat.
 *
 * @deprecated
 * @throws {500} Something went wrong.
 * @throws {400} Bad Request.
 */
@Post()
async create(): Promise<Cat> {}
```

#### Sử dụng CLI plugin

Để kích hoạt plugin, mở `nest-cli.json` (nếu bạn sử dụng [Nest CLI](/cli/overview)) và thêm cấu hình `plugins` sau:

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/swagger"]
  }
}
```

Bạn có thể sử dụng thuộc tính `options` để tùy chỉnh hành vi của plugin.

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": [
      {
        "name": "@nestjs/swagger",
        "options": {
          "classValidatorShim": false,
          "introspectComments": true,
          "skipAutoHttpCode": true
        }
      }
    ]
  }
}
```

Thuộc tính `options` phải thoả mãn interface sau:

```typescript
export interface PluginOptions {
  dtoFileNameSuffix?: string[];
  controllerFileNameSuffix?: string[];
  classValidatorShim?: boolean;
  dtoKeyOfComment?: string;
  controllerKeyOfComment?: string;
  introspectComments?: boolean;
  skipAutoHttpCode?: boolean;
  esmCompatible?: boolean;
}
```

<table>
  <tr>
    <th>Tùy chọn</th>
    <th>Mặc định</th>
    <th>Mô tả</th>
  </tr>
  <tr>
    <td><code>dtoFileNameSuffix</code></td>
    <td><code>['.dto.ts', '.entity.ts']</code></td>
    <td>Hậu tố file DTO (Data Transfer Object)</td>
  </tr>
  <tr>
    <td><code>controllerFileNameSuffix</code></td>
    <td><code>.controller.ts</code></td>
    <td>Hậu tố file Controller</td>
  </tr>
  <tr>
    <td><code>classValidatorShim</code></td>
    <td><code>true</code></td>
    <td>Nếu được đặt thành true, module sẽ tái sử dụng các decorators validation <code>class-validator</code> (ví dụ: <code>@Max(10)</code> sẽ thêm <code>max: 10</code> vào định nghĩa schema)</td>
  </tr>
  <tr>
    <td><code>dtoKeyOfComment</code></td>
    <td><code>'description'</code></td>
    <td>Khóa thuộc tính để đặt văn bản comment vào trên <code>ApiProperty</code>.</td>
  </tr>
  <tr>
    <td><code>controllerKeyOfComment</code></td>
    <td><code>'summary'</code></td>
    <td>Khóa thuộc tính để đặt văn bản comment vào trên <code>ApiOperation</code>.</td>
  </tr>
  <tr>
    <td><code>introspectComments</code></td>
    <td><code>false</code></td>
    <td>Nếu được đặt thành true, plugin sẽ tạo descriptions và example values cho các thuộc tính dựa trên comments</td>
  </tr>
  <tr>
    <td><code>skipAutoHttpCode</code></td>
    <td><code>false</code></td>
    <td>Vô hiệu hóa việc tự động thêm <code>@HttpCode()</code> trong controllers</td>
  </tr>
  <tr>
    <td><code>esmCompatible</code></td>
    <td><code>false</code></td>
    <td>Nếu được đặt thành true, giải quyết các lỗi syntax gặp phải khi sử dụng ESM (<code>&#123; "type": "module" &#125;</code>).</td>
  </tr>
</table>

Đảm bảo xóa thư mục `/dist` và rebuild ứng dụng của bạn bất cứ khi nào các tùy chọn plugin được cập nhật.
Nếu bạn không sử dụng CLI mà thay vào đó có cấu hình `webpack` tùy chỉnh, bạn có thể sử dụng plugin này kết hợp với `ts-loader`:

```javascript
getCustomTransformers: (program: any) => ({
  before: [require('@nestjs/swagger/plugin').before({}, program)]
}),
```

#### SWC builder

Đối với các thiết lập tiêu chuẩn (non-monorepo), để sử dụng CLI Plugins với SWC builder, bạn cần kích hoạt type checking, như được mô tả [ở đây](/recipes/swc#type-checking).

```bash
$ nest start -b swc --type-check
```

Đối với các thiết lập monorepo, hãy làm theo hướng dẫn [ở đây](/recipes/swc#monorepo-and-cli-plugins).

```bash
$ npx ts-node src/generate-metadata.ts
# OR npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

Bây giờ, file metadata được serialize phải được tải bởi phương thức `SwaggerModule#loadPluginMetadata`, như được hiển thị dưới đây:

```typescript
import metadata from './metadata'; // <-- file tự động tạo bởi "PluginMetadataGenerator"

await SwaggerModule.loadPluginMetadata(metadata); // <-- ở đây
const document = SwaggerModule.createDocument(app, config);
```

#### Tích hợp với `ts-jest` (e2e tests)

Để chạy e2e tests, `ts-jest` biên dịch các file mã nguồn của bạn trên bay, trong bộ nhớ. Điều này có nghĩa là, nó không sử dụng trình biên dịch Nest CLI và không áp dụng bất kỳ plugin nào hoặc thực hiện các biến đổi AST.

Để kích hoạt plugin, tạo file sau trong thư mục e2e tests của bạn:

```javascript
const transformer = require('@nestjs/swagger/plugin');

module.exports.name = 'nestjs-swagger-transformer';
// bạn nên thay đổi số phiên bản bất cứ khi nào bạn thay đổi cấu hình dưới đây - nếu không, jest sẽ không phát hiện thay đổi
module.exports.version = 1;

module.exports.factory = (cs) => {
  return transformer.before(
    {
      // tùy chọn @nestjs/swagger/plugin (có thể để trống)
    },
    cs.program, // "cs.tsCompiler.program" cho các phiên bản cũ hơn của Jest (<= v27)
  );
};
```

Với điều này đã sẵn sàng, import AST transformer trong file cấu hình `jest` của bạn. Theo mặc định (trong ứng dụng khởi động), file cấu hình e2e tests nằm dưới thư mục `test` và được đặt tên là `jest-e2e.json`.

Nếu bạn sử dụng `jest@<29`, thì sử dụng đoạn mã dưới đây.

```json
{
  ... // cấu hình khác
  "globals": {
    "ts-jest": {
      "astTransformers": {
        "before": ["<path to the file created above>"]
      }
    }
  }
}
```

Nếu bạn sử dụng `jest@^29`, thì sử dụng đoạn mã dưới đây, vì cách tiếp cận trước đó đã bị deprecated.

```json
{
  ... // cấu hình khác
  "transform": {
    "^.+\\.(t|j)s$": [
      "ts-jest",
      {
        "astTransformers": {
          "before": ["<path to the file created above>"]
        }
      }
    ]
  }
}
```

#### Xử lý sự cố `jest` (e2e tests)

Trong trường hợp `jest` không dường như nhận các thay đổi cấu hình của bạn, có thể là Jest đã **cache** kết quả build. Để áp dụng cấu hình mới, bạn cần xóa thư mục cache của Jest.

Để xóa thư mục cache, chạy lệnh sau trong thư mục dự án NestJS của bạn:

```bash
$ npx jest --clearCache
```

Trong trường hợp xóa cache tự động thất bại, bạn vẫn có thể thủ công xóa thư mục cache với các lệnh sau:

```bash
# Tìm thư mục cache của jest (thường là /tmp/jest_rs)
# bằng cách chạy lệnh sau trong thư mục gốc dự án NestJS của bạn
$ npx jest --showConfig | grep cache
# ví dụ kết quả:
#   "cache": true,
#   "cacheDirectory": "/tmp/jest_rs"

# Xóa hoặc làm trống thư mục cache của Jest
$ rm -rf  <cacheDirectory value>
# ví dụ:
# rm -rf /tmp/jest_rs
```