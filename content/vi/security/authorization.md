### Authorization

**Authorization** đề cập đến quá trình xác định xem một user có thể làm gì. Ví dụ, một administrative user được phép tạo, chỉnh sửa, và xóa bài viết. Một non-administrative user chỉ được authorized để đọc các bài viết.

Authorization là orthogonal và độc lập từ authentication. Tuy nhiên, authorization yêu cầu một cơ chế authentication.

Có nhiều cách tiếp cận và chiến lược khác nhau để xử lý authorization. Cách tiếp cận được chọn cho bất kỳ dự án nào phụ thuộc vào các yêu cầu ứng dụng cụ thể của dự án đó. Chương này trình bày một vài cách tiếp cận để authorization có thể được điều chỉnh theo nhiều yêu cầu khác nhau.

#### Basic RBAC implementation

Role-based access control (**RBAC**) là một cơ chế access-control policy-neutral được định nghĩa xung quanh roles và privileges. Trong phần này, chúng ta sẽ chứng minh cách triển khai một cơ chế RBAC rất cơ bản bằng cách sử dụng Nest [guards](/guards).

Trước tiên, hãy tạo một enum `Role` đại diện cho các roles trong hệ thống:

```typescript
@@filename(role.enum)
export enum Role {
  User = 'user',
  Admin = 'admin',
}
```

> info **Gợi ý** Trong các hệ thống tinh vi hơn, bạn có thể lưu trữ roles trong một database, hoặc kéo chúng từ nhà cung cấp authentication bên ngoài.

Với điều này đã có, chúng ta có thể tạo một decorator `@Roles()`. Decorator này cho phép chỉ định các roles nào được yêu cầu để truy cập các tài nguyên cụ thể.

```typescript
@@filename(roles.decorator)
import { SetMetadata } from '@nestjs/common';
import { Role } from '../enums/role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
@@switch
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles) => SetMetadata(ROLES_KEY, roles);
```

Bây giờ chúng ta có một custom decorator `@Roles()`, chúng ta có thể sử dụng nó để decorate bất kỳ route handler nào.

```typescript
@@filename(cats.controller)
@Post()
@Roles(Role.Admin)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles(Role.Admin)
@Bind(Body())
create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

Cuối cùng, chúng ta tạo một class `RolesGuard` sẽ so sánh các roles được gán cho user hiện tại với các roles thực tế được yêu cầu bởi route hiện tại đang được xử lý. Để truy cập role(s) của route (metadata tùy chỉnh), chúng ta sẽ sử dụng class helper `Reflector`, được cung cấp out-of-the-box bởi framework và expose từ package `@nestjs/core`.

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
@Dependencies(Reflector)
export class RolesGuard {
  constructor(reflector) {
    this.reflector = reflector;
  }

  canActivate(context) {
    const requiredRoles = this.reflector.getAllAndOverride(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles.includes(role));
  }
}
```

> info **Gợi ý** Tham khảo phần [Reflection and metadata](/fundamentals/execution-context#reflection-and-metadata) của chương Execution context để biết thêm chi tiết về việc sử dụng `Reflector` theo cách nhạy cảm với ngữ cảnh.

> warning **Lưu ý** Ví dụ này được đặt tên là "**cơ bản**" vì chúng ta chỉ kiểm tra sự hiện diện của các roles ở mức route handler. Trong các ứng dụng thực tế, bạn có thể có endpoints/handlers liên quan đến một số thao tác, trong đó mỗi thao tác yêu cầu một tập hợp permissions cụ thể. Trong trường hợp đó, bạn sẽ phải cung cấp một cơ chế để kiểm tra roles ở đâu đó trong business-logic của mình, làm cho nó khó bảo trì hơn một chút vì sẽ không có nơi tập trung liên kết permissions với các hành động cụ thể.

Trong ví dụ này, chúng ta giả định rằng `request.user` chứa user instance và các roles được phép (dưới thuộc tính `roles`). Trong ứng dụng của bạn, bạn có thể sẽ tạo liên kết đó trong custom **authentication guard** của mình - xem chương [authentication](/security/authentication) để biết thêm chi tiết.

Để đảm bảo ví dụ này hoạt động, class `User` của bạn phải trông như sau:

```typescript
class User {
  // ...other properties
  roles: Role[];
}
```

Cuối cùng, đảm bảo đăng ký `RolesGuard`, ví dụ, ở mức controller, hoặc globally:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: RolesGuard,
  },
],
```

Khi một user với các quyền không đủ yêu cầu một endpoint, Nest tự động trả về phản hồi sau:

```typescript
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

