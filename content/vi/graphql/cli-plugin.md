### CLI Plugin

> warning **Cảnh báo** Chương này chỉ áp dụng cho phương pháp code first.

Hệ thống phản ánh siêu dữ liệu của TypeScript có một số hạn chế khiến nó không thể, ví dụ, xác định các thuộc tính mà một lớp bao gồm hoặc nhận biết liệu một thuộc tính nhất định là tùy chọn hay bắt buộc. Tuy nhiên, một số ràng buộc này có thể được giải quyết tại thời gian biên dịch. Nest cung cấp một plugin nâng cao quy trình biên dịch TypeScript để giảm lượng code boilerplate cần thiết.

> info **Gợi ý** Plugin này là **opt-in**. Nếu bạn thích, bạn có thể khai báo tất cả các decorator thủ công, hoặc chỉ các decorator cụ thể nơi bạn cần chúng.

#### Tổng quan

Plugin GraphQL sẽ tự động:

- chú thích tất cả các thuộc tính lớp đối tượng đầu vào, đối tượng và args với `@Field` trừ khi `@HideField` được sử dụng
- đặt thuộc tính `nullable` tùy thuộc vào dấu hỏi (ví dụ `name?: string` sẽ đặt `nullable: true`)
- đặt thuộc tính `type` tùy thuộc vào loại (hỗ trợ mảng cũng)
- tạo mô tả cho các thuộc tính dựa trên bình luận (nếu `introspectComments` được đặt thành `true`)

Vui lòng lưu ý rằng tên tệp của bạn **phải có** một trong các hậu tố sau để được phân tích bởi plugin: `['.input.ts', '.args.ts', '.entity.ts', '.model.ts']` (ví dụ, `author.entity.ts`). Nếu bạn đang sử dụng một hậu tố khác, bạn có thể điều chỉnh hành vi của plugin bằng cách chỉ định tùy chọn `typeFileNameSuffix` (xem dưới đây).

Với những gì chúng ta đã học cho đến nay, bạn phải nhân đôi rất nhiều code để cho gói biết loại của bạn nên được khai báo trong GraphQL như thế nào. Ví dụ, bạn có thể định nghĩa một lớp `Author` đơn giản như sau:

```typescript
@@filename(authors/models/author.model)
@ObjectType()
export class Author {
  @Field(type => ID)
  id: number;

  @Field({ nullable: true })
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  @Field(type => [Post])
  posts: Post[];
}
```

Mặc dù không phải là một vấn đề đáng kể với các dự án kích thước trung bình, nó trở nên dài dòng và khó duy trì khi bạn có một tập hợp lớn các lớp.

Bằng cách kích hoạt plugin GraphQL, định nghĩa lớp trên có thể được khai báo đơn giản:

```typescript
@@filename(authors/models/author.model)
@ObjectType()
export class Author {
  @Field(type => ID)
  id: number;
  firstName?: string;
  lastName?: string;
  posts: Post[];
}
```

Plugin thêm các decorator phù hợp ngay lập tức dựa trên **Cây Cú Pháp Trừu Tượng**. Do đó, bạn sẽ không phải đấu tranh với các decorator `@Field` phân tán trong toàn bộ code.

> info **Gợi ý** Plugin sẽ tự động tạo bất kỳ thuộc tính GraphQL nào bị thiếu, nhưng nếu bạn cần ghi đè chúng, chỉ cần đặt chúng rõ ràng qua `@Field()`.

#### Nội quan bình luận

Với tính năng nội quan bình luận được kích hoạt, plugin CLI sẽ tạo mô tả cho các trường dựa trên bình luận.

Ví dụ, cho một thuộc tính `roles` mẫu:

```typescript
/**
 * A list of user's roles
 */
@Field(() => [String], {
  description: `A list of user's roles`
})
roles: string[];
```

Bạn phải nhân đôi các giá trị mô tả. Với `introspectComments` được kích hoạt, plugin CLI có thể trích xuất các bình luận này và tự động cung cấp mô tả cho các thuộc tính. Bây giờ, trường trên có thể được khai báo đơn giản như sau:

```typescript
/**
 * A list of user's roles
 */
