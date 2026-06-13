### Guards

Guard là một class được annotate với decorator `@Injectable()`, implements interface `CanActivate`.

<figure><img class="illustrative-image" src="/assets/Guards_1.png" /></figure>

Guards có một **trách nhiệm đơn lẻ**. Chúng xác định xem một yêu cầu nhất định có được xử lý bởi route handler hay không, tùy thuộc vào một số điều kiện nhất định (như permissions, roles, ACLs, v.v.) có mặt tại run-time. Điều này thường được gọi là **authorization**. Authorization (và người anh em của nó, **authentication**, với nó thường hợp tác) thường được xử lý bởi middleware trong các ứng dụng Express truyền thống. Middleware là một lựa chọn tốt cho authentication, vì những thứ như token validation và attaching properties đến đối tượng `request` không kết nối mạnh với một route context cụ thể (và metadata của nó).

Nhưng middleware, theo bản chất của nó, là dumb. Nó không biết handler nào sẽ được thực thi sau khi gọi function `next()`. Mặt khác, **Guards** có quyền truy cập vào instance `ExecutionContext`, và do đó biết chính xác những gì sẽ được thực thi tiếp theo. Chúng được thiết kế, giống như exception filters, pipes, và interceptors, để cho phép bạn chèn logic xử lý tại đúng điểm trong chu kỳ request/response, và làm như vậy theo cách declarative. Điều này giúp giữ code của bạn DRY và declarative.

> info **Gợi ý** Guards được thực thi **sau** tất cả middleware, nhưng **trước** bất kỳ interceptor hoặc pipe nào.

#### Authorization guard

Như đã đề cập, **authorization** là một trường hợp sử dụng tuyệt vời cho Guards vì các route cụ thể chỉ nên có sẵn khi caller (thường là một authenticated user cụ thể) có đủ permissions. `AuthGuard` mà chúng ta sẽ xây dựng bây giờ giả định một authenticated user (và do đó, một token được attached đến request headers). Nó sẽ extract và validate token, và sử dụng thông tin được extract để xác định xem yêu cầu có thể tiếp tục hay không.

```typescript
@@filename(auth.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class AuthGuard {
  async canActivate(context) {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

> info **Gợi ý** Nếu bạn đang tìm kiếm một ví dụ thực tế về cách implement một authentication mechanism trong ứng dụng của bạn, hãy truy cập chương này. Tương tự, để có ví dụ authorization tinh vi hơn, hãy kiểm tra trang này.

Logic bên trong function `validateRequest()` có thể đơn giản hoặc tinh vi tùy theo nhu cầu. Điểm chính của ví dụ này là cho thấy guards phù hợp vào chu kỳ request/response như thế nào.

Mọi guard phải implement một function `canActivate()`. Function này nên trả về một boolean, chỉ định xem yêu cầu hiện tại có được phép hay không. Nó có thể trả về response either synchronously hoặc asynchronously (qua một `Promise` hoặc `Observable`). Nest sử dụng giá trị trả về để kiểm soát hành động tiếp theo:

- nếu nó trả về `true`, yêu cầu sẽ được xử lý.
- nếu nó trả về `false`, Nest sẽ từ chối yêu cầu.

<app-banner-enterprise></app-banner-enterprise>

#### Execution context

Function `canActivate()` nhận một đối số đơn lẻ, instance `ExecutionContext`. `ExecutionContext` kế thừa từ `ArgumentsHost`. Chúng ta đã thấy `ArgumentsHost` trước đó trong chương exception filters. Trong mẫu trên, chúng ta chỉ sử dụng cùng các helper methods được định nghĩa trên `ArgumentsHost` mà chúng ta đã sử dụng trước đó, để lấy một tham chiếu đến đối tượng `Request`. Bạn có thể tham khảo lại phần **Arguments host** của chương exception filters để biết thêm về chủ đề này.

Bằng cách mở rộng `ArgumentsHost`, `ExecutionContext` cũng thêm một số helper methods mới cung cấp chi tiết bổ sung về quy trình thực thi hiện tại. Các chi tiết này có thể hữu ích trong việc xây dựng các guards generic hơn có thể hoạt động trên một tập hợp rộng các controllers, methods, và execution contexts. Tìm hiểu thêm về `ExecutionContext` ở đây.

#### Role-based authentication

Hãy xây dựng một guard chức năng hơn chỉ cho phép truy cập chỉ đến người dùng với một role cụ thể. Chúng ta sẽ bắt đầu với một guard template cơ bản, và xây dựng trên nó trong các phần sắp tới. Hiện tại, nó cho phép tất cả các yêu cầu tiếp tục:

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class RolesGuard {
  canActivate(context) {
    return true;
  }
}
```

#### Binding guards

Giống như pipes và exception filters, guards có thể là **controller-scoped**, method-scoped, hoặc global-scoped. Dưới đây, chúng ta thiết lập một guard controller-scoped sử dụng decorator `@UseGuards()`. Decorator này có thể nhận một đối số đơn lẻ, hoặc một danh sách các đối số được phân tách bằng dấu phẩy. Điều này cho phép bạn dễ dàng áp dụng tập hợp guards phù hợp với một khai báo.

```typescript
@@filename()
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

> info **Gợi ý** Decorator `@UseGuards()` được import từ package `@nestjs/common`.

Ở trên, chúng ta truyền class `RolesGuard` (thay vì một instance), để lại trách nhiệm instantiation cho framework và cho phép dependency injection. Như với pipes và exception filters, chúng ta cũng có thể truyền một instance in-place:

```typescript
@@filename()
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

Cấu trúc trên attach guard đến mọi handler được khai báo bởi controller này. Nếu chúng ta muốn guard chỉ áp dụng cho một method đơn lẻ, chúng ta áp dụng decorator `@UseGuards()` ở **mức method**.

