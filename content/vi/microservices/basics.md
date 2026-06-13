### Tổng quan

Ngoài các kiến trúc ứng dụng truyền thống (đôi khi được gọi là monolithic), Nest hỗ trợ sẵn phong cách phát triển kiến trúc microservice. Hầu hết các khái niệm được thảo luận ở nơi khác trong tài liệu này, chẳng hạn như dependency injection, decorators, exception filters, pipes, guards và interceptors, đều áp dụng tương đương cho microservices. Bất cứ khi nào có thể, Nest trừu tượng hóa các chi tiết triển khai để cùng một thành phần có thể chạy trên các nền tảng dựa trên HTTP, WebSockets và Microservices. Phần này bao gồm các khía cạnh của Nest đặc thù cho microservices.

Trong Nest, microservice về cơ bản là một ứng dụng sử dụng một lớp **transport** khác với HTTP.

<figure><img class="illustrative-image" src="/assets/Microservices_1.png" /></figure>

Nest hỗ trợ một số triển khai lớp transport tích hợp sẵn, được gọi là **transporters**, chịu trách nhiệm truyền tải tin nhắn giữa các thể hiện microservice khác nhau. Hầu hết các transporter hỗ trợ sẵn cả phong cách tin nhắn **request-response** và **event-based**. Nest trừu tượng hóa chi tiết triển khai của từng transporter phía sau một interface chính tắc cho cả tin nhắn request-response và event-based. Điều này giúp dễ dàng chuyển đổi từ một lớp transport sang lớp khác — ví dụ để tận dụng các tính năng độ tin cậy hoặc hiệu suất cụ thể của một lớp transport cụ thể — mà không ảnh hưởng đến mã ứng dụng của bạn.

#### Cài đặt

Để bắt đầu xây dựng microservices, trước tiên hãy cài đặt gói cần thiết:

```bash
$ npm i --save @nestjs/microservices
```

#### Bắt đầu

