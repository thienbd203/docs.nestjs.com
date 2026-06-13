### Injection scopes

Đối với những người đến từ các nền tảng ngôn ngữ lập trình khác, có thể bất ngờ khi biết rằng trong Nest, hầu hết mọi thứ đều được chia sẻ trên các incoming requests. Chúng ta có một connection pool đến database, các singleton services với global state, v.v. Hãy nhớ rằng Node.js không tuân theo mô hình Multi-Threaded Stateless request/response trong đó mỗi request được xử lý bởi một thread riêng biệt. Do đó, việc sử dụng các singleton instances là hoàn toàn **an toàn** cho các ứng dụng của chúng ta.

Tuy nhiên, có những trường hợp ngoại lệ khi lifetime dựa trên request có thể là hành vi mong muốn, ví dụ, per-request caching trong các ứng dụng GraphQL, request tracking, và multi-tenancy. Injection scopes cung cấp một cơ chế để có được hành động lifetime provider mong muốn.

#### Provider scope

Một provider có thể có bất kỳ scope nào sau đây:

<table>
  <tr>
    <td><code>DEFAULT</code></td>
    <td>Một instance duy nhất của provider được chia sẻ trên toàn bộ ứng dụng. Instance lifetime được gắn trực tiếp với lifecycle của ứng dụng. Khi ứng dụng đã được bootstrapped, tất cả các singleton providers đã được instantiate. Singleton scope được sử dụng theo mặc định.</td>
  </tr>
  <tr>
    <td><code>REQUEST</code></td>
    <td>Một instance mới của provider được tạo riêng cho mỗi incoming <strong>request</strong>. Instance được garbage-collected sau khi request đã hoàn thành xử lý.</td>
  </tr>
  <tr>
    <td><code>TRANSIENT</code></td>
    <td>Transient providers không được chia sẻ giữa các consumers. Mỗi consumer inject một transient provider sẽ nhận được một instance mới, dành riêng.</td>
  </tr>
</table>

> info **Hint** Sử dụng singleton scope được **khuyến nghị** cho hầu hết các use cases. Việc chia sẻ providers giữa các consumers và giữa các requests có nghĩa là một instance có thể được cached và việc khởi tạo của nó chỉ xảy ra một lần, trong quá trình khởi động ứng dụng.

#### Usage

Chỉ định injection scope bằng cách truyền thuộc tính `scope` vào đối tượng options của decorator `@Injectable()`:

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {}
```

Tương tự, đối với [custom providers](/fundamentals/custom-providers), đặt thuộc tính `scope` trong dạng dài cho đăng ký provider:

```typescript
{
  provide: 'CACHE_MANAGER',
  useClass: CacheManager,
  scope: Scope.TRANSIENT,
}
```

> info **Hint** Import enum `Scope` từ `@nestjs/common`

Singleton scope được sử dụng theo mặc định và không cần phải khai báo. Nếu bạn muốn khai báo một provider là singleton scoped, sử dụng giá trị `Scope.DEFAULT` cho thuộc tính `scope`.

> warning **Notice** Websocket Gateways không nên sử dụng request-scoped providers vì chúng phải hoạt động như singletons. Mỗi gateway đóng gói một socket thực và không thể được instantiate nhiều lần. Giới hạn này cũng áp dụng cho một số providers khác, như [_Passport strategies_](../security/authentication#request-scoped-strategies) hoặc _Cron controllers_.

#### Controller scope

Controllers cũng có thể có scope, áp dụng cho tất cả các request method handlers được khai báo trong controller đó. Giống như provider scope, scope của một controller khai báo lifetime của nó. Đối với một controller request-scoped, một instance mới được tạo cho mỗi inbound request, và được garbage-collected khi request đã hoàn thành xử lý.

Khai báo controller scope với thuộc tính `scope` của đối tượng `ControllerOptions`:

```typescript
@Controller({
  path: 'cats',
  scope: Scope.REQUEST,
})
export class CatsController {}
```

#### Scope hierarchy

Scope `REQUEST` lan truyền lên chuỗi injection. Một controller phụ thuộc vào một provider request-scoped sẽ tự trở thành request-scoped.

Hãy tưởng tượng dependency graph sau: `CatsController <- CatsService <- CatsRepository`. Nếu `CatsService` là request-scoped (và các cái khác là default singletons), `CatsController` sẽ trở thành request-scoped vì nó phụ thuộc vào service được inject. `CatsRepository`, không phụ thuộc, sẽ vẫn là singleton-scoped.

Các dependency transient-scoped không tuân theo pattern đó. Nếu một `DogsService` singleton-scoped inject một provider `LoggerService` transient, nó sẽ nhận được một instance mới của nó. Tuy nhiên, `DogsService` sẽ vẫn là singleton-scoped, vì vậy việc inject nó ở bất kỳ đâu sẽ _không_ resolve thành một instance mới của `DogsService`. Trong trường hợp đó là hành vi mong muốn, `DogsService` phải được đánh dấu rõ ràng là `TRANSIENT` cũng vậy.

<app-banner-courses></app-banner-courses>

#### Request provider

Trong một ứng dụng dựa trên HTTP server (ví dụ, sử dụng `@nestjs/platform-express` hoặc `@nestjs/platform-fastify`), bạn có thể muốn truy cập tham chiếu đến request object gốc khi sử dụng request-scoped providers. Bạn có thể làm điều này bằng cách inject object `REQUEST`.

Provider `REQUEST` vốn dĩ là request-scoped, nghĩa là bạn không cần chỉ định scope `REQUEST` một cách rõ ràng khi sử dụng nó. Ngoài ra, ngay cả khi bạn cố gắng làm như vậy, nó sẽ bị bỏ qua. Bất kỳ provider nào phụ thuộc vào một provider request-scoped sẽ tự động áp dụng request scope, và hành vi này không thể thay đổi.

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(REQUEST) private request: Request) {}
}
```

