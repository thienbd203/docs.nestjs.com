### Modules

Module là một class được annotate với decorator `@Module()`. Decorator này cung cấp metadata mà **Nest** sử dụng để tổ chức và quản lý cấu trúc ứng dụng một cách hiệu quả.

<figure><img class="illustrative-image" src="/assets/Modules_1.png" /></figure>

Mọi ứng dụng Nest đều có ít nhất một module, module **root**, đóng vai trò là điểm bắt đầu để Nest xây dựng **application graph**. Graph này là một cấu trúc nội bộ mà Nest sử dụng để giải quyết các mối quan hệ và dependencies giữa các modules và providers. Trong khi các ứng dụng nhỏ có thể chỉ có một root module, điều này thường không phải là trường hợp. Modules được **khuyến nghị mạnh** như một cách hiệu quả để tổ chức các components của bạn. Đối với hầu hết các ứng dụng, bạn có thể sẽ có nhiều modules, mỗi module đóng gói một tập hợp **khả năng** liên quan chặt chẽ.

Decorator `@Module()` nhận một đối tượng duy nhất với các thuộc tính mô tả module:

||               |                                                                                                                                                                                                          |
|| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|| `providers`   | các providers sẽ được khởi tạo bởi Nest injector và có thể được chia sẻ ít nhất trên module này                                                                                          |
|| `controllers` | tập hợp các controllers được định nghĩa trong module này cần được khởi tạo                                                                                                                              |
|| `imports`     | danh sách các imported modules export các providers cần thiết trong module này                                                                                                                 |
|| `exports`     | tập hợp con của `providers` được cung cấp bởi module này và nên có sẵn trong các modules khác import module này. Bạn có thể sử dụng chính provider hoặc chỉ token của nó (`provide` value) |

Module **đóng gói** providers theo mặc định, có nghĩa là bạn chỉ có thể inject các providers là một phần của module hiện tại hoặc được export rõ ràng từ các modules imported khác. Các providers được export từ một module về cơ bản đóng vai trò là interface công khai hoặc API của module.

#### Feature modules

Trong ví dụ của chúng ta, `CatsController` và `CatsService` liên quan chặt chẽ và phục vụ cùng một domain ứng dụng. Đó là hợp lý để nhóm chúng vào một feature module. Feature module tổ chức code liên quan đến một tính năng cụ thể, giúp duy trì các ranh giới rõ ràng và tổ chức tốt hơn. Điều này đặc biệt quan trọng khi ứng dụng hoặc nhóm phát triển, và nó phù hợp với các nguyên tắc SOLID.

Tiếp theo, chúng ta sẽ tạo `CatsModule` để minh họa cách nhóm controller và service.

```typescript
@@filename(cats/cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

> info **Gợi ý** Để tạo một module sử dụng CLI, chỉ cần thực thi lệnh `$ nest g module cats`.

Ở trên, chúng ta định nghĩa `CatsModule` trong file `cats.module.ts`, và di chuyển mọi thứ liên quan đến module này vào thư mục `cats`. Điều cuối cùng chúng ta cần làm là import module này vào root module (`AppModule`, được định nghĩa trong file `app.module.ts`).

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

Đây là cách cấu trúc thư mục của chúng ta trông như bây giờ:

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
      <div class="item">cats.module.ts</div>
      <div class="item">cats.service.ts</div>
    </div>
    <div class="item">app.module.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

#### Shared modules

Trong Nest, các modules là **singletons** theo mặc định, và do đó bạn có thể chia sẻ cùng một instance của bất kỳ provider nào giữa nhiều modules một cách dễ dàng.

<figure><img class="illustrative-image" src="/assets/Shared_Module_1.png" /></figure>

Mọi module tự động là một **shared module**. Một khi được tạo, nó có thể được tái sử dụng bởi bất kỳ module nào. Hãy tưởng tượng rằng chúng ta muốn chia sẻ một instance của `CatsService` giữa một số modules khác. Để làm điều đó, trước tiên chúng ta cần **export** provider `CatsService` bằng cách thêm nó vào mảng `exports` của module, như được hiển thị dưới đây:

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

Bây giờ bất kỳ module nào import `CatsModule` đều có truy cập đến `CatsService` và sẽ chia sẻ cùng một instance với tất cả các modules khác import nó.

Nếu chúng ta đăng ký trực tiếp `CatsService` trong mỗi module yêu cầu nó, nó thực sự sẽ hoạt động, nhưng nó sẽ dẫn đến mỗi module nhận được instance riêng của `CatsService`. Điều này có thể dẫn đến tăng sử dụng bộ nhớ vì nhiều instances của cùng một service được tạo ra, và nó cũng có thể gây ra hành vi không mong đợi, chẳng hạn as inconsistency trạng thái nếu service duy trì bất kỳ trạng thái nội bộ nào.

Bằng cách đóng gói `CatsService` bên trong một module, chẳng hạn như `CatsModule`, và export nó, chúng ta đảm bảo rằng cùng một instance của `CatsService` được tái sử dụng trên tất cả các modules import `CatsModule`. Điều này không chỉ giảm tiêu thụ bộ nhớ mà còn dẫn đến hành vi dễ dự đoán hơn, vì tất cả các modules chia sẻ cùng một instance, làm cho nó dễ dàng hơn để quản lý các trạng thái hoặc tài nguyên chia sẻ. Đây là một trong những lợi ích chính của tính mô đun và dependency injection trong các framework như NestJS—cho phép các services được chia sẻ hiệu quả trên toàn bộ ứng dụng.

<app-banner-devtools></app-banner-devtools>

#### Module re-exporting

Như đã thấy ở trên, Modules có thể export các providers nội bộ của chúng. Ngoài ra, chúng có thể re-export các modules mà chúng import. Trong ví dụ dưới đây, `CommonModule` vừa được import vào **và** export từ `CoreModule`, làm cho nó có sẵn cho các modules khác import module này.

```typescript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

