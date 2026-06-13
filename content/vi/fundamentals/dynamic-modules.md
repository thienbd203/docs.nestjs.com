### Dynamic modules

[Chương Modules](/modules) bao gồm các cơ bản của Nest modules, và bao gồm một giới thiệu ngắn về [dynamic modules](https://docs.nestjs.com/modules#dynamic-modules). Chương này mở rộng chủ đề của dynamic modules. Khi hoàn thành, bạn nên có sự hiểu biết tốt về chúng là gì và cách và khi nào sử dụng chúng.

#### Introduction

Hầu hết các ví dụ code ứng dụng trong phần **Overview** của tài liệu sử dụng các module thông thường, hoặc static. Modules định nghĩa các nhóm các components như [providers](/providers) và [controllers](/controllers) phù hợp với nhau như một phần mô-đun của một ứng dụng tổng thể. Chúng cung cấp một ngữ cảnh thực thi, hoặc scope, cho các components này. Ví dụ, các providers được định nghĩa trong một module hiển thị với các thành viên khác của module mà không cần export chúng. Khi một provider cần hiển thị bên ngoài một module, nó trước tiên được export từ module chủ của nó, và sau đó được import vào module tiêu thụ của nó.

Hãy đi qua một ví dụ quen thuộc.

Đầu tiên, chúng ta sẽ định nghĩa một `UsersModule` để cung cấp và export một `UsersService`. `UsersModule` là module **chủ** cho `UsersService`.

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

Tiếp theo, chúng ta sẽ định nghĩa một `AuthModule`, import `UsersModule`, làm cho các providers được export của `UsersModule` có sẵn bên trong `AuthModule`:

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

Các cấu trúc này cho phép chúng ta inject `UsersService` vào, ví dụ, `AuthService` được lưu trữ trong `AuthModule`:

```typescript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  /*
    Implementation that makes use of this.usersService
  */
}
```

Chúng ta sẽ tham chiếu điều này như là ràng buộc module **static**. Tất cả thông tin Nest cần để wire cùng nhau các module đã được khai báo trong module chủ và module tiêu thụ. Hãy giải những gì đang xảy ra trong quá trình này. Nest làm cho `UsersService` có sẵn bên trong `AuthModule` bằng cách:

1. Instantiating `UsersModule`, bao gồm transitively importing các module khác mà `UsersModule` chính nó tiêu thụ, và transitively resolving bất kỳ dependencies nào (xem [Custom providers](https://docs.nestjs.com/fundamentals/custom-providers)).
2. Instantiating `AuthModule`, và làm cho các providers được export của `UsersModule` có sẵn cho các components trong `AuthModule` (giống như thể chúng đã được khai báo trong `AuthModule`).
3. Injecting một instance của `UsersService` vào `AuthService`.

#### Dynamic module use case

Với ràng buộc module tĩnh, không có cơ hội cho module tiêu thụ để **ảnh hưởng** cách các providers từ module chủ được cấu hình. Tại sao điều này quan trọng? Xem xét trường hợp chúng ta có một module mục đích chung cần hoạt động khác nhau trong các use case khác nhau. Điều này tương tự như khái niệm của một "plugin" trong nhiều hệ thống, trong đó một cơ sở chung yêu cầu một số cấu hình trước khi nó có thể được sử dụng bởi một tiêu thụ.

Một ví dụ tốt với Nest là một **configuration module**. Nhiều ứng dụng thấy hữu ích để externalize các chi tiết cấu hình bằng cách sử dụng một module cấu hình. Điều này giúp dễ dàng thay đổi động các cài đặt ứng dụng trong các triển khai khác nhau: ví dụ, một database phát triển cho các nhà phát triển, một database staging cho môi trường staging/test, v.v. Bằng cách ủy quyền quản lý các tham số cấu hình cho một module cấu hình, mã nguồn ứng dụng vẫn độc lập với các tham số cấu hình.

Thách thức là module cấu hình chính nó, vì nó là generic (tương tự như một "plugin"), cần được tùy chỉnh bởi module tiêu thụ của nó. Đây là lúc _dynamic modules_ phát huy tác dụng. Sử dụng các tính năng dynamic module, chúng ta có thể làm cho module cấu hình của chúng ta **dynamic** để module tiêu thụ có thể sử dụng một API để kiểm soát cách module cấu hình được tùy chỉnh tại thời điểm nó được import.

Nói cách khác, dynamic modules cung cấp một API để import một module vào một module khác, và tùy chỉnh các thuộc tính và hành vi của module đó khi nó được import, trái ngược với việc sử dụng các ràng buộc tĩnh mà chúng ta đã thấy cho đến nay.

<app-banner-devtools></app-banner-devtools>

#### Config module example

Chúng tôi sẽ sử dụng phiên bản cơ bản của ví dụ code từ [chương cấu hình](https://docs.nestjs.com/techniques/configuration#service) cho phần này. Phiên bản hoàn chỉnh như của cuối chương này có sẵn như một ví dụ hoạt động [tại đây](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules).

Yêu cầu của chúng ta là làm cho `ConfigModule` chấp nhận một object `options` để tùy chỉnh nó. Đây là tính năng chúng ta muốn hỗ trợ. Mẫu cơ bản hard-code vị trí của file `.env` để nằm trong thư mục gốc dự án. Hãy giả sử chúng ta muốn làm cho điều đó có thể cấu hình, để bạn có thể quản lý các file `.env` của mình trong bất kỳ thư mục nào bạn chọn. Ví dụ, hãy tưởng tượng bạn muốn lưu trữ các file `.env` khác nhau của mình trong một thư mục dưới thư mục gốc dự án gọi là `config` (tức là một thư mục anh em với `src`). Bạn muốn có thể chọn các thư mục khác nhau khi sử dụng `ConfigModule` trong các dự án khác nhau.

Dynamic modules cho chúng ta khả năng truyền tham số vào module đang được import để chúng ta có thể thay đổi hành vi của nó. Hãy xem điều này hoạt động như thế nào. Nó hữu ích nếu chúng ta bắt đầu từ mục tiêu cuối cùng về cách điều này có thể trông từ góc nhìn của module tiêu thụ, và sau đó làm việc ngược lại. Đầu tiên, hãy nhanh chóng xem lại ví dụ về việc _tĩnh_ import `ConfigModule` (tức là một cách tiếp cận không có khả năng ảnh hưởng đến hành vi của module được import). Chú ý kỹ đến mảng `imports` trong decorator `@Module()`:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Hãy xem xét một import module _dynamic_, nơi chúng ta đang truyền vào một object cấu hình, có thể trông như thế nào. So sánh sự khác biệt trong mảng `imports` giữa hai ví dụ này:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Hãy xem điều gì đang xảy ra trong ví dụ dynamic ở trên. Các thành phần chuyển động là gì?

1. `ConfigModule` là một class thông thường, vì vậy chúng ta có thể suy luận rằng nó phải có một **phương thức static** gọi là `register()`. Chúng ta biết nó là static vì chúng ta đang gọi nó trên class `ConfigModule`, không phải trên một **instance** của class. Lưu ý: phương thức này, mà chúng ta sẽ tạo sớm, có thể có bất kỳ tên tùy ý nào, nhưng theo quy ước chúng ta nên gọi nó là `forRoot()` hoặc `register()`.
2. Phương thức `register()` được định nghĩa bởi chúng ta, vì vậy chúng ta có thể chấp nhận bất kỳ đối số đầu vào nào chúng ta thích. Trong trường hợp này, chúng ta sẽ chấp nhận một object `options` đơn giản với các thuộc tính phù hợp, là trường hợp điển hình.
3. Chúng ta có thể suy luận rằng phương thức `register()` phải trả về một cái gì đó như một `module` vì giá trị trả về của nó xuất hiện trong danh sách `imports` quen thuộc, mà chúng ta đã thấy cho đến nay bao gồm một danh sách các module.

Thực tế, những gì phương thức `register()` của chúng ta sẽ trả về là một `DynamicModule`. Một dynamic module không hơn gì một module được tạo tại run-time, với các thuộc tính chính xác giống như một module static, cộng với một thuộc tính bổ sung gọi là `module`. Hãy nhanh chóng xem lại một khai báo module static mẫu, chú ý kỹ đến các options module được truyền vào decorator:

```typescript
@Module({
  imports: [DogsModule],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
```

Dynamic modules phải trả về một object với interface chính xác giống nhau, cộng với một thuộc tính bổ sung gọi là `module`. Thuộc tính `module` đóng vai trò như tên của module, và nên giống với tên class của module, như được hiển thị trong ví dụ dưới đây.

> info **Hint** Đối với một dynamic module, tất cả các thuộc tính của object options module là tùy chọn **ngoại trừ** `module`.

Vậy về phương thức static `register()`? Bây giờ chúng ta có thể thấy rằng công việc của nó là trả về một object có interface `DynamicModule`. Khi chúng ta gọi nó, chúng ta thực sự đang cung cấp một module cho danh sách `imports`, tương tự như cách chúng ta sẽ làm như vậy trong trường hợp tĩnh bằng cách liệt kê một tên class module. Nói cách khác, API dynamic module đơn giản trả về một module, nhưng thay vì cố định các thuộc tính trong decorator `@Module`, chúng ta chỉ định chúng theo chương trình.

Vẫn có một vài chi tiết để bao gồm để giúp làm cho bức tranh hoàn chỉnh:

1. Bây giờ chúng ta có thể nói rằng thuộc tính `imports` của decorator `@Module()` có thể nhận không chỉ một tên class module (ví dụ, `imports: [UsersModule]`), mà cũng là một hàm **trả về** một dynamic module (ví dụ, `imports: [ConfigModule.register(...)]`).
2. Một dynamic module có thể tự import các module khác. Chúng tôi sẽ không làm như vậy trong ví dụ này, nhưng nếu dynamic module phụ thuộc vào các providers từ các module khác, bạn sẽ import chúng sử dụng thuộc tính `imports` tùy chọn. Một lần nữa, điều này hoàn toàn tương tự như cách bạn sẽ khai báo metadata cho một module static sử dụng decorator `@Module()`.

Được trang bị sự hiểu biết này, bây giờ chúng ta có thể xem việc khai báo `ConfigModule` dynamic của chúng ta phải trông như thế nào. Hãy thử một chút.

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(): DynamicModule {
    return {
      module: ConfigModule,
      providers: [ConfigService],
      exports: [ConfigService],
    };
  }
}
```

Bây giờ nên rõ ràng cách các phần kết nối với nhau. Gọi `ConfigModule.register(...)` trả về một object `DynamicModule` với các thuộc tính về cơ bản giống như những thuộc tính mà, cho đến nay, chúng tôi đã cung cấp như metadata thông qua decorator `@Module()`.

> info **Hint** Import `DynamicModule` từ `@nestjs/common`.

Module dynamic của chúng ta chưa thú vị lắm, tuy nhiên, vì chúng tôi chưa giới thiệu bất kỳ khả năng nào để **cấu hình** nó như chúng tôi đã nói chúng ta muốn làm. Hãy giải quyết điều đó tiếp theo.

#### Module configuration

Giải pháp rõ ràng để tùy chỉnh hành vi của `ConfigModule` là truyền cho nó một object `options` trong phương thức static `register()`, như chúng tôi đã đoán ở trên. Hãy xem lại thuộc tính `imports` của module tiêu thụ của chúng ta:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Điều đó xử lý việc truyền một object `options` vào module dynamic của chúng ta một cách đẹp mắt. Làm thế nào chúng ta sau đó sử dụng object `options` đó trong `ConfigModule`? Hãy xem xét điều đó một chút. Chúng ta biết rằng `ConfigModule` của chúng ta về cơ bản là một chủ để cung cấp và export một service injectable - `ConfigService` - để sử dụng bởi các providers khác. Thực tế là `ConfigService` của chúng ta cần đọc object `options` để tùy chỉnh hành vi của nó. Hãy giả sử trong giây lát rằng chúng ta biết cách lấy `options` từ phương thức `register()` vào `ConfigService`. Với giả định đó, chúng ta có thể thực hiện một vài thay đổi cho service để tùy chỉnh hành vi của nó dựa trên các thuộc tính từ object `options`. (**Lưu ý**: trong thời điểm hiện tại, vì chúng tôi _chưa_ thực sự xác định cách truyền nó, chúng ta chỉ hard-code `options`. Chúng tôi sẽ sửa điều này trong một phút).

```typescript
import { Injectable } from '@nestjs/common';
import * as fs from 'node:fs';
import * as path from 'node:path';
import * as dotenv from 'dotenv';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const options = { folder: './config' };

    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

Bây giờ `ConfigService` của chúng ta biết cách tìm file `.env` trong thư mục chúng ta đã chỉ định trong `options`.

Nhiệm vụ còn lại của chúng ta là bằng cách nào inject object `options` từ bước `register()` vào `ConfigService` của chúng ta. Và tất nhiên, chúng ta sẽ sử dụng _dependency injection_ để làm điều đó. Đây là một điểm chính, vì vậy hãy đảm bảo bạn hiểu nó. `ConfigModule` của chúng ta đang cung cấp `ConfigService`. `ConfigService` lần lượt phụ thuộc vào object `options` chỉ được cung cấp tại run-time. Vì vậy, tại run-time, chúng ta sẽ cần trước tiên bind object `options` vào Nest IoC container, và sau đó có Nest inject nó vào `ConfigService` của chúng ta. Hãy nhớ từ chương **Custom providers** rằng các providers có thể [bao gồm bất kỳ giá trị nào](https://docs.nestjs.com/fundamentals/custom-providers#non-service-based-providers) không chỉ các services, vì vậy chúng ta ổn khi sử dụng dependency injection để xử lý một object `options` đơn giản.

Hãy giải quyết việc bind object options vào IoC container trước. Chúng ta làm điều này trong phương thức static `register()` của chúng ta. Hãy nhớ rằng chúng ta đang xây dựng động một module, và một trong các thuộc tính của một module là danh sách các providers của nó. Vì vậy những gì chúng ta cần làm là định nghĩa object options của chúng ta như một provider. Điều này sẽ làm cho nó có thể inject vào `ConfigService`, mà chúng ta sẽ tận dụng trong bước tiếp theo. Trong code dưới đây, chú ý đến mảng `providers`:

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(options: Record<string, any>): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}
```

Bây giờ chúng ta có thể hoàn thành quá trình bằng cách inject provider `'CONFIG_OPTIONS'` vào `ConfigService`. Nhớ lại rằng khi chúng ta định nghĩa một provider sử dụng một token không phải class chúng ta cần sử dụng decorator `@Inject()` [như được mô tả ở đây](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens).

```typescript
import * as fs from 'node:fs';
import * as path from 'node:path';
import * as dotenv from 'dotenv';
import { Injectable, Inject } from '@nestjs/common';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor(@Inject('CONFIG_OPTIONS') private options: Record<string, any>) {
    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

Một lưu ý cuối cùng: để đơn giản chúng ta đã sử dụng một token injection dựa trên chuỗi (`'CONFIG_OPTIONS'`) ở trên, nhưng thực hành tốt nhất là định nghĩa nó như một hằng số (hoặc `Symbol`) trong một file riêng biệt, và import file đó. Ví dụ:

```typescript
export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';
```

#### Example

Một ví dụ đầy đủ của code trong chương này có thể được tìm thấy [tại đây](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules).

#### Community guidelines

Bạn có thể đã thấy sử dụng cho các phương thức như `forRoot`, `register`, và `forFeature` xung quanh một số package `@nestjs/` và có thể tự hỏi sự khác biệt cho tất cả các phương thức này là gì. Không có quy tắc cứng về điều này, nhưng các package `@nestjs/` cố gắng theo các hướng dẫn này:

Khi tạo một module với:

- `register`, bạn đang mong đợi cấu hình một dynamic module với một cấu hình cụ thể để sử dụng chỉ bởi module gọi. Ví dụ, với `@nestjs/axios` của Nest: `HttpModule.register({{ '{' }} baseUrl: 'someUrl' {{ '}' }})`. Nếu, trong một module khác bạn sử dụng `HttpModule.register({{ '{' }} baseUrl: 'somewhere else' {{ '}' }})`, nó sẽ có cấu hình khác. Bạn có thể làm điều này cho bao nhiêu module tùy thích.

- `forRoot`, bạn đang mong đợi cấu hình một dynamic module một lần và tái sử dụng cấu hình đó ở nhiều nơi (mặc dù có thể vô thức vì nó được trừu tượng hóa). Đây là lý do bạn có một `GraphQLModule.forRoot()`, một `TypeOrmModule.forRoot()`, v.v.

- `forFeature`, bạn đang mong đợi sử dụng cấu hình của một dynamic module của `forRoot` nhưng cần sửa đổi một số cấu hình cụ thể cho nhu cầu của module gọi (tức là repository nào module này nên có quyền truy cập, hoặc context mà một logger nên sử dụng.)

Tất cả những điều này, thường xuyên, có các đối tác `async` của chúng cũng vậy, `registerAsync`, `forRootAsync`, và `forFeatureAsync`, có nghĩa là điều tương tự, nhưng sử dụng Dependency Injection của Nest cho cấu hình cũng vậy.

#### Configurable module builder

Vì việc tạo thủ công các dynamic modules có cấu hình cao mà expose các phương thức `async` (`registerAsync`, `forRootAsync`, v.v.) khá phức tạp, đặc biệt đối với người mới, Nest expose class `ConfigurableModuleBuilder` giúp tạo điều kiện cho quá trình này và cho phép bạn xây dựng một "blueprint" module chỉ trong vài dòng code.

Ví dụ, hãy lấy ví dụ chúng ta đã sử dụng ở trên (`ConfigModule`) và chuyển đổi nó để sử dụng `ConfigurableModuleBuilder`. Trước khi bắt đầu, hãy đảm bảo chúng ta tạo một interface chuyên dụng đại diện cho những options mà `ConfigModule` của chúng ta nhận vào.

```typescript
export interface ConfigModuleOptions {
  folder: string;
}
```

Với điều này tại chỗ, tạo một file chuyên dụng mới (cùng với file `config.module.ts` hiện có) và đặt tên là `config.module-definition.ts`. Trong file này, hãy sử dụng `ConfigurableModuleBuilder` để xây dựng định nghĩa `ConfigModule`.

```typescript
@@filename(config.module-definition)
import { ConfigurableModuleBuilder } from '@nestjs/common';
import { ConfigModuleOptions } from './interfaces/config-module-options.interface';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
@@switch
import { ConfigurableModuleBuilder } from '@nestjs/common';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().build();
```

Bây giờ hãy mở file `config.module.ts` và sửa đổi implementation của nó để tận dụng `ConfigurableModuleClass` được tạo tự động:

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {}
```

Mở rộng `ConfigurableModuleClass` có nghĩa là `ConfigModule` giờ cung cấp không chỉ phương thức `register` (như trước đó với implementation tùy chỉnh), mà cũng phương thức `registerAsync` cho phép tiêu thụ cấu hình module một cách async, ví dụ, bằng cách cung cấp các async factories:

```typescript
@Module({
  imports: [
    ConfigModule.register({ folder: './config' }),
    // or alternatively:
    // ConfigModule.registerAsync({
    //   useFactory: () => {
    //     return {
      //       folder: './config',
    //     }
    //   },
    //   inject: [...any extra dependencies...]
    // }),
  ],
})
export class AppModule {}
```

Phương thức `registerAsync` nhận object sau làm đối số:

```typescript
{
  /**
   * Injection token resolving to a class that will be instantiated as a provider.
   * The class must implement the corresponding interface.
   */
  useClass?: Type<
    ConfigurableModuleOptionsFactory<ModuleOptions, FactoryClassMethodKey>
  >;
  /**
   * Function returning options (or a Promise resolving to options) to configure the
   * module.
   */
  useFactory?: (...args: any[]) => Promise<ModuleOptions> | ModuleOptions;
  /**
   * Dependencies that a Factory may inject.
   */
  inject?: FactoryProvider['inject'];
  /**
   * Injection token resolving to an existing provider. The provider must implement
   * the corresponding interface.
   */
  useExisting?: Type<
    ConfigurableModuleOptionsFactory<ModuleOptions, FactoryClassMethodKey>
  >;
}
```

Hãy đi qua các thuộc tính ở trên từng cái một:

- `useFactory` - một hàm trả về object cấu hình. Nó có thể là đồng bộ hoặc không đồng bộ. Để inject dependencies vào hàm factory, sử dụng thuộc tính `inject`. Chúng tôi đã sử dụng biến thể này trong ví dụ trên.
- `inject` - một mảng các dependencies sẽ được inject vào hàm factory. Thứ tự của các dependencies phải khớp với thứ tự của các tham số trong hàm factory.
- `useClass` - một class sẽ được instantiate như một provider. Class phải implement interface tương ứng. Thông thường, đây là một class cung cấp một phương thức `create()` trả về object cấu hình. Đọc thêm về điều này trong phần [Custom method key](/fundamentals/dynamic-modules#custom-method-key) bên dưới.
- `useExisting` - một biến thể của `useClass` cho phép bạn sử dụng một provider hiện có thay vì hướng dẫn Nest tạo một instance mới của class. Điều này hữu ích khi bạn muốn sử dụng một provider đã được đăng ký trong module. Hãy nhớ rằng class phải implement cùng interface với interface được sử dụng trong `useClass` (và do đó nó phải cung cấp phương thức `create()`, trừ khi bạn ghi đè tên phương thức mặc định, xem phần [Custom method key](/fundamentals/dynamic-modules#custom-method-key) bên dưới).

Luôn chọn một trong các tùy chọn ở trên (`useFactory`, `useClass`, hoặc `useExisting`), vì chúng loại trừ lẫn nhau.

Cuối cùng, hãy cập nhật class `ConfigService` để inject provider options module được tạo thay vì `'CONFIG_OPTIONS'` mà chúng ta đã sử dụng cho đến nay.

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) { ... }
}
```

#### Custom method key

`ConfigurableModuleClass` theo mặc định cung cấp các phương thức `register` và đối tác của nó `registerAsync`. Để sử dụng một tên phương thức khác, sử dụng phương thức `ConfigurableModuleBuilder#setClassMethodName`, như sau:

```typescript
@@filename(config.module-definition)
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setClassMethodName('forRoot').build();
@@switch
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().setClassMethodName('forRoot').build();
```

Cấu trúc này sẽ hướng dẫn `ConfigurableModuleBuilder` tạo ra một class expose `forRoot` và `forRootAsync` thay thế. Ví dụ:

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ folder: './config' }), // <-- note the use of "forRoot" instead of "register"
    // or alternatively:
    // ConfigModule.forRootAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...any extra dependencies...]
    // }),
  ],
})
export class AppModule {}
```

#### Custom options factory class

Vì phương thức `registerAsync` (hoặc `forRootAsync` hoặc bất kỳ tên nào khác, tùy thuộc vào cấu hình) cho phép tiêu thụ truyền một định nghĩa provider resolve thành cấu hình module, một tiêu thụ thư viện có thể cung cấp một class để được sử dụng để xây dựng object cấu hình.

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory,
    }),
  ],
})
export class AppModule {}
```