Do sự khác biệt nền tảng/giao thức cơ bản, bạn truy cập inbound request một chút khác nhau cho các ứng dụng Microservice hoặc GraphQL. Trong các ứng dụng [GraphQL](/graphql/quick-start), bạn inject `CONTEXT` thay vì `REQUEST`:

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT } from '@nestjs/graphql';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private context) {}
}
```

Sau đó bạn cấu hình giá trị `context` của bạn (trong `GraphQLModule`) để chứa `request` làm property của nó.

#### Inquirer provider

Nếu bạn muốn lấy class nơi một provider được xây dựng, ví dụ trong các providers logging hoặc metrics, bạn có thể inject token `INQUIRER`.

```typescript
import { Inject, Injectable, Scope } from '@nestjs/common';
import { INQUIRER } from '@nestjs/core';

@Injectable({ scope: Scope.TRANSIENT })
export class HelloService {
  constructor(@Inject(INQUIRER) private parentClass: object) {}

  sayHello(message: string) {
    console.log(`${this.parentClass?.constructor?.name}: ${message}`);
  }
}
```

Và sau đó sử dụng nó như sau:

```typescript
import { Injectable } from '@nestjs/common';
import { HelloService } from './hello.service';

@Injectable()
export class AppService {
  constructor(private helloService: HelloService) {}

  getRoot(): string {
    this.helloService.sayHello('My name is getRoot');

    return 'Hello world!';
  }
}
```

Trong ví dụ trên khi `AppService#getRoot` được gọi, `"AppService: My name is getRoot"` sẽ được logged vào console.

#### Performance

Việc sử dụng request-scoped providers sẽ có ảnh hưởng đến hiệu suất ứng dụng. Mặc dù Nest cố gắng cache nhiều metadata nhất có thể, nó vẫn sẽ phải tạo một instance của class của bạn trên mỗi request. Do đó, nó sẽ làm chậm thời gian phản hồi trung bình và kết quả benchmarking tổng thể. Trừ khi một provider phải là request-scoped, rất khuyến nghị rằng bạn sử dụng singleton scope mặc định.

> info **Hint** Mặc dù tất cả nghe có vẻ khá đáng sợ, một ứng dụng được thiết kế đúng cách tận dụng request-scoped providers không nên chậm hơn ~5% về độ trễ.

#### Durable providers

Request-scoped providers, như được đề cập trong phần trên, có thể dẫn đến độ trễ tăng vì có ít nhất 1 provider request-scoped (được inject vào controller instance, hoặc sâu hơn - được inject vào một trong các providers của nó) làm cho controller cũng trở thành request-scoped. Điều đó có nghĩa là nó phải được tạo lại (instantiate) cho mỗi request riêng lẻ (và garbage collected sau đó). Bây giờ, điều đó cũng có nghĩa là, giả sử cho 30k requests song song, sẽ có 30k instances ngắn hạn của controller (và các providers request-scoped của nó).

Việc có một provider chung mà hầu hết các providers phụ thuộc vào (hãy nghĩ về một database connection, hoặc một logger service), tự động chuyển đổi tất cả các providers đó thành các providers request-scoped cũng vậy. Điều này có thể tạo ra thách thức trong **các ứng dụng multi-tenant**, đặc biệt là đối với những ứng dụng có một provider "data source" request-scoped trung tâm lấy headers/token từ request object và dựa trên các giá trị của nó, truy xuất database connection/schema tương ứng (cụ thể cho tenant đó).

Ví dụ, giả sử bạn có một ứng dụng được sử dụng thay thế bởi 10 khách hàng khác nhau. Mỗi khách hàng có **data source riêng biệt của riêng họ**, và bạn muốn đảm bảo khách hàng A sẽ không bao giờ có thể truy cập database của khách hàng B. Một cách để đạt được điều này có thể là khai báo một provider "data source" request-scoped - dựa trên request object - xác định "khách hàng hiện tại" là gì và truy xuất database tương ứng của nó. Với cách tiếp cận này, bạn có thể biến ứng dụng của mình thành một ứng dụng multi-tenant chỉ trong vài phút. Nhưng, một nhược điểm lớn của cách tiếp cận này là vì rất có thể một phần lớn các thành phần của ứng dụng phụ thuộc vào provider "data source", chúng sẽ trở thành "request-scoped" một cách ngầm định, và do đó bạn chắc chắn sẽ thấy ảnh hưởng trong hiệu suất ứng dụng của bạn.

