### Lifecycle Events

Một ứng dụng Nest, cũng như mọi phần tử ứng dụng, có một lifecycle được quản lý bởi Nest. Nest cung cấp **lifecycle hooks** mang lại khả năng nhìn thấy các sự kiện lifecycle chính, và khả năng hành động (chạy code đã đăng ký trên modules, providers hoặc controllers) khi chúng xảy ra.

#### Lifecycle sequence

Sơ đồ sau đây mô tả chuỗi các sự kiện lifecycle chính của ứng dụng, từ thời điểm ứng dụng được bootstrapped cho đến khi node process exits. Chúng ta có thể chia tổng thể lifecycle thành ba giai đoạn: **initializing**, **running** và **terminating**. Sử dụng lifecycle này, bạn có thể lập kế hoạch cho việc khởi tạo modules và services thích hợp, quản lý các kết nối hoạt động, và gracefully shutdown ứng dụng của bạn khi nó nhận được một termination signal.

<figure><img class="illustrative-image" src="/assets/lifecycle-events.png" /></figure>

#### Lifecycle events

Các sự kiện lifecycle xảy ra trong quá trình application bootstrapping và shutdown. Nest gọi các methods lifecycle hook đã đăng ký trên modules, providers và controllers tại mỗi sự kiện lifecycle sau (**shutdown hooks** cần được kích hoạt trước, như được mô tả ở dưới). Như được hiển thị trong sơ đồ trên, Nest cũng gọi các methods underlying thích hợp để bắt đầu lắng nghe các kết nối, và để dừng lắng nghe các kết nối.

Trong bảng sau, `onModuleInit` và `onApplicationBootstrap` chỉ được kích hoạt nếu bạn rõ ràng gọi `app.init()` hoặc `app.listen()`.

Trong bảng sau, `onModuleDestroy`, `beforeApplicationShutdown` và `onApplicationShutdown` chỉ được kích hoạt nếu bạn rõ ràng gọi `app.close()` hoặc nếu process nhận được một system signal đặc biệt (như SIGTERM) và bạn đã đúng cách gọi `enableShutdownHooks` tại application bootstrap (xem phần **Application shutdown** dưới đây).

|| Lifecycle hook method           | Lifecycle event triggering the hook method call                                                                                                                                                                   |
|| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|| `onModuleInit()`                | Called once the host module's dependencies have been resolved.                                                                                                                                                    |
|| `onApplicationBootstrap()`      | Called once all modules have been initialized, but before listening for connections.                                                                                                                              |
|| `onModuleDestroy()`\*           | Called after a termination signal (e.g., `SIGTERM`) has been received.                                                                                                                                            |
|| `beforeApplicationShutdown()`\* | Called after all `onModuleDestroy()` handlers have completed (Promises resolved or rejected);<br />once complete (Promises resolved or rejected), all existing connections will be closed (`app.close()` called). |
|| `onApplicationShutdown()`\*     | Called after connections close (`app.close()` resolves).                                                                                                                                                          |

\* For these events, if you're not calling `app.close()` explicitly, you must opt-in to make them work with system signals such as `SIGTERM`. See [Application shutdown](fundamentals/lifecycle-events#application-shutdown) below.

> warning **Cảnh báo** Các lifecycle hooks được liệt kê ở trên không được kích hoạt cho các classes **request-scoped**. Các classes request-scoped không được gắn với application lifecycle và lifespan của chúng là không thể dự đoán. Chúng được tạo ra độc quyền cho mỗi request và tự động được garbage-collected sau khi response được gửi.

> info **Gợi ý** Thực thi của `onModuleInit()` và `onApplicationBootstrap()` trực tiếp phụ thuộc vào thứ tự của module imports, awaiting hook trước đó.

#### Usage

Mỗi lifecycle hook được đại diện bởi một interface. Các interfaces về mặt kỹ thuật là tùy chọn vì chúng không tồn tại sau khi biên dịch TypeScript. Tuy nhiên, đó là thực hành tốt để sử dụng chúng để hưởng lợi từ strong typing và editor tooling. Để đăng ký một lifecycle hook, implement interface thích hợp. Ví dụ, để đăng ký một method được gọi trong quá trình module initialization trên một class cụ thể (ví dụ, Controller, Provider hoặc Module), implement interface `OnModuleInit` bằng cách cung cấp một method `onModuleInit()`, như được hiển thị dưới đây:

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

Cả hai hooks `OnModuleInit` và `OnApplicationBootstrap` cho phép bạn defer quá trình application initialization (trả về một `Promise` hoặc đánh dấu method như `async` và `await` một asynchronous method completion trong body của method).

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

Các hooks `onModuleDestroy()`, `beforeApplicationShutdown()` và `onApplicationShutdown()` được gọi trong terminating phase (để đáp ứng một cuộc gọi rõ ràng đến `app.close()` hoặc khi nhận được system signals như SIGTERM nếu opt-in). Tính năng này thường được sử dụng với [Kubernetes](https://kubernetes.io/) để quản lý lifecycles của containers, bởi [Heroku](https://www.heroku.com/) cho dynos hoặc các dịch vụ tương tự.

Shutdown hook listeners tiêu thụ system resources, vì vậy chúng bị vô hiệu hóa theo mặc định. Để sử dụng shutdown hooks, bạn **bắt buộc kích hoạt listeners** bằng cách gọi `enableShutdownHooks()`:

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

> warning **Cảnh báo** Do các hạn chế vốn có của platform, NestJS có hỗ trợ hạn chế cho application shutdown hooks trên Windows. Bạn có thể mong đợi `SIGINT` hoạt động, cũng như `SIGBREAK` và ở một mức độ nào đó `SIGHUP` - đọc thêm. Tuy nhiên `SIGTERM` sẽ không bao giờ hoạt động trên Windows vì việc giết một process trong task manager là vô điều kiện, "nghĩa là không có cách nào cho một ứng dụng để phát hiện hoặc ngăn chặn nó". Đây là một số tài liệu liên quan từ libuv để tìm hiểu thêm về cách `SIGINT`, `SIGBREAK` và các cái khác được xử lý trên Windows. Ngoài ra, xem tài liệu Node.js của Process Signal Events

> info **Thông tin** `enableShutdownHooks` tiêu thụ bộ nhớ bằng cách bắt đầu listeners. Trong trường hợp bạn đang chạy nhiều ứng dụng Nest trong một Node process đơn lẻ (ví dụ, khi chạy các bài kiểm tra song song với Jest), Node có thể phàn nàn về các quá trình listener quá mức. Vì lý do này, `enableShutdownHooks` không được kích hoạt theo mặc định. Hãy lưu ý điều kiện này khi bạn đang chạy nhiều instances trong một Node process đơn lẻ.

Khi ứng dụng nhận được một termination signal nó sẽ gọi bất kỳ methods `onModuleDestroy()`, `beforeApplicationShutdown()`, sau đó `onApplicationShutdown()` đã đăng ký (trong chuỗi được mô tả ở trên) với signal tương ứng như tham số đầu tiên. Nếu một function đã đăng ký await một asynchronous call (trả về một promise), Nest sẽ không tiếp tục trong chuỗi cho đến khi promise được resolved hoặc rejected.

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

> info **Thông tin** Gọi `app.close()` không kết thúc Node process nhưng chỉ kích hoạt các hooks `onModuleDestroy()` và `onApplicationShutdown()`, vì vậy nếu có một số intervals, các background tasks chạy dài, v.v. process sẽ không tự động được kết thúc.