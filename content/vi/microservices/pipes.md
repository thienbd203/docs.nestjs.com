### Pipes

Không có sự khác biệt cơ bản giữa [pipes thông thường](/pipes) và microservices pipes. Sự khác biệt duy nhất là thay vì throwing `HttpException`, bạn nên sử dụng `RpcException`.

> info **Gợi ý** Lớp `RpcException` được expose từ gói `@nestjs/microservices`.

#### Binding pipes

Ví dụ sau sử dụng một pipe phạm vi phương thức được khởi tạo thủ công. Giống như với các ứng dụng dựa trên HTTP, bạn cũng có thể sử dụng pipes phạm vi controller (tức là tiền tố lớp controller với một decorator `@UsePipes()`).

```typescript
@@filename()
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new RpcException(errors) }))
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new RpcException(errors) }))
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```