Nhưng nếu chúng ta có một giải pháp tốt hơn? Vì chúng ta chỉ có 10 khách hàng, chúng ta không thể có 10 [DI sub-trees](/fundamentals/module-ref#resolving-scoped-providers) riêng biệt cho mỗi khách hàng (thay vì tạo lại mỗi cây cho mỗi request)? Nếu các providers của bạn không phụ thuộc vào bất kỳ property nào thực sự duy nhất cho mỗi request liên tiếp (ví dụ, request UUID) mà thay vào đó có một số thuộc tính cụ thể cho phép chúng ta tổng hợp (phân loại) chúng, không có lý do gì để _tạo lại DI sub-tree_ trên mỗi incoming request.

Và đó chính là lúc **durable providers** phát huy tác dụng.

Trước khi chúng ta bắt đầu đánh dấu providers là durable, chúng ta phải trước tiên đăng ký một **chiến lược** hướng dẫn Nest những "thuộc tính request chung" là gì, cung cấp logic nhóm requests - liên kết chúng với các DI sub-tree tương ứng của chúng.

```typescript
import {
  HostComponentInfo,
  ContextId,
  ContextIdFactory,
  ContextIdStrategy,
} from '@nestjs/core';
import { Request } from 'express';

const tenants = new Map<string, ContextId>();

export class AggregateByTenantContextIdStrategy implements ContextIdStrategy {
  attach(contextId: ContextId, request: Request) {
    const tenantId = request.headers['x-tenant-id'] as string;
    let tenantSubTreeId: ContextId;

    if (tenants.has(tenantId)) {
      tenantSubTreeId = tenants.get(tenantId);
    } else {
      tenantSubTreeId = ContextIdFactory.create();
      tenants.set(tenantId, tenantSubTreeId);
    }

    // If tree is not durable, return the original "contextId" object
    return (info: HostComponentInfo) =>
      info.isTreeDurable ? tenantSubTreeId : contextId;
  }
}
```

> info **Hint** Tương tự như request scope, durability lan truyền lên chuỗi injection. Điều đó có nghĩa là nếu A phụ thuộc vào B được đánh dấu là `durable`, A ngầm định trở thành durable cũng vậy (trừ khi `durable` được đặt rõ ràng thành `false` cho provider A).

> warning **Warning** Lưu ý chiến lược này không lý tưởng cho các ứng dụng hoạt động với một số lượng lớn tenants.

Giá trị được trả về từ phương thức `attach` hướng dẫn Nest identifier context nào nên được sử dụng cho một host nhất định. Trong trường hợp này, chúng tôi chỉ định rằng `tenantSubTreeId` nên được sử dụng thay vì object `contextId` gốc, được tạo tự động, khi thành phần host (ví dụ, controller request-scoped) được đánh dấu là durable (bạn có thể tìm hiểu cách đánh dấu providers là durable bên dưới). Ngoài ra, trong ví dụ trên, **không có payload** nào sẽ được đăng ký (nơi payload = provider `REQUEST`/`CONTEXT` đại diện cho "root" - parent của sub-tree).

Nếu bạn muốn đăng ký payload cho một cây durable, sử dụng cấu trúc sau thay thế:

```typescript
// The return of `AggregateByTenantContextIdStrategy#attach` method:
return {
  resolve: (info: HostComponentInfo) =>
    info.isTreeDurable ? tenantSubTreeId : contextId,
  payload: { tenantId },
};
```

Bây giờ bất cứ khi nào bạn inject provider `REQUEST` (hoặc `CONTEXT` cho các ứng dụng GraphQL) sử dụng `@Inject(REQUEST)`/`@Inject(CONTEXT)`, object `payload` sẽ được inject (bao gồm một property duy nhất - `tenantId` trong trường hợp này).

Được rồi, với chiến lược này tại chỗ, bạn có thể đăng ký nó ở đâu đó trong code của bạn (vì nó áp dụng toàn cầu anyway), vì vậy ví dụ, bạn có thể đặt nó trong file `main.ts`:

```typescript
ContextIdFactory.apply(new AggregateByTenantContextIdStrategy());
```

> info **Hint** Class `ContextIdFactory` được import từ package `@nestjs/core`.

Chừng nào việc đăng ký xảy ra trước khi bất kỳ request nào truy cập ứng dụng của bạn, mọi thứ sẽ hoạt động như dự định.

Cuối cùng, để biến một provider thường thành một provider durable, chỉ cần đặt flag `durable` thành `true` và thay đổi scope của nó thành `Scope.REQUEST` (không cần thiết nếu scope REQUEST đã có trong chuỗi injection):

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST, durable: true })
export class CatsService {}
```

Tương tự, đối với [custom providers](/fundamentals/custom-providers), đặt thuộc tính `durable` trong dạng dài cho đăng ký provider:

```typescript
{
  provide: 'foobar',
  useFactory: () => { ... },
  scope: Scope.REQUEST,
  durable: true,
}
```