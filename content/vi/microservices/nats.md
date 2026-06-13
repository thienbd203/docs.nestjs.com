### NATS

[NATS](https://nats.io) là một hệ thống nhắn tin open source đơn giản, an toàn và hiệu suất cao cho các ứng dụng cloud native, nhắn tin IoT và kiến trúc microservices. Máy chủ NATS được viết bằng ngôn ngữ lập trình Go, nhưng các thư viện client để tương tác với máy chủ có sẵn cho hàng chục ngôn ngữ lập trình chính. NATS hỗ trợ cả giao hàng **At Most Once** và **At Least Once**. Nó có thể chạy ở bất cứ đâu, từ các máy chủ lớn và các thể hiện cloud, qua các gateway biên và thậm chí các thiết bị Internet of Things.

#### Cài đặt

Để bắt đầu xây dựng microservices dựa trên NATS, trước tiên hãy cài đặt gói cần thiết:

```bash
$ npm i --save nats
```

#### Tổng quan

Để sử dụng transporter NATS, truyền đối tượng options sau đến phương thức `createMicroservice()`:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
  },
});
```

> info **Gợi ý** Enum `Transport` được import từ gói `@nestjs/microservices`.

#### Tùy chọn

Đối tượng `options` là cụ thể cho transporter được chọn. Transporter <strong>NATS</strong> expose các thuộc tính được mô tả [ở đây](https://github.com/nats-io/node-nats#connection-options) cũng như các thuộc tính sau:

<table>
  <tr>
    <td><code>queue</code></td>
    <td>Hàng đợi mà server của bạn nên subscribe vào (để <code>undefined</code> để bỏ qua thiết lập này). Đọc thêm về các nhóm hàng đợi NATS <a href="https://docs.nestjs.com/microservices/nats#queue-groups">dưới đây</a>.
    </td>
  </tr>
  <tr>
    <td><code>gracefulShutdown</code></td>
    <td>Kích hoạt graceful shutdown. Khi được kích hoạt, server trước tiên unsubscribe từ tất cả các kênh trước khi đóng kết nối. Mặc định là <code>false</code>.
  </tr>
  <tr>
    <td><code>gracePeriod</code></td>
    <td>Thời gian tính bằng mili giây để chờ server sau khi unsubscribe từ tất cả các kênh. Mặc định là <code>10000</code> ms.
  </tr>
</table>

#### Client

Giống như các transporter microservice khác, bạn có <a href="https://docs.nestjs.com/microservices/basics#client">nhiều tùy chọn</a> để tạo một thể hiện NATS `ClientProxy`.

Một phương pháp để tạo một thể hiện là sử dụng `ClientsModule`. Để tạo một thể hiện client với `ClientsModule`, import nó và sử dụng phương thức `register()` để truyền một đối tượng options với các thuộc tính giống nhau được hiển thị ở trên trong phương thức `createMicroservice()`, cũng như một thuộc tính `name` để được sử dụng như injection token. Đọc thêm về `ClientsModule` <a href="https://docs.nestjs.com/microservices/basics#client">ở đây</a>.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.NATS,
        options: {
          servers: ['nats://localhost:4222'],
        }
      },
    ]),
  ]
  ...
})
```

Các tùy chọn khác để tạo một client (hoặc `ClientProxyFactory` hoặc `@Client()`) cũng có thể được sử dụng. Bạn có thể đọc về chúng <a href="https://docs.nestjs.com/microservices/basics#client">ở đây</a>.

#### Request-response

Đối với phong cách tin nhắn **request-response** ([đọc thêm](https://docs.nestjs.com/microservices/basics#request-response)), transporter NATS không sử dụng cơ chế [Request-Reply](https://docs.nats.io/nats-concepts/reqreply) tích hợp sẵn của NATS. Thay vào đó, một "request" được xuất bản trên một subject nhất định sử dụng phương thức `publish()` với một tên subject phản hồi duy nhất, và các responders lắng nghe trên subject đó và gửi phản hồi đến subject phản hồi. Các subject phản hồi được định hướng trở lại requestor một cách động, bất kể vị trí của cả hai bên.

#### Event-based

Đối với phong cách tin nhắn **event-based** ([đọc thêm](https://docs.nestjs.com/microservices/basics#event-based)), transporter NATS sử dụng cơ chế [Publish-Subscribe](https://docs.nats.io/nats-concepts/pubsub) tích hợp sẵn của NATS. Một publisher gửi một tin nhắn trên một subject và bất kỳ subscriber hoạt động nào lắng nghe trên subject đó nhận tin nhắn. Subscribers cũng có thể đăng ký quan tâm trong các subjects wildcard hoạt động một chút như một biểu thức chính quy. Pattern one-to-many này đôi khi được gọi là fan-out.

#### Nhóm hàng đợi

NATS cung cấp một tính năng load balancing tích hợp sẵn gọi là [distributed queues](https://docs.nats.io/nats-concepts/queue). Để tạo một subscription hàng đợi, sử dụng thuộc tính `queue` như sau:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
    queue: 'cats_queue',
  },
});
```

#### Context

Trong các kịch bản phức tạp hơn, bạn có thể cần truy cập thông tin bổ sung về yêu cầu đến. Khi sử dụng transporter NATS, bạn có thể truy cập đối tượng `NatsContext`.

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Subject: ${context.getSubject()}`);
}
```

> info **Gợi ý** `@Payload()`, `@Ctx()` và `NatsContext` được import từ gói `@nestjs/microservices`.

#### Wildcards

Một subscription có thể là cho một subject cụ thể, hoặc nó có thể bao gồm wildcards.

```typescript
@@filename()
@MessagePattern('time.us.*')
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`); // e.g. "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('time.us.*')
getDate(data, context) {
  console.log(`Subject: ${context.getSubject()}`); // e.g. "time.us.east"
  return new Date().toLocaleTimeString(...);
}
```

