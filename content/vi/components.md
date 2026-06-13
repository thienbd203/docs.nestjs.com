### Providers

Providers là một khái niệm cốt lõi trong Nest. Nhiều lớp Nest cơ bản, chẳng hạn như services, repositories, factories và helpers, có thể được coi như providers. Ý tưởng chính đằng sau một provider là nó có thể được **inject** như một dependency, cho phép các đối tượng hình thành các mối quan hệ khác nhau với nhau. Trách nhiệm của "kết nối" các đối tượng này chủ yếu được xử lý bởi hệ thống thời gian chạy Nest.

<figure><img class="illustrative-image" src="/assets/Components_1.png" /></figure>

Trong chương trước, chúng tôi đã tạo một `CatsController` đơn giản. Controllers nên xử lý các yêu cầu HTTP và ủy quyền các nhiệm vụ phức tạp hơn cho **providers**. Providers là các lớp JavaScript thuần túy được khai báo như `providers` trong một module NestJS. Để biết thêm chi tiết, hãy tham khảo chương "Modules".

> info **Gợi ý** Vì Nest cho phép bạn thiết kế và tổ chức các dependencies theo cách hướng đối tượng, chúng tôi khuyến nghị mạnh mẽ việc tuân thủ [nguyên tắc SOLID](https://en.wikipedia.org/wiki/SOLID).

#### Services

Hãy bắt đầu bằng cách tạo một `CatsService` đơn giản. Service này sẽ xử lý lưu trữ và truy xuất dữ liệu, và sẽ được sử dụng bởi `CatsController`. Vì vai trò của nó trong việc quản lý logic ứng dụng, nó là một ứng cử viên lý tưởng để được định nghĩa như một provider.

```typescript
@@filename(cats.service)
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor() {
    this.cats = [];
  }

  create(cat) {
    this.cats.push(cat);
  }

  findAll() {
    return this.cats;
  }
}
```

> info **Gợi ý** Để tạo một service sử dụng CLI, chỉ cần thực thi lệnh `$ nest g service cats`.

`CatsService` của chúng ta là một lớp cơ bản với một thuộc tính và hai phương thức. Phần bổ sung chính ở đây là decorator `@Injectable()`. Decorator này gắn metadata vào lớp, báo hiệu rằng `CatsService` là một lớp có thể được quản lý bởi container [IoC](https://en.wikipedia.org/wiki/Inversion_of_control) của Nest.

Ngoài ra, ví dụ này sử dụng một interface `Cat`, có thể trông giống như sau:

```typescript
@@filename(interfaces/cat.interface)
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

Bây giờ chúng ta có một lớp service để truy xuất cats, hãy sử dụng nó bên trong `CatsController`:

```typescript
@@filename(cats.controller)
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
@@switch
import { Controller, Get, Post, Body, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Post()
  @Bind(Body())
  async create(createCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

`CatsService` được **inject** thông qua constructor của lớp. Lưu ý việc sử dụng từ khóa `private`. Cách viết tắt này cho phép chúng ta vừa khai báo vừa khởi tạo thành viên `catsService` trên cùng một dòng, hợp lý hóa quá trình.

#### Dependency injection

Nest được xây dựng xung quanh mẫu thiết kế mạnh mẽ được gọi là **Dependency Injection**. Chúng tôi khuyến nghị mạnh mẽ việc đọc một bài viết tuyệt vời về khái niệm này trong tài liệu chính thức của [Angular](https://angular.dev/guide/di).

Trong Nest, nhờ vào khả năng của TypeScript, quản lý các dependencies là đơn giản vì chúng được giải quyết dựa trên kiểu của chúng. Trong ví dụ dưới đây, Nest sẽ giải quyết `catsService` bằng cách tạo và trả về một instance của `CatsService` (hoặc, trong trường hợp một singleton, trả về instance hiện có nếu nó đã được yêu cầu ở nơi khác). Dependency này sau đó được inject vào constructor của controller của bạn (hoặc được gán cho thuộc tính được chỉ định):

```typescript
constructor(private catsService: CatsService) {}
```

#### Phạm vi

Các providers thường có một thời gian tồn tại ("phạm vi") phù hợp với vòng đời ứng dụng. Khi ứng dụng được khởi tạo, mỗi dependency phải được giải quyết, nghĩa là mọi provider được khởi tạo. Tương tự, khi ứng dụng tắt, tất cả các providers bị hủy. Tuy nhiên, cũng có thể làm cho một provider trở thành **phạm vi request**, nghĩa là thời gian tồn tại của nó được gắn với một yêu cầu cụ thể thay vì vòng đời của ứng dụng. Bạn có thể tìm hiểu thêm về các kỹ thuật này trong chương [Injection Scopes](/fundamentals/injection-scopes).

<app-banner-courses></app-banner-courses>

#### Custom providers

Nest đi kèm với một container inversion of control ("IoC") tích hợp quản lý các mối quan hệ giữa các providers. Tính năng này là nền tảng của dependency injection, nhưng thực tế nó mạnh hơn nhiều so với chúng tôi đã bao gồm cho đến nay. Có một số cách để định nghĩa một provider: bạn có thể sử dụng các giá trị thuần túy, các lớp, và cả factories đồng bộ hoặc bất đồng bộ. Để biết thêm ví dụ về việc định nghĩa providers, hãy xem chương [Dependency Injection](/fundamentals/dependency-injection).

#### Optional providers

Đôi khi, bạn có thể có các dependencies không luôn cần phải được giải quyết. Ví dụ, lớp của bạn có thể phụ thuộc vào một **đối tượng cấu hình**, nhưng nếu không có đối tượng nào được cung cấp, các giá trị mặc định nên được sử dụng. Trong những trường hợp như vậy, dependency được coi là tùy chọn, và sự thiếu vắng của provider cấu hình không nên dẫn đến lỗi.

Để đánh dấu một provider là tùy chọn, sử dụng decorator `@Optional()` trong chữ ký constructor.

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

Trong ví dụ trên, chúng tôi đang sử dụng một provider tùy chỉnh, đó là lý do chúng tôi bao gồm **token** tùy chỉnh `HTTP_OPTIONS`. Các ví dụ trước đã chứng minh injection dựa trên constructor, trong đó một dependency được chỉ định thông qua một lớp trong constructor. Để biết thêm chi tiết về các providers tùy chỉnh và cách các token liên quan của chúng hoạt động, hãy xem chương [Custom Providers](/fundamentals/custom-providers).

#### Injection dựa trên thuộc tính

Kỹ thuật chúng tôi đã sử dụng cho đến nay được gọi là injection dựa trên constructor, trong đó các providers được inject thông qua phương thức constructor. Trong một số trường hợp cụ thể, **injection dựa trên thuộc tính** có thể hữu ích. Ví dụ, nếu lớp cấp cao nhất của bạn phụ thuộc vào một hoặc nhiều providers, chuyển chúng tất cả lên thông qua `super()` trong các lớp con có thể trở nên cồng kềnh. Để tránh điều này, bạn có thể sử dụng decorator `@Inject()` trực tiếp ở cấp thuộc tính.

```typescript
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

> warning **Cảnh báo** Nếu lớp của bạn không mở rộng một lớp khác, nói chung tốt hơn nên sử dụng injection **dựa trên constructor**. Constructor chỉ định rõ các dependencies nào được yêu cầu, cung cấp khả năng hiển thị tốt hơn và làm cho mã dễ hiểu hơn so với các thuộc tính lớp được chú thích với `@Inject`.

#### Đăng ký provider

Bây giờ chúng tôi đã định nghĩa một provider (`CatsService`) và một consumer (`CatsController`), chúng ta cần đăng ký service với Nest để nó có thể xử lý việc injection. Điều này được thực hiện bằng cách chỉnh sửa file module (`app.module.ts`) và thêm service vào mảng `providers` trong decorator `@Module()`.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

Bây giờ Nest sẽ có thể giải quyết các dependencies của lớp `CatsController`.

Tại thời điểm này, cấu trúc thư mục của chúng ta nên trông như sau:

<div class="file-tree">
<div class="item">src</div>
<div class="children">
<div class="item">cats</div>
<div class="children">
<div class="item">dto</div>
<div class="children">
<div class="item">create-cat.dto.ts</div>
</div>
<div class="item">interfaces</div>
<div class="children">
<div class="item">cat.interface.ts</div>
</div>
<div class="item">cats.controller.ts</div>
<div class="item">cats.service.ts</div>
</div>
<div class="item">app.module.ts</div>
<div class="item">main.ts</div>
</div>
</div>

#### Khởi tạo thủ công

Cho đến nay, chúng tôi đã bao gồm cách Nest tự động xử lý hầu hết các chi tiết của việc giải quyết dependencies. Tuy nhiên, trong một số trường hợp, bạn có thể cần bước ra khỏi hệ thống Dependency Injection tích hợp và truy xuất hoặc khởi tạo các providers thủ công. Hai kỹ thuật như vậy được thảo luận ngắn gọn dưới đây.

- Để truy xuất các instances hiện có hoặc khởi tạo các providers động, bạn có thể sử dụng [Module reference](https://docs.nestjs.com/fundamentals/module-ref).
- Để lấy các providers trong hàm `bootstrap()` (ví dụ, cho các ứng dụng độc lập hoặc để sử dụng một service cấu hình trong quá trình khởi tạo), hãy xem [Ứng dụng độc lập](https://docs.nestjs.com/standalone-applications).