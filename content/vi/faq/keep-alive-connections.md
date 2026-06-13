### Kết nối keep alive

Theo mặc định, các adapter HTTP của NestJS sẽ đợi cho đến khi phản hồi kết thúc trước khi đóng ứng dụng. Nhưng đôi khi, hành vi này không được mong muốn, hoặc không mong đợi. Có thể có một số yêu cầu sử dụng các header `Connection: Keep-Alive` tồn tại trong một thời gian dài.

Đối với các kịch bản này nơi bạn luôn muốn ứng dụng của mình thoát mà không đợi các yêu cầu kết thúc, bạn có thể bật tùy chọn `forceCloseConnections` khi tạo ứng dụng NestJS của bạn.

> warning **Mẹo** Hầu hết người dùng sẽ không cần bật tùy chọn này. Nhưng triệu chứng cần tùy chọn này là ứng dụng của bạn sẽ không thoát khi bạn mong đợi. Thường là khi `app.enableShutdownHooks()` được bật và bạn nhận thấy rằng ứng dụng không đang khởi động lại/thoát. Rất có thể trong khi chạy ứng dụng NestJS trong quá trình phát triển với `--watch`.

#### Sử dụng

Trong file `main.ts` của bạn, bật tùy chọn khi tạo ứng dụng NestJS của bạn:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    forceCloseConnections: true,
  });
  await app.listen(process.env.PORT ?? 3000);
}

bootstrap();
```