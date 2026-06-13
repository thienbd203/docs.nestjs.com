### gRPC

[gRPC](https://github.com/grpc/grpc-node) là một framework RPC hiện đại, open source, hiệu suất cao có thể chạy trong bất kỳ môi trường nào. Nó có thể kết nối hiệu quả các dịch vụ trong và qua các trung tâm dữ liệu với hỗ trợ có thể cắm (pluggable) cho load balancing, tracing, health checking và authentication.

Giống như nhiều hệ thống RPC, gRPC dựa trên khái niệm định nghĩa một dịch vụ theo các hàm (phương thức) có thể được gọi từ xa. Đối với mỗi phương thức, bạn định nghĩa các tham số và loại trả về. Dịch vụ, tham số và loại trả về được định nghĩa trong các tệp `.proto` sử dụng cơ chế <a href="https://protobuf.dev">protocol buffers</a> trung lập ngôn ngữ open source của Google.

Với transporter gRPC, Nest sử dụng các tệp `.proto` để động liên kết clients và servers để giúp triển khai dễ dàng các cuộc gọi thủ tục từ xa, tự động serialize và deserialize dữ liệu có cấu trúc.

#### Cài đặt

Để bắt đầu xây dựng microservices dựa trên gRPC, trước tiên hãy cài đặt các gói cần thiết:

```bash
$ npm i --save @grpc/grpc-js @grpc/proto-loader
```

#### Tổng quan

Giống như các triển khai lớp transport microservices Nest khác, bạn chọn cơ chế transporter gRPC bằng thuộc tính `transport` của đối tượng options được truyền đến phương thức `createMicroservice()`. Trong ví dụ sau, chúng ta sẽ thiết lập một hero service. Thuộc tính `options` cung cấp metadata về dịch vụ đó; các thuộc tính của nó được mô tả <a href="microservices/grpc#options">dưới đây</a>.

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.GRPC,
  options: {
    package: 'hero',
    protoPath: join(__dirname, 'hero/hero.proto'),
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.GRPC,
  options: {
    package: 'hero',
    protoPath: join(__dirname, 'hero/hero.proto'),
  },
});
```

> info **Gợi ý** Hàm `join()` được import từ gói `path`; enum `Transport` được import từ gói `@nestjs/microservices`.

Trong tệp `nest-cli.json`, chúng ta thêm thuộc tính `assets` cho phép chúng ta phân phối các tệp không phải TypeScript, và `watchAssets` - để bật watching tất cả các tài sản không phải TypeScript. Trong trường hợp của chúng ta, chúng ta muốn các tệp `.proto` được tự động sao chép vào thư mục `dist`.

```json
{
  "compilerOptions": {
    "assets": ["**/*.proto"],
    "watchAssets": true
  }
}
```

#### Tùy chọn

Đối tượng tùy chọn transporter <strong>gRPC</strong> expose các thuộc tính được mô tả dưới đây.

<table>
  <tr>
    <td><code>package</code></td>
    <td>Tên gói Protobuf (khớp với thiết lập <code>package</code> từ tệp <code>.proto</code>). Bắt buộc</td>
  </tr>
  <tr>
    <td><code>protoPath</code></td>
    <td>
      Đường dẫn tuyệt đối (hoặc tương đối với thư mục gốc) đến tệp
      <code>.proto</code>. Bắt buộc
    </td>
  </tr>
  <tr>
    <td><code>url</code></td>
    <td>Url kết nối. Chuỗi ở định dạng <code>địa chỉ ip/tên dns:port</code> (ví dụ, <code>'0.0.0.0:50051'</code> cho một Docker server) định nghĩa địa chỉ/port mà transporter thiết lập kết nối. Tùy chọn. Mặc định là <code>'localhost:5000'</code></td>
  </tr>
  <tr>
    <td><code>protoLoader</code></td>
    <td>Tên gói NPM cho tiện ích tải các tệp <code>.proto</code>. Tùy chọn. Mặc định là <code>'@grpc/proto-loader'</code></td>
  </tr>
  <tr>
    <td><code>loader</code></td>
    <td>
      Tùy chọn <code>@grpc/proto-loader</code>. Các thứ này cung cấp kiểm soát chi tiết về hành vi của các tệp <code>.proto</code>. Tùy chọn. Xem
      <a
        href="https://github.com/grpc/grpc-node/blob/master/packages/proto-loader/README.md"
        rel="nofollow"
        target="_blank"
        >ở đây</a
      > để biết thêm chi tiết
    </td>
  </tr>
  <tr>
    <td><code>credentials</code></td>
    <td>
      Thông tin xác thực server. Tùy chọn. <a
        href="https://grpc.io/grpc/node/grpc.ServerCredentials.html"
        rel="nofollow"
        target="_blank"
        >Đọc thêm ở đây</a
      >
    </td>
  </tr>
</table>

#### Dịch vụ gRPC mẫu

Hãy định nghĩa dịch vụ gRPC mẫu của chúng ta gọi là `HeroesService`. Trong đối tượng `options` ở trên, thuộc tính `protoPath` đặt một đường dẫn đến tệp định nghĩa `.proto` `hero.proto`. Tệp `hero.proto` được cấu trúc sử dụng <a href="https://developers.google.com/protocol-buffers">protocol buffers</a>. Đây là cách nó trông:

```typescript
// hero/hero.proto
syntax = "proto3";

