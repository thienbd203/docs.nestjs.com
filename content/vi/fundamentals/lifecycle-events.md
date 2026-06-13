### Lifecycle Events

Một ứng dụng Nest, cũng như mọi thành phần ứng dụng, có một lifecycle được quản lý bởi Nest. Nest cung cấp **lifecycle hooks** cho phép nhìn thấy các sự kiện lifecycle quan trọng, và khả năng thực hiện hành động (chạy code đã đăng ký trên các module, providers hoặc controllers của bạn) khi chúng xảy ra.

#### Lifecycle sequence

Sơ đồ sau đây mô tả chuỗi các sự kiện lifecycle quan trọng của ứng dụng, từ thời điểm ứng dụng được bootstrapped cho đến khi process node thoát. Chúng ta có thể chia toàn bộ lifecycle thành ba giai đoạn: **initializing** (khởi tạo), **running** (đang chạy) và **terminating** (đang kết thúc). Sử dụng lifecycle này, bạn có thể lập kế hoạch cho việc khởi tạo module và service phù hợp, quản lý các kết nối hoạt động, và tắt ứng dụng một cách graceful khi nó nhận được tín hiệu kết thúc.

<figure><img class="illustrative-image" src="/assets/lifecycle-events.png" /></figure>

#### Lifecycle events

Lifecycle events xảy ra trong quá trình bootstrapping và tắt ứng dụng. Nest gọi các phương thức lifecycle hook đã đăng ký trên module, providers và controllers tại mỗi sự kiện lifecycle sau (**shutdown hooks** cần được bật trước, như mô tả [bên dưới](https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown)). Như được hiển thị trong sơ đồ trên, Nest cũng gọi các phương thức cơ bản thích hợp để bắt đầu lắng nghe kết nối, và để ngừng lắng nghe kết nối.

Trong bảng sau, `onModuleInit` và `onApplicationBootstrap` chỉ được kích hoạt nếu bạn gọi rõ ràng `app.init()` hoặc `app.listen()`.

Trong bảng sau, `onModuleDestroy`, `beforeApplicationShutdown` và `onApplicationShutdown` chỉ được kích hoạt nếu bạn gọi rõ ràng `app.close()` hoặc nếu process nhận được tín hiệu hệ thống đặc biệt (như SIGTERM) và bạn đã gọi đúng `enableShutdownHooks` tại bootstrap ứng dụng (xem phần **Application shutdown** bên dưới).

|| Lifecycle hook method           | Lifecycle event kích hoạt lời gọi hook method                                                                                                                                                                   |
|| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|| `onModuleInit()`                | Được gọi sau khi các dependency của module chủ đã được giải quyết.                                                                                                                                                     |
|| `onApplicationBootstrap()`      | Được gọi sau khi tất cả các module đã được khởi tạo, nhưng trước khi lắng nghe kết nối.                                                                                                                                   |
|| `onModuleDestroy()`\*           | Được gọi sau khi nhận được tín hiệu kết thúc (ví dụ, `SIGTERM`).                                                                                                                                                          |
|| `beforeApplicationShutdown()`\* | Được gọi sau khi tất cả các handler `onModuleDestroy()` đã hoàn thành (Promises được resolve hoặc reject);<br />sau khi hoàn thành (Promises được resolve hoặc reject), tất cả các kết nối hiện có sẽ bị đóng (`app.close()` được gọi). |
|| `onApplicationShutdown()`\*     | Được gọi sau khi kết nối đóng (`app.close()` được resolve).                                                                                                                                                              |

