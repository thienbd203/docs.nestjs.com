### MQTT

[MQTT](https://mqtt.org/) (Message Queuing Telemetry Transport) là một giao thức nhắn tin open source, nhẹ, được tối ưu hóa cho độ trễ thấp. Giao thức này cung cấp một cách có thể mở rộng và hiệu quả về chi phí để kết nối các thiết bị sử dụng mô hình **publish/subscribe**. Một hệ thống giao tiếp được xây dựng trên MQTT bao gồm máy chủ xuất bản, một broker và một hoặc nhiều clients. Nó được thiết kế cho các thiết bị bị hạn chế và mạng có băng thông thấp, độ trễ cao hoặc không đáng tin cậy.

#### Cài đặt

Để bắt đầu xây dựng microservices dựa trên MQTT, trước tiên hãy cài đặt gói cần thiết:

```bash
$ npm i --save mqtt
```

#### Tổng quan

Để sử dụng transporter MQTT, truyền đối tượng options sau đến phương thức `createMicroservice()`:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
  },
});
```

> info **Gợi ý** Enum `Transport` được import từ gói `@nestjs/microservices`.

#### Tùy chọn

Đối tượng `options` là cụ thể cho transporter được chọn. Transporter <strong>MQTT</strong> expose các thuộc tính được mô tả [ở đây](https://github.com/mqttjs/MQTT.js/#mqttclientstreambuilder-options).

#### Client

Giống như các transporter microservice khác, bạn có <a href="https://docs.nestjs.com/microservices/basics#client">nhiều tùy chọn</a> để tạo một thể hiện MQTT `ClientProxy`.

Một phương pháp để tạo một thể hiện là sử dụng `ClientsModule`. Để tạo một thể hiện client với `ClientsModule`, import nó và sử dụng phương thức `register()` để truyền một đối tượng options với các thuộc tính giống nhau được hiển thị ở trên trong phương thức `createMicroservice()`, cũng như một thuộc tính `name` để được sử dụng như injection token. Đọc thêm về `ClientsModule` <a href="https://docs.nestjs.com/microservices/basics#client">ở đây</a>.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.MQTT,
        options: {
          url: 'mqtt://localhost:1883',
        }
      },
    ]),
  ]
  ...
})
```

Các tùy chọn khác để tạo một client (hoặc `ClientProxyFactory` hoặc `@Client()`) cũng có thể được sử dụng. Bạn có thể đọc về chúng <a href="https://docs.nestjs.com/microservices/basics#client">ở đây</a>.

#### Context

Trong các kịch bản phức tạp hơn, bạn có thể cần truy cập thông tin bổ sung về yêu cầu đến. Khi sử dụng transporter MQTT, bạn có thể truy cập đối tượng `MqttContext`.

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

> info **Gợi ý** `@Payload()`, `@Ctx()` và `MqttContext` được import từ gói `@nestjs/microservices`.

Để truy cập [packet](https://github.com/mqttjs/mqtt-packet) mqtt gốc, sử dụng phương thức `getPacket()` của đối tượng `MqttContext`, như sau:

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
  console.log(context.getPacket());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getPacket());
}
```

#### Wildcards

Một subscription có thể là cho một topic cụ thể, hoặc nó có thể bao gồm wildcards. Hai wildcards có sẵn, `+` và `#`. `+` là một wildcard cấp đơn, trong khi `#` là một wildcard đa cấp bao gồm nhiều cấp topic.

```typescript
@@filename()
@MessagePattern('sensors/+/temperature/+')
getTemperature(@Ctx() context: MqttContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
@@switch
@Bind(Ctx())
@MessagePattern('sensors/+/temperature/+')
getTemperature(context) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

#### Quality of Service (QoS)

Bất kỳ subscription nào được tạo với decorators `@MessagePattern` hoặc `@EventPattern` sẽ subscribe với QoS 0. Nếu QoS cao hơn được yêu cầu, nó có thể được đặt toàn cục sử dụng khối `subscribeOptions` khi thiết lập kết nối như sau:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
    subscribeOptions: {
      qos: 2
    },
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
    subscribeOptions: {
      qos: 2
    },
  },
});
```

#### QoS theo pattern

Bạn có thể override QoS subscription MQTT trên cơ sở theo pattern bằng cách cung cấp `qos` trong trường `extras` của decorator pattern. Khi không được chỉ định, `subscribeOptions.qos` toàn cục được sử dụng như mặc định.

