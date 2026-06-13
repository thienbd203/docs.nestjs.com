### Queues

Queues là một mẫu thiết kế mạnh mẽ giúp bạn giải quyết các thách thức phổ biến về mở rộng và hiệu suất ứng dụng. Một số ví dụ về vấn đề mà Queues có thể giúp bạn giải quyết là:

- Làm mượt các đỉnh xử lý. Ví dụ, nếu người dùng có thể khởi tạo các tasks tốn nhiều tài nguyên vào thời điểm tùy ý, bạn có thể thêm các tasks này vào một queue thay vì thực hiện chúng đồng bộ. Sau đó bạn có thể có các worker processes kéo tasks từ queue theo cách được kiểm soát. Bạn có thể dễ dàng thêm các Queue consumers mới để mở rộng xử lý tasks backend khi ứng dụng mở rộng.
- Phá vỡ các tasks nguyên khối có thể chặn vòng lặp sự kiện Node.js. Ví dụ, nếu một yêu cầu người dùng yêu cầu công việc tốn CPU như chuyển đổi âm thanh, bạn có thể ủy quyền task này cho các processes khác, giải phóng các processes hướng người dùng để vẫn phản hồi.
- Cung cấp một kênh giao tiếp đáng tin cậy trên các dịch vụ khác nhau. Ví dụ, bạn có thể queue tasks (jobs) trong một process hoặc service, và tiêu thụ chúng trong một khác. Bạn có thể được thông báo (bằng cách lắng nghe các sự kiện trạng thái) khi hoàn thành, lỗi hoặc các thay đổi trạng thái khác trong vòng đời job từ bất kỳ process hoặc service nào. Khi Queue producers hoặc consumers thất bại, trạng thái của chúng được bảo toàn và xử lý task có thể khởi động lại tự động khi các nodes được khởi động lại.

Nest cung cấp package `@nestjs/bullmq` cho tích hợp BullMQ và package `@nestjs/bull` cho tích hợp Bull. Cả hai package đều là các abstraction/wrappers trên cùng các thư viện tương ứng, được phát triển bởi cùng một đội ngũ. Bull hiện đang ở chế độ bảo trì, với đội ngũ tập trung vào việc sửa lỗi, trong khi BullMQ đang được phát triển tích cực, có triển khai TypeScript hiện đại và một tập hợp tính năng khác nhau. Nếu Bull đáp ứng nhu cầu của bạn, nó vẫn là một lựa chọn đáng tin cậy và đã được thử nghiệm trong thực chiến. Các package Nest giúp dễ dàng tích hợp cả BullMQ hoặc Bull Queues vào ứng dụng Nest của bạn một cách thân thiện.