#### Dependency injection

Một class module có thể **inject** các providers cũng được (ví dụ, cho mục đích cấu hình):

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
@@switch
import { Module, Dependencies } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
@Dependencies(CatsService)
export class CatsModule {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

Tuy nhiên, các class module chính chúng không thể được inject như các providers do circular dependency.

#### Global modules

Nếu bạn phải import cùng một tập hợp các modules ở khắp mọi nơi, nó có thể trở nên tẻ nhạt. Khác với Nest, các `providers` của Angular được đăng ký trong phạm vi toàn cầu. Một khi được định nghĩa, chúng có sẵn ở khắp mọi nơi. Nest, tuy nhiên, đóng gói các providers bên trong phạm vi module. Bạn không thể sử dụng các providers của một module ở nơi khác mà không trước tiên import module đóng gói đó.

Khi bạn muốn cung cấp một tập hợp các providers nên có sẵn ở khắp mọi nơi out-of-the-box (ví dụ, helpers, các kết nối database, v.v.), hãy làm cho module **global** với decorator `@Global()`.

```typescript
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

Decorator `@Global()` làm cho module có phạm vi toàn cầu. Global modules nên được đăng ký **chỉ một lần**, thường bởi root hoặc core module. Trong ví dụ trên, provider `CatsService` sẽ có mặt ở khắp mọi nơi, và các modules muốn inject service sẽ không cần import `CatsModule` trong mảng imports của chúng.

> info **Gợi ý** Làm cho mọi thứ toàn cầu không được khuyến nghị như một thực hành thiết kế. Trong khi global modules có thể giúp giảm boilerplate, nói chung tốt hơn là sử dụng mảng `imports` để làm cho API của một module có sẵn cho các modules khác theo một cách được kiểm soát và rõ ràng. Cách tiếp cận này cung cấp cấu trúc và khả năng bảo trì tốt hơn, đảm bảo rằng chỉ các phần cần thiết của module được chia sẻ với những người khác trong khi tránh kết nối không cần thiết giữa các phần không liên quan của ứng dụng.

#### Dynamic modules

Dynamic modules trong Nest cho phép bạn tạo các modules có thể được cấu hình tại runtime. Điều này đặc biệt hữu ích khi bạn cần cung cấp các modules linh hoạt, có thể tùy chỉnh mà các providers có thể được tạo dựa trên một số tùy chọn hoặc cấu hình. Dưới đây là tổng quan ngắn gọn về cách **dynamic modules** hoạt động.

```typescript
@@filename()
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
@@switch
import { Module } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options) {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

> info **Gợi ý** Method `forRoot()` có thể trả về một dynamic module đồng bộ hoặc không đồng bộ (tức là, qua một `Promise`).

Module này định nghĩa provider `Connection` theo mặc định (trong metadata decorator `@Module()`), nhưng thêm vào đó - tùy thuộc vào các đối tượng `entities` và `options` được truyền vào method `forRoot()` - expose một tập hợp các providers, ví dụ, repositories. Lưu ý rằng các thuộc tính được trả về bởi dynamic module **mở rộng** (thay vì ghi đè) metadata module cơ bản được định nghĩa trong decorator `@Module()`. Đó là cách cả provider `Connection` được khai báo tĩnh **và** các providers repository được tạo động được export từ module.

Nếu bạn muốn đăng ký một dynamic module trong phạm vi toàn cầu, đặt thuộc tính `global` thành `true`.

```typescript
{
  global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

> warning **Cảnh báo** Như đã đề cập ở trên, làm cho mọi thứ toàn cầu **không phải là một quyết định thiết kế tốt**.

`DatabaseModule` có thể được import và cấu hình theo cách sau:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

Nếu bạn muốn re-export một dynamic module, bạn có thể bỏ qua cuộc gọi method `forRoot()` trong mảng exports:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```

Chương Dynamic modules bao gồm chủ đề này chi tiết hơn, và bao gồm một ví dụ hoạt động.

> info **Gợi ý** Tìm hiểu cách xây dựng các dynamic modules có thể tùy chỉnh cao với việc sử dụng `ConfigurableModuleBuilder` ở đây trong chương này.