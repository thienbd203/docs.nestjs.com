### Extensions

> warning **Cảnh báo** Chương này chỉ áp dụng cho phương pháp code first.

Extensions là một **tính năng cấp thấp, nâng cao** cho phép bạn định nghĩa dữ liệu tùy ý trong cấu hình loại. Đính kèm siêu dữ liệu tùy chỉnh vào các trường nhất định cho phép bạn tạo các giải pháp tổng quát, tinh vi hơn. Ví dụ, với extensions, bạn có thể định nghĩa các vai trò cấp trường cần thiết để truy cập các trường cụ thể. Các vai trò như vậy có thể được phản ánh tại thời gian chạy để xác định liệu người gọi có đủ quyền để truy xuất một trường cụ thể hay không.

#### Thêm siêu dữ liệu tùy chỉnh

Để đính kèm siêu dữ liệu tùy chỉnh cho một trường, sử dụng decorator `@Extensions()` được xuất từ gói `@nestjs/graphql`.

```typescript
@Field()
@Extensions({ role: Role.ADMIN })
password: string;
```

Trong ví dụ trên, chúng ta đã gán thuộc tính siêu dữ liệu `role` giá trị của `Role.ADMIN`. `Role` là một enum TypeScript đơn giản nhóm tất cả các vai trò người dùng có sẵn trong hệ thống của chúng ta.

Lưu ý, ngoài việc đặt siêu dữ liệu trên các trường, bạn có thể sử dụng decorator `@Extensions()` ở cấp lớp và cấp phương thức (ví dụ, trên bộ xử lý query).

#### Sử dụng siêu dữ liệu tùy chỉnh

Logic tận dụng siêu dữ liệu tùy chỉnh có thể phức tạp tùy theo nhu cầu. Ví dụ, bạn có thể tạo một interceptor đơn giản lưu trữ/ghi lại các sự kiện cho mỗi lời gọi phương thức, hoặc một [field middleware](/graphql/field-middleware) khớp các vai trò cần thiết để truy xuất một trường với quyền của người gọi (hệ thống quyền cấp trường).

Để minh họa, hãy định nghĩa một `checkRoleMiddleware` so sánh vai trò của người dùng (hardcoded ở đây) với một vai trò cần thiết để truy cập một trường mục tiêu:

```typescript
export const checkRoleMiddleware: FieldMiddleware = async (
  ctx: MiddlewareContext,
  next: NextFn,
) => {
  const { info } = ctx;
  const { extensions } = info.parentType.getFields()[info.fieldName];

  /**
   * Trong một ứng dụng thực tế, biến "userRole"
   * nên đại diện cho vai trò của người gọi (người dùng) (ví dụ, "ctx.user.role").
   */
  const userRole = Role.USER;
  if (userRole === extensions.role) {
    // hoặc chỉ "return null" để bỏ qua
    throw new ForbiddenException(
      `User does not have sufficient permissions to access "${info.fieldName}" field.`,
    );
  }
  return next();
};
```

Với điều này đã có sẵn, chúng ta có thể đăng ký một middleware cho trường `password`, như sau:

```typescript
@Field({ middleware: [checkRoleMiddleware] })
@Extensions({ role: Role.ADMIN })
password: string;
```