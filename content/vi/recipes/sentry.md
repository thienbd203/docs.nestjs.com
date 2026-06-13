### Sentry

[Sentry](https://sentry.io) là một nền tảng theo dõi lỗi và giám sát hiệu suất giúp các nhà phát triển xác định và sửa các vấn đề trong thời gian thực. Công thức này cho thấy cách tích hợp [NestJS SDK](https://docs.sentry.io/platforms/javascript/guides/nestjs/) của Sentry với ứng dụng NestJS của bạn.

#### Cài đặt

Trước tiên, cài đặt các phụ thuộc cần thiết:

```bash
$ npm install --save @sentry/nestjs @sentry/profiling-node
```

> info **Gợi ý** `@sentry/profiling-node` là tùy chọn, nhưng được khuyến nghị cho hiệu suất profiling.

#### Thiết lập cơ bản

Để bắt đầu với Sentry, bạn sẽ cần tạo một file tên là `instrument.ts` nên được nhập trước bất kỳ module nào khác trong ứng dụng của bạn:

```typescript
@@filename(instrument)
const Sentry = require("@sentry/nestjs");
const { nodeProfilingIntegration } = require("@sentry/profiling-node");

// Đảm bảo gọi điều này trước khi yêu cầu bất kỳ module nào khác!
Sentry.init({
  dsn: SENTRY_DSN,
  integrations: [
    // Thêm integration Profiling của chúng tôi
    nodeProfilingIntegration(),
  ],

  // Thêm Tracing bằng cách đặt tracesSampleRate
  // Chúng tôi khuyến nghị điều chỉnh giá trị này trong production
  tracesSampleRate: 1.0,

  // Đặt tốc độ lấy mẫu cho profiling
  // Điều này liên quan đến tracesSampleRate
  profilesSampleRate: 1.0,
});
```

Cập nhật file `main.ts` của bạn để nhập `instrument.ts` trước các nhập khác:

```typescript
@@filename(main)
// Nhập cái này trước!
import "./instrument";

// Bây giờ nhập các module khác
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}

bootstrap();
```

Sau đó, thêm `SentryModule` như một module gốc vào module chính của bạn:

```typescript
@@filename(app.module)
import { Module } from "@nestjs/common";
import { SentryModule } from "@sentry/nestjs/setup";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";

@Module({
  imports: [
    SentryModule.forRoot(),
    // ...các module khác
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

#### Xử lý ngoại lệ

Nếu bạn đang sử dụng một bộ lọc ngoại lệ catch-all toàn cầu (đó là một bộ lọc được đăng ký với `app.useGlobalFilters()` hoặc một bộ lọc được đăng ký trong các provider module ứng dụng của bạn được chú thích với decorator `@Catch()` không có đối số), thêm decorator `@SentryExceptionCaptured()` vào phương thức `catch()` của bộ lọc. Decorator này sẽ báo cáo tất cả các lỗi không mong đợi được nhận bởi bộ lọc lỗi toàn cầu của bạn đến Sentry:

```typescript
import { Catch, ExceptionFilter } from '@nestjs/common';
import { SentryExceptionCaptured } from '@sentry/nestjs';

@Catch()
export class YourCatchAllExceptionFilter implements ExceptionFilter {
  @SentryExceptionCaptured()
  catch(exception, host): void {
    // triển khai của bạn ở đây
  }
}
```

Theo mặc định, chỉ các ngoại lệ không được xử lý không bị bắt bởi bộ lọc lỗi được báo cáo cho Sentry. `HttpExceptions` (bao gồm [các dẫn xuất](https://docs.nestjs.com/exception-filters#built-in-http-exceptions)) cũng không được bắt theo mặc định vì chúng chủ yếu hoạt động như các phương tiện luồng điều khiển.

Nếu bạn không có một bộ lọc ngoại lệ catch-all toàn cầu, thêm `SentryGlobalFilter` vào các provider của module chính của bạn. Bộ lọc này sẽ báo cáo bất kỳ lỗi không được xử lý nào không bị bắt bởi các bộ lọc lỗi khác đến Sentry.

> warning **Cảnh báo** `SentryGlobalFilter` cần được đăng ký trước bất kỳ bộ lọc ngoại lệ nào khác.

```typescript
@@filename(app.module)
import { Module } from "@nestjs/common";
import { APP_FILTER } from "@nestjs/core";
import { SentryGlobalFilter } from "@sentry/nestjs/setup";

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: SentryGlobalFilter,
    },
    // ..các provider khác
  ],
})
export class AppModule {}
```

#### Stack traces có thể đọc được

Tùy thuộc vào cách bạn đã thiết lập dự án của mình, các stack traces trong các lỗi Sentry của bạn có thể sẽ không trông giống với mã thực của bạn.

Để khắc phục điều này, tải lên source maps của bạn đến Sentry. Cách dễ nhất để làm điều này là sử dụng Sentry Wizard:

```bash
npx @sentry/wizard@latest -i sourcemaps
```

#### Kiểm tra tích hợp

Để xác minh tích hợp Sentry của bạn đang hoạt động, bạn có thể thêm một endpoint kiểm tra ném một lỗi:

```typescript
@Get("debug-sentry")
getError() {
  throw new Error("My first Sentry error!");
}
```

Truy cập `/debug-sentry` trong ứng dụng của bạn, và bạn nên thấy lỗi xuất hiện trong bảng điều khiển Sentry của bạn.

### Tóm tắt

Để biết tài liệu hoàn chỉnh về NestJS SDK của Sentry, bao gồm các tùy chọn cấu hình nâng cao và tính năng, hãy truy cập [tài liệu Sentry chính thức](https://docs.sentry.io/platforms/javascript/guides/nestjs/).

Trong khi lỗi phần mềm là thứ của Sentry, chúng tôi vẫn viết chúng. Nếu bạn gặp bất kỳ vấn đề nào trong khi cài đặt SDK của chúng tôi, vui lòng mở [GitHub Issue](https://github.com/getsentry/sentry-javascript/issues) hoặc liên hệ trên [Discord](https://discord.com/invite/sentry).