package hero;

service HeroesService {
  rpc FindOne (HeroById) returns (Hero) {}
}

message HeroById {
  int32 id = 1;
}

message Hero {
  int32 id = 1;
  string name = 2;
}
```

`HeroesService` của chúng ta expose một phương thức `FindOne()`. Phương thức này mong đợi một đối số đầu vào của loại `HeroById` và trả về một tin nhắn `Hero` (protocol buffers sử dụng các phần tử `message` để định nghĩa cả loại tham số và loại trả về).

Tiếp theo, chúng ta cần triển khai dịch vụ. Để định nghĩa một handler đáp ứng định nghĩa này, chúng ta sử dụng decorator `@GrpcMethod()` trong một controller, như được hiển thị dưới đây. Decorator này cung cấp metadata cần thiết để khai báo một phương thức như một phương thức dịch vụ gRPC.

> info **Gợi ý** Decorator `@MessagePattern()` (<a href="microservices/basics#request-response">đọc thêm</a>) được giới thiệu trong các chương microservices trước không được sử dụng với microservices dựa trên gRPC. Decorator `@GrpcMethod()` hiệu quả thay thế vị trí của nó cho microservices dựa trên gRPC.

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService', 'FindOne')
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService', 'FindOne')
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

> info **Gợi ý** Decorator `@GrpcMethod()` được import từ gói `@nestjs/microservices`, trong khi `Metadata` và `ServerUnaryCall` từ gói `grpc`.

Decorator được hiển thị ở trên nhận hai đối số. Đối số đầu tiên là tên dịch vụ (ví dụ, `'HeroesService'`), tương ứng với định nghĩa dịch vụ `HeroesService` trong `hero.proto`. Đối số thứ hai (chuỗi `'FindOne'`) tương ứng với phương thức rpc `FindOne()` được định nghĩa trong `HeroesService` trong tệp `hero.proto`.

Phương thức handler `findOne()` nhận ba đối số, `data` được truyền từ caller, `metadata` lưu trữ metadata yêu cầu gRPC và `call` để lấy các thuộc tính đối tượng `GrpcCall` như `sendMetadata` để gửi metadata đến client.

Cả hai đối số decorator `@GrpcMethod()` đều là tùy chọn. Nếu được gọi mà không có đối số thứ hai (ví dụ, `'FindOne'`), Nest sẽ tự động liên kết phương thức rpc tệp `.proto` với handler dựa trên việc chuyển đổi tên handler thành upper camel case (ví dụ, handler `findOne` được liên kết với định nghĩa cuộc gọi rpc `FindOne`). Điều này được hiển thị dưới đây.

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService')
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService')
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

Bạn cũng có thể bỏ qua đối số `@GrpcMethod()` đầu tiên. Trong trường hợp này, Nest tự động liên kết handler với định nghĩa dịch vụ từ tệp định nghĩa proto dựa trên tên **lớp** nơi handler được định nghĩa. Ví dụ, trong mã sau, lớp `HeroesService` liên kết các phương thức handler của nó với định nghĩa dịch vụ `HeroesService` trong tệp `hero.proto` dựa trên việc khớp tên `'HeroesService'`.

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

#### Client

Các ứng dụng Nest có thể hoạt động như gRPC clients, tiêu thụ các dịch vụ được định nghĩa trong các tệp `.proto`. Bạn truy cập các dịch vụ từ xa thông qua một đối tượng `ClientGrpc`. Bạn có thể lấy một đối tượng `ClientGrpc` theo một số cách.

Kỹ thuật được ưu tiên là import `ClientsModule`. Sử dụng phương thức `register()` để liên kết một gói dịch vụ được định nghĩa trong một tệp `.proto` với một injection token, và để cấu hình dịch vụ. Thuộc tính `name` là injection token. Đối với dịch vụ gRPC, sử dụng `transport: Transport.GRPC`. Thuộc tính `options` là một đối tượng với các thuộc tính giống nhau được mô tả <a href="microservices/grpc#options">ở trên</a>.

```typescript
imports: [
  ClientsModule.register([
    {
      name: 'HERO_PACKAGE',
      transport: Transport.GRPC,
      options: {
        package: 'hero',
        protoPath: join(__dirname, 'hero/hero.proto'),
      },
    },
  ]),
];
```

> info **Gợi ý** Phương thức `register()` nhận một mảng các đối tượng. Đăng ký nhiều gói bằng cách cung cấp một danh sách được phân tách bằng dấu phẩy của các đối tượng đăng ký.

Sau khi đăng ký, chúng ta có thể inject đối tượng `ClientGrpc` đã cấu hình với `@Inject()`. Sau đó chúng ta sử dụng phương thức `getService()` của đối tượng `ClientGrpc` để truy xuất thể hiện dịch vụ, như được hiển thị dưới đây.

```typescript
@Injectable()
export class AppService implements OnModuleInit {
  private heroesService: HeroesService;