> info **Gợi ý** Nếu bạn muốn trả về một phản hồi lỗi khác, bạn nên throw exception cụ thể của riêng bạn thay vì trả về một giá trị boolean.

<app-banner-courses-auth></app-banner-courses-auth>

#### Claims-based authorization

Khi một identity được tạo, nó có thể được gán một hoặc nhiều claims được phát hành bởi một bên đáng tin cậy. Một claim là một cặp tên-giá trị đại diện cho những gì subject có thể làm, không phải subject là gì.

Để triển khai một Claims-based authorization trong Nest, bạn có thể làm theo các bước tương tự mà chúng ta đã trình bày ở trên trong phần [RBAC](/security/authorization#basic-rbac-implementation) với một sự khác biệt quan trọng: thay vì kiểm tra các roles cụ thể, bạn nên so sánh **permissions**. Mỗi user sẽ có một tập hợp permissions được gán. Tương tự, mỗi tài nguyên/endpoint sẽ định nghĩa các permissions nào được yêu cầu (ví dụ, thông qua một decorator `@RequirePermissions()` chuyên dụng) để truy cập chúng.

```typescript
@@filename(cats.controller)
@Post()
@RequirePermissions(Permission.CREATE_CAT)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@RequirePermissions(Permission.CREATE_CAT)
@Bind(Body())
create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **Gợi ý** Trong ví dụ trên, `Permission` (tương tự như `Role` mà chúng ta đã trình bày trong phần RBAC) là một TypeScript enum chứa tất cả các permissions có sẵn trong hệ thống của bạn.

#### Integrating CASL

[CASL](https://casl.js.org/) là một thư viện authorization isomorphic hạn chế những tài nguyên mà một client nhất định được phép truy cập. Nó được thiết kế để có thể được chấp nhận tăng dần và có thể dễ dàng mở rộng giữa một authorization dựa trên claim đơn giản và một authorization dựa trên subject và attribute đầy đủ tính năng.

Để bắt đầu, trước tiên cài đặt package `@casl/ability`:

```bash
$ npm i @casl/ability
```

> info **Gợi ý** Trong ví dụ này, chúng ta đã chọn CASL, nhưng bạn có thể sử dụng bất kỳ thư viện nào khác như `accesscontrol` hoặc `acl`, tùy thuộc vào sở thích và nhu cầu dự án của bạn.

Sau khi cài đặt hoàn tất, để minh họa cơ chế của CASL, chúng ta sẽ định nghĩa hai class entity: `User` và `Article`.

```typescript
class User {
  id: number;
  isAdmin: boolean;
}
```

Class `User` bao gồm hai thuộc tính, `id`, là một định danh user duy nhất, và `isAdmin`, chỉ ra liệu một user có quyền quản trị viên hay không.

```typescript
class Article {
  id: number;
  isPublished: boolean;
  authorId: number;
}
```

Class `Article` có ba thuộc tính, lần lượt là `id`, `isPublished`, và `authorId`. `id` là một định danh bài viết duy nhất, `isPublished` chỉ ra liệu một bài viết đã được xuất bản hay chưa, và `authorId`, là ID của một user đã viết bài viết đó.

Bây giờ hãy xem xét và tinh chỉnh các yêu cầu của chúng ta cho ví dụ này:

- Admins có thể quản lý (tạo/đọc/cập nhật/xóa) tất cả các entities
- Users có quyền truy cập read-only cho mọi thứ
- Users có thể cập nhật các bài viết của họ (`article.authorId === userId`)
- Các bài viết đã xuất bản không thể bị xóa (`article.isPublished === true`)

Với điều này trong tâm trí, chúng ta có thể bắt đầu bằng cách tạo một enum `Action` đại diện cho tất cả các hành động có thể mà users có thể thực hiện với các entities:

```typescript
export enum Action {
  Manage = 'manage',
  Create = 'create',
  Read = 'read',
  Update = 'update',
  Delete = 'delete',
}
```

> warning **Lưu ý** `manage` là một từ khóa đặc biệt trong CASL đại diện cho "bất kỳ hành động nào".

Để đóng gói thư viện CASL, hãy tạo `CaslModule` và `CaslAbilityFactory` ngay bây giờ.

```bash
$ nest g module casl
$ nest g class casl/casl-ability.factory
```

Với điều này đã có, chúng ta có thể định nghĩa phương thức `createForUser()` trên `CaslAbilityFactory`. Phương thức này sẽ tạo đối tượng `Ability` cho một user nhất định:

```typescript
type Subjects = InferSubjects<typeof Article | typeof User> | 'all';

export type AppAbility = MongoAbility<[Action, Subjects]>;

@Injectable()
export class CaslAbilityFactory {
  createForUser(user: User) {
    const { can, cannot, build } = new AbilityBuilder(createMongoAbility);

    if (user.isAdmin) {
      can(Action.Manage, 'all'); // read-write access to everything
    } else {
      can(Action.Read, 'all'); // read-only access to everything
    }

    can(Action.Update, Article, { authorId: user.id });
    cannot(Action.Delete, Article, { isPublished: true });

    return build({
      // Read https://casl.js.org/v6/en/guide/subject-type-detection#use-classes-as-subject-types for details
      detectSubjectType: (item) =>
        item.constructor as ExtractSubjectType<Subjects>,
    });
  }
}
```

> warning **Lưu ý** `all` là một từ khóa đặc biệt trong CASL đại diện cho "bất kỳ subject nào".

> info **Gợi ý** Kể từ CASL v6, `MongoAbility` phục vụ như class ability mặc định, thay thế `Ability` kế thừa để hỗ trợ tốt hơn các permissions dựa trên điều kiện bằng cách sử dụng cú pháp giống MongoDB. Mặc dù tên, nó không bị ràng buộc với MongoDB — nó hoạt động với bất kỳ loại dữ liệu nào bằng cách đơn giản so sánh các đối tượng với các điều kiện được viết bằng cú pháp giống Mongo.

> info **Gợi ý** Các class `MongoAbility`, `AbilityBuilder`, `AbilityClass`, và `ExtractSubjectType` được xuất từ package `@casl/ability`.

> info **Gợi ý** Tùy chọn `detectSubjectType` cho phép CASL hiểu cách lấy subject type khỏi một đối tượng. Để biết thêm thông tin, hãy đọc [tài liệu CASL](https://casl.js.org/v6/en/guide/subject-type-detection#use-classes-as-subject-types).

Trong ví dụ trên, chúng ta đã tạo instance `MongoAbility` bằng cách sử dụng class `AbilityBuilder`. Như bạn có thể đã đoán, `can` và `cannot` chấp nhận các đối số giống nhau nhưng có ý nghĩa khác nhau, `can` cho phép bạn thực hiện một hành động trên subject được chỉ định và `cannot` cấm nó. Cả hai có thể chấp nhận tối đa 4 đối số. Để tìm hiểu thêm về các hàm này, hãy truy cập [tài liệu CASL chính thức](https://casl.js.org/v6/en/guide/intro).

Cuối cùng, đảm bảo thêm `CaslAbilityFactory` vào các mảng `providers` và `exports` trong định nghĩa module `CaslModule`:

```typescript
import { Module } from '@nestjs/common';
import { CaslAbilityFactory } from './casl-ability.factory';

@Module({
  providers: [CaslAbilityFactory],
  exports: [CaslAbilityFactory],
})
export class CaslModule {}
```

Với điều này đã có, chúng ta có thể inject `CaslAbilityFactory` vào bất kỳ class nào bằng cách sử dụng constructor injection tiêu chuẩn miễn là `CaslModule` được import trong ngữ cảnh host:

```typescript
constructor(private caslAbilityFactory: CaslAbilityFactory) {}
```

Sau đó sử dụng nó trong một class như sau.

```typescript
const ability = this.caslAbilityFactory.createForUser(user);
if (ability.can(Action.Read, 'all')) {
  // "user" has read access to everything
}
```

> info **Gợi ý** Tìm hiểu thêm về class `MongoAbility` trong [tài liệu CASL chính thức](https://casl.js.org/v6/en/guide/intro).

Ví dụ, hãy nói rằng chúng ta có một user không phải là admin. Trong trường hợp đó, user nên có thể đọc các bài viết, nhưng việc tạo bài viết mới hoặc xóa các bài viết hiện có nên bị cấm.

```typescript
const user = new User();
user.isAdmin = false;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Read, Article); // true
ability.can(Action.Delete, Article); // false
ability.can(Action.Create, Article); // false
```

> info **Gợi ý** Mặc dù cả hai class `MongoAbility` và `AbilityBuilder` đều cung cấp các phương thức `can` và `cannot`, chúng có mục đích khác nhau và chấp nhận các đối số hơi khác nhau.

Ngoài ra, như chúng ta đã chỉ định trong các yêu cầu của mình, user nên có thể cập nhật các bài viết của họ:

```typescript
const user = new User();
user.id = 1;