|\* Đối với các sự kiện này, nếu bạn không gọi `app.close()` một cách rõ ràng, bạn phải opt-in để chúng hoạt động với các tín hiệu hệ thống như `SIGTERM`. Xem [Application shutdown](fundamentals/lifecycle-events#application-shutdown) bên dưới.

> warning **Warning** Các lifecycle hooks được liệt kê ở trên không được kích hoạt cho các class **request-scoped**. Class request-scoped không bị ràng buộc với lifecycle của ứng dụng và lifespan của chúng không thể dự đoán được. Chúng chỉ được tạo cho mỗi request và tự động được garbage-collected sau khi response được gửi đi.

> info **Hint** Thứ tự thực thi của `onModuleInit()` và `onApplicationBootstrap()` phụ thuộc trực tiếp vào thứ tự imports của module, chờ hook trước đó.

#### Usage

Mỗi lifecycle hook được đại diện bởi một interface. Interface về mặt kỹ thuật là tùy chọn vì chúng không tồn tại sau khi biên dịch TypeScript. Tuy nhiên, đó là thực hành tốt để sử dụng chúng để hưởng lợi từ strong typing và công cụ editor. Để đăng ký một lifecycle hook, implement interface thích hợp. Ví dụ, để đăng ký một phương thức được gọi trong quá trình khởi tạo module trên một class cụ thể (ví dụ, Controller, Provider hoặc Module), implement interface `OnModuleInit` bằng cách cung cấp một phương thức `onModuleInit()`, như được hiển thị bên dưới:

```typescript
@@filename()
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class UsersService implements OnModuleInit {
  onModuleInit() {
    console.log(`The module has been initialized.`);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  onModuleInit() {
    console.log(`The module has been initialized.`);
  }
}
```

#### Asynchronous initialization

Cả hai hook `OnModuleInit` và `OnApplicationBootstrap` cho phép bạn trì hoãn quá trình khởi tạo ứng dụng (trả về một `Promise` hoặc đánh dấu phương thức là `async` và `await` việc hoàn thành của một phương thức async trong phần thân phương thức).

```typescript
@@filename()
async onModuleInit(): Promise<void> {
  await this.fetch();
}
@@switch
async onModuleInit() {
  await this.fetch();
}
```

#### Application shutdown

Các hook `onModuleDestroy()`, `beforeApplicationShutdown()` và `onApplicationShutdown()` được gọi trong giai đoạn terminating (để phản hồi việc gọi rõ ràng `app.close()` hoặc khi nhận được tín hiệu hệ thống như SIGTERM nếu opt-in). Tính năng này thường được sử dụng với [Kubernetes](https://kubernetes.io/) để quản lý lifecycles của container, bởi [Heroku](https://www.heroku.com/) cho dynos hoặc các dịch vụ tương tự.

Shutdown hook listeners tiêu thụ tài nguyên hệ thống, vì vậy chúng bị tắt theo mặc định. Để sử dụng shutdown hooks, bạn **phải bật listeners** bằng cách gọi `enableShutdownHooks()`:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Starts listening for shutdown hooks
  app.enableShutdownHooks();

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> warning **warning** Do các hạn chế nền tảng vốn có, NestJS có hỗ trợ hạn chế cho application shutdown hooks trên Windows. Bạn có thể mong đợi `SIGINT` hoạt động, cũng như `SIGBREAK` và ở một mức độ nào đó `SIGHUP` - [đọc thêm](https://nodejs.org/api/process.html#process_signal_events). Tuy nhiên `SIGTERM` sẽ không bao giờ hoạt động trên Windows vì việc giết một process trong task manager là không điều kiện, "tức là không có cách nào để ứng dụng phát hiện hoặc ngăn chặn nó". Dưới đây là một số [tài liệu liên quan](https://docs.libuv.org/en/v1.x/signal.html) từ libuv để tìm hiểu thêm về cách `SIGINT`, `SIGBREAK` và các tín hiệu khác được xử lý trên Windows. Ngoài ra, xem tài liệu Node.js về [Process Signal Events](https://nodejs.org/api/process.html#process_signal_events)

> info **Info** `enableShutdownHooks` tiêu thụ bộ nhớ bằng cách bắt đầu listeners. Trong trường hợp bạn chạy nhiều ứng dụng Nest trong một process Node duy nhất (ví dụ, khi chạy các test song song với Jest), Node có thể phàn nàn về quá nhiều listener processes. Vì lý do này, `enableShutdownHooks` không được bật theo mặc định. Hãy aware về điều kiện này khi bạn chạy nhiều instances trong một process Node duy nhất.

Khi ứng dụng nhận được tín hiệu kết thúc, nó sẽ gọi bất kỳ phương thức `onModuleDestroy()`, `beforeApplicationShutdown()`, sau đó `onApplicationShutdown()` nào đã đăng ký (theo chuỗi được mô tả ở trên) với tín hiệu tương ứng làm tham số đầu tiên. Nếu một hàm đã đăng ký await một lời gọi async (trả về một promise), Nest sẽ không tiếp tục trong chuỗi cho đến khi promise được resolve hoặc reject.

```typescript
@@filename()
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal: string) {
    console.log(signal); // e.g. "SIGINT"
  }
}
@@switch
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal) {
    console.log(signal); // e.g. "SIGINT"
  }
}
```

> info **Info** Việc gọi `app.close()` không kết thúc process Node nhưng chỉ kích hoạt các hook `onModuleDestroy()` và `onApplicationShutdown()`, vì vậy nếu có một số intervals, các background tasks chạy lâu, v.v., process sẽ không được tự động kết thúc.