```typescript
@@filename()
@EventPattern('critical-events', { extras: { qos: 2 } })
handleCriticalEvent(@Payload() data: any) {
  // Subscription này sử dụng QoS 2
}

@EventPattern('metrics', { extras: { qos: 0 } })
handleMetrics(@Payload() data: any) {
  // Subscription này sử dụng QoS 0
}
@@switch
@Bind(Payload())
@EventPattern('critical-events', { extras: { qos: 2 } })
handleCriticalEvent(data) {
  // Subscription này sử dụng QoS 2
}

@Bind(Payload())
@EventPattern('metrics', { extras: { qos: 0 } })
handleMetrics(data) {
  // Subscription này sử dụng QoS 0
}
```

> info **Gợi ý** Cấu hình QoS theo pattern không ảnh hưởng đến hành vi hiện có. Khi `extras.qos` không được chỉ định, subscription sử dụng giá trị `subscribeOptions.qos` toàn cục.

#### Record builders

Để cấu hình các tùy chọn tin nhắn (điều chỉnh mức QoS, đặt cờ Retain hoặc DUP, hoặc thêm các thuộc tính bổ sung vào payload), bạn có thể sử dụng lớp `MqttRecordBuilder`. Ví dụ, để đặt `QoS` thành `2` sử dụng phương thức `setQoS`, như sau:

```typescript
const userProperties = { 'x-version': '1.0.0' };
const record = new MqttRecordBuilder(':cat:')
  .setProperties({ userProperties })
  .setQoS(1)
  .build();
client.send('replace-emoji', record).subscribe(...);
```

> info **Gợi ý** Lớp `MqttRecordBuilder` được xuất từ gói `@nestjs/microservices`.

Và bạn có thể đọc các tùy chọn này ở phía server cũng vậy, bằng cách truy cập `MqttContext`.

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: MqttContext): string {
  const { properties: { userProperties } } = context.getPacket();
  return userProperties['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const { properties: { userProperties } } = context.getPacket();
  return userProperties['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

Trong một số trường hợp bạn có thể muốn cấu hình user properties cho nhiều yêu cầu, bạn có thể truyền các tùy chọn này đến `ClientProxyFactory`.

```typescript
import { Module } from '@nestjs/common';
import { ClientProxyFactory, Transport } from '@nestjs/microservices';

@Module({
  providers: [
    {
      provide: 'API_v1',
      useFactory: () =>
        ClientProxyFactory.create({
          transport: Transport.MQTT,
          options: {
            url: 'mqtt://localhost:1833',
            userProperties: { 'x-version': '1.0.0' },
          },
        }),
    },
  ],
})
export class ApiModule {}
```

#### Cập nhật trạng thái thể hiện

Để nhận cập nhật thời gian thực về kết nối và trạng thái của thể hiện driver cơ bản, bạn có thể subscribe vào stream `status`. Stream này cung cấp các cập nhật trạng thái cụ thể cho driver được chọn. Đối với driver MQTT, stream `status` emit các sự kiện `connected`, `disconnected`, `reconnecting`, và `closed`.

```typescript
this.client.status.subscribe((status: MqttStatus) => {
  console.log(status);
});
```

> info **Gợi ý** Kiểu `MqttStatus` được import từ gói `@nestjs/microservices`.

Tương tự, bạn có thể subscribe vào stream `status` của server để nhận thông báo về trạng thái của server.

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: MqttStatus) => {
  console.log(status);
});
```

#### Lắng nghe các sự kiện MQTT

Trong một số trường hợp, bạn có thể muốn lắng nghe các sự kiện nội bộ được emit bởi microservice. Ví dụ, bạn có thể lắng nghe sự kiện `error` để kích hoạt các thao tác bổ sung khi xảy ra lỗi. Để làm điều này, sử dụng phương thức `on()`, như được hiển thị dưới đây:

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

Tương tự, bạn có thể lắng nghe các sự kiện nội bộ của server:

```typescript
server.on<MqttEvents>('error', (err) => {
  console.error(err);
});
```

> info **Gợi ý** Kiểu `MqttEvents` được import từ gói `@nestjs/microservices`.

#### Truy cập driver cơ bản

Đối với các trường hợp sử dụng nâng cao hơn, bạn có thể cần truy cập thể hiện driver cơ bản. Điều này có thể hữu ích cho các kịch bản như đóng thủ công kết nối hoặc sử dụng các phương thức cụ thể của driver. Tuy nhiên, hãy nhớ rằng đối với hầu hết các trường hợp, bạn **không nên cần** truy cập trực tiếp driver.

Để làm điều này, bạn có thể sử dụng phương thức `unwrap()`, trả về thể hiện driver cơ bản. Tham số kiểu generic nên chỉ định loại thể hiện driver bạn mong đợi.

```typescript
const mqttClient = this.client.unwrap<import('mqtt').MqttClient>();
```

Tương tự, bạn có thể truy cập thể hiện driver cơ bản của server:

```typescript
const mqttClient = server.unwrap<import('mqtt').MqttClient>();
```