const article = new Article();
article.authorId = user.id;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Update, article); // true

article.authorId = 2;
ability.can(Action.Update, article); // false
```

Như bạn có thể thấy, instance `MongoAbility` cho phép chúng ta kiểm tra permissions theo cách rất dễ đọc. Tương tự, `AbilityBuilder` cho phép chúng ta định nghĩa permissions (và chỉ định các điều kiện khác nhau) theo cách tương tự. Để tìm thêm ví dụ, hãy truy cập tài liệu chính thức.

#### Advanced: Implementing a `PoliciesGuard`

Trong phần này, chúng ta sẽ chứng minh cách xây dựng một guard tinh vi hơn một chút, kiểm tra xem một user có đáp ứng các **authorization policies** cụ thể có thể được cấu hình ở mức method hay không (bạn có thể mở rộng nó để tôn trọng các policies được cấu hình ở mức class cũng vậy). Trong ví dụ này, chúng ta sẽ sử dụng package CASL chỉ cho mục đích minh họa, nhưng việc sử dụng thư viện này không được yêu cầu. Ngoài ra, chúng ta sẽ sử dụng provider `CaslAbilityFactory` mà chúng ta đã tạo trong phần trước.

Trước tiên, hãy làm rõ các yêu cầu. Mục tiêu là cung cấp một cơ chế cho phép chỉ định các kiểm tra policy cho mỗi route handler. Chúng ta sẽ hỗ trợ cả objects và functions (cho các kiểm tra đơn giản hơn và cho những người thích code phong cách functional hơn).

Hãy bắt đầu bằng cách định nghĩa các interfaces cho policy handlers:

```typescript
import { AppAbility } from '../casl/casl-ability.factory';

