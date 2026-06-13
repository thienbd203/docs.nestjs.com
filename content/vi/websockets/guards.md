### Guards

Không có sự khác biệt cơ bản giữa web sockets guards và [guards ứng dụng HTTP thông thường](/guards). Sự khác biệt duy nhất là thay vì throwing `HttpException`, bạn nên sử dụng `WsException`.

> info **Gợi ý** Lớp `WsException` được expose từ gói `@nestjs/websockets`.

#### Binding guards

Ví dụ sau sử dụng một guard phạm vi phương thức. Giống như với các ứng dụng dựa trên HTTP, bạn cũng có thể sử dụng guards phạm vi gateway (tức là tiền tố lớp gateway với một decorator `@UseGuards()`).

```typescript
@@filename()
@UseGuards(AuthGuard)
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UseGuards(AuthGuard)
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```