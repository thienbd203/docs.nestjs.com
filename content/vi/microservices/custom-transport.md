### Transporter tùy chỉnh

Nest cung cấp nhiều loại **transporters** tích hợp sẵn, cũng như một API cho phép các nhà phát triển xây dựng các chiến lược transport tùy chỉnh mới.
Transporters cho phép bạn kết nối các thành phần qua mạng bằng một lớp giao tiếp có thể cắm (pluggable) và một giao thức tin nhắn cấp ứng dụng rất đơn giản (đọc bài viết đầy đủ [ở đây](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3)).

> info **Gợi ý** Xây dựng một microservice với Nest không nhất thiết có nghĩa là bạn phải sử dụng gói `@nestjs/microservices`. Ví dụ, nếu bạn muốn giao tiếp với các dịch vụ bên ngoài (giả sử các microservices khác được viết bằng các ngôn ngữ khác), bạn có thể không cần tất cả các tính năng được cung cấp bởi thư viện `@nestjs/microservice`.
> Trên thực tế, nếu bạn không cần decorators (`@EventPattern` hoặc `@MessagePattern`) cho phép bạn định nghĩa subscribers một cách khai báo, chạy một [Ứng dụng Standalone](/application-context) và duy trì thủ công kết nối/subscribe vào các kênh nên đủ cho hầu hết các trường hợp sử dụng và sẽ cung cấp cho bạn sự linh hoạt hơn.