interface IPolicyHandler {
  handle(ability: AppAbility): boolean;
}

type PolicyHandlerCallback = (ability: AppAbility) => boolean;

export type PolicyHandler = IPolicyHandler | PolicyHandlerCallback;
```

Như đã đề cập ở trên, chúng ta đã cung cấp hai cách có thể để định nghĩa một policy handler, một object (instance của một class triển khai interface `IPolicyHandler`) và một function (thỏa mãn type `PolicyHandlerCallback`).

Với điều này đã có, chúng ta có thể tạo một decorator `@CheckPolicies()`. Decorator này cho phép chỉ định các policies nào phải được đáp ứng để truy cập các tài nguyên cụ thể.

```typescript
export const CHECK_POLICIES_KEY = 'check_policy';
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(CHECK_POLICIES_KEY, handlers);
```

Bây giờ hãy tạo một `PoliciesGuard` sẽ trích xuất và thực thi tất cả các policy handlers được gán cho một route handler.

```typescript
@Injectable()
export class PoliciesGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private caslAbilityFactory: CaslAbilityFactory,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const policyHandlers =
      this.reflector.get<PolicyHandler[]>(
        CHECK_POLICIES_KEY,
        context.getHandler(),
      ) || [];

    const { user } = context.switchToHttp().getRequest();
    const ability = this.caslAbilityFactory.createForUser(user);

    return policyHandlers.every((handler) =>
      this.execPolicyHandler(handler, ability),
    );
  }

  private execPolicyHandler(handler: PolicyHandler, ability: AppAbility) {
    if (typeof handler === 'function') {
      return handler(ability);
    }
    return handler.handle(ability);
  }
}
```

> info **Gợi ý** Trong ví dụ này, chúng ta giả định rằng `request.user` chứa user instance. Trong ứng dụng của bạn, bạn có thể sẽ tạo liên kết đó trong custom **authentication guard** của mình - xem chương [authentication](/security/authentication) để biết thêm chi tiết.

Hãy phân tích ví dụ này. `policyHandlers` là một mảng các handlers được gán cho phương thức thông qua decorator `@CheckPolicies()`. Tiếp theo, chúng ta sử dụng phương thức `CaslAbilityFactory#create` xây dựng đối tượng `Ability`, cho phép chúng ta xác minh liệu một user có đủ permissions để thực hiện các hành động cụ thể hay không. Chúng ta đang truyền đối tượng này đến policy handler là một function hoặc một instance của một class triển khai `IPolicyHandler`, exposing phương thức `handle()` trả về một boolean. Cuối cùng, chúng ta sử dụng phương thức `Array#every` để đảm bảo rằng mọi handler trả về giá trị `true`.

