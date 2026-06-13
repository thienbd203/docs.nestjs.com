### Kafka

[Kafka](https://kafka.apache.org/) là một nền tảng streaming phân tán, open source có ba khả năng chính:

- Xuất bản và subscribe vào các luồng bản ghi, tương tự như một hàng đợi tin nhắn hoặc hệ thống nhắn tin doanh nghiệp.
- Lưu trữ các luồng bản ghi theo cách bền vững chịu lỗi.
- Xử lý các luồng bản ghi khi chúng xảy ra.

Dự án Kafka nhằm cung cấp một nền tảng thống nhất, thông lượng cao, độ trễ thấp để xử lý các luồng dữ liệu thời gian thực. Nó tích hợp rất tốt với Apache Storm và Spark để phân tích dữ liệu streaming thời gian thực.

#### Cài đặt

Để bắt đầu xây dựng microservices dựa trên Kafka, trước tiên hãy cài đặt gói cần thiết:

```bash
$ npm i --save kafkajs
```

#### Tổng quan

Giống như các triển khai lớp transport microservices Nest khác, bạn chọn cơ chế transporter Kafka bằng thuộc tính `transport` của đối tượng options được truyền đến phương thức `createMicroservice()`, cùng với thuộc tính `options` tùy chọn, như được hiển thị dưới đây:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    }
  }
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    }
  }
});
```

> info **Gợi ý** Enum `Transport` được import từ gói `@nestjs/microservices`.

#### Tùy chọn

Thuộc tính `options` là cụ thể cho transporter được chọn. Transporter <strong>Kafka</strong> expose các thuộc tính được mô tả dưới đây.

<table>
  <tr>
    <td><code>client</code></td>
    <td>Tùy chọn cấu hình client (đọc thêm
      <a
        href="https://kafka.js.org/docs/configuration"
        rel="nofollow"
        target="blank"
        >ở đây</a
      >)</td>
  </tr>
  <tr>
    <td><code>consumer</code></td>
    <td>Tùy chọn cấu hình consumer (đọc thêm
      <a
        href="https://kafka.js.org/docs/consuming#a-name-options-a-options"
        rel="nofollow"
        target="blank"
        >ở đây</a
      >)</td>
  </tr>
  <tr>
    <td><code>run</code></td>
    <td>Tùy chọn cấu hình chạy (đọc thêm
      <a
        href="https://kafka.js.org/docs/consuming"
        rel="nofollow"
        target="blank"
        >ở đây</a
      >)</td>
  </tr>
  <tr>
    <td><code>subscribe</code></td>
    <td>Tùy chọn cấu hình subscribe (đọc thêm
      <a
        href="https://kafka.js.org/docs/consuming#frombeginning"
        rel="nofollow"
        target="blank"
        >ở đây</a
      >)</td>
  </tr>
  <tr>
    <td><code>producer</code></td>
    <td>Tùy chọn cấu hình producer (đọc thêm
      <a
        href="https://kafka.js.org/docs/producing#options"
        rel="nofollow"
        target="blank"
        >ở đây</a
      >)</td>
  </tr>
  <tr>
    <td><code>send</code></td>
    <td>Tùy chọn cấu hình gửi (đọc thêm
      <a
        href="https://kafka.js.org/docs/producing#options"
        rel="nofollow"
        target="blank"
        >ở đây</a
      >)</td>
  </tr>
  <tr>
    <td><code>producerOnlyMode</code></td>
    <td>Cờ tính năng để bỏ qua đăng ký nhóm consumer và chỉ hoạt động như một producer (<code>boolean</code>)</td>
  </tr>
  <tr>
    <td><code>postfixId</code></td>
    <td>Thay đổi hậuuffix của giá trị clientId (<code>string</code>)</td>
  </tr>
</table>

#### Client

Có một sự khác biệt nhỏ trong Kafka so với các transporter microservice khác. Thay vì lớp `ClientProxy`, chúng ta sử dụng lớp `ClientKafkaProxy`.

Giống như các transporter microservice khác, bạn có <a href="https://docs.nestjs.com/microservices/basics#client">nhiều tùy chọn</a> để tạo một thể hiện `ClientKafkaProxy`.

Một phương pháp để tạo một thể hiện là sử dụng `ClientsModule`. Để tạo một thể hiện client với `ClientsModule`, import nó và sử dụng phương thức `register()` để truyền một đối tượng options với các thuộc tính giống nhau được hiển thị ở trên trong phương thức `createMicroservice()`, cũng như một thuộc tính `name` để được sử dụng như injection token. Đọc thêm về `ClientsModule` <a href="https://docs.nestjs.com/microservices/basics#client">ở đây</a>.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'HERO_SERVICE',
        transport: Transport.KAFKA,
        options: {
          client: {
            clientId: 'hero',
            brokers: ['localhost:9092'],
          },
          consumer: {
            groupId: 'hero-consumer'
          }
        }
      },
    ]),
  ]
  ...
})
```