  constructor(@Inject('HERO_PACKAGE') private client: ClientGrpc) {}

  onModuleInit() {
    this.heroesService = this.client.getService<HeroesService>('HeroesService');
  }

  getHero(): Observable<string> {
    return this.heroesService.findOne({ id: 1 });
  }
}
```

> error **Cảnh báo** gRPC Client sẽ không gửi các trường chứa dấu gạch dưới `_` trong tên của chúng trừ khi tùy chọn `keepCase` được đặt thành `true` trong cấu hình proto loader (`options.loader.keepcase` trong cấu hình transporter microservice).

Lưu ý rằng có một sự khác biệt nhỏ so với kỹ thuật được sử dụng trong các phương thức transport microservice khác. Thay vì lớp `ClientProxy`, chúng ta sử dụng lớp `ClientGrpc`, cung cấp phương thức `getService()`. Phương thức generic `getService()` nhận một tên dịch vụ làm đối số và trả về thể hiện của nó (nếu có sẵn).

Ngoài ra, bạn có thể sử dụng decorator `@Client()` để khởi tạo một đối tượng `ClientGrpc`, như sau:

```typescript
@Injectable()
export class AppService implements OnModuleInit {
  @Client({
    transport: Transport.GRPC,
    options: {
      package: 'hero',
      protoPath: join(__dirname, 'hero/hero.proto'),
    },
  })
  client: ClientGrpc;

  private heroesService: HeroesService;

  onModuleInit() {
    this.heroesService = this.client.getService<HeroesService>('HeroesService');
  }

