### Gateways

Hầu hết các khái niệm được thảo luận ở nơi khác trong tài liệu này, chẳng hạn như dependency injection, decorators, exception filters, pipes, guards và interceptors, đều áp dụng tương đương cho gateways. Bất cứ khi nào có thể, Nest trừu tượng hóa các chi tiết triển khai để cùng một thành phần có thể chạy trên các nền tảng dựa trên HTTP, WebSockets và Microservices. Phần này bao gồm các khía cạnh của Nest đặc thù cho WebSockets.

Trong Nest, một gateway đơn giản là một lớp được chú thích với decorator `@WebSocketGateway()Về mặt kỹ thuật, gateways là platform-agnostic điều này làm cho chúng tương thích với bất kỳ thư viện WebSockets nào khi một adapter được tạo. Có hai nền tảng WS được hỗ trợ tích hợp sẵn: [socket.io](https://github.com/socketio/socket.io) và [ws](https://github.com/websockets/ws). Bạn có thể chọn cái phù hợp nhất với nhu cầu của mình. Ngoài ra, bạn có thể xây dựng adapter của riêng mình bằng cách theo hướng dẫn [này](/websockets/adapter).

<figure><img class="illustrative-image" src="/assets/Gateways_1.png" /></figure>

> info **Gợi ý** Gateways có thể được coi là [providers](/providers); điều này có nghĩa là chúng có thể inject dependencies thông qua constructor lớp. Ngoài ra, gateways cũng có thể được inject bởi các lớp khác (providers và controllers).

#### Cài đặt

Để bắt đầu xây dựng các ứng dụng dựa trên WebSockets, trước tiên hãy cài đặt gói cần thiết:

```bash
@@filename()
$ npm i --save @nestjs/websockets @nestjs/platform-socket.io
@@switch
$ npm i --save @nestjs/websockets @nestjs/platform-socket.io
```

#### Tổng quan

Nhìn chung, mỗi gateway lắng nghe trên cùng một cổng với **HTTP server**, trừ khi ứng dụng của bạn không phải là ứng dụng web, hoặc bạn đã thay đổi cổng thủ công. Hành vi mặc định này có thể được sửa đổi bằng cách truyền một đối số đến decorator `@WebSocketGateway(80)` trong đó `80` là một số cổng được chọn. Bạn cũng có thể đặt một [namespace](https://socket.io/docs/v4/namespaces/) được sử dụng bởi gateway bằng cách sử dụng cấu trúc sau:

```typescript
@WebSocketGateway(80, { namespace: 'events' })
```

> warning **Cảnh báo** Gateways không được khởi tạo cho đến khi chúng được tham chiếu trong mảng providers của một module hiện có.

Bạn có thể truyền bất kỳ [tùy chọn](https://socket.io/docs/v4/server-options/) được hỗ trợ nào đến constructor socket bằng đối số thứ hai cho decorator `@WebSocketGateway()`, như được hiển thị dưới đây:

```typescript
@WebSocketGateway(81, { transports: ['websocket'] })
```

Gateway hiện đang lắng nghe, nhưng chúng ta chưa subscribe vào bất kỳ tin nhắn đến nào. Hãy tạo một handler sẽ subscribe vào các tin nhắn `events` và phản hồi cho người dùng với cùng một dữ liệu chính xác.

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(@MessageBody() data: string): string {
  return data;
}
@@switch
@Bind(MessageBody())
@SubscribeMessage('events')
handleEvent(data) {
  return data;
}
```

> info **Gợi ý** Decorators `@SubscribeMessage()` và `@MessageBody()` được import từ gói `@nestjs/websockets`.

Sau khi gateway được tạo, chúng ta có thể đăng ký nó trong module của chúng ta.

```typescript
import { Module } from '@nestjs/common';
import { EventsGateway } from './events.gateway';

@@filename(events.module)
@Module({
  providers: [EventsGateway]
})
export class EventsModule {}
```