Các tùy chọn khác để tạo một client (hoặc `ClientProxyFactory` hoặc `@Client()`) cũng có thể được sử dụng. Bạn có thể đọc về chúng <a href="https://docs.nestjs.com/microservices/basics#client">ở đây</a>.

Sử dụng decorator `@Client()` như sau:

```typescript
@Client({
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero',
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer'
    }
  }
})
client: ClientKafkaProxy;
```

#### Pattern tin nhắn

Pattern tin nhắn microservice Kafka sử dụng hai topics cho các kênh yêu cầu và phản hồi. Phương thức `ClientKafkaProxy.send()` gửi tin nhắn với một [return address](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ReturnAddress.html) bằng cách liên kết một [correlation id](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html), topic phản hồi và partition phản hồi với tin nhắn yêu cầu. Điều này yêu cầu thể hiện `ClientKafkaProxy` phải được subscribe vào topic phản hồi và được gán cho ít nhất một partition trước khi gửi tin nhắn.

Sau đó, bạn cần có ít nhất một partition topic phản hồi cho mỗi ứng dụng Nest đang chạy. Ví dụ, nếu bạn đang chạy 4 ứng dụng Nest nhưng topic phản hồi chỉ có 3 partitions, thì 1 trong số các ứng dụng Nest sẽ gặp lỗi khi cố gắng gửi tin nhắn.

Khi các thể hiện `ClientKafkaProxy` mới được khởi chạy, chúng tham gia nhóm consumer và subscribe vào các topics tương ứng của họ. Quá trình này kích hoạt một rebalance của các partitions topic được gán cho consumers của nhóm consumer.

Thông thường, các partitions topic được gán sử dụng partitioner round robin, gán các partitions topic cho một tập hợp các consumers được sắp xếp theo tên consumer được đặt ngẫu nhiên khi khởi động ứng dụng. Tuy nhiên, khi một consumer mới tham gia nhóm consumer, consumer mới có thể được định vị ở bất cứ đâu trong tập hợp các consumers. Điều này tạo ra một điều kiện mà các consumers hiện có có thể được gán các partitions khác nhau khi consumer hiện có được định vị sau consumer mới. Kết quả là, các consumers được gán các partitions khác nhau sẽ mất các tin nhắn phản hồi của các yêu cầu được gửi trước rebalance.

Để ngăn chặn consumers `ClientKafkaProxy` mất các tin nhắn phản hồi, một partitioner tùy chỉnh tích hợp sẵn đặc thù của Nest được sử dụng. Partitioner tùy chỉnh này gán các partitions cho một tập hợp các consumers được sắp xếp theo timestamps độ phân giải cao (`process.hrtime()`) được đặt khi khởi động ứng dụng.

#### Subscription phản hồi tin nhắn