Cả BullMQ và Bull đều sử dụng [Redis](https://redis.io/) để duy trì dữ liệu job, vì vậy bạn sẽ cần có Redis được cài đặt trên hệ thống của bạn. Vì chúng được hỗ trợ bởi Redis, kiến trúc Queue của bạn có thể hoàn toàn được phân phối và độc lập nền tảng. Ví dụ, bạn có thể có một số Queue <a href="techniques/queues#producers">producers</a> và <a href="techniques/queues#consumers">consumers</a> và <a href="techniques/queues#event-listeners">listeners</a> chạy trong Nest trên một (hoặc nhiều) nodes, và các producers, consumers và listeners khác chạy trên các nền tảng Node.js khác trên các nodes mạng khác.

Chương này bao gồm các package `@nestjs/bullmq` và `@nestjs/bull`. Chúng tôi cũng khuyến nghị đọc tài liệu [BullMQ](https://docs.bullmq.io/) và [Bull](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md) để biết thêm thông tin nền và chi tiết triển khai cụ thể.

#### Cài đặt BullMQ

Để bắt đầu sử dụng BullMQ, trước tiên chúng ta cài đặt các dependency cần thiết.

```bash
$ npm install --save @nestjs/bullmq bullmq
```

Sau khi quá trình cài đặt hoàn tất, chúng ta có thể nhập `BullModule` vào `AppModule` gốc.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';

@Module({
  imports: [
    BullModule.forRoot({
      connection: {
        host: 'localhost',
        port: 6379,
      },
    }),
  ],
})
export class AppModule {}
```

Phương thức `forRoot()` được sử dụng để đăng ký một đối tượng cấu hình package `bullmq` sẽ được sử dụng bởi tất cả các queues được đăng ký trong ứng dụng (trừ khi được chỉ định khác). Để tham khảo của bạn, sau đây là một số thuộc tính trong đối tượng cấu hình:

- `connection: ConnectionOptions` - Tùy chọn để cấu hình kết nối Redis. Xem [Connections](https://docs.bullmq.io/guide/connections) để biết thêm thông tin. Tùy chọn.
- `prefix: string` - Tiền tố cho tất cả các khóa queue. Tùy chọn.
- `defaultJobOptions: JobOpts` - Tùy chọn để kiểm soát các cài đặt mặc định cho các jobs mới. Xem [JobOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd) để biết thêm thông tin. Tùy chọn.
- `settings: AdvancedSettings` - Cài đặt cấu hình Queue nâng cao. Các cài đặt này thường không nên thay đổi. Xem [AdvancedSettings](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue) để biết thêm thông tin. Tùy chọn.
- `extraOptions` - Các tùy chọn bổ sung cho khởi tạo module. Xem [Đăng ký thủ công](https://docs.nestjs.com/techniques/queues#manual-registration)

Tất cả các tùy chọn là tùy chọn, cung cấp kiểm soát chi tiết về hành vi queue. Các tùy chọn này được truyền trực tiếp đến constructor `Queue` của BullMQ. Đọc thêm về các tùy chọn này và các tùy chọn khác [ở đây](https://api.docs.bullmq.io/interfaces/v4.QueueOptions.html).

Để đăng ký một queue, nhập dynamic module `BullModule.registerQueue()`, như sau:

```typescript
BullModule.registerQueue({
  name: 'audio',
});
```

> info **Gợi ý** Tạo nhiều queues bằng cách truyền nhiều đối tượng cấu hình được phân tách bằng dấu phẩy đến phương thức `registerQueue()`.

Phương thức `registerQueue()` được sử dụng để khởi tạo và/hoặc đăng ký queues. Queues được chia sẻ trên các modules và processes kết nối đến cùng cơ sở dữ liệu Redis cơ bản với cùng thông tin xác thực. Mỗi queue là duy nhất theo thuộc tính tên của nó. Tên queue được sử dụng như một injection token (để inject queue vào controllers/providers) và như một đối số cho các decorator để liên kết các lớp consumer và listeners với queues.

Bạn cũng có thể ghi đè một số tùy chọn được cấu hình trước cho một queue cụ thể, như sau:

```typescript
BullModule.registerQueue({
  name: 'audio',
  connection: {
    port: 6380,
  },
});
```

BullMQ cũng hỗ trợ các mối quan hệ parent - child giữa các jobs. Chức năng này cho phép tạo các flows nơi các jobs là nút của các cây có độ sâu tùy ý. Để đọc thêm về chúng, kiểm tra [ở đây](https://docs.bullmq.io/guide/flows).

Để thêm một flow, bạn có thể làm như sau:

```typescript
BullModule.registerFlowProducer({
  name: 'flowProducerName',
});
```

Vì các jobs được duy trì trong Redis, mỗi khi một queue được đặt tên cụ thể được khởi tạo (ví dụ, khi một ứng dụng được khởi động/khởi động lại), nó cố gắng xử lý bất kỳ jobs cũ nào có thể tồn tại từ một session chưa hoàn thành trước đó.

Mỗi queue có thể có một hoặc nhiều producers, consumers, và listeners. Consumers truy xuất các jobs từ queue theo một thứ tự cụ thể: FIFO (mặc định), LIFO, hoặc theo các ưu tiên. Kiểm soát thứ tự xử lý queue được thảo luận <a href="techniques/queues#consumers">ở đây</a>.

<app-banner-enterprise></app-banner-enterprise>

#### Cấu hình được đặt tên

Nếu các queues của bạn kết nối đến nhiều instances Redis khác nhau, bạn có thể sử dụng một kỹ thuật gọi là **cấu hình được đặt tên**. Tính năng này cho phép bạn đăng ký một số cấu hình dưới các khóa được chỉ định, sau đó bạn có thể tham chiếu trong các tùy chọn queue.

Ví dụ, giả sử rằng bạn có một instance Redis bổ sung (ngoài mặc định) được sử dụng bởi một vài queues được đăng ký trong ứng dụng của bạn, bạn có thể đăng ký cấu hình của nó như sau:

```typescript
BullModule.forRoot('alternative-config', {
  connection: {
    port: 6381,
  },
});
```

Trong ví dụ trên, `'alternative-config'` chỉ là một khóa cấu hình (nó có thể là bất kỳ chuỗi tùy ý nào).

Với điều này đã sẵn sàng, bây giờ bạn có thể trỏ đến cấu hình này trong đối tượng tùy chọn `registerQueue()`:

```typescript
BullModule.registerQueue({
  configKey: 'alternative-config',
  name: 'video',
});
```

#### Producers

Job producers thêm jobs vào queues. Producers thường là các services ứng dụng (Nest [providers](/providers)). Để thêm jobs vào một queue, trước tiên hãy inject queue vào service như sau:

```typescript
import { Injectable } from '@nestjs/common';
import { Queue } from 'bullmq';
import { InjectQueue } from '@nestjs/bullmq';

@Injectable()
export class AudioService {
  constructor(@InjectQueue('audio') private audioQueue: Queue) {}
}
```

> info **Gợi ý** Decorator `@InjectQueue()` xác định queue theo tên của nó, như được cung cấp trong cuộc gọi phương thức `registerQueue()` (ví dụ, `'audio'`).

Bây giờ, thêm một job bằng cách gọi phương thức `add()` của queue, truyền một đối tượng job do người dùng định nghĩa. Jobs được đại diện như các đối tượng JavaScript có thể tuần tự hóa (vì đó là cách chúng được lưu trữ trong cơ sở dữ liệu Redis). Hình dạng của job bạn truyền là tùy ý; sử dụng nó để đại diện cho ngữ nghĩa của đối tượng job của bạn. Bạn cũng cần đặt tên cho nó. Điều này cho phép bạn tạo các <a href="techniques/queues#consumers">consumers</a> chuyên biệt sẽ chỉ xử lý các jobs với một tên cụ thể.

```typescript
const job = await this.audioQueue.add('transcode', {
  foo: 'bar',
});
```

#### Tùy chọn job

Jobs có thể có các tùy chọn bổ sung liên kết với chúng. Truyền một đối tượng tùy chọn sau đối số `job` trong phương thức `Queue.add()`. Một số thuộc tính tùy chọn job là:

- `priority`: `number` - Giá trị ưu tiên tùy chọn. Phạm vi từ 1 (ưu tiên cao nhất) đến MAX_INT (ưu tiên thấp nhất). Lưu ý rằng sử dụng các ưu tiên có tác động nhẹ đến hiệu suất, vì vậy hãy sử dụng chúng một cách thận trọng.
- `delay`: `number` - Một khoảng thời gian (mili giây) để đợi cho đến khi job này có thể được xử lý. Lưu ý rằng để có độ trễ chính xác, cả máy chủ và máy khách nên có đồng hồ của chúng được đồng bộ hóa.
- `attempts`: `number` - Tổng số lần thử job cho đến khi nó hoàn thành.
- `repeat`: `RepeatOpts` - Lặp lại job theo một đặc tả cron. Xem [RepeatOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd).
- `backoff`: `number | BackoffOpts` - Cài đặt backoff cho các thử lại tự động nếu job thất bại. Xem [BackoffOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd).
- `lifo`: `boolean` - Nếu true, thêm job vào cuối bên phải của queue thay vì bên trái (mặc định false).
- `jobId`: `number` | `string` - Ghi đè ID job - theo mặc định, ID job là một số nguyên duy nhất, nhưng bạn có thể sử dụng cài đặt này để ghi đè nó. Nếu bạn sử dụng tùy chọn này, trách nhiệm của bạn là đảm bảo jobId là duy nhất. Nếu bạn cố gắng thêm một job với một id đã tồn tại, nó sẽ không được thêm.
- `removeOnComplete`: `boolean | number` - Nếu true, loại bỏ job khi nó hoàn thành thành công. Một số chỉ định số lượng jobs để giữ. Hành vi mặc định là giữ job trong tập hợp hoàn thành.
- `removeOnFail`: `boolean | number` - Nếu true, loại bỏ job khi nó thất bại sau tất cả các lần thử. Một số chỉ định số lượng jobs để giữ. Hành vi mặc định là giữ job trong tập hợp thất bại.
- `stackTraceLimit`: `number` - Giới hạn số lượng dòng stack trace sẽ được ghi lại trong stacktrace.

Dưới đây là một vài ví dụ về tùy chỉnh jobs với các tùy chọn job.

Để trì hoãn việc bắt đầu một job, sử dụng thuộc tính cấu hình `delay`.

```typescript
const job = await this.audioQueue.add(
  'transcode',
  {
    foo: 'bar',
  },
  { delay: 3000 }, // trì hoãn 3 giây
);
```

Để thêm một job vào cuối bên phải của queue (xử lý job dưới dạng **LIFO** (Last In First Out)), đặt thuộc tính `lifo` của đối tượng cấu hình thành `true`.

```typescript
const job = await this.audioQueue.add(
  'transcode',
  {
    foo: 'bar',
  },
  { lifo: true },
);
```

Để ưu tiên một job, sử dụng thuộc tính `priority`.

```typescript
const job = await this.audioQueue.add(
  'transcode',
  {
    foo: 'bar',
  },
  { priority: 2 },
);
```

Để biết danh sách đầy đủ các tùy chọn, kiểm tra tài liệu API [ở đây](https://api.docs.bullmq.io/types/v4.JobsOptions.html) và [ở đây](https://api.docs.bullmq.io/interfaces/v4.BaseJobOptions.html).

#### Consumers

Một consumer là một **class** định nghĩa các phương thức xử lý jobs được thêm vào queue, hoặc lắng nghe các sự kiện trên queue, hoặc cả hai. Khai báo một class consumer sử dụng decorator `@Processor()` như sau:

```typescript
import { Processor } from '@nestjs/bullmq';

@Processor('audio')
export class AudioConsumer {}
```

> info **Gợi ý** Consumers phải được đăng ký như `providers` để package `@nestjs/bullmq` có thể chọn chúng.

Trong đó đối số chuỗi của decorator (ví dụ, `'audio'`) là tên của queue để được liên kết với các phương thức class.

```typescript
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('audio')
export class AudioConsumer extends WorkerHost {
  async process(job: Job<any, any, string>): Promise<any> {
    let progress = 0;
    for (let i = 0; i < 100; i++) {
      await doSomething(job.data);
      progress += 1;
      await job.updateProgress(progress);
    }
    return {};
  }
}
```

Phương thức process được gọi bất cứ khi nào worker rảnh và có jobs để xử lý trong queue. Phương thức handler này nhận đối tượng `job` làm đối số duy nhất của nó. Giá trị được trả về bởi phương thức handler được lưu trữ trong đối tượng job và có thể được truy cập sau này, ví dụ trong một listener cho sự kiện hoàn thành.

Các đối tượng `Job` có nhiều phương thức cho phép bạn tương tác với trạng thái của chúng. Ví dụ, mã ở trên sử dụng phương thức `updateProgress()` để cập nhật tiến độ của job. Xem [ở đây](https://api.docs.bullmq.io/classes/v4.Job.html) để biết tham chiếu API đối tượng `Job` đầy đủ.

Trong phiên bản cũ hơn, Bull, bạn có thể chỉ định rằng một phương thức handler job sẽ xử lý **chỉ** các jobs của một loại cụ thể (các jobs với một `name` cụ thể) bằng cách truyền `name` đó đến decorator `@Process()` như được hiển thị dưới đây.

> warning **Cảnh báo** Điều này không hoạt động với BullMQ, hãy tiếp tục đọc.

```typescript
@Process('transcode')
async transcode(job: Job<unknown>) { ... }
```

Hành vi này không được hỗ trợ trong BullMQ do sự nhầm lẫn nó tạo ra. Thay vào đó, bạn cần các trường hợp switch để gọi các services hoặc logic khác nhau cho mỗi tên job:

```typescript
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('audio')
export class AudioConsumer extends WorkerHost {
  async process(job: Job<any, any, string>): Promise<any> {
    switch (job.name) {
      case 'transcode': {
        let progress = 0;
        for (i = 0; i < 100; i++) {
          await doSomething(job.data);
          progress += 1;
          await job.progress(progress);
        }
        return {};
      }
      case 'concatenate': {
        await doSomeLogic2();
        break;
      }
    }
  }
}
```

Điều này được bao phủ trong phần [named processor](https://docs.bullmq.io/patterns/named-processor) của tài liệu BullMQ.

#### Consumers phạm vi request

Khi một consumer được đánh dấu là phạm vi request (tìm hiểu thêm về các phạm vi injection [ở đây](/fundamentals/injection-scopes#provider-scope)), một instance mới của class sẽ được tạo riêng cho mỗi job. Instance sẽ được thu gom rác sau khi job đã hoàn thành.

```typescript
@Processor({
  name: 'audio',
  scope: Scope.REQUEST,
})
```

Vì các lớp consumer phạm vi request được khởi tạo động và phạm vi đến một job duy nhất, bạn có thể inject một `JOB_REF` thông qua constructor bằng cách sử dụng cách tiếp cận tiêu chuẩn.

```typescript
constructor(@Inject(JOB_REF) jobRef: Job) {
  console.log(jobRef);
}
```

> info **Gợi ý** Token `JOB_REF` được nhập từ package `@nestjs/bullmq`.

#### Event listeners

BullMQ tạo ra một tập hợp các sự kiện hữu ích khi các thay đổi trạng thái queue và/hoặc job xảy ra. Các sự kiện này có thể được đăng ký ở mức Worker bằng cách sử dụng decorator `@OnWorkerEvent(event)`, hoặc ở mức Queue với một class listener chuyên dụng và decorator `@OnQueueEvent(event)`.

Các sự kiện Worker phải được khai báo trong một class <a href="techniques/queues#consumers">consumer</a> (tức là trong một class được trang trí với decorator `@Processor()`). Để lắng nghe một sự kiện, sử dụng decorator `@OnWorkerEvent(event)` với sự kiện bạn muốn được xử lý. Ví dụ, để lắng nghe sự kiện được phát ra khi một job vào trạng thái active trong queue `audio`, sử dụng cấu trúc sau:

```typescript
import { Processor, Process, OnWorkerEvent } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('audio')
export class AudioConsumer {
  @OnWorkerEvent('active')
  onActive(job: Job) {
    console.log(
      `Processing job ${job.id} of type ${job.name} with data ${job.data}...`,
    );
  }

  // ...
}
```

Bạn có thể thấy danh sách đầy đủ các sự kiện và các đối số của chúng như các thuộc tính của WorkerListener [ở đây](https://api.docs.bullmq.io/interfaces/v4.WorkerListener.html).

Các listeners QueueEvent phải sử dụng decorator `@QueueEventsListener(queue)` và mở rộng class `QueueEventsHost` được cung cấp bởi `@nestjs/bullmq`. Để lắng nghe một sự kiện, sử dụng decorator `@OnQueueEvent(event)` với sự kiện bạn muốn được xử lý. Ví dụ, để lắng nghe sự kiện được phát ra khi một job vào trạng thái active trong queue `audio`, sử dụng cấu trúc sau:

```typescript
import {
  QueueEventsHost,
  QueueEventsListener,
  OnQueueEvent,
} from '@nestjs/bullmq';

@QueueEventsListener('audio')
export class AudioEventsListener extends QueueEventsHost {
  @OnQueueEvent('active')
  onActive(job: { jobId: string; prev?: string }) {
    console.log(`Processing job ${job.jobId}...`);
  }

  // ...
}
```

> info **Gợi ý** QueueEvent Listeners phải được đăng ký như `providers` để package `@nestjs/bullmq` có thể chọn chúng.

Bạn có thể thấy danh sách đầy đủ các sự kiện và các đối số của chúng như các thuộc tính của QueueEventsListener [ở đây](https://api.docs.bullmq.io/interfaces/v4.QueueEventsListener.html).

#### Quản lý Queue

Queues có một API cho phép bạn thực hiện các chức năng quản lý như tạm dừng và tiếp tục, truy xuất số lượng jobs trong các trạng thái khác nhau, và nhiều hơn nữa. Bạn có thể tìm thấy API queue đầy đủ [ở đây](https://api.docs.bullmq.io/classes/v4.Queue.html). Gọi bất kỳ phương thức nào trong số này trực tiếp trên đối tượng `Queue`, như được hiển thị dưới đây với các ví dụ tạm dừng/tiếp tục.

Tạm dừng một queue với cuộc gọi phương thức `pause()`. Một queue bị tạm dừng sẽ không xử lý các jobs mới cho đến khi được tiếp tục, nhưng các jobs hiện đang được xử lý sẽ tiếp tục cho đến khi chúng được hoàn thành.

```typescript
await audioQueue.pause();
```

Để tiếp tục một queue bị tạm dừng, sử dụng phương thức `resume()`, như sau:

```typescript
await audioQueue.resume();
```

#### Các processes riêng biệt

Job handlers cũng có thể được chạy trong một process riêng (được fork) ([nguồn](https://docs.bullmq.io/guide/workers/sandboxed-processors)). Điều này có một số lợi ích:

- Process được sandboxed nên nếu nó bị sập, nó không ảnh hưởng đến worker.
- Bạn có thể chạy mã chặn mà không ảnh hưởng đến queue (các jobs sẽ không bị treo).
- Sử dụng tốt hơn nhiều CPU đa lõi.
- Ít kết nối đến redis.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { join } from 'node:path';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'audio',
      processors: [join(__dirname, 'processor.js')],
    }),
  ],
})
export class AppModule {}
```

> warning **Cảnh báo** Vui lòng lưu ý rằng vì hàm của bạn đang được thực thi trong một process được fork, Dependency Injection (và container IoC) sẽ không khả dụng. Điều đó có nghĩa là hàm processor của bạn sẽ cần chứa (hoặc tạo) tất cả các instances của các dependencies bên ngoài mà nó cần.

#### Cấu hình async

Bạn có thể muốn truyền các tùy chọn `bullmq` một cách bất đồng bộ thay vì tĩnh. Trong trường hợp này, sử dụng phương thức `forRootAsync()` cung cấp một số cách để xử lý cấu hình async. Tương tự, nếu bạn muốn truyền các tùy chọn queue một cách bất đồng bộ, sử dụng phương thức `registerQueueAsync()`.

Một cách tiếp cận là sử dụng một hàm factory:

```typescript
BullModule.forRootAsync({
  useFactory: () => ({
    connection: {
      host: 'localhost',
      port: 6379,
    },
  }),
});
```

Factory của chúng ta hoạt động giống như bất kỳ [asynchronous provider](https://docs.nestjs.com/fundamentals/async-providers) nào khác (ví dụ, nó có thể là `async` và có thể inject dependencies thông qua `inject`).

```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    connection: {
      host: configService.get('QUEUE_HOST'),
      port: configService.get('QUEUE_PORT'),
    },
  }),
  inject: [ConfigService],
});
```

Ngoài ra, bạn có thể sử dụng cú pháp `useClass`:

```typescript
BullModule.forRootAsync({
  useClass: MyBullMQConfigService,
});
```

> info **Gợi ý** Để tìm hiểu thêm về các tùy chọn cấu hình BullMQ, hãy xem tài liệu [BullMQ](https://docs.bullmq.io/).