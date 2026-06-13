### Field middleware

> warning **Cảnh báo** Chương này chỉ áp dụng cho phương pháp code first.

Field Middleware cho phép bạn chạy code tùy ý **trước hoặc sau** khi một trường được giải quyết. Một field middleware có thể được sử dụng để chuyển đổi kết quả của một trường, xác thực các đối số của một trường, hoặc thậm chí kiểm tra các vai trò cấp trường (ví dụ, cần thiết để truy cập một trường mục tiêu mà một hàm middleware được thực thi).

Bạn có thể kết nối nhiều hàm middleware vào một trường. Trong trường hợp này, chúng sẽ được gọi tuần tự dọc theo chuỗi nơi middleware trước đó quyết định gọi cái tiếp theo. Thứ tự của các hàm middleware trong mảng `middleware` là quan trọng. Resolver đầu tiên là lớp "ngoài cùng", vì vậy nó được thực thi đầu tiên và cuối cùng (tương tự đến gói `graphql-middleware`). Resolver thứ hai là lớp "ngoài cùng thứ hai", vì vậy nó được thực thi thứ hai và thứ hai từ cuối cùng.

#### Bắt đầu

Hãy bắt đầu bằng cách tạo một middleware đơn giản sẽ ghi lại giá trị trường trước khi nó được gửi lại cho client:

```typescript
import { FieldMiddleware, MiddlewareContext, NextFn } from '@nestjs/graphql';

const loggerMiddleware: FieldMiddleware = async (
  ctx: MiddlewareContext,
  next: NextFn,
) => {
  const value = await next();
  console.log(value);
  return value;
};
```

> info **Gợi ý** `MiddlewareContext` là một đối tượng bao gồm các đối số giống nhau thường được nhận bởi hàm resolver GraphQL (`{{ '{' }} source, args, context, info {{ '}' }}`), trong khi `NextFn` là một hàm cho phép bạn thực thi middleware tiếp theo trong stack (liên kết với trường này) hoặc resolver trường thực tế.

> warning **Cảnh báo** Các hàm field middleware không thể inject dependencies hay truy cập container DI của Nest vì chúng được thiết kế để rất nhẹ và không nên thực hiện bất kỳ hoạt động có thể tốn thời gian nào (như lấy dữ liệu từ cơ sở dữ liệu). Nếu bạn cần gọi các dịch vụ bên ngoài/truy vấn dữ liệu từ nguồn dữ liệu, bạn nên làm điều đó trong một guard/interceptor liên kết với bộ xử lý query/mutation gốc và gán nó cho đối tượng `context` mà bạn có thể truy cập từ trong field middleware (cụ thể, từ đối tượng `MiddlewareContext`).

Lưu ý rằng field middleware phải khớp với interface `FieldMiddleware`. Trong ví dụ trên, chúng ta đầu tiên chạy hàm `next()` (thực thi resolver trường thực tế và trả về một giá trị trường) và sau đó, chúng ta ghi lại giá trị này vào terminal của chúng ta. Ngoài ra, giá trị được trả về từ hàm middleware hoàn toàn ghi đè giá trị trước đó và vì chúng ta không muốn thực hiện bất kỳ thay đổi nào, chúng ta chỉ đơn giản trả về giá trị gốc.

Với điều này đã có sẵn, chúng ta có thể đăng ký middleware của mình trực tiếp trong decorator `@Field()`, như sau:

```typescript
@ObjectType()
export class Recipe {
  @Field({ middleware: [loggerMiddleware] })
  title: string;
}
```

Bây giờ bất cứ khi nào chúng ta yêu cầu trường `title` của loại đối tượng `Recipe`, giá trị trường gốc sẽ được ghi lại vào console.

> info **Gợi ý** Để tìm hiểu cách bạn có thể triển khai một hệ thống quyền cấp trường với việc sử dụng tính năng [extensions](/graphql/extensions), hãy kiểm tra phần [này](/graphql/extensions#using-custom-metadata).

> warning **Cảnh báo** Field middleware chỉ có thể được áp dụng cho các lớp `ObjectType`. Để biết thêm chi tiết, hãy kiểm tra [vấn đề này](https://github.com/nestjs/graphql/issues/2446).

Ngoài ra, như đã đề cập ở trên, chúng ta có thể kiểm soát giá trị của trường từ trong hàm middleware. Để mục đích minh họa, hãy viết hoa tiêu đề của một công thức (nếu có):

```typescript
const value = await next();
return value?.toUpperCase();
```

Trong trường hợp này, mọi tiêu đề sẽ được tự động viết hoa, khi được yêu cầu.

Tương tự, bạn có thể liên kết một field middleware với một resolver trường tùy chỉnh (một phương thức được chú thích với decorator `@ResolveField()`), như sau:

```typescript
@ResolveField(() => String, { middleware: [loggerMiddleware] })
title() {
  return 'Placeholder';
}
```

> warning **Cảnh báo** Trong trường hợp enhancers được kích hoạt ở cấp resolver trường ([đọc thêm](/graphql/other-features#execute-enhancers-at-the-field-resolver-level)), các hàm field middleware sẽ chạy trước bất kỳ interceptors, guards, v.v., **liên kết với phương thức** (nhưng sau các enhancers cấp gốc được đăng ký cho các bộ xử lý query hoặc mutation).

#### Field middleware toàn cục

Ngoài việc liên kết một middleware trực tiếp với một trường cụ thể, bạn cũng có thể đăng ký một hoặc nhiều hàm middleware toàn cục. Trong trường hợp này, chúng sẽ được tự động kết nối với tất cả các trường của các loại đối tượng của bạn.

```typescript
GraphQLModule.forRoot({
  autoSchemaFile: 'schema.gql',
  buildSchemaOptions: {
    fieldMiddleware: [loggerMiddleware],
  },
}),
```

> info **Gợi ý** Các hàm field middleware được đăng ký toàn cục sẽ được thực thi **trước** các hàm được đăng ký cục bộ (những hàm liên kết trực tiếp với các trường cụ thể).