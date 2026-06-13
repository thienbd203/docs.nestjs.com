### Adapters

Module WebSockets là platform-agnostic, do đó, bạn có thể mang thư viện của riêng mình (hoặc thậm chí là một triển khai native) bằng cách sử dụng interface `WebSocketAdapter`. Interface này bắt buộc triển khai một vài phương thức được mô tả trong bảng sau:

<table>
  <tr>
    <td><code>create</code></td>
    <td>Tạo một thể hiện socket dựa trên các đối số được truyền</td>
  </tr>
  <tr>
    <td><code>bindClientConnect</code></td>
    <td>Liên kết sự kiện kết nối client</td>
  </tr>
  <tr>
    <td><code>bindClientDisconnect</code></td>
    <td>Liên kết sự kiện ngắt kết nối client (tùy chọn*)</td>
  </tr>
  <tr>
    <td><code>bindMessageHandlers</code></td>
    <td>Liên kết tin nhắn đến với message handler tương ứng</td>
  </tr>
  <tr>
    <td><code>close</code></td>
    <td>Chấm dứt một thể hiện server</td>
  </tr>
</table>

#### Mở rộng socket.io

Gói [socket.io](https://github.com/socketio/socket.io) được bọc trong lớp `IoAdapter`. Điều gì sẽ xảy ra nếu bạn muốn nâng cao chức năng cơ bản của adapter? Ví dụ, các yêu cầu kỹ thuật của bạn yêu cầu khả năng broadcast sự kiện qua nhiều thể hiện load-balanced của dịch vụ web của bạn. Để làm điều này, bạn có thể mở rộng `IoAdapter` và override một phương thức duy nhất có trách nhiệm khởi tạo các server socket.io mới. Nhưng trước hết, hãy cài đặt gói cần thiết.

> warning **Cảnh báo** Để sử dụng socket.io với nhiều thể hiện load-balanced, bạn phải tắt polling bằng cách đặt `transports: ['websocket']` trong cấu hình socket.io client của bạn hoặc bạn phải kích hoạt định tuyến dựa trên cookie trong load balancer của bạn. Redis một mình là không đủ. Xem [ở đây](https://socket.io/docs/v4/using-multiple-nodes/#enabling-sticky-session) để biết thêm thông tin.

```bash
$ npm i --save redis socket.io @socket.io/redis-adapter
```

Sau khi gói được cài đặt, chúng ta có thể tạo một lớp `RedisIoAdapter`.

```typescript
import { IoAdapter } from '@nestjs/platform-socket.io';
import { ServerOptions } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;

  async connectToRedis(): Promise<void> {
    const pubClient = createClient({ url: `redis://localhost:6379` });
    const subClient = pubClient.duplicate();

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
}
```

Sau đó, chỉ cần chuyển sang Redis adapter mới được tạo của bạn.

```typescript
const app = await NestFactory.create(AppModule);
const redisIoAdapter = new RedisIoAdapter(app);
await redisIoAdapter.connectToRedis();

app.useWebSocketAdapter(redisIoAdapter);
```

#### Thư viện ws

Một adapter có sẵn khác là `WsAdapter` đóng vai trò như một proxy giữa framework và tích hợp thư viện [ws](https://github.com/websockets/ws) cực nhanh và được kiểm tra kỹ. Adapter này hoàn toàn tương thích với WebSockets native trình duyệt và nhanh hơn nhiều so với gói socket.io. Không may, nó có ít chức năng hơn đáng kể có sẵn ngay lập tức. Trong một số trường hợp, bạn không nhất thiết cần chúng.

> info **Gợi ý** Thư viện `ws` không hỗ trợ namespaces (kênh giao tiếp phổ biến bởi `socket.io`). Tuy nhiên, để somehow mô phỏng tính năng này, bạn có thể mount nhiều server `ws` trên các đường dẫn khác nhau (ví dụ: `@WebSocketGateway({{ '{' }} path: '/users' {{ '}' }})`).

Để sử dụng `ws`, trước tiên chúng ta phải cài đặt gói cần thiết:

```bash
$ npm i --save @nestjs/platform-ws
```

Sau khi gói được cài đặt, chúng ta có thể chuyển adapter:

```typescript
const app = await NestFactory.create(AppModule);
app.useWebSocketAdapter(new WsAdapter(app));
```

> info **Gợi ý** `WsAdapter` được import từ `@nestjs/platform-ws`.

`wsAdapter` được thiết kế để xử lý các tin nhắn trong định dạng `{{ '{' }} event: string, data: any {{ '}' }}`. Nếu bạn cần nhận và xử lý tin nhắn trong một định dạng khác, bạn sẽ cần cấu hình một message parser để chuyển đổi chúng sang định dạng cần thiết này.

```typescript
const wsAdapter = new WsAdapter(app, {
  // Để xử lý tin nhắn trong định dạng [event, data]
  messageParser: (data) => {
    const [event, payload] = JSON.parse(data.toString());
    return { event, data: payload };
  },
});
```

Ngoài ra, bạn có thể cấu hình message parser sau khi adapter được tạo bằng cách sử dụng phương thức `setMessageParser`.

#### Nâng cao (adapter tùy chỉnh)

Để minh họa, chúng ta sẽ tích hợp thủ công thư viện [ws](https://github.com/websockets/ws). Như đã đề cập, adapter cho thư viện này đã được tạo và được expose từ gói `@nestjs/platform-ws` như một lớp `WsAdapter`. Đây là cách triển khai đơn giản hóa có thể trông như thế này:

```typescript
@@filename(ws-adapter)
import * as WebSocket from 'ws';
import { WebSocketAdapter, INestApplicationContext } from '@nestjs/common';
import { MessageMappingProperties } from '@nestjs/websockets';
import { Observable, fromEvent, EMPTY } from 'rxjs';
import { mergeMap, filter } from 'rxjs/operators';

export class WsAdapter implements WebSocketAdapter {
  constructor(private app: INestApplicationContext) {}

  create(port: number, options: any = {}): any {
    return new WebSocket.Server({ port, ...options });
  }

  bindClientConnect(server, callback: Function) {
    server.on('connection', callback);
  }

  bindMessageHandlers(
    client: WebSocket,
    handlers: MessageMappingProperties[],
    process: (data: any) => Observable<any>,
  ) {
    fromEvent(client, 'message')
      .pipe(
        mergeMap(data => this.bindMessageHandler(data, handlers, process)),
        filter(result => result),
      )
      .subscribe(response => client.send(JSON.stringify(response)));
  }

  bindMessageHandler(
    buffer,
    handlers: MessageMappingProperties[],
    process: (data: any) => Observable<any>,
  ): Observable<any> {
    const message = JSON.parse(buffer.data);
    const messageHandler = handlers.find(
      handler => handler.message === message.event,
    );
    if (!messageHandler) {
      return EMPTY;
    }
    return process(messageHandler.callback(message.data));
  }

  close(server) {
    server.close();
  }
}
```

> info **Gợi ý** Khi bạn muốn tận dụng thư viện [ws](https://github.com/websockets/ws), sử dụng `WsAdapter` tích hợp sẵn thay vì tạo cái của riêng bạn.

Sau đó, chúng ta có thể thiết lập một adapter tùy chỉnh sử dụng phương thức `useWebSocketAdapter()`:

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.useWebSocketAdapter(new WsAdapter(app));
```

#### Ví dụ

Một ví dụ hoạt động sử dụng `WsAdapter` có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/16-gateways-ws).