roles: string[];
```

#### Sử dụng plugin CLI

Để kích hoạt plugin, mở `nest-cli.json` (nếu bạn sử dụng [Nest CLI](/cli/overview)) và thêm cấu hình `plugins` sau:

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/graphql"]
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
        "name": "@nestjs/graphql",
        "options": {
          "typeFileNameSuffix": [".input.ts", ".args.ts"],
          "introspectComments": true
        }
      }
    ]
  }

}
```

Thuộc tính `options` phải đáp ứng interface sau:

```typescript
export interface PluginOptions {
  typeFileNameSuffix?: string[];
  introspectComments?: boolean;
}
```

<table>
  <tr>
    <th>Tùy chọn</th>
    <th>Mặc định</th>
    <th>Mô tả</th>
  </tr>
  <tr>
    <td><code>typeFileNameSuffix</code></td>
    <td><code>['.input.ts', '.args.ts', '.entity.ts', '.model.ts']</code></td>
    <td>Hậu tố tệp loại GraphQL</td>
  </tr>
  <tr>
    <td><code>introspectComments</code></td>
      <td><code>false</code></td>
      <td>Nếu được đặt thành true, plugin sẽ tạo mô tả cho các thuộc tính dựa trên bình luận</td>
  </tr>
</table>

Nếu bạn không sử dụng CLI nhưng thay vào đó có cấu hình `webpack` tùy chỉnh, bạn có thể sử dụng plugin này kết hợp với `ts-loader`:

```javascript
getCustomTransformers: (program: any) => ({
  before: [require('@nestjs/graphql/plugin').before({}, program)]
}),
```

#### SWC builder

Đối với các thiết lập tiêu chuẩn (non-monorepo), để sử dụng CLI Plugins với builder SWC, bạn cần kích hoạt kiểm tra loại, như được mô tả [ở đây](/recipes/swc#type-checking).

```bash
$ nest start -b swc --type-check
```

Đối với các thiết lập monorepo, hãy làm theo hướng dẫn [ở đây](/recipes/swc#monorepo-and-cli-plugins).

```bash
$ npx ts-node src/generate-metadata.ts
# HOẶC npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

Bây giờ, tệp siêu dữ liệu được serialize phải được tải bởi phương thức `GraphQLModule`, như được hiển thị dưới đây:

```typescript
import metadata from './metadata'; // <-- tệp được tạo tự động bởi "PluginMetadataGenerator"

GraphQLModule.forRoot<...>({
  ..., // các tùy chọn khác
  metadata,
}),
```

#### Tích hợp với `ts-jest` (kiểm tra e2e)

Khi chạy kiểm tra e2e với plugin này được kích hoạt, bạn có thể gặp vấn đề với biên dịch schema. Ví dụ, một trong những lỗi phổ biến nhất là:

```json
Object type <name> must define one or more fields.
```

Điều này xảy ra vì cấu hình `jest` không import plugin `@nestjs/graphql/plugin` ở bất cứ đâu.

Để khắc phục điều này, tạo tệp sau trong thư mục kiểm tra e2e của bạn:

```javascript
const transformer = require('@nestjs/graphql/plugin');

module.exports.name = 'nestjs-graphql-transformer';
// bạn nên thay đổi số phiên bản bất cứ khi nào bạn thay đổi cấu hình dưới đây - nếu không, jest sẽ không phát hiện thay đổi
module.exports.version = 1;

module.exports.factory = (cs) => {
  return transformer.before(
    {
      // tùy chọn @nestjs/graphql/plugin (có thể trống)
    },
    cs.program, // "cs.tsCompiler.program" cho các phiên bản cũ hơn của Jest (<= v27)
  );
};
```

Với điều này đã có sẵn, import transformer AST trong tệp cấu hình `jest` của bạn. Theo mặc định (trong ứng dụng khởi động), tệp cấu hình kiểm tra e2e nằm dưới thư mục `test` và được đặt tên `jest-e2e.json`.

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

Nếu bạn sử dụng `jest@^29`, sau đó sử dụng đoạn mã dưới đây, vì cách tiếp cận trước đã bị deprecated.

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