> warning **Lưu ý** Phần này chỉ liên quan nếu bạn sử dụng phong cách tin nhắn [request-response](/microservices/basics#request-response) (với decorator `@MessagePattern` và phương thức `ClientKafkaProxy.send`). Subscribe vào topic phản hồi không cần thiết cho giao tiếp [event-based](/microservices/basics#event-based) (decorator `@EventPattern` và phương thức `ClientKafkaProxy.emit`).

Lớp `ClientKafkaProxy` cung cấp phương thức `subscribeToResponseOf()`. Phương thức `subscribeToResponseOf()` nhận tên topic của một yêu cầu làm đối số và thêm tên topic phản hồi có nguồn gốc vào một tập hợp các topics phản hồi. Phương thức này được yêu cầu khi triển khai pattern tin nhắn.

```typescript
@@filename(heroes.controller)
onModuleInit() {
  this.client.subscribeToResponseOf('hero.kill.dragon');
}
```

Nếu thể hiện `ClientKafkaProxy` được tạo không đồng bộ, phương thức `subscribeToResponseOf()` phải được gọi trước khi gọi phương thức `connect()`.

```typescript
@@filename(heroes.controller)
async onModuleInit() {
  this.client.subscribeToResponseOf('hero.kill.dragon');
  await this.client.connect();
}
```

#### Incoming

Nest nhận các tin nhắn Kafka đến như một đối tượng với các thuộc tính `key`, `value`, và `headers` có giá trị loại `Buffer`. Nest sau đó phân tích các giá trị này bằng cách chuyển đổi các buffers thành chuỗi. Nếu chuỗi là "giống đối tượng", Nest cố gắng phân tích chuỗi như `JSON`. Giá trị `value` sau đó được truyền đến handler liên quan của nó.

#### Outgoing

Nest gửi các tin nhắn Kafka đi sau một quá trình serialization khi xuất bản sự kiện hoặc gửi tin nhắn. Điều này xảy ra trên các đối số được truyền đến các phương thức `ClientKafkaProxy` `emit()` và `send()` hoặc trên các giá trị được trả về từ một phương thức `@MessagePattern`. Quá trình serialization này "stringify" các đối tượng không phải là chuỗi hoặc buffers bằng cách sử dụng `JSON.stringify()` hoặc phương thức prototype `toString()`.

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const dragonId = message.dragonId;
    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];
    return items;
  }
}
```

> info **Gợi ý** `@Payload()` được import từ gói `@nestjs/microservices`.

Các tin nhắn đi cũng có thể được keyed bằng cách truyền một đối tượng với các thuộc tính `key` và `value`. Keying tin nhắn rất quan trọng để đáp ứng [yêu cầu co-partitioning](https://docs.confluent.io/current/ksql/docs/developer-guide/partition-data.html#co-partitioning-requirements).

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const realm = 'Nest';
    const heroId = message.heroId;
    const dragonId = message.dragonId;

    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];

    return {
      headers: {
        realm
      },
      key: heroId,
      value: items
    }
  }
}
```

Ngoài ra, các tin nhắn được truyền theo định dạng này cũng có thể chứa các headers tùy chỉnh được đặt trong thuộc tính hash `headers`. Các giá trị thuộc tính hash header phải là loại `string` hoặc loại `Buffer`.

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const realm = 'Nest';
    const heroId = message.heroId;
    const dragonId = message.dragonId;

    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];

    return {
      headers: {
        kafka_nestRealm: realm
      },
      key: heroId,
      value: items
    }
  }
}
```

#### Event-based

Mặc dù phương pháp request-response là lý tưởng để trao đổi tin nhắn giữa các dịch vụ, nó ít phù hợp hơn khi phong cách tin nhắn của bạn là event-based (về mặt lý tưởng cho Kafka) - khi bạn chỉ muốn xuất bản sự kiện **mà không chờ phản hồi**. Trong trường hợp đó, bạn không muốn chi phí được yêu cầu bởi request-response để duy trì hai topics.

Kiểm tra hai phần này để tìm hiểu thêm về điều này: [Tổng quan: Event-based](/microservices/basics#event-based) và [Tổng quan: Publishing events](/microservices/basics#publishing-events).

#### Context

Trong các kịch bản phức tạp hơn, bạn có thể cần truy cập thông tin bổ sung về yêu cầu đến. Khi sử dụng transporter Kafka, bạn có thể truy cập đối tượng `KafkaContext`.

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('hero.kill.dragon')
killDragon(message, context) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

> info **Gợi ý** `@Payload()`, `@Ctx()` và `KafkaContext` được import từ gói `@nestjs/microservices`.

Để truy cập đối tượng Kafka `IncomingMessage` gốc, sử dụng phương thức `getMessage()` của đối tượng `KafkaContext`, như sau:

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  const originalMessage = context.getMessage();
  const partition = context.getPartition();
  const { headers, timestamp } = originalMessage;
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('hero.kill.dragon')
killDragon(message, context) {
  const originalMessage = context.getMessage();
  const partition = context.getPartition();
  const { headers, timestamp } = originalMessage;
}
```

Ở đó `IncomingMessage` đáp ứng interface sau:

```typescript
interface IncomingMessage {
  topic: string;
  partition: number;
  timestamp: string;
  size: number;
  attributes: number;
  offset: string;
  key: any;
  value: any;
  headers: Record<string, any>;
}
```

Nếu handler của bạn liên quan đến thời gian xử lý chậm cho mỗi tin nhắn nhận được, bạn nên cân nhắc sử dụng callback `heartbeat`. Để truy xuất hàm `heartbeat`, sử dụng phương thức `getHeartbeat()` của `KafkaContext`, như sau:

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
async killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  const heartbeat = context.getHeartbeat();

  // Do some slow processing
  await doWorkPart1();

  // Send heartbeat to not exceed the sessionTimeout
  await heartbeat();

  // Do some slow processing again
  await doWorkPart2();
}
```

#### Quy ước đặt tên

Các thành phần microservice Kafka nối thêm một mô tả về vai trò tương ứng của họ vào các tùy chọn `client.clientId` và `consumer.groupId` để ngăn chặn va chạm giữa các thành phần client và server microservice Nest. Theo mặc định, các thành phần `ClientKafkaProxy` nối thêm `-client` và các thành phần `ServerKafka` nối thêm `-server` vào cả hai tùy chọn này. Lưu ý cách các giá trị được cung cấp dưới đây được chuyển đổi theo cách đó (như được hiển thị trong các chú thích).

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero', // hero-server
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer' // hero-consumer-server
    },
  }
});
```

Và cho client:

```typescript
@@filename(heroes.controller)
@Client({
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero', // hero-client
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer' // hero-consumer-client
    }
  }
})
client: ClientKafkaProxy;
```

> info **Gợi ý** Quy ước đặt tên client và consumer Kafka có thể được tùy chỉnh bằng cách mở rộng `ClientKafkaProxy` và `KafkaServer` trong custom provider của riêng bạn và override constructor.

Vì pattern tin nhắn microservice Kafka sử dụng hai topics cho các kênh yêu cầu và phản hồi, một pattern phản hồi nên được dẫn xuất từ topic yêu cầu. Theo mặc định, tên topic phản hồi là tổng hợp của tên topic yêu cầu với `.reply` được nối thêm.

```typescript
@@filename(heroes.controller)
onModuleInit() {
  this.client.subscribeToResponseOf('hero.get'); // hero.get.reply
}
```

> info **Gợi ý** Quy ước đặt tên topic phản hồi Kafka có thể được tùy chỉnh bằng cách mở rộng `ClientKafkaProxy` trong custom provider của riêng bạn và override phương thức `getResponsePatternName`.

#### Exceptions có thể thử lại

Tương tự như các transporter khác, tất cả các exceptions không được xử lý đều được tự động bọc vào một `RpcException` và chuyển đổi sang định dạng "thân thiện với người dùng". Tuy nhiên, có các trường hợp cạnh khi bạn có thể muốn bỏ qua cơ chế này và để exceptions được tiêu dùng bởi driver `kafkajs` thay thế. Throwing một exception khi xử lý tin nhắn hướng dẫn `kafkajs` để **retry** nó (redeliver nó) điều này có nghĩa là mặc dù handler tin nhắn (hoặc sự kiện) đã được kích hoạt, offset sẽ không được commit đến Kafka.

> warning **Cảnh báo** Đối với event handlers (giao tiếp event-based), tất cả các exceptions không được xử lý được coi là **retriable exceptions** theo mặc định.

Để làm điều này, bạn có thể sử dụng một lớp chuyên dụng gọi là `KafkaRetriableException`, như sau:

```typescript
throw new KafkaRetriableException('...');
```

> info **Gợi ý** Lớp `KafkaRetriableException` được xuất từ gói `@nestjs/microservices`.

### Xử lý exception tùy chỉnh

Cùng với các cơ chế xử lý lỗi mặc định, bạn có thể tạo một Exception Filter tùy chỉnh cho các sự kiện Kafka để quản lý logic retry. Ví dụ, ví dụ dưới đây trình bày cách bỏ qua một sự kiện có vấn đề sau một số lần thử lại có thể cấu hình:

```typescript
import { Catch, ArgumentsHost, Logger } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';
import { KafkaContext } from '@nestjs/microservices';
import { Producer } from 'kafkajs';

@Catch()
export class KafkaMaxRetryExceptionFilter extends BaseExceptionFilter {
  private readonly logger = new Logger(KafkaMaxRetryExceptionFilter.name);

  constructor(
    private readonly producer: Producer,
    private readonly maxRetries: number,
    // Optional custom function executed when max retries are exceeded
    private readonly skipHandler?: (message: any) => Promise<void>,
  ) {
    super();
  }

  async catch(exception: unknown, host: ArgumentsHost) {
    const kafkaContext = host.switchToRpc().getContext<KafkaContext>();
    const message = kafkaContext.getMessage();
    const currentRetryCount = this.getRetryCountFromContext(kafkaContext);

    if (currentRetryCount >= this.maxRetries) {
      this.logger.warn(
        `Max retries (${
          this.maxRetries
        }) exceeded for message: ${JSON.stringify(message)}`,
      );

      if (this.skipHandler) {
        try {
          await this.skipHandler(message);
        } catch (err) {
          this.logger.error('Error in skipHandler:', err);
        }
      }

      try {
        await this.commitOffset(kafkaContext);
      } catch (commitError) {
        this.logger.error('Failed to commit offset:', commitError);
      }
      return; // Stop propagating the exception
    }

    // Republish the message to the same topic with incremented retry count
    try {
      await this.republishWithRetry(kafkaContext, currentRetryCount + 1);
      await this.commitOffset(kafkaContext);
    } catch (republishError) {
      this.logger.error('Failed to republish message for retry:', republishError);
      // Fall back to default exception handling
      super.catch(exception, host);
    }
  }

  private getRetryCountFromContext(context: KafkaContext): number {
    const headers = context.getMessage().headers || {};
    const retryHeader = headers['retry-count'];
    if (!retryHeader) {
      return 0;
    }
    // Header values are Buffers, so convert to string first
    const value = Buffer.isBuffer(retryHeader)
      ? retryHeader.toString()
      : String(retryHeader);
    return parseInt(value, 10) || 0;
  }

  private async republishWithRetry(
    context: KafkaContext,
    retryCount: number,
  ): Promise<void> {
    const topic = context.getTopic();
    const message = context.getMessage();

    await this.producer.send({
      topic,
      messages: [
        {
          key: message.key,
          value: message.value,
          headers: {
            ...message.headers,
            'retry-count': retryCount.toString(),
          },
        },
      ],
    });
  }

  private async commitOffset(context: KafkaContext): Promise<void> {
    const consumer = context.getConsumer();
    if (!consumer) {
      throw new Error('Consumer instance is not available from KafkaContext.');
    }

    const topic = context.getTopic();
    const partition = context.getPartition();
    const message = context.getMessage();
    const offset = message.offset;

    if (!topic || partition === undefined || offset === undefined) {
      throw new Error(
        'Incomplete Kafka message context for committing offset.',
      );
    }

    await consumer.commitOffsets([
      {
        topic,
        partition,
        // When committing an offset, commit the next number (i.e., current offset + 1)
        offset: (Number(offset) + 1).toString(),
      },
    ]);
  }
}
```

Filter này cung cấp một cách để thử lại xử lý một sự kiện Kafka lên đến một số lần có thể cấu hình. Khi một exception xảy ra, nó republish tin nhắn đến cùng một topic với một header `retry-count` được tăng, sau đó commit offset hiện tại. Khi số lần thử lại tối đa được đạt đến, nó kích hoạt một `skipHandler` tùy chỉnh (nếu được cung cấp) và commit offset, hiệu quả bỏ qua sự kiện có vấn đề. Điều này cho phép các sự kiện tiếp theo được xử lý không bị gián đoạn.

Bạn có thể tích hợp filter này bằng cách đăng ký nó toàn cục hoặc ở cấp độ controller. Lưu ý rằng bạn cần cung cấp một thể hiện producer Kafka:

```typescript
@@filename(kafka-retry.filter)
import { Inject, Injectable } from '@nestjs/common';
import { Producer } from 'kafkajs';

@Injectable()
export class AppKafkaRetryFilter extends KafkaMaxRetryExceptionFilter {
  constructor(@Inject('KAFKA_PRODUCER') producer: Producer) {
    super(producer, 5); // maxRetries = 5
  }
}
@@switch
import { Inject, Injectable } from '@nestjs/common';

@Injectable()
export class AppKafkaRetryFilter extends KafkaMaxRetryExceptionFilter {
  constructor(@Inject('KAFKA_PRODUCER') producer) {
    super(producer, 5); // maxRetries = 5
  }
}
```