Với một transporter tùy chỉnh, bạn có thể tích hợp bất kỳ hệ thống/giao thức tin nhắn nào (bao gồm Google Cloud Pub/Sub, Amazon Kinesis và những cái khác) hoặc mở rộng cái hiện có, thêm các tính năng bổ sung ở trên (ví dụ, [QoS](https://github.com/mqttjs/MQTT.js/blob/master/README.md#qos) cho MQTT).

> info **Gợi ý** Để hiểu rõ hơn cách microservices Nest hoạt động và cách bạn có thể mở rộng khả năng của các transporter hiện có, chúng tôi khuyến nghị đọc chuỗi bài viết [NestJS Microservices in Action](https://dev.to/johnbiundo/series/4724) và [Advanced NestJS Microservices](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l).

#### Tạo chiến lược

Đầu tiên, hãy định nghĩa một lớp đại diện cho transporter tùy chỉnh của chúng ta.

```typescript
import { CustomTransportStrategy, Server } from '@nestjs/microservices';

class GoogleCloudPubSubServer
  extends Server
  implements CustomTransportStrategy
{
  /**
   * Được kích hoạt khi bạn chạy "app.listen()".
   */
  listen(callback: () => void) {
    callback();
  }

  /**
   * Được kích hoạt khi tắt ứng dụng.
   */
  close() {}

  /**
   * Bạn có thể bỏ qua phương thức này nếu bạn không muốn người dùng transporter
   * có thể đăng ký event listeners. Hầu hết các triển khai tùy chỉnh
   * sẽ không cần cái này.
   */
  on(event: string, callback: Function) {
    throw new Error('Method not implemented.');
  }

  /**
   * Bạn có thể bỏ qua phương thức này nếu bạn không muốn người dùng transporter
   * có thể truy xuất server native cơ bản. Hầu hết các triển khai tùy chỉnh
   * sẽ không cần cái này.
   */
  unwrap<T = never>(): T {
    throw new Error('Method not implemented.');
  }
}
```

> warning **Cảnh báo** Xin lưu ý, chúng tôi sẽ không triển khai một Google Cloud Pub/Sub server đầy đủ tính năng trong chương này vì điều này sẽ yêu cầu đi sâu vào các chi tiết kỹ thuật cụ thể của transporter.

Trong ví dụ trên, chúng ta đã khai báo lớp `GoogleCloudPubSubServer` và cung cấp các phương thức `listen()` và `close()` được thực thi bởi interface `CustomTransportStrategy`.
Ngoài ra, lớp của chúng ta mở rộng lớp `Server` được import từ gói `@nestjs/microservices` cung cấp một số phương thức hữu ích, ví dụ, các phương thức được sử dụng bởi runtime của Nest để đăng ký message handlers. Ngoài ra, trong trường hợp bạn muốn mở rộng khả năng của một chiến lược transport hiện có, bạn có thể mở rộng lớp server tương ứng, ví dụ, `ServerRedis`.
Theo quy ước, chúng ta thêm hậu tố `"Server"` vào lớp của mình vì nó sẽ chịu trách nhiệm subscribe vào các tin nhắn/sự kiện (và phản hồi chúng, nếu cần).

Với điều này đã có sẵn, bây giờ chúng ta có thể sử dụng chiến lược tùy chỉnh của mình thay vì sử dụng một transporter tích hợp sẵn, như sau:

```typescript
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    strategy: new GoogleCloudPubSubServer(),
  },
);
```

Về cơ bản, thay vì truyền đối tượng options transporter bình thường với các thuộc tính `transport` và `options`, chúng ta truyền một thuộc tính duy nhất, `strategy`, có giá trị là một thể hiện của lớp transporter tùy chỉnh của chúng ta.

Quay lại lớp `GoogleCloudPubSubServer` của chúng ta, trong một ứng dụng thực tế, chúng ta sẽ thiết lập một kết nối đến message broker/dịch vụ bên ngoài của mình và đăng ký subscribers/lắng nghe các kênh cụ thể trong phương thức `listen()` (và sau đó xóa subscriptions & đóng kết nối trong phương thức teardown `close()`),
nhưng vì điều này đòi hỏi sự hiểu biết tốt về cách microservices Nest giao tiếp với nhau, chúng tôi khuyến nghị đọc chuỗi bài viết [này](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l).
Trong chương này thay vào đó, chúng ta sẽ tập trung vào các khả năng mà lớp `Server` cung cấp và cách bạn có thể tận dụng chúng để xây dựng các chiến lược tùy chỉnh.

Ví dụ, hãy nói rằng ở đâu đó trong ứng dụng của chúng ta, message handler sau được định nghĩa:

```typescript
@MessagePattern('echo')
echo(@Payload() data: object) {
  return data;
}
```

Message handler này sẽ được đăng ký tự động bởi runtime của Nest. Với lớp `Server`, bạn có thể thấy các pattern tin nhắn nào đã được đăng ký và cũng truy cập và thực thi các phương thức thực tế được gán cho chúng.
Để kiểm tra điều này, hãy thêm một `console.log` đơn giản bên trong phương thức `listen()` trước khi hàm `callback` được gọi:

```typescript
listen(callback: () => void) {
  console.log(this.messageHandlers);
  callback();
}
```

Sau khi ứng dụng của bạn khởi động lại, bạn sẽ thấy log sau trong terminal của mình:

```typescript
Map { 'echo' => [AsyncFunction] { isEventHandler: false } }
```

> info **Gợi ý** Nếu chúng ta sử dụng decorator `@EventPattern`, bạn sẽ thấy cùng một output, nhưng với thuộc tính `isEventHandler` được đặt thành `true`.

Như bạn có thể thấy, thuộc tính `messageHandlers` là một tập hợp `Map` của tất cả các message (và event) handlers, trong đó patterns được sử dụng làm keys.
Bây giờ, bạn có thể sử dụng một key (ví dụ, `"echo"`) để nhận một tham chiếu đến message handler:

```typescript
async listen(callback: () => void) {
  const echoHandler = this.messageHandlers.get('echo');
  console.log(await echoHandler('Hello world!'));
  callback();
}
```

Khi chúng ta thực thi `echoHandler` truyền một chuỗi tùy ý làm đối số (`"Hello world!"` ở đây), chúng ta nên thấy nó trong console:

```json
Hello world!
```

Điều này có nghĩa là method handler của chúng ta đã được thực thi đúng cách.

Khi sử dụng `CustomTransportStrategy` với [Interceptors](/interceptors), các handlers được bọc vào các RxJS streams. Điều này có nghĩa là bạn cần subscribe vào chúng để thực thi logic underlying của streams (ví dụ, tiếp tục vào logic controller sau khi một interceptor đã được thực thi).

Một ví dụ về điều này có thể được thấy dưới đây:

```typescript
async listen(callback: () => void) {
  const echoHandler = this.messageHandlers.get('echo');
  const streamOrResult = await echoHandler('Hello World');
  if (isObservable(streamOrResult)) {
    streamOrResult.subscribe();
  }
  callback();
}
```

#### Client proxy

Như chúng ta đã đề cập trong phần đầu tiên, bạn không nhất thiết phải sử dụng gói `@nestjs/microservices` để tạo microservices, nhưng nếu bạn quyết định làm như vậy và bạn cần tích hợp một chiến lược tùy chỉnh, bạn sẽ cần cung cấp một lớp "client" cũng vậy.

> info **Gợi ý** Một lần nữa, triển khai một lớp client đầy đủ tính năng tương thích với tất cả các tính năng `@nestjs/microservices` (ví dụ, streaming) đòi hỏi sự hiểu biết tốt về các kỹ thuật giao tiếp được sử dụng bởi framework. Để tìm hiểu thêm, hãy kiểm tra bài viết [này](https://dev.to/nestjs/part-4-basic-client-component-16f9).

Để giao tiếp với một dịch vụ bên ngoài/emit & xuất bản tin nhắn (hoặc sự kiện), bạn có thể sử dụng gói SDK cụ thể cho thư viện, hoặc triển khai một lớp client tùy chỉnh mở rộng `ClientProxy`, như sau:

```typescript
import { ClientProxy, ReadPacket, WritePacket } from '@nestjs/microservices';

class GoogleCloudPubSubClient extends ClientProxy {
  async connect(): Promise<any> {}
  async close() {}
  async dispatchEvent(packet: ReadPacket<any>): Promise<any> {}
  publish(
    packet: ReadPacket<any>,
    callback: (packet: WritePacket<any>) => void,
  ): Function {}
  unwrap<T = never>(): T {
    throw new Error('Method not implemented.');
  }
}
```

> warning **Cảnh báo** Xin lưu ý, chúng tôi sẽ không triển khai một Google Cloud Pub/Sub client đầy đủ tính năng trong chương này vì điều này sẽ yêu cầu đi sâu vào các chi tiết kỹ thuật cụ thể của transporter.

Như bạn có thể thấy, lớp `ClientProxy` yêu cầu chúng ta cung cấp một số phương thức để thiết lập & đóng kết nối và xuất bản tin nhắn (`publish`) và sự kiện (`dispatchEvent`).
Lưu ý, nếu bạn không cần hỗ trợ phong cách giao tiếp request-response, bạn có thể để phương thức `publish()` trống. Tương tự, nếu bạn không cần hỗ trợ giao tiếp dựa trên sự kiện, bỏ qua phương thức `dispatchEvent()`.

Để quan sát những gì và khi nào những phương thức này được thực thi, hãy thêm nhiều cuộc gọi `console.log`, như sau:

```typescript
class GoogleCloudPubSubClient extends ClientProxy {
  async connect(): Promise<any> {
    console.log('connect');
  }

  async close() {
    console.log('close');
  }

  async dispatchEvent(packet: ReadPacket<any>): Promise<any> {
    return console.log('event to dispatch: ', packet);
  }

  publish(
    packet: ReadPacket<any>,
    callback: (packet: WritePacket<any>) => void,
  ): Function {
    console.log('message:', packet);

    // Trong một ứng dụng thực tế, hàm "callback" nên được thực thi
    // với payload được gửi lại từ responder. Ở đây, chúng ta sẽ chỉ mô phỏng (độ trễ 5 giây)
    // rằng phản hồi đã đến bằng cách truyền cùng một "data" như chúng ta đã truyền ban đầu.
    //
    // Bool "isDisposed" trên WritePacket cho biết phản hồi rằng không có dữ liệu nào khác
    // được mong đợi. Nếu không được gửi hoặc là false, điều này sẽ chỉ emit dữ liệu đến Observable.
    setTimeout(() => callback({
      response: packet.data,
      isDisposed: true,
    }), 5000);

    return () => console.log('teardown');
  }

  unwrap<T = never>(): T {
    throw new Error('Method not implemented.');
  }
}
```

Với điều này đã có sẵn, hãy tạo một thể hiện của lớp `GoogleCloudPubSubClient` và chạy phương thức `send()` (bạn có thể đã thấy trong các chương trước), subscribe vào stream observable được trả về.

```typescript
const googlePubSubClient = new GoogleCloudPubSubClient();
googlePubSubClient
  .send('pattern', 'Hello world!')
  .subscribe((response) => console.log(response));
```

Bây giờ, bạn nên thấy output sau trong terminal của mình:

```typescript
connect
message: { pattern: 'pattern', data: 'Hello world!' }
Hello world! // <-- sau 5 giây
```

Để kiểm tra xem phương thức "teardown" của chúng ta (mà phương thức `publish()` của chúng ta trả về) có được thực thi đúng cách không, hãy áp dụng một operator timeout cho stream của chúng ta, đặt nó thành 2 giây để đảm bảo nó ném sớm hơn khi `setTimeout` của chúng ta gọi hàm `callback`.

```typescript
const googlePubSubClient = new GoogleCloudPubSubClient();
googlePubSubClient
  .send('pattern', 'Hello world!')
  .pipe(timeout(2000))
  .subscribe(
    (response) => console.log(response),
    (error) => console.error(error.message),
  );
```

> info **Gợi ý** Operator `timeout` được import từ gói `rxjs/operators`.

Với operator `timeout` được áp dụng, output terminal của bạn sẽ trông như sau:

```typescript
connect
message: { pattern: 'pattern', data: 'Hello world!' }
teardown // <-- teardown
Timeout has occurred
```

Để dispatch một sự kiện (thay vì gửi một tin nhắn), sử dụng phương thức `emit()`:

```typescript
googlePubSubClient.emit('event', 'Hello world!');
```

Và đây là những gì bạn sẽ thấy trong console:

```typescript
connect
event to dispatch:  { pattern: 'event', data: 'Hello world!' }
```

#### Serialization tin nhắn

Nếu bạn cần thêm một số logic tùy chỉnh xung quanh serialization của các phản hồi ở phía client, bạn có thể sử dụng một lớp tùy chỉnh mở rộng lớp `ClientProxy` hoặc một trong các lớp con của nó. Để sửa đổi các yêu cầu thành công, bạn có thể override phương thức `serializeResponse`, và để sửa đổi bất kỳ lỗi nào đi qua client này, bạn có thể override phương thức `serializeError`. Để sử dụng lớp tùy chỉnh này, bạn có thể truyền chính lớp đó vào phương thức `ClientsModule.register()` bằng thuộc tính `customClass`. Dưới đây là một ví dụ về một `ClientProxy` tùy chỉnh serialize mỗi lỗi thành một `RpcException`.

```typescript
@@filename(error-handling.proxy)
import { ClientTCP, RpcException } from '@nestjs/microservices';

class ErrorHandlingProxy extends ClientTCP {
  serializeError(err: Error) {
    return new RpcException(err);
  }
}
```

và sau đó sử dụng nó trong `ClientsModule` như sau:

```typescript
@@filename(app.module)
@Module({
  imports: [
    ClientsModule.register([{
      name: 'CustomProxy',
      customClass: ErrorHandlingProxy,
    }]),
  ]
})
export class AppModule
```

> info **Gợi ý** Đây là chính lớp đó được truyền vào `customClass`, không phải một thể hiện của lớp. Nest sẽ tạo thể hiện dưới mui cho bạn, và sẽ truyền bất kỳ tùy chọn nào được đưa đến thuộc tính `options` đến `ClientProxy` mới.