#### Record builders

Để cấu hình các tùy chọn tin nhắn, bạn có thể sử dụng lớp `NatsRecordBuilder` (lưu ý: điều này có thể thực hiện cho các luồng event-based cũng vậy). Ví dụ, để thêm header `x-version`, sử dụng phương thức `setHeaders`, như sau:

```typescript
import * as nats from 'nats';

// somewhere in your code
const headers = nats.headers();
headers.set('x-version', '1.0.0');

const record = new NatsRecordBuilder(':cat:').setHeaders(headers).build();
this.client.send('replace-emoji', record).subscribe(...);
```

> info **Gợi ý** Lớp `NatsRecordBuilder` được xuất từ gói `@nestjs/microservices`.

Và bạn có thể đọc các headers này ở phía server cũng vậy, bằng cách truy cập `NatsContext`, như sau:

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: NatsContext): string {
  const headers = context.getHeaders();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const headers = context.getHeaders();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

Trong một số trường hợp bạn có thể muốn cấu hình headers cho nhiều yêu cầu, bạn có thể truyền các thứ này như tùy chọn đến `ClientProxyFactory`:

```typescript
import { Module } from '@nestjs/common';
import { ClientProxyFactory, Transport } from '@nestjs/microservices';

@Module({
  providers: [
    {
      provide: 'API_v1',
      useFactory: () =>
        ClientProxyFactory.create({
          transport: Transport.NATS,
          options: {
            servers: ['nats://localhost:4222'],
            headers: { 'x-version': '1.0.0' },
          },
        }),
    },
  ],
})
export class ApiModule {}
```

#### Cập nhật trạng thái thể hiện

Để nhận cập nhật thời gian thực về kết nối và trạng thái của thể hiện driver cơ bản, bạn có thể subscribe vào stream `status`. Stream này cung cấp các cập nhật trạng thái cụ thể cho driver được chọn. Đối với driver NATS, stream `status` emit các sự kiện `connected`, `disconnected`, và `reconnecting`.

```typescript
this.client.status.subscribe((status: NatsStatus) => {
  console.log(status);
});
```

> info **Gợi ý** Kiểu `NatsStatus` được import từ gói `@nestjs/microservices`.

Tương tự, bạn có thể subscribe vào stream `status` của server để nhận thông báo về trạng thái của server.

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: NatsStatus) => {
  console.log(status);
});
```

#### Lắng nghe các sự kiện Nats

Trong một số trường hợp, bạn có thể muốn lắng nghe các sự kiện nội bộ được emit bởi microservice. Ví dụ, bạn có thể lắng nghe sự kiện `error` để kích hoạt các thao tác bổ sung khi xảy ra lỗi. Để làm điều này, sử dụng phương thức `on()`, như được hiển thị dưới đây:

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

Tương tự, bạn có thể lắng nghe các sự kiện nội bộ của server:

```typescript
server.on<NatsEvents>('error', (err) => {
  console.error(err);
});
```

> info **Gợi ý** Kiểu `NatsEvents` được import từ gói `@nestjs/microservices`.

#### Truy cập driver cơ bản

Đối với các trường hợp sử dụng nâng cao hơn, bạn có thể cần truy cập thể hiện driver cơ bản. Điều này có thể hữu ích cho các kịch bản như đóng thủ công kết nối hoặc sử dụng các phương thức cụ thể của driver. Tuy nhiên, hãy nhớ rằng đối với hầu hết các trường hợp, bạn **không nên cần** truy cập trực tiếp driver.

Để làm điều này, bạn có thể sử dụng phương thức `unwrap()`, trả về thể hiện driver cơ bản. Tham số kiểu generic nên chỉ định loại thể hiện driver bạn mong đợi.

```typescript
const natsConnection = this.client.unwrap<import('nats').NatsConnection>();
```

Tương tự, bạn có thể truy cập thể hiện driver cơ bản của server:

```typescript
const natsConnection = server.unwrap<import('nats').NatsConnection>();
```