Để khởi tạo một microservice, sử dụng phương thức `createMicroservice()` của lớp `NestFactory`:

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
    },
  );
  await app.listen();
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.TCP,
  });
  await app.listen();
}
bootstrap();
```

> info **Gợi ý** Microservices sử dụng lớp transport **TCP** theo mặc định.

Đối số thứ hai của phương thức `createMicroservice()` là một đối tượng `options`. Đối tượng này có thể bao gồm hai thành viên:

<table>
  <tr>
    <td><code>transport</code></td>
    <td>Chỉ định transporter (ví dụ, <code>Transport.NATS</code>)</td>
  </tr>
  <tr>
    <td><code>options</code></td>
    <td>Một đối tượng options cụ thể cho transporter xác định hành vi của transporter</td>
  </tr>
</table>
<p>
  Đối tượng <code>options</code> là cụ thể cho transporter được chọn. Transporter <strong>TCP</strong> expose
  các thuộc tính được mô tả dưới đây.  Đối với các transporter khác (ví dụ, Redis, MQTT, v.v.), xem chương liên quan để biết mô tả về các tùy chọn có sẵn.
</p>
<table>
  <tr>
    <td><code>host</code></td>
    <td>Hostname kết nối</td>
  </tr>
  <tr>
    <td><code>port</code></td>
    <td>Port kết nối</td>
  </tr>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>Số lần thử lại tin nhắn (mặc định: <code>0</code>)</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>Độ trễ giữa các lần thử lại tin nhắn (ms) (mặc định: <code>0</code>)</td>
  </tr>
  <tr>
    <td><code>serializer</code></td>
    <td><a href="https://github.com/nestjs/nest/blob/master/packages/microservices/interfaces/serializer.interface.ts" target="_blank">Serializer</a> tùy chỉnh cho tin nhắn đi</td>
  </tr>
  <tr>
    <td><code>deserializer</code></td>
    <td><a href="https://github.com/nestjs/nest/blob/master/packages/microservices/interfaces/deserializer.interface.ts" target="_blank">Deserializer</a> tùy chỉnh cho tin nhắn đến</td>
  </tr>
  <tr>
    <td><code>socketClass</code></td>
    <td>Một Socket tùy chỉnh mở rộng <code>TcpSocket</code> (mặc định: <code>JsonSocket</code>)</td>
  </tr>
  <tr>
    <td><code>tlsOptions</code></td>
    <td>Tùy chọn để cấu hình giao thức tls</td>
  </tr>
</table>

> info **Gợi ý** Các thuộc tính trên là cụ thể cho transporter TCP. Để biết thông tin về các tùy chọn có sẵn cho các transporter khác, tham khảo chương liên quan.

#### Pattern tin nhắn và sự kiện

Microservices nhận ra cả tin nhắn và sự kiện bằng **patterns**. Pattern là một giá trị đơn giản, ví dụ, một đối tượng literal hoặc một chuỗi. Patterns được tự động serialize và gửi qua mạng cùng với phần dữ liệu của tin nhắn. Bằng cách này, người gửi tin nhắn và người tiêu dùng có thể phối hợp các yêu cầu nào được tiêu dùng bởi các handler nào.

#### Request-response

Phong cách tin nhắn request-response rất hữu ích khi bạn cần **trao đổi** tin nhắn giữa các dịch vụ bên ngoài khác nhau. Paradigm này đảm bảo rằng dịch vụ thực sự đã nhận được tin nhắn (mà không yêu cầu bạn triển khai thủ công một giao thức xác nhận). Tuy nhiên, cách tiếp cận request-response có thể không phải lúc nào cũng phù hợp nhất. Ví dụ, các transporter streaming, chẳng hạn như [Kafka](https://docs.confluent.io/3.0.0/streams/) hoặc [NATS streaming](https://github.com/nats-io/node-nats-streaming), sử dụng tính bền vững dựa trên log, được tối ưu hóa để giải quyết một tập hợp các thách thức khác, phù hợp hơn với paradigm tin nhắn sự kiện (xem [event-based messaging](https://docs.nestjs.com/microservices/basics#event-based) để biết thêm chi tiết).

Để kích hoạt kiểu tin nhắn request-response, Nest tạo hai kênh logic: một để chuyển dữ liệu và một khác để chờ phản hồi đến. Đối với một số transport cơ bản, như [NATS](https://nats.io/), hỗ trợ kênh kép này được cung cấp sẵn. Đối với những cái khác, Nest bù đắp bằng cách tạo thủ công các kênh riêng biệt. Mặc dù điều này hiệu quả, nó có thể gây ra một số chi phí. Do đó, nếu bạn không yêu cầu phong cách tin nhắn request-response, bạn có thể muốn xem xét sử dụng phương pháp event-based.

Để tạo một message handler dựa trên paradigm request-response, sử dụng decorator `@MessagePattern()`, được import từ gói `@nestjs/microservices`. Decorator này chỉ nên được sử dụng trong các lớp [controller](https://docs.nestjs.com/controllers), vì chúng đóng vai trò là các điểm nhập cho ứng dụng của bạn. Sử dụng nó trong providers sẽ không có hiệu lực, vì chúng sẽ bị bỏ qua bởi runtime của Nest.

```typescript
@@filename(math.controller)
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data: number[]): number {
    return (data || []).reduce((a, b) => a + b);
  }
}
@@switch
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data) {
    return (data || []).reduce((a, b) => a + b);
  }
}
```

Trong mã trên, `accumulate()` **message handler** lắng nghe các tin nhắn khớp với pattern tin nhắn `{{ '{' }} cmd: 'sum' {{ '}' }}`. Message handler nhận một đối số duy nhất, `data` được truyền từ client. Trong trường hợp này, dữ liệu là một mảng các số cần được tích lũy.

#### Phản hồi không đồng bộ

Message handlers có thể phản hồi đồng bộ hoặc **không đồng bộ**, nghĩa là các phương thức `async` được hỗ trợ.

```typescript
@@filename()
@MessagePattern({ cmd: 'sum' })
async accumulate(data: number[]): Promise<number> {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@MessagePattern({ cmd: 'sum' })
async accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```

Message handler cũng có thể trả về một `Observable`, trong trường hợp đó các giá trị kết quả sẽ được emit cho đến khi stream hoàn thành.

```typescript
@@filename()
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): Observable<number> {
  return from([1, 2, 3]);
}
@@switch
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): Observable<number> {
  return from([1, 2, 3]);
}
```

Trong ví dụ trên, message handler sẽ phản hồi **ba lần**, một lần cho mỗi mục trong mảng.

#### Event-based

Mặc dù phương pháp request-response hoàn hảo để trao đổi tin nhắn giữa các dịch vụ, nó ít phù hợp hơn cho tin nhắn dựa trên sự kiện — khi bạn chỉ muốn xuất bản **sự kiện** mà không chờ phản hồi. Trong những trường hợp như vậy, chi phí duy trì hai kênh cho request-response là không cần thiết.

Ví dụ, nếu bạn muốn thông báo cho một dịch vụ khác rằng một điều kiện cụ thể đã xảy ra trong phần này của hệ thống, phong cách tin nhắn event-based là lý tưởng.

Để tạo một event handler, bạn có thể sử dụng decorator `@EventPattern()`, được import từ gói `@nestjs/microservices`.

```typescript
@@filename()
@EventPattern('user_created')
async handleUserCreated(data: Record<string, unknown>) {
  // business logic
}
@@switch
@EventPattern('user_created')
async handleUserCreated(data) {
  // business logic
}
```

> info **Gợi ý** Bạn có thể đăng ký nhiều event handler cho một **pattern sự kiện duy nhất**, và tất cả chúng sẽ được kích hoạt tự động song song.

`handleUserCreated()` **event handler** lắng nghe sự kiện `'user_created'`. Event handler nhận một đối số duy nhất, `data` được truyền từ client (trong trường hợp này, một event payload đã được gửi qua mạng).

<app-banner-enterprise></app-banner-enterprise>

#### Chi tiết yêu cầu bổ sung

Trong các kịch bản nâng cao hơn, bạn có thể cần truy cập các chi tiết bổ sung về yêu cầu đến. Ví dụ, khi sử dụng NATS với các subscription wildcard, bạn có thể muốn truy xuất chủ đề gốc mà producer đã gửi tin nhắn đến. Tương tự, với Kafka, bạn có thể cần truy cập các header của tin nhắn. Để đạt được điều này, bạn có thể tận dụng các decorator tích hợp sẵn như được hiển thị dưới đây:

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

> info **Gợi ý** `@Payload()`, `@Ctx()` và `NatsContext` được import từ `@nestjs/microservices`.

> info **Gợi ý** Bạn cũng có thể truyền một khóa thuộc tính vào decorator `@Payload()` để trích xuất một thuộc tính cụ thể từ đối tượng payload đến, ví dụ, `@Payload('id')`.

#### Client (lớp producer)

Một ứng dụng Nest client có thể trao đổi tin nhắn hoặc xuất bản sự kiện đến một microservice Nest bằng lớp `ClientProxy`. Lớp này cung cấp một số phương thức, chẳng hạn như `send()` (cho tin nhắn request-response) và `emit()` (cho tin nhắn dựa trên sự kiện), cho phép giao tiếp với một microservice từ xa. Bạn có thể lấy một thể hiện của lớp này theo các cách sau:

Một cách tiếp cận là import `ClientsModule`, expose phương thức tĩnh `register()`. Phương thức này nhận một mảng các đối tượng đại diện cho các transporter microservice. Mỗi đối tượng phải bao gồm một thuộc tính `name`, và tùy chọn một thuộc tính `transport` (mặc định là `Transport.TCP`), cũng như một thuộc tính `options` tùy chọn.

Thuộc tính `name` đóng vai trò là một **injection token**, mà bạn có thể sử dụng để inject một thể hiện của `ClientProxy` ở bất cứ đâu cần thiết. Giá trị của thuộc tính `name` này có thể là bất kỳ chuỗi hoặc JavaScript symbol tùy ý, như được mô tả [ở đây](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens).

Thuộc tính `options` là một đối tượng bao gồm các thuộc tính giống nhau mà chúng ta đã thấy trong phương thức `createMicroservice()` trước đó.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      { name: 'MATH_SERVICE', transport: Transport.TCP },
    ]),
  ],
})
```