```typescript
@@filename(my-event.handler)
@Controller()
@UseFilters(AppKafkaRetryFilter)
export class MyEventHandler {
  @EventPattern('your-topic')
  async handleEvent(@Payload() data: any, @Ctx() context: KafkaContext) {
    // Your event processing logic...
  }
}
@@switch
@Controller()
@UseFilters(AppKafkaRetryFilter)
export class MyEventHandler {
  @Bind(Payload(), Ctx())
  @EventPattern('your-topic')
  async handleEvent(data, context) {
    // Your event processing logic...
  }
}
```

Đảm bảo cung cấp producer Kafka trong module của bạn:

```typescript
@@filename(app.module)
import { Kafka } from 'kafkajs';

@Module({
  providers: [
    AppKafkaRetryFilter,
    {
      provide: 'KAFKA_PRODUCER',
      useFactory: async () => {
        const kafka = new Kafka({ brokers: ['localhost:9092'] });
        const producer = kafka.producer();
        await producer.connect();
        return producer;
      },
    },
  ],
})
export class AppModule {}
```

#### Commit offsets

Committing offsets là thiết yếu khi làm việc với Kafka. Theo mặc định, các tin nhắn sẽ được tự động commit sau một thời gian cụ thể. Để biết thêm thông tin, hãy truy cập [tài liệu KafkaJS](https://kafka.js.org/docs/consuming#autocommit). `KafkaContext` cung cấp một cách để truy cập consumer hoạt động để commit offsets thủ công. Consumer là consumer KafkaJS và hoạt động như [triển khai KafkaJS gốc](https://kafka.js.org/docs/consuming#manual-committing).

```typescript
@@filename()
@EventPattern('user.created')
async handleUserCreated(@Payload() data: IncomingMessage, @Ctx() context: KafkaContext) {
  // business logic

  const { offset } = context.getMessage();
  const partition = context.getPartition();
  const topic = context.getTopic();
  const consumer = context.getConsumer();
  await consumer.commitOffsets([{ topic, partition, offset }])
}
@@switch
@Bind(Payload(), Ctx())
@EventPattern('user.created')
async handleUserCreated(data, context) {
  // business logic

  const { offset } = context.getMessage();
  const partition = context.getPartition();
  const topic = context.getTopic();
  const consumer = context.getConsumer();
  await consumer.commitOffsets([{ topic, partition, offset }])
}
```

Để tắt auto-committing của tin nhắn, đặt `autoCommit: false` trong cấu hình `run`, như sau:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    },
    run: {
      autoCommit: false
    }
  }
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    },
    run: {
      autoCommit: false
    }
  }
});
```

#### Cập nhật trạng thái thể hiện

Để nhận cập nhật thời gian thực về kết nối và trạng thái của thể hiện driver cơ bản, bạn có thể subscribe vào stream `status`. Stream này cung cấp các cập nhật trạng thái cụ thể cho driver được chọn. Đối với driver Kafka, stream `status` emit các sự kiện `connected`, `disconnected`, `rebalancing`, `crashed`, và `stopped`.

```typescript
this.client.status.subscribe((status: KafkaStatus) => {
  console.log(status);
});
```

> info **Gợi ý** Kiểu `KafkaStatus` được import từ gói `@nestjs/microservices`.

Tương tự, bạn có thể subscribe vào stream `status` của server để nhận thông báo về trạng thái của server.

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: KafkaStatus) => {
  console.log(status);
});
```

#### Producer và consumer cơ bản

Đối với các trường hợp sử dụng nâng cao hơn, bạn có thể cần truy cập các thể hiện producer và consumer cơ bản. Điều này có thể hữu ích cho các kịch bản như đóng thủ công kết nối hoặc sử dụng các phương thức cụ thể của driver. Tuy nhiên, hãy nhớ rằng đối với hầu hết các trường hợp, bạn **không nên cần** truy cập trực tiếp driver.

Để làm điều này, bạn có thể sử dụng các getters `producer` và `consumer` được expose bởi thể hiện `ClientKafkaProxy`.

```typescript
const producer = this.client.producer;
const consumer = this.client.consumer;
```