  getHero(): Observable<string> {
    return this.heroesService.findOne({ id: 1 });
  }
}
```

Cuối cùng, đối với các kịch bản phức tạp hơn, chúng ta có thể inject một client được cấu hình động sử dụng lớp `ClientProxyFactory` như được mô tả <a href="/microservices/basics#client">ở đây</a>.

Trong cả hai trường hợp, chúng ta kết thúc với một tham chiếu đến đối tượng proxy `HeroesService` của chúng ta, expose cùng một tập hợp các phương thức được định nghĩa bên trong tệp `.proto`. Bây giờ, khi chúng ta truy cập đối tượng proxy này (tức là `heroesService`), hệ thống gRPC tự động serialize các yêu cầu, chuyển tiếp chúng đến hệ thống từ xa, trả về một phản hồi, và deserialize phản hồi. Vì gRPC bảo vệ chúng ta khỏi các chi tiết giao tiếp mạng này, `heroesService` trông và hoạt động như một provider cục bộ.

Lưu ý, tất cả các phương thức dịch vụ đều là **lower camel cased** (để theo quy ước tự nhiên của ngôn ngữ). Vì vậy, ví dụ, trong khi định nghĩa `HeroesService` tệp `.proto` của chúng ta chứa hàm `FindOne()`, thể hiện `heroesService` sẽ cung cấp phương thức `findOne()`.

```typescript
interface HeroesService {
  findOne(data: { id: number }): Observable<any>;
}
```

Một message handler cũng có thể trả về một `Observable`, trong trường hợp đó các giá trị kết quả sẽ được emit cho đến khi stream hoàn thành.

```typescript
@@filename(heroes.controller)
@Get()
call(): Observable<any> {
  return this.heroesService.findOne({ id: 1 });
}
@@switch
@Get()
call() {
  return this.heroesService.findOne({ id: 1 });
}
```

Để gửi metadata gRPC (cùng với yêu cầu), bạn có thể truyền một đối số thứ hai, như sau:

```typescript
call(): Observable<any> {
  const metadata = new Metadata();
  metadata.add('Set-Cookie', 'yummy_cookie=choco');

  return this.heroesService.findOne({ id: 1 }, metadata);
}
```

> info **Gợi ý** Lớp `Metadata` được import từ gói `grpc`.

Xin lưu ý rằng điều này sẽ yêu cầu cập nhật interface `HeroesService` mà chúng ta đã định nghĩa một vài bước trước.

#### Ví dụ

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/04-grpc).

#### gRPC Reflection

[Thông số gRPC Server Reflection](https://grpc.io/docs/guides/reflection/#overview) là một tiêu chuẩn cho phép gRPC clients yêu cầu chi tiết về API mà server expose, tương tự như exposing một tài liệu OpenAPI cho một REST API. Điều này có thể làm cho việc làm việc với các công cụ debug của nhà phát triển như grpc-ui hoặc postman dễ dàng hơn đáng kể.

Để thêm hỗ trợ gRPC reflection vào server của bạn, trước tiên hãy cài đặt gói triển khai cần thiết:

```bash
$ npm i --save @grpc/reflection
```

Sau đó nó có thể được hook vào server gRPC sử dụng hook `onLoadPackageDefinition` trong các tùy chọn server gRPC của bạn, như sau:

```typescript
@@filename(main)
import { ReflectionService } from '@grpc/reflection';

const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  options: {
    onLoadPackageDefinition: (pkg, server) => {
      new ReflectionService(pkg).addToServer(server);
    },
  },
});
```

Bây giờ server của bạn sẽ phản hồi các tin nhắn yêu cầu chi tiết API sử dụng thông số reflection.

#### gRPC Streaming

gRPC tự nó hỗ trợ các kết nối sống dài hạn, theo quy ước được gọi là `streams`. Streams hữu ích cho các trường hợp như Chatting, Observations hoặc chuyển dữ liệu Chunk-data. Tìm thêm chi tiết trong tài liệu chính thức [ở đây](https://grpc.io/docs/guides/concepts/).

Nest hỗ trợ các handler stream GRPC theo hai cách có thể:

- Handler RxJS `Subject` + `Observable`: có thể hữu ích để viết phản hồi ngay bên trong một phương thức Controller hoặc để được truyền xuống đến consumer `Subject`/`Observable`
- Handler stream cuộc gọi GRPC thuần túy: có thể hữu ích để được truyền đến một số executor sẽ xử lý phần còn lại của dispatch cho handler stream `Duplex` tiêu chuẩn Node.

<app-banner-enterprise></app-banner-enterprise>

#### Ví dụ streaming

Hãy định nghĩa một dịch vụ gRPC mẫu mới gọi là `HelloService`. Tệp `hello.proto` được cấu trúc sử dụng <a href="https://developers.google.com/protocol-buffers">protocol buffers</a>. Đây là cách nó trông:

```typescript
// hello/hello.proto
syntax = "proto3";

package hello;