Ngoài ra, bạn có thể sử dụng phương thức `registerAsync()` nếu bạn cần cung cấp cấu hình hoặc thực hiện bất kỳ quy trình không đồng bộ nào khác trong quá trình thiết lập.

```typescript
@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        imports: [ConfigModule],
        name: 'MATH_SERVICE',
        useFactory: async (configService: ConfigService) => ({
          transport: Transport.TCP,
          options: {
            url: configService.get('URL'),
          },
        }),
        inject: [ConfigService],
      },
    ]),
  ],
})
```

Sau khi module đã được import, bạn có thể inject một thể hiện của `ClientProxy` được cấu hình với các tùy chọn được chỉ định cho transporter `'MATH_SERVICE'` bằng decorator `@Inject()`.

```typescript
constructor(
  @Inject('MATH_SERVICE') private client: ClientProxy,
) {}
```

> info **Gợi ý** Các lớp `ClientsModule` và `ClientProxy` được import từ gói `@nestjs/microservices`.

Đôi khi, bạn có thể cần lấy cấu hình transporter từ một dịch vụ khác (như `ConfigService`), thay vì hard-code nó trong ứng dụng client của bạn. Để đạt được điều này, bạn có thể đăng ký một [custom provider](/fundamentals/custom-providers) bằng lớp `ClientProxyFactory`. Lớp này cung cấp phương thức tĩnh `create()` chấp nhận một đối tượng options transporter và trả về một thể hiện `ClientProxy` tùy chỉnh.

