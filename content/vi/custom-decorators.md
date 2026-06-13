### Custom route decorators

Nest được xây dựng xung quanh một tính năng ngôn ngữ gọi là **decorators**. Decorators là một khái niệm nổi tiếng trong nhiều ngôn ngữ lập trình phổ biến, nhưng trong thế giới JavaScript, chúng vẫn còn tương đối mới. Để hiểu rõ hơn cách decorators hoạt động, chúng tôi khuyến nghị đọc bài viết này. Đây là một định nghĩa đơn giản:

<blockquote class="external">
  Một decorator ES2016 là một expression trả về một function và có thể nhận một target, name và property descriptor như các đối số.
  Bạn áp dụng nó bằng cách thêm tiền tố decorator với một ký tự <code>@</code> và đặt điều này ở trên cùng của những gì
  bạn đang cố gắng decorate. Decorators có thể được định nghĩa cho một class, một method hoặc một property.
</blockquote>

#### Param decorators

Nest cung cấp một tập hợp các **param decorators** hữu ích mà bạn có thể sử dụng cùng với các HTTP route handlers. Dưới đây là danh sách các decorators được cung cấp và các đối tượng Express thuần (hoặc Fastify) mà chúng đại diện

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td>
    </tr>
    <tr>
      <td><code>@Response(), @Res()</code></td>
      <td><code>res</code></td>
    </tr>
    <tr>
      <td><code>@Next()</code></td>
      <td><code>next</code></td>
    </tr>
    <tr>
      <td><code>@Session()</code></td>
      <td><code>req.session</code></td>
    </tr>
    <tr>
      <td><code>@Param(param?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[param]</code></td>
    </tr>
    <tr>
      <td><code>@Body(param?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[param]</code></td>
    </tr>
    <tr>
      <td><code>@Query(param?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[param]</code></td>
    </tr>
    <tr>
      <td><code>@Headers(param?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[param]</code></td>
    </tr>
    <tr>
      <td><code>@Ip()</code></td>
      <td><code>req.ip</code></td>
    </tr>
    <tr>
      <td><code>@HostParam()</code></td>
      <td><code>req.hosts</code></td>
    </tr>
  </tbody>
</table>

Ngoài ra, bạn có thể tạo **custom decorators** của riêng bạn. Tại sao điều này hữu ích?

Trong thế giới node.js, thực hành phổ biến là attach các thuộc tính đến đối tượng **request**. Sau đó bạn manually extract chúng trong mỗi route handler, sử dụng code như sau:

```typescript
const user = req.user;
```

Để làm cho code của bạn dễ đọc và transparent hơn, bạn có thể tạo một decorator `@User()` và tái sử dụng nó trên tất cả các controllers của bạn.

```typescript
@@filename(user.decorator)
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

Sau đó, bạn có thể đơn giản sử dụng nó ở bất cứ nơi nào phù hợp với yêu cầu của bạn.

```typescript
@@filename()
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}
@@switch
@Get()
@Bind(User())
async findOne(user) {
  console.log(user);
}
```

#### Passing data

Khi hành vi của decorator của bạn phụ thuộc vào một số điều kiện, bạn có thể sử dụng parameter `data` để truyền một đối số đến factory function của decorator. Một trường hợp sử dụng cho điều này là một custom decorator extract các thuộc tính từ đối tượng request theo key. Hãy giả sử rằng authentication layer của chúng ta validates các yêu cầu và attach một user entity đến đối tượng request. User entity cho một yêu cầu được authenticated có thể trông như:

```json
{
  "id": 101,
  "firstName": "Alan",
  "lastName": "Turing",
  "email": "alan@email.com",
  "roles": ["admin"]
}
```

Hãy định nghĩa một decorator nhận một property name như key, và trả về giá trị liên quan nếu nó tồn tại (hoặc undefined nếu nó không tồn tại, hoặc nếu đối tượng `user` chưa được tạo).

```typescript
@@filename(user.decorator)
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
@@switch
import { createParamDecorator } from '@nestjs/common';

export const User = createParamDecorator((data, ctx) => {
  const request = ctx.switchToHttp().getRequest();
  const user = request.user;

  return data ? user && user[data] : user;
});
```

Đây là cách bạn có thể sau đó truy cập một thuộc tính cụ thể qua decorator `@User()` trong controller:

```typescript
@@filename()
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
@@switch
@Get()
@Bind(User('firstName'))
async findOne(firstName) {
  console.log(`Hello ${firstName}`);
}
```

Bạn có thể sử dụng cùng decorator này với các keys khác nhau để truy cập các thuộc tính khác nhau. Nếu đối tượng `user` deep hoặc phức tạp, điều này có thể làm cho việc implement request handler dễ dàng và dễ đọc hơn.

> info **Gợi ý** Đối với người dùng TypeScript, lưu ý rằng `createParamDecorator<T>()` là một generic. Điều này có nghĩa là bạn có thể enforce type safety một cách rõ ràng, ví dụ `createParamDecorator<string>((data, ctx) => ...)`. Ngoài ra, chỉ định một loại parameter trong factory function, ví dụ `createParamDecorator((data: string, ctx) => ...)`. Nếu bạn bỏ qua cả hai, loại cho `data` sẽ là `any`.

#### Working with pipes

Nest xử lý custom param decorators theo cùng fashion như các built-in ones (`@Body()`, `@Param()` và `@Query()`). Điều này có nghĩa là pipes được thực thi cho các custom annotated parameters cũng như (trong các ví dụ của chúng ta, đối số `user`). Hơn nữa, bạn có thể áp dụng pipe trực tiếp đến custom decorator:

```typescript
@@filename()
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {
  console.log(user);
}
@@switch
@Get()
@Bind(User(new ValidationPipe({ validateCustomDecorators: true })))
async findOne(user) {
  console.log(user);
}
```

> info **Gợi ý** Lưu ý rằng option `validateCustomDecorators` phải được đặt thành true. `ValidationPipe` không validate các đối số được annotated với custom decorators theo mặc định.

#### Decorator composition

Nest cung cấp một helper method để compose nhiều decorators. Ví dụ, giả sử bạn muốn kết hợp tất cả các decorators liên quan đến authentication thành một decorator đơn lẻ. Điều này có thể được thực hiện với cấu trúc sau:

```typescript
@@filename(auth.decorator)
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
@@switch
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

Sau đó bạn có thể sử dụng custom decorator `@Auth()` này như sau:

```typescript
@Get('users')
@Auth('admin')
findAllUsers() {}
```

Điều này có hiệu quả áp dụng tất cả bốn decorators với một khai báo đơn lẻ.

> warning **Cảnh báo** Decorator `@ApiHideProperty()` từ package `@nestjs/swagger` không thể composition và sẽ không hoạt động đúng với function `applyDecorators`.