Cuối cùng, để test guard này, bind nó đến bất kỳ route handler nào, và đăng ký một inline policy handler (cách tiếp cận functional), như sau:

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies((ability: AppAbility) => ability.can(Action.Read, Article))
findAll() {
  return this.articlesService.findAll();
}
```

Ngoài ra, chúng ta có thể định nghĩa một class triển khai interface `IPolicyHandler`:

```typescript
export class ReadArticlePolicyHandler implements IPolicyHandler {
  handle(ability: AppAbility) {
    return ability.can(Action.Read, Article);
  }
}
```

Và sử dụng nó như sau:

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies(new ReadArticlePolicyHandler())
findAll() {
  return this.articlesService.findAll();
}
```

> warning **Lưu ý** Vì chúng ta phải khởi tạo policy handler tại chỗ bằng cách sử dụng từ khóa `new`, class `ReadArticlePolicyHandler` không thể sử dụng Dependency Injection. Điều này có thể được giải quyết bằng phương thức `ModuleRef#get` (đọc thêm [ở đây](/fundamentals/module-ref)). Về cơ bản, thay vì đăng ký các functions và instances thông qua decorator `@CheckPolicies()`, bạn phải cho phép truyền một `Type<IPolicyHandler>`. Sau đó, bên trong guard của bạn, bạn có thể lấy một instance bằng cách sử dụng một tham chiếu type: `moduleRef.get(YOUR_HANDLER_TYPE)` hoặc thậm chí khởi tạo nó động bằng cách sử dụng phương thức `ModuleRef#create`.