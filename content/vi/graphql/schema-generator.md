### Tạo SDL

> warning **Cảnh báo** Chương này chỉ áp dụng cho phương pháp code first.

Để tạo thủ công một schema SDL GraphQL (tức là, không chạy ứng dụng, kết nối với cơ sở dữ liệu, hook up resolvers, v.v.), sử dụng `GraphQLSchemaBuilderModule`.

```typescript
async function generateSchema() {
  const app = await NestFactory.create(GraphQLSchemaBuilderModule);
  await app.init();

  const gqlSchemaFactory = app.get(GraphQLSchemaFactory);
  const schema = await gqlSchemaFactory.create([RecipesResolver]);
  console.log(printSchema(schema));
}
```

> info **Gợi ý** `GraphQLSchemaBuilderModule` và `GraphQLSchemaFactory` được nhập từ gói `@nestjs/graphql`. Hàm `printSchema` được nhập từ gói `graphql`.

#### Sử dụng

Phương thức `gqlSchemaFactory.create()` nhận một mảng các tham chiếu lớp resolver. Ví dụ:

```typescript
const schema = await gqlSchemaFactory.create([
  RecipesResolver,
  AuthorsResolver,
  PostsResolver,
]);
```

Nó cũng nhận một đối số thứ hai tùy chọn với một mảng các lớp scalar:

```typescript
const schema = await gqlSchemaFactory.create(
  [RecipesResolver, AuthorsResolver, PostsResolver],
  [DurationScalar, DateScalar],
);
```

Cuối cùng, bạn có thể chuyển một đối tượng options:

```typescript
const schema = await gqlSchemaFactory.create([RecipesResolver], {
  skipCheck: true,
  orphanedTypes: [],
});
```

- `skipCheck`: bỏ qua xác thực schema; boolean, mặc định là `false`
- `orphanedTypes`: danh sách các lớp không được tham chiếu rõ ràng (không phải là một phần của đồ thị đối tượng) để được tạo. Thông thường, nếu một lớp được khai báo nhưng không được tham chiếu trong đồ thị, nó bị bỏ qua. Giá trị thuộc tính là một mảng các tham chiếu lớp.