### Pipes

Không có sự khác biệt cơ bản giữa [pipes thông thường](/pipes) và web sockets pipes. Sự khác biệt duy nhất là thay vì throwing `HttpException`, bạn nên sử dụng `WsException`. Ngoài ra, tất cả các pipes sẽ chỉ được áp dụng cho tham số `data` (vì việc xác thực hoặc chuyển đổi thể hiện `client` là vô ích).

> info **Gợi ý** Lớp `WsException` được expose từ gói `@nestjs/websockets`.

#### Binding pipes

Ví dụ sau sử dụng một pipe phạm vi phương thức được khởi tạo thủ công. Giống như với các ứng dụng dựa trên HTTP, bạn cũng có thể sử dụng pipes phạm vi gateway (tức là tiền tố lớp gateway với một decorator `@UsePipes()`).

```typescript
@@filename()
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new WsException(errors) }))
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new WsException(errors) }))
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```