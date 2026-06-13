### Serve Static

Để phục vụ nội dung tĩnh như một Single Page Application (SPA) chúng ta có thể sử dụng `ServeStaticModule` từ gói [`@nestjs/serve-static`](https://www.npmjs.com/package/@nestjs/serve-static).

#### Cài đặt

Trước tiên chúng ta cần cài đặt gói cần thiết:

```bash
$ npm install --save @nestjs/serve-static
```

#### Bootstrap

Khi quá trình cài đặt hoàn tất, chúng ta có thể nhập `ServeStaticModule` vào `AppModule` gốc và cấu hình nó bằng cách chuyển một đối tượng cấu hình vào phương thức `forRoot()`.

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'path';

@Module({
  imports: [
    ServeStaticModule.forRoot({
      rootPath: join(__dirname, '..', 'client'),
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Với điều này đã có, xây dựng trang web tĩnh và đặt nội dung của nó ở vị trí được chỉ định bởi thuộc tính `rootPath`.

#### Cấu hình

[ServeStaticModule](https://github.com/nestjs/serve-static) có thể được cấu hình với nhiều tùy chọn để tùy chỉnh hành vi của nó.
Bạn có thể đặt đường dẫn để render ứng dụng tĩnh của bạn, chỉ định các đường dẫn bị loại trừ, bật hoặc tắt việc đặt header phản hồi Cache-Control, v.v. Xem danh sách đầy đủ các tùy chọn [ở đây](https://github.com/nestjs/serve-static/blob/master/lib/interfaces/serve-static-options.interface.ts).

> warning **Lưu ý** `renderPath` mặc định của Ứng dụng Tĩnh là `*` (tất cả các đường dẫn), và module sẽ gửi các file "index.html" trong phản hồi.
> Nó cho phép bạn tạo định tuyến Client-Side cho SPA của bạn. Các đường dẫn, được chỉ định trong các controller của bạn sẽ fallback đến máy chủ.
> Bạn có thể thay đổi hành vi này đặt `serveRoot`, `renderPath` kết hợp chúng với các tùy chọn khác.
> Ngoài ra, tùy chọn `serveStaticOptions.fallthrough` đã được triển khai trong adapter Fastify để mô phỏng hành vi fallthrough của Express và cần được đặt thành `true` để gửi `index.html` thay vì lỗi 404 cho route không tồn tại.

#### Ví dụ

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/24-serve-static).