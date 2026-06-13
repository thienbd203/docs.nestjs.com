### Interceptors

Không có sự khác biệt giữa [interceptors thông thường](/interceptors) và web sockets interceptors. Ví dụ sau sử dụng một interceptor phạm vi phương thức được khởi tạo thủ công. Giống như với các ứng dụng dựa trên HTTP, bạn cũng có thể sử dụng interceptors phạm vi gateway (tức là tiền tố lớp gateway với một decorator `@UseInterceptors()`).

```typescript
@@filename()
@UseInterceptors(new TransformInterceptor())
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UseInterceptors(new TransformInterceptor())
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```