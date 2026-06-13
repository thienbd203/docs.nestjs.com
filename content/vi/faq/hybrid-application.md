### Ứng dụng lai

Một ứng dụng lai là một ứng dụng lắng nghe các yêu cầu từ hai hoặc nhiều nguồn khác nhau. Điều này có thể kết hợp một máy chủ HTTP với một bộ lắng nghe microservice hoặc thậm chí chỉ là nhiều bộ lắng nghe microservice khác nhau. Phương thức `createMicroservice` mặc định không cho phép nhiều máy chủ vì vậy trong trường hợp này mỗi microservice phải được tạo và khởi động thủ công. Để làm điều này, instance `INestApplication` có thể được kết nối với các instance `INestMicroservice` thông qua phương thức `connectMicroservice()`.

```typescript
const app = await NestFactory.create(AppModule);
const microservice = app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.TCP,
});

await app.startAllMicroservices();
await app.listen(3001);
```

> info **Gợi ý** phương thức `app.listen(port)` khởi động một máy chủ HTTP trên địa chỉ được chỉ định. Nếu ứng dụng của bạn không xử lý các yêu cầu HTTP thì bạn nên sử dụng phương thức `app.init()` thay thế.

Để kết nối nhiều instance microservice, phát hành lệnh gọi đến `connectMicroservice()` cho mỗi microservice:

```typescript
const app = await NestFactory.create(AppModule);
// microservice #1
const microserviceTcp = app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.TCP,
  options: {
    port: 3001,
  },
});
// microservice #2
const microserviceRedis = app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});

await app.startAllMicroservices();
await app.listen(3001);
```

Để bind `@MessagePattern()` chỉ đến một chiến lược transport (ví dụ, MQTT) trong một ứng dụng lai với nhiều microservice, chúng ta có thể truyền đối số thứ hai kiểu `Transport` là một enum với tất cả các chiến lược transport tích hợp sẵn được định nghĩa.

```typescript
@@filename()
@MessagePattern('time.us.*', Transport.NATS)
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`); // e.g. "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@MessagePattern({ cmd: 'time.us' }, Transport.TCP)
getTCPDate(@Payload() data: number[]) {
  return new Date().toLocaleTimeString(...);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('time.us.*', Transport.NATS)
getDate(data, context) {
  console.log(`Subject: ${context.getSubject()}`); // e.g. "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@Bind(Payload(), Ctx())
@MessagePattern({ cmd: 'time.us' }, Transport.TCP)
getTCPDate(data) {
  return new Date().toLocaleTimeString(...);
}
```

> info **Gợi ý** `@Payload()`, `@Ctx()`, `Transport` và `NatsContext` được nhập từ `@nestjs/microservices`.

#### Chia sẻ cấu hình

Theo mặc định, một ứng dụng lai sẽ không kế thừa các pipe, interceptor, guard và filter toàn cầu được cấu hình cho ứng dụng chính (dựa trên HTTP).
Để kế thừa các thuộc tính cấu hình này từ ứng dụng chính, đặt thuộc tính `inheritAppConfig` trong đối số thứ hai (một đối tượng tùy chọn tùy chọn) của lệnh gọi `connectMicroservice()`, như sau:

```typescript
const microservice = app.connectMicroservice<MicroserviceOptions>(
  {
    transport: Transport.TCP,
  },
  { inheritAppConfig: true },
);
```