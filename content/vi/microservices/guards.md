### Guards

Không có sự khác biệt cơ bản giữa microservices guards và [guards ứng dụng HTTP thông thường](/guards).
Sự khác biệt duy nhất là thay vì throwing `HttpException`, bạn nên sử dụng `RpcException`.

> info **Gợi ý** Lớp `RpcException` được expose từ gói `@nestjs/microservices`.

#### Binding guards

Ví dụ sau sử dụng một guard phạm vi phương thức. Giống như với các ứng dụng dựa trên HTTP, bạn cũng có thể sử dụng guards phạm vi controller (tức là tiền tố lớp controller với một decorator `@UseGuards()`).

```typescript
@@filename()
@UseGuards(AuthGuard)
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseGuards(AuthGuard)
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```