Class này, theo mặc định, phải cung cấp phương thức `create()` trả về một object cấu hình module. Tuy nhiên, nếu thư viện của bạn tuân theo một quy ước đặt tên khác, bạn có thể thay đổi hành vi đó và hướng dẫn `ConfigurableModuleBuilder` mong đợi một phương thức khác, ví dụ, `createConfigOptions`, sử dụng phương thức `ConfigurableModuleBuilder#setFactoryMethodName`:

```typescript
@@filename(config.module-definition)
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setFactoryMethodName('createConfigOptions').build();
@@switch
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().setFactoryMethodName('createConfigOptions').build();
```

Bây giờ, class `ConfigModuleOptionsFactory` phải expose phương thức `createConfigOptions` (thay vì `create`):

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory, // <-- this class must provide the "createConfigOptions" method
    }),
  ],
})
export class AppModule {}
```

#### Extra options

Có các trường hợp ngoại lệ khi module của bạn có thể cần nhận các options bổ sung xác định cách nó được cho là hoạt động (một ví dụ tốt của một option như vậy là flag `isGlobal` - hoặc chỉ `global`) mà cùng lúc đó, không nên được bao gồm trong provider `MODULE_OPTIONS_TOKEN` (vì chúng không liên quan đến các services/providers được đăng ký trong module đó, ví dụ, `ConfigService` không cần biết liệu module chủ của nó được đăng ký như một global module hay không).

Trong các trường hợp như vậy, phương thức `ConfigurableModuleBuilder#setExtras` có thể được sử dụng. Xem ví dụ sau:

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>()
    .setExtras(
      {
        isGlobal: true,
      },
      (definition, extras) => ({
        ...definition,
        global: extras.isGlobal,
      }),
    )
    .build();
