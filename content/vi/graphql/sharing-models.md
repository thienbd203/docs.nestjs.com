### Chia sẻ mô hình

> warning **Cảnh báo** Chương này chỉ áp dụng cho phương pháp code first.

Một trong những lợi thế lớn nhất của việc sử dụng Typescript cho backend của dự án của bạn là khả năng tái sử dụng cùng các mô hình trong một ứng dụng frontend dựa trên Typescript, bằng cách sử dụng một gói Typescript chung.

Nhưng có một vấn đề: các mô hình được tạo sử dụng phương pháp code first được trang trí nặng nề với các decorator liên quan đến GraphQL. Những decorator đó không liên quan trong frontend, tác động tiêu cực đến hiệu suất.

#### Sử dụng model shim

Để giải quyết vấn đề này, NestJS cung cấp một "shim" cho phép bạn thay thế các decorator gốc bằng code không hoạt động bằng cách sử dụng cấu hình `webpack` (hoặc tương tự).
Để sử dụng shim này, cấu hình một alias giữa gói `@nestjs/graphql` và shim.

Ví dụ, đối với webpack điều này được giải quyết theo cách này:

```typescript
resolve: { // see: https://webpack.js.org/configuration/resolve/
  alias: {
      "@nestjs/graphql": path.resolve(__dirname, "../node_modules/@nestjs/graphql/dist/extra/graphql-model-shim")
  }
}
```

> info **Gợi ý** Gói [TypeORM](/techniques/database) có một shim tương tự có thể được tìm thấy [ở đây](https://github.com/typeorm/typeorm/blob/master/extra/typeorm-model-shim.js).