Bạn cũng có thể truyền một khóa thuộc tính vào decorator để trích xuất nó từ body tin nhắn đến:

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(@MessageBody('id') id: number): number {
  // id === messageBody.id
  return id;
}
@@switch
@Bind(MessageBody('id'))
@SubscribeMessage('events')
handleEvent(id) {
  // id === messageBody.id
  return id;
}
```

Nếu bạn thích không sử dụng decorators, mã sau là tương đương về mặt chức năng:

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(client: Socket, data: string): string {
  return data;
}
@@switch
@SubscribeMessage('events')
handleEvent(client, data) {
  return data;
}
```

Trong ví dụ trên, hàm `handleEvent()` nhận hai đối số. Đối số đầu tiên là một [thể hiện socket](https://socket.io/docs/v4/server-api/#socket) cụ thể nền tảng, trong khi đối số thứ hai là dữ liệu nhận được từ client. Cách tiếp cận này không được khuyến nghị, vì nó yêu cầu mocking thể hiện `socket` trong mỗi unit test.

Sau khi tin nhắn `events` được nhận, handler gửi một acknowledgment với cùng dữ liệu được gửi qua mạng. Ngoài ra, có thể emit tin nhắn sử dụng cách tiếp cận cụ thể thư viện, ví dụ, bằng cách sử dụng phương thức `client.emit()`. Để truy cập một thể hiện socket được kết nối, sử dụng decorator `@ConnectedSocket()`.

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(
  @MessageBody() data: string,
  @ConnectedSocket() client: Socket,
): string {
  return data;
}
@@switch
@Bind(MessageBody(), ConnectedSocket())
@SubscribeMessage('events')
handleEvent(data, client) {
  return data;
}
```

> info **Gợi ý** Decorator `@ConnectedSocket()` được import từ gói `@nestjs/websockets`.

Tuy nhiên, trong trường hợp này, bạn sẽ không thể tận dụng interceptors. Nếu bạn không muốn phản hồi cho người dùng, bạn có thể chỉ cần bỏ qua câu lệnh `return` (hoặc explicitly trả về một giá trị "falsy", ví dụ `undefined`).

Bây giờ khi một client emit tin nhắn như sau:

```typescript
socket.emit('events', { name: 'Nest' });
```

Phương thức `handleEvent()` sẽ được thực thi. Để lắng nghe các tin nhắn được emit từ trong handler ở trên, client phải đính kèm một listener acknowledgment tương ứng:

```typescript
socket.emit('events', { name: 'Nest' }, (data) => console.log(data));
```

Trong khi trả về một giá trị từ một message handler implicitly gửi một acknowledgement, các kịch bản nâng cao thường yêu cầu kiểm soát trực tiếp trên callback acknowledgment.

Decorator tham số `@Ack()` cho phép inject hàm callback `ack` trực tiếp vào một message handler.
Nếu không sử dụng decorator, callback này được truyền làm đối số thứ ba của phương thức.

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(
  @MessageBody() data: string,
  @Ack() ack: (response: { status: string; data: string }) => void,
) {
  ack({ status: 'received', data });
}
@@switch
@Bind(MessageBody(), Ack())
@SubscribeMessage('events')
handleEvent(data, ack) {
  ack({ status: 'received', data });
}
```

#### Đa phản hồi

Acknowledgment chỉ được dispatch một lần. Hơn nữa, nó không được hỗ trợ bởi triển khai WebSockets native. Để giải quyết giới hạn này, bạn có thể trả về một đối tượng bao gồm hai thuộc tính. `event` là tên của sự kiện được emit và `data` phải được chuyển tiếp đến client.

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(@MessageBody() data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@Bind(MessageBody())
@SubscribeMessage('events')
handleEvent(data) {
  const event = 'events';
  return { event, data };
}
```

> info **Gợi ý** Interface `WsResponse` được import từ gói `@nestjs/websockets`.

> warning **Cảnh báo** Bạn nên trả về một thể hiện lớp triển khai `WsResponse` nếu trường `data` của bạn phụ thuộc vào `ClassSerializerInterceptor`, vì nó bỏ qua các phản hồi đối tượng JavaScript thuần túy.

Để lắng nghe các phản hồi đến, client phải áp dụng một event listener khác.

```typescript
socket.on('events', (data) => console.log(data));
```

#### Phản hồi không đồng bộ

Message handlers có thể phản hồi đồng bộ hoặc **không đồng bộ**. Do đó, các phương thức `async` được hỗ trợ. Message handler cũng có thể trả về một `Observable`, trong trường hợp đó các giá trị kết quả sẽ được emit cho đến khi stream hoàn thành.

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
onEvent(@MessageBody() data: unknown): Observable<WsResponse<number>> {
  const event = 'events';
  const response = [1, 2, 3];

  return from(response).pipe(
    map(data => ({ event, data })),
  );
}
@@switch
@Bind(MessageBody())
@SubscribeMessage('events')
onEvent(data) {
  const event = 'events';
  const response = [1, 2, 3];

  return from(response).pipe(
    map(data => ({ event, data })),
  );
}
```

Trong ví dụ trên, message handler sẽ phản hồi **3 lần** (với mỗi mục từ mảng).

#### Lifecycle hooks

Có 3 lifecycle hooks hữu ích có sẵn. Tất cả chúng có các interface tương ứng và được mô tả trong bảng sau:

<table>
  <tr>
    <td>
      <code>OnGatewayInit</code>
    </td>
    <td>
      Bắt buộc triển khai phương thức <code>afterInit()</code>. Nhận thể hiện server cụ thể thư viện làm đối số (và
      spread phần còn lại nếu cần).
    </td>
  </tr>
  <tr>
    <td>
      <code>OnGatewayConnection</code>
    </td>
    <td>
      Bắt buộc triển khai phương thức <code>handleConnection()</code>. Nhận thể hiện socket client cụ thể thư viện làm
      đối số.
    </td>
  </tr>
  <tr>
    <td>
      <code>OnGatewayDisconnect</code>
    </td>
    <td>
      Bắt buộc triển khai phương thức <code>handleDisconnect()</code>. Nhận thể hiện socket client cụ thể thư viện làm
      đối số.
    </td>
  </tr>
</table>

> info **Gợi ý** Mỗi interface lifecycle được expose từ gói `@nestjs/websockets`.

#### Server và Namespace

Thỉnh thoảng, bạn có thể muốn có quyền truy cập trực tiếp đến thể hiện server **cụ thể nền tảng** native. Tham chiếu đến đối tượng này được truyền làm đối số đến phương thức `afterInit()` (interface `OnGatewayInit`). Một tùy chọn khác là sử dụng decorator `@WebSocketServer()`.

```typescript
@WebSocketServer()
server: Server;
```

Ngoài ra, bạn có thể truy xuất namespace tương ứng sử dụng thuộc tính `namespace`, như sau:

```typescript
@WebSocketGateway({ namespace: 'my-namespace' })
export class EventsGateway {
  @WebSocketServer()
  namespace: Namespace;
}
```

Decorator `@WebSocketServer()` inject một thể hiện server bằng cách tham chiếu metadata được lưu trữ bởi decorator `@WebSocketGateway()`. Nếu bạn cung cấp tùy chọn namespace cho decorator `@WebSocketGateway()`, decorator `@WebSocketServer()` trả về một thể hiện `Namespace` thay vì một thể hiện `Server`.

> warning **Lưu ý** Decorator `@WebSocketServer()` được import từ gói `@nestjs/websockets`.

Nest sẽ tự động gán thể hiện server đến thuộc tính này khi nó sẵn sàng để sử dụng.

<app-banner-enterprise></app-banner-enterprise>

#### Ví dụ

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/02-gateways).