```

Trong ví dụ trên, đối số đầu tiên được truyền vào phương thức `setExtras` là một object chứa các giá trị mặc định cho các thuộc tính "extra". Đối số thứ hai là một hàm nhận các định nghĩa module được tạo tự động (với `provider`, `exports`, v.v.) và object `extras` đại diện cho các thuộc tính bổ sung (hoặc được chỉ định bởi tiêu thụ hoặc mặc định). Giá trị trả về của hàm này là một định nghĩa module đã sửa đổi. Trong ví dụ cụ thể này, chúng ta lấy thuộc tính `extras.isGlobal` và gán nó cho thuộc tính `global` của định nghĩa module (mà lần lượt xác định liệu một module có global hay không, đọc thêm [tại đây](/modules#dynamic-modules)).

Bây giờ khi tiêu thụ module này, flag bổ sung `isGlobal` có thể được truyền vào, như sau:

```typescript
@Module({
  imports: [
    ConfigModule.register({
      isGlobal: true,
      folder: './config',
    }),
  ],
})
export class AppModule {}
```

Tuy nhiên, vì `isGlobal` được khai báo như một thuộc tính "extra", nó sẽ không có sẵn trong provider `MODULE_OPTIONS_TOKEN`:

```typescript
@Injectable()
export class ConfigService {
  constructor(
    @Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions,
  ) {
    // "options" object will not have the "isGlobal" property
    // ...
  }
}
```

#### Extending auto-generated methods

Các phương thức static được tạo tự động (`register`, `registerAsync`, v.v.) có thể được mở rộng nếu cần, như sau:

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import {
  ConfigurableModuleClass,
  ASYNC_OPTIONS_TYPE,
  OPTIONS_TYPE,
} from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {
  static register(options: typeof OPTIONS_TYPE): DynamicModule {
    return {
      // your custom logic here
      ...super.register(options),
    };
  }

  static registerAsync(options: typeof ASYNC_OPTIONS_TYPE): DynamicModule {
    return {
      // your custom logic here
      ...super.registerAsync(options),
    };
  }
}
```

Lưu ý việc sử dụng các loại `OPTIONS_TYPE` và `ASYNC_OPTIONS_TYPE` phải được export từ file định nghĩa module:

```typescript
export const {
  ConfigurableModuleClass,
  MODULE_OPTIONS_TOKEN,
  OPTIONS_TYPE,
  ASYNC_OPTIONS_TYPE,
} = new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
```