```typescript
@Module({
  providers: [
    {
      provide: 'MATH_SERVICE',
      useFactory: (configService: ConfigService) => {
        const mathSvcOptions = configService.getMathSvcOptions();
        return ClientProxyFactory.create(mathSvcOptions);
      },
      inject: [ConfigService],
    }
  ]
  ...
})
```

> info **Gợi ý** `ClientProxyFactory` được import từ gói `@nestjs/microservices`.

Một tùy chọn khác là sử dụng decorator thuộc tính `@Client()`.

```typescript
@Client({ transport: Transport.TCP })
client: ClientProxy;
```

> info **Gợi ý** Decorator `@Client()` được import từ gói `@nestjs/microservices`.

Sử dụng decorator `@Client()` không phải là kỹ thuật được ưu tiên, vì nó khó kiểm tra hơn và khó chia sẻ một thể hiện client hơn.

`ClientProxy` là **lazy**. Nó không khởi tạo kết nối ngay lập tức. Thay vào đó, nó sẽ được thiết lập trước cuộc gọi microservice đầu tiên, và sau đó được tái sử dụng qua từng cuộc gọi tiếp theo. Tuy nhiên, nếu bạn muốn trì hoãn quá trình bootstrap ứng dụng cho đến khi kết nối được thiết lập, bạn có thể khởi tạo thủ công một kết nối bằng phương thức `connect()` của đối tượng `ClientProxy` bên trong lifecycle hook `OnApplicationBootstrap`.

```typescript
@@filename()
async onApplicationBootstrap() {
  await this.client.connect();
}
```

Nếu kết nối không thể được tạo, phương thức `connect()` sẽ reject với đối tượng lỗi tương ứng.

#### Gửi tin nhắn

`ClientProxy` expose phương thức `send()`. Phương thức này nhằm gọi microservice và trả về một `Observable` với phản hồi của nó. Do đó, chúng ta có thể dễ dàng subscribe vào các giá trị được emit.

```typescript
@@filename()
accumulate(): Observable<number> {
  const pattern = { cmd: 'sum' };
  const payload = [1, 2, 3];
  return this.client.send<number>(pattern, payload);
}
@@switch
accumulate() {
  const pattern = { cmd: 'sum' };
  const payload = [1, 2, 3];
  return this.client.send(pattern, payload);
}
```

Phương thức `send()` nhận hai đối số, `pattern` và `payload`. `pattern` nên khớp với một được định nghĩa trong decorator `@MessagePattern()`. `payload` là một tin nhắn mà chúng ta muốn truyền tải đến microservice từ xa. Phương thức này trả về một **cold `Observable`**, nghĩa là bạn phải explicitly subscribe vào nó trước khi tin nhắn được gửi.

#### Xuất bản sự kiện

Để gửi một sự kiện, sử dụng phương thức `emit()` của đối tượng `ClientProxy`. Phương thức này xuất bản một sự kiện đến message broker.

```typescript
@@filename()
async publish() {
  this.client.emit<number>('user_created', new UserCreatedEvent());
}
@@switch
async publish() {
  this.client.emit('user_created', new UserCreatedEvent());
}
```

Phương thức `emit()` nhận hai đối số: `pattern` và `payload`. `pattern` nên khớp với một được định nghĩa trong decorator `@EventPattern()`, trong khi `payload` đại diện cho dữ liệu sự kiện mà bạn muốn truyền tải đến microservice từ xa. Phương thức này trả về một **hot `Observable`** (trái ngược với cold `Observable` được trả về bởi `send()`), nghĩa là bất kể bạn có explicitly subscribe vào observable hay không, proxy sẽ ngay lập tức cố gắng giao sự kiện.

<app-banner-devtools></app-banner-devtools>

#### Request-scoping

Đối với những người đến từ các nền tảng ngôn ngữ lập trình khác, có thể ngạc nhiên khi biết rằng trong Nest, hầu hết mọi thứ được chia sẻ qua các yêu cầu đến. Điều này bao gồm một pool kết nối đến cơ sở dữ liệu, các dịch vụ singleton với trạng thái toàn cục, và nhiều hơn nữa. Hãy nhớ rằng Node.js không theo mô hình stateless đa luồng request/response, nơi mỗi yêu cầu được xử lý bởi một luồng riêng biệt. Kết quả là, sử dụng các thể hiện singleton là **an toàn** cho các ứng dụng của chúng ta.

Tuy nhiên, có các trường hợp cạnh mà vòng đời dựa trên yêu cầu cho handler có thể là mong muốn. Điều này có thể bao gồm các kịch bản như caching theo yêu cầu trong các ứng dụng GraphQL, theo dõi yêu cầu, hoặc multi-tenancy. Bạn có thể tìm hiểu thêm về cách kiểm soát scopes [ở đây](/fundamentals/injection-scopes).

Handlers và providers request-scoped có thể inject `RequestContext` bằng decorator `@Inject()` kết hợp với token `CONTEXT`:

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT, RequestContext } from '@nestjs/microservices';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private ctx: RequestContext) {}
}
```

Điều này cung cấp quyền truy cập vào đối tượng `RequestContext`, có hai thuộc tính:

```typescript
export interface RequestContext<T = any> {
  pattern: string | Record<string, any>;
  data: T;
}
```

Thuộc tính `data` là message payload được gửi bởi message producer. Thuộc tính `pattern` là pattern được sử dụng để xác định một handler phù hợp để xử lý tin nhắn đến.

#### Cập nhật trạng thái thể hiện

Để nhận cập nhật thời gian thực về kết nối và trạng thái của thể hiện driver cơ bản, bạn có thể subscribe vào stream `status`. Stream này cung cấp các cập nhật trạng thái cụ thể cho driver được chọn. Ví dụ, nếu bạn đang sử dụng transporter TCP (mặc định), stream `status` emit các sự kiện `connected` và `disconnected`.

```typescript
this.client.status.subscribe((status: TcpStatus) => {
  console.log(status);
});
```

> info **Gợi ý** Kiểu `TcpStatus` được import từ gói `@nestjs/microservices`.

Tương tự, bạn có thể subscribe vào stream `status` của server để nhận thông báo về trạng thái của server.

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: TcpStatus) => {
  console.log(status);
});
```

#### Lắng nghe các sự kiện nội bộ

Trong một số trường hợp, bạn có thể muốn lắng nghe các sự kiện nội bộ được emit bởi microservice. Ví dụ, bạn có thể lắng nghe sự kiện `error` để kích hoạt các thao tác bổ sung khi xảy ra lỗi. Để làm điều này, sử dụng phương thức `on()`, như được hiển thị dưới đây:

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

Tương tự, bạn có thể lắng nghe các sự kiện nội bộ của server:

```typescript
server.on<TcpEvents>('error', (err) => {
  console.error(err);
});
```

> info **Gợi ý** Kiểu `TcpEvents` được import từ gói `@nestjs/microservices`.

#### Truy cập driver cơ bản

Đối với các trường hợp sử dụng nâng cao hơn, bạn có thể cần truy cập thể hiện driver cơ bản. Điều này có thể hữu ích cho các kịch bản như đóng thủ công kết nối hoặc sử dụng các phương thức cụ thể của driver. Tuy nhiên, hãy nhớ rằng đối với hầu hết các trường hợp, bạn **không nên cần** truy cập trực tiếp driver.

Để làm điều này, bạn có thể sử dụng phương thức `unwrap()`, trả về thể hiện driver cơ bản. Tham số kiểu generic nên chỉ định loại thể hiện driver bạn mong đợi.

```typescript
const netServer = this.client.unwrap<Server>();
```

Ở đây, `Server` là một kiểu được import từ module `net`.

Tương tự, bạn có thể truy cập thể hiện driver cơ bản của server:

```typescript
const netServer = server.unwrap<Server>();
```

#### Xử lý timeout

Trong các hệ thống phân tán, microservices có thể đôi khi bị down hoặc không khả dụng. Để ngăn chặn việc chờ đợi vô thời hạn, bạn có thể sử dụng timeouts. Timeout là một mô hình rất hữu ích khi giao tiếp với các dịch vụ khác. Để áp dụng timeouts cho các cuộc gọi microservice của bạn, bạn có thể sử dụng operator `timeout` của [RxJS](https://rxjs.dev). Nếu microservice không phản hồi trong thời gian được chỉ định, một ngoại lệ được ném, mà bạn có thể bắt và xử lý phù hợp.

Để triển khai điều này, bạn sẽ cần sử dụng gói [`rxjs`](https://github.com/ReactiveX/rxjs). Chỉ cần sử dụng operator `timeout` bên trong pipe:

```typescript
@@filename()
this.client
  .send<TResult, TInput>(pattern, data)
  .pipe(timeout(5000));
@@switch
this.client
  .send(pattern, data)
  .pipe(timeout(5000));
```

> info **Gợi ý** Operator `timeout` được import từ gói `rxjs/operators`.

Sau 5 giây, nếu microservice không phản hồi, nó sẽ ném một lỗi.

#### Hỗ trợ TLS

Khi giao tiếp bên ngoài một mạng riêng tư, điều quan trọng là phải mã hóa lưu lượng để đảm bảo bảo mật. Trong NestJS, điều này có thể đạt được với TLS qua TCP bằng module [TLS](https://nodejs.org/api/tls.html) tích hợp sẵn của Node. Nest cung cấp hỗ trợ tích hợp sẵn cho TLS trong transport TCP của nó, cho phép chúng ta mã hóa giao tiếp giữa microservices hoặc clients.

Để kích hoạt TLS cho một server TCP, bạn sẽ cần cả một private key và một certificate ở định dạng PEM. Các thứ này được thêm vào options của server bằng cách thiết lập `tlsOptions` và chỉ định các file key và cert, như được hiển thị dưới đây:

```typescript
import * as fs from 'fs';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

async function bootstrap() {
  const key = fs.readFileSync('<pathToKeyFile>', 'utf8').toString();
  const cert = fs.readFileSync('<pathToCertFile>', 'utf8').toString();

  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
      options: {
        tlsOptions: {
          key,
          cert,
        },
      },
    },
  );

  await app.listen();
}
bootstrap();
```

Để một client giao tiếp an toàn qua TLS, chúng ta cũng định nghĩa đối tượng `tlsOptions` nhưng lần này với certificate CA. Đây là certificate của authority đã ký certificate của server. Điều này đảm bảo rằng client tin tưởng certificate của server và có thể thiết lập một kết nối an toàn.

```typescript
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.TCP,
        options: {
          tlsOptions: {
            ca: [fs.readFileSync('<pathToCaFile>', 'utf-8').toString()],
          },
        },
      },
    ]),
  ],
})
export class AppModule {}
```

Bạn cũng có thể truyền một mảng các CA nếu thiết lập của bạn liên quan đến nhiều authority được tin cậy.

Sau khi mọi thứ được thiết lập, bạn có thể inject `ClientProxy` như bình thường bằng decorator `@Inject()` để sử dụng client trong các dịch vụ của bạn. Điều này đảm bảo giao tiếp được mã hóa qua các microservices NestJS của bạn, với module `TLS` của Node xử lý các chi tiết mã hóa.

Để biết thêm thông tin, tham khảo [tài liệu TLS](https://nodejs.org/api/tls.html) của Node.

#### Cấu hình động

Khi một microservice cần được cấu hình bằng `ConfigService` (từ gói `@nestjs/config`), nhưng context injection chỉ có sẵn sau khi thể hiện microservice được tạo, `AsyncMicroserviceOptions` cung cấp một giải pháp. Cách tiếp cận này cho phép cấu hình động, đảm bảo tích hợp mượt mà với `ConfigService`.

```typescript
import { ConfigService } from '@nestjs/config';
import { AsyncMicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<AsyncMicroserviceOptions>(
    AppModule,
    {
      useFactory: (configService: ConfigService) => ({
        transport: Transport.TCP,
        options: {
          host: configService.get<string>('HOST'),
          port: configService.get<number>('PORT'),
        },
      }),
      inject: [ConfigService],
    },
  );

  await app.listen();
}
bootstrap();
```