service HelloService {
  rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```

> info **Gợi ý** Phương thức `LotsOfGreetings` có thể được triển khai đơn giản với decorator `@GrpcMethod` (như trong các ví dụ trên) vì stream được trả về có thể emit nhiều giá trị.

Dựa trên tệp `.proto` này, hãy định nghĩa interface `HelloService`:

```typescript
interface HelloService {
  bidiHello(upstream: Observable<HelloRequest>): Observable<HelloResponse>;
  lotsOfGreetings(
    upstream: Observable<HelloRequest>,
  ): Observable<HelloResponse>;

interface HelloRequest {
  greeting: string;
}

interface HelloResponse {
  reply: string;
}
```

> info **Gợi ý** Interface proto có thể được tạo tự động bởi gói [ts-proto](https://github.com/stephenh/ts-proto), tìm hiểu thêm [ở đây](https://github.com/stephenh/ts-proto/blob/main/NESTJS.markdown).

#### Chiến lược Subject

Decorator `@GrpcStreamMethod()` cung cấp tham số hàm như một RxJS `Observable`. Do đó, chúng ta có thể nhận và xử lý nhiều tin nhắn.

```typescript
@GrpcStreamMethod()
bidiHello(messages: Observable<any>, metadata: Metadata, call: ServerDuplexStream<any, any>): Observable<any> {
  const subject = new Subject();

  const onNext = message => {
    console.log(message);
    subject.next({
      reply: 'Hello, world!'
    });
  };
  const onComplete = () => subject.complete();
  messages.subscribe({
    next: onNext,
    complete: onComplete,
  });


  return subject.asObservable();
}
```

> warning **Cảnh báo** Để hỗ trợ tương tác full-duplex với decorator `@GrpcStreamMethod()`, phương thức controller phải trả về một RxJS `Observable`.

> info **Gợi ý** Các lớp/interface `Metadata` và `ServerUnaryCall` được import từ gói `grpc`.

Theo định nghĩa dịch vụ (trong tệp `.proto`), phương thức `BidiHello` nên stream các yêu cầu đến dịch vụ. Để gửi nhiều tin nhắn không đồng bộ đến stream từ một client, chúng ta tận dụng một lớp RxJS `ReplaySubject`.

```typescript
const helloService = this.client.getService<HelloService>('HelloService');
const helloRequest$ = new ReplaySubject<HelloRequest>();

helloRequest$.next({ greeting: 'Hello (1)!' });
helloRequest$.next({ greeting: 'Hello (2)!' });
helloRequest$.complete();

return helloService.bidiHello(helloRequest$);
```

Trong ví dụ trên, chúng ta đã viết hai tin nhắn vào stream (cuộc gọi `next()`) và thông báo cho dịch vụ rằng chúng ta đã hoàn thành gửi dữ liệu (cuộc gọi `complete()`).

#### Handler stream cuộc gọi

Khi giá trị trả về phương thức được định nghĩa là `stream`, decorator `@GrpcStreamCall()` cung cấp tham số hàm như `grpc.ServerDuplexStream`, hỗ trợ các phương thức tiêu chuẩn như `.on('data', callback)`, `.write(message)` hoặc `.cancel()`. Tài liệu đầy đủ về các phương thức có sẵn có thể được tìm thấy [ở đây](https://grpc.github.io/grpc/node/grpc-ClientDuplexStream.html).

Ngoài ra, khi giá trị trả về phương thức không phải là `stream`, decorator `@GrpcStreamCall()` cung cấp hai tham số hàm, tương ứng `grpc.ServerReadableStream` (đọc thêm [ở đây](https://grpc.github.io/grpc/node/grpc-ServerReadableStream.html)) và `callback`.

Hãy bắt đầu với việc triển khai `BidiHello` nên hỗ trợ tương tác full-duplex.

```typescript
@GrpcStreamCall()
bidiHello(requestStream: any) {
  requestStream.on('data', message => {
    console.log(message);
    requestStream.write({
      reply: 'Hello, world!'
    });
  });
}
```

> info **Gợi ý** Decorator này không yêu cầu bất kỳ tham số trả về cụ thể nào được cung cấp. Nó được mong đợi rằng stream sẽ được xử lý tương tự như bất kỳ loại stream tiêu chuẩn nào khác.

Trong ví dụ trên, chúng ta đã sử dụng phương thức `write()` để viết các đối tượng vào stream phản hồi. Callback được truyền vào phương thức `.on()` làm tham số thứ hai sẽ được gọi mỗi lần dịch vụ của chúng ta nhận một đoạn dữ liệu mới.

Hãy triển khai phương thức `LotsOfGreetings`.

```typescript
@GrpcStreamCall()
lotsOfGreetings(requestStream: any, callback: (err: unknown, value: HelloResponse) => void) {
  requestStream.on('data', message => {
    console.log(message);
  });
  requestStream.on('end', () => callback(null, { reply: 'Hello, world!' }));
}
```

Ở đây chúng ta đã sử dụng hàm `callback` để gửi phản hồi sau khi xử lý `requestStream` đã hoàn thành.

#### Health checks

Khi chạy một ứng dụng gRPC trong một orchestrator như Kubernetes, bạn có thể cần biết liệu nó đang chạy và ở trạng thái khỏe mạnh. [Thông số gRPC Health Check](https://grpc.io/docs/guides/health-checking/) là một tiêu chuẩn cho phép gRPC clients expose trạng thái sức khỏe của họ để cho phép orchestrator hành động phù hợp.

Để thêm hỗ trợ gRPC health check, trước tiên hãy cài đặt gói [grpc-node](https://github.com/grpc/grpc-node/tree/master/packages/grpc-health-check):

```bash
$ npm i --save grpc-health-check
```

Sau đó nó có thể được hook vào dịch vụ gRPC sử dụng hook `onLoadPackageDefinition` trong các tùy chọn server gRPC của bạn, như sau. Lưu ý rằng `protoPath` cần có cả health check và hero package.

```typescript
@@filename(main)
import { HealthImplementation, protoPath as healthCheckProtoPath } from 'grpc-health-check';

const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  options: {
    protoPath: [
      healthCheckProtoPath,
      protoPath: join(__dirname, 'hero/hero.proto'),
    ],
    onLoadPackageDefinition: (pkg, server) => {
      const healthImpl = new HealthImplementation({
        '': 'UNKNOWN',
      });

      healthImpl.addToServer(server);
      healthImpl.setStatus('', 'SERVING');
    },
  },
});
```

> info **Gợi ý** [gRPC health probe](https://github.com/grpc-ecosystem/grpc-health-probe) là một CLI hữu ích để kiểm tra gRPC health checks trong một môi trường được container hóa.

#### gRPC Metadata

Metadata là thông tin về một cuộc gọi RPC cụ thể dưới dạng một danh sách các cặp key-value, trong đó các keys là chuỗi và các giá trị thường là chuỗi nhưng có thể là dữ liệu nhị phân. Metadata là opaque đối với chính gRPC - nó cho phép client cung cấp thông tin liên quan đến cuộc gọi đến server và ngược lại. Metadata có thể bao gồm các token xác thực, định danh yêu cầu và các thẻ cho mục đích giám sát, và thông tin dữ liệu như số lượng bản ghi trong một tập dữ liệu.

Để đọc metadata trong handler `@GrpcMethod()`, sử dụng đối số thứ hai (metadata), có loại `Metadata` (được import từ gói `grpc`).

Để gửi lại metadata từ handler, sử dụng phương thức `ServerUnaryCall#sendMetadata()` (đối số handler thứ ba).

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const serverMetadata = new Metadata();
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];

    serverMetadata.add('Set-Cookie', 'yummy_cookie=choco');
    call.sendMetadata(serverMetadata);

    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data, metadata, call) {
    const serverMetadata = new Metadata();
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];

    serverMetadata.add('Set-Cookie', 'yummy_cookie=choco');
    call.sendMetadata(serverMetadata);

    return items.find(({ id }) => id === data.id);
  }
}
```

Tương tự, để đọc metadata trong các handler được chú thích với handler `@GrpcStreamMethod()` ([chiến lược subject](microservices/grpc#subject-strategy)), sử dụng đối số thứ hai (metadata), có loại `Metadata` (được import từ gói `grpc`).

Để gửi lại metadata từ handler, sử dụng phương thức `ServerDuplexStream#sendMetadata()` (đối số handler thứ ba).

Để đọc metadata từ bên trong [call stream handlers](microservices/grpc#call-stream-handler) (các handler được chú thích với decorator `@GrpcStreamCall()`), lắng nghe sự kiện `metadata` trên tham chiếu `requestStream`, như sau:

```typescript
requestStream.on('metadata', (metadata: Metadata) => {
  const meta = metadata.get('X-Meta');
});
```