Để thiết lập một global guard, sử dụng method `useGlobalGuards()` của instance ứng dụng Nest:

```typescript
@@filename()
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

> warning **Lưu ý** Trong trường hợp hybrid apps method `useGlobalGuards()` không thiết lập guards cho gateways và microservices theo mặc định (xem Hybrid application để biết thông tin về cách thay đổi hành vi này). Đối với các ứng dụng microservice "standard" (non-hybrid), `useGlobalGuards()` mount guards global.

Global guards được sử dụng trên toàn bộ ứng dụng, cho mọi controller và mọi route handler. Về mặt dependency injection, global guards được đăng ký từ bên ngoài bất kỳ module nào (với `useGlobalGuards()` như trong ví dụ trên) không thể inject dependencies vì điều này được thực hiện bên ngoài context của bất kỳ module nào. Để giải quyết vấn đề này, bạn có thể thiết lập một guard trực tiếp từ bất kỳ module nào sử dụng cấu trúc sau:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

> info **Gợi ý** Khi sử dụng cách tiếp cận này để thực hiện dependency injection cho guard, lưu ý rằng bất kể module nơi cấu trúc này được sử dụng, guard thực tế là global. Nên làm điều này ở đâu? Chọn module nơi guard (`RolesGuard` trong ví dụ trên) được định nghĩa. Ngoài ra, `useClass` không phải là cách duy nhất để xử lý đăng ký custom provider. Tìm hiểu thêm ở đây.

#### Setting roles per handler

`RolesGuard` của chúng ta đang hoạt động, nhưng nó chưa rất thông minh. Chúng ta chưa tận dụng tính năng guard quan trọng nhất - execution context. Nó chưa biết về roles, hoặc roles nào được phép cho mỗi handler. `CatsController`, ví dụ, có thể có các permission schemes khác nhau cho các routes khác nhau. Một số có thể chỉ có sẵn cho admin user, và những khác có thể mở cho mọi người. Làm thế nào chúng ta có thể match roles đến routes theo cách linh hoạt và có thể tái sử dụng?

Đây là nơi **custom metadata** phát huy tác dụng (tìm hiểu thêm ở đây). Nest cung cấp khả năng attach custom **metadata** đến route handlers thông qua decorators được tạo qua static method `Reflector.createDecorator`, hoặc decorator built-in `@SetMetadata()`.

Ví dụ, hãy tạo một decorator `@Roles()` sử dụng method `Reflector.createDecorator` sẽ attach metadata đến handler. `Reflector` được cung cấp out-of-the-box bởi framework và exposed từ package `@nestjs/core`.

```ts
@@filename(roles.decorator)
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

Decorator `Roles` ở đây là một function nhận một đối số đơn lẻ của loại `string[]`.

Bây giờ, để sử dụng decorator này, chúng ta chỉ cần annotate handler với nó:

```typescript
@@filename(cats.controller)
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles(['admin'])
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

Ở đây chúng ta đã attach metadata decorator `Roles` đến method `create()`, chỉ định rằng chỉ người dùng với role `admin` nên được phép truy cập route này.

Ngoài ra, thay vì sử dụng method `Reflector.createDecorator`, chúng ta có thể sử dụng decorator built-in `@SetMetadata()`. Tìm hiểu thêm ở đây.

#### Putting it all together

Hãy quay lại và kết nối điều này với `RolesGuard` của chúng ta. Hiện tại, nó chỉ trả về `true` trong mọi trường hợp, cho phép mọi yêu cầu tiếp tục. Chúng ta muốn làm cho giá trị trả có điều kiện dựa trên việc so sánh **roles được gán cho user hiện tại** với các roles thực tế cần thiết bởi route hiện tại đang được xử lý. Để truy cập role(s) của route (custom metadata), chúng ta sẽ sử dụng helper class `Reflector` một lần nữa, như sau:

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
@Dependencies(Reflector)
export class RolesGuard {
  constructor(reflector) {
    this.reflector = reflector;
  }

  canActivate(context) {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

> info **Gợi ý** Trong thế giới node.js, thực hành phổ biến là attach user được authorized đến đối tượng `request`. Do đó, trong mẫu code của chúng ta ở trên, chúng ta giả định rằng `request.user` chứa instance user và roles được phép. Trong ứng dụng của bạn, bạn có thể sẽ tạo liên kết đó trong custom **authentication guard** (hoặc middleware) của bạn. Kiểm tra chương này để biết thêm thông tin về chủ đề này.

> warning **Cảnh báo** Logic bên trong function `matchRoles()` có thể đơn giản hoặc tinh vi tùy theo nhu cầu. Điểm chính của ví dụ này là cho thấy guards phù hợp vào chu kỳ request/response như thế nào.

Tham khảo phần Reflection and metadata của chương **Execution context** để biết thêm chi tiết về việc sử dụng `Reflector` theo cách context-sensitive.

Khi một user với các đặc quyền không đủ yêu cầu một endpoint, Nest tự động trả về response sau:

```typescript
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

Lưu ý rằng behind the scenes, khi một guard trả về `false`, framework throw một `ForbiddenException`. Nếu bạn muốn trả về một error response khác, bạn nên throw exception cụ thể của riêng bạn. Ví dụ:

```typescript
throw new UnauthorizedException();
```

Bất kỳ exception nào được throw bởi một guard sẽ được xử lý bởi exceptions layer (global exceptions filter và bất kỳ exceptions filters nào được áp dụng cho context hiện tại).

> info **Gợi ý** Nếu bạn đang tìm kiếm một ví dụ thực tế về cách implement authorization, hãy kiểm tra chương này.