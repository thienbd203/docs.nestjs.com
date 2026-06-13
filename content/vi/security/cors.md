### CORS

Cross-origin resource sharing (CORS) là một cơ chế cho phép các tài nguyên được yêu cầu từ một domain khác. Dưới lớp vỏ, Nest sử dụng Express [cors](https://github.com/expressjs/cors) hoặc Fastify [@fastify/cors](https://github.com/fastify/fastify-cors) packages tùy thuộc vào nền tảng cơ bản. Các packages này cung cấp các tùy chọn khác nhau mà bạn có thể tùy chỉnh dựa trên các yêu cầu của mình.

#### Getting started

Để bật CORS, gọi phương thức `enableCors()` trên đối tượng ứng dụng Nest.

```typescript
const app = await NestFactory.create(AppModule);
app.enableCors();
await app.listen(process.env.PORT ?? 3000);
```

Phương thức `enableCors()` nhận một đối số đối tượng cấu hình tùy chọn. Các thuộc tính có sẵn của đối tượng này được mô tả trong tài liệu [CORS](https://github.com/expressjs/cors#configuration-options) chính thức. Một cách khác là truyền một [callback function](https://github.com/expressjs/cors#configuring-cors-asynchronously) cho phép bạn định nghĩa đối tượng cấu hình một cách không đồng bộ dựa trên yêu cầu (trong thời gian thực).

Ngoài ra, hãy bật CORS thông qua đối tượng tùy chọn của phương thức `create()`. Đặt thuộc tính `cors` thành `true` để bật CORS với các cài đặt mặc định.
Hoặc, truyền một [đối tượng cấu hình CORS](https://github.com/expressjs/cors#configuration-options) hoặc [callback function](https://github.com/expressjs/cors#configuring-cors-asynchronously) như giá trị thuộc tính `cors` để tùy chỉnh hành vi của nó.

```typescript
const app = await NestFactory.create(AppModule, { cors: true });
await app.listen(process.env.PORT ?? 3000);
```