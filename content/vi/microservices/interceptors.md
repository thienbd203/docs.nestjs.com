### Interceptors

Không có sự khác biệt giữa [interceptors thông thường](/interceptors) và microservices interceptors. Ví dụ sau sử dụng một interceptor phạm vi phương thức được khởi tạo thủ công. Giống như với các ứng dụng dựa trên HTTP, bạn cũng có thể sử dụng interceptors phạm vi controller (tức là tiền tố lớp controller với một decorator `@UseInterceptors()`).

```typescript
@@filename()
@UseInterceptors(new TransformInterceptor())
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseInterceptors(new TransformInterceptor())
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```