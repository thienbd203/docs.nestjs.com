### Lên lịch task

Lên lịch task cho phép bạn lên lịch mã tùy ý (phương thức/hàm) để thực thi tại một ngày/giờ cố định, tại các khoảng lặp lại, hoặc một lần sau một khoảng thời gian được chỉ định. Trong thế giới Linux, điều này thường được xử lý bởi các package như [cron](https://en.wikipedia.org/wiki/Cron) ở cấp hệ điều hành. Đối với các ứng dụng Node.js, có một số package mô phỏng chức năng kiểu cron. Nest cung cấp package `@nestjs/schedule`, tích hợp với package [cron](https://github.com/kelektiv/node-cron) Node.js phổ biến. Chúng tôi sẽ bao gồm package này trong chương hiện tại.

#### Cài đặt

Để bắt đầu sử dụng nó, trước tiên chúng ta cài đặt các dependency cần thiết.

```bash
$ npm install --save @nestjs/schedule
```

Để kích hoạt lên lịch job, nhập `ScheduleModule` vào `AppModule` gốc và chạy phương thức tĩnh `forRoot()` như được hiển thị dưới đây:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot()
  ],
})
export class AppModule {}
```

Cuộc gọi `.forRoot()` khởi tạo trình lập lịch và đăng ký bất kỳ <a href="techniques/task-scheduling#declarative-cron-jobs">cron jobs</a>, <a href="techniques/task-scheduling#declarative-timeouts">timeouts</a> và <a href="techniques/task-scheduling#declarative-intervals">intervals</a> khai báo tồn tại trong ứng dụng của bạn. Đăng ký diễn ra khi lifecycle hook `onApplicationBootstrap` xảy ra, đảm bảo rằng tất cả các modules đã tải và khai báo bất kỳ jobs được lên lịch nào.

#### Cron jobs khai báo

Một cron job lên lịch một hàm tùy ý (cuộc gọi phương thức) để chạy tự động. Cron jobs có thể chạy:

- Một lần, tại một ngày/giờ được chỉ định.
- Trên cơ sở lặp lại; các jobs lặp lại có thể chạy tại một thời điểm cụ thể trong một khoảng được chỉ định (ví dụ, mỗi giờ một lần, mỗi tuần một lần, mỗi 5 phút một lần)

Khai báo một cron job với decorator `@Cron()` đi trước định nghĩa phương thức chứa mã để được thực thi, như sau:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron('45 * * * * *')
  handleCron() {
    this.logger.debug('Called when the current second is 45');
  }
}
```

Trong ví dụ này, phương thức `handleCron()` sẽ được gọi mỗi khi giây hiện tại là `45`. Nói cách khác, phương thức sẽ chạy một lần mỗi phút, tại dấu 45 giây.

Decorator `@Cron()` hỗ trợ các [mẫu cron](http://crontab.org/) tiêu chuẩn sau:

- Dấu hoa thị (ví dụ: `*`)
- Phạm vi (ví dụ: `1-3,5`)
- Bước (ví dụ: `*/2`)

Trong ví dụ trên, chúng ta đã truyền `45 * * * * *` đến decorator. Khóa sau đây cho thấy cách mỗi vị trí trong chuỗi mẫu cron được hiểu:

<pre class="language-javascript"><code class="language-javascript">
* * * * * *
|| | | | | |
|| | | | | day of week
|| | | | months
|| | | day of month
|| | hours
|| minutes
seconds (optional)
</code></pre>

Một số mẫu cron mẫu là:

<table>
  <tbody>
    <tr>
      <td><code>* * * * * *</code></td>
      <td>mỗi giây</td>
    </tr>
    <tr>
      <td><code>45 * * * * *</code></td>
      <td>mỗi phút, tại giây thứ 45</td>
    </tr>
    <tr>
      <td><code>0 10 * * * *</code></td>
      <td>mỗi giờ, tại đầu phút thứ 10</td>
    </tr>
    <tr>
      <td><code>0 */30 9-17 * * *</code></td>
      <td>mỗi 30 phút giữa 9 sáng và 5 chiều</td>
    </tr>
   <tr>
      <td><code>0 30 11 * * 1-5</code></td>
      <td>Thứ Hai đến Thứ Sáu lúc 11:30 sáng</td>
    </tr>
  </tbody>
</table>

Package `@nestjs/schedule` cung cấp một enum tiện ích với các mẫu cron phổ biến. Bạn có thể sử dụng enum này như sau:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron(CronExpression.EVERY_30_SECONDS)
  handleCron() {
    this.logger.debug('Called every 30 seconds');
  }
}
```

Trong ví dụ này, phương thức `handleCron()` sẽ được gọi mỗi `30` giây. Nếu một exception xảy ra, nó sẽ được ghi vào bảng điều khiển, vì mọi phương thức được chú thích với `@Cron()` tự động được bọc trong một khối `try-catch`.

Ngoài ra, bạn có thể cung cấp một đối tượng `Date` JavaScript đến decorator `@Cron()`. Làm như vậy làm cho job thực thi chính xác một lần, tại ngày được chỉ định.

> info **Gợi ý** Sử dụng số học ngày JavaScript để lên lịch các jobs liên quan đến ngày hiện tại. Ví dụ, `@Cron(new Date(Date.now() + 10 * 1000))` để lên lịch một job chạy 10 giây sau khi ứng dụng bắt đầu.

Ngoài ra, bạn có thể cung cấp các tùy chọn bổ sung làm tham số thứ hai cho decorator `@Cron()`.

<table>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>
        Hữu ích để truy cập và kiểm soát một cron job sau khi nó được khai báo.
      </td>
    </tr>
    <tr>
      <td><code>timeZone</code></td>
      <td>
        Chỉ định múi giờ cho việc thực thi. Điều này sẽ sửa đổi thời gian thực tế liên quan đến múi giờ của bạn. Nếu múi giờ không hợp lệ, một lỗi được ném. Bạn có thể kiểm tra tất cả các múi giờ có sẵn tại trang web <a href="http://momentjs.com/timezone/">Moment Timezone</a>.
      </td>
    </tr>
    <tr>
      <td><code>utcOffset</code></td>
      <td>
        Điều này cho phép bạn chỉ định độ lệch của múi giờ của bạn thay vì sử dụng tham số <code>timeZone</code>.
      </td>
    </tr>
    <tr>
      <td><code>waitForCompletion</code></td>
      <td>
        Nếu <code>true</code>, không có các instance bổ sung nào của cron job sẽ chạy cho đến khi callback onTick hiện tại đã hoàn thành. Bất kỳ thực thi được lên lịch mới nào xảy ra trong khi cron job hiện tại đang chạy sẽ bị bỏ qua hoàn toàn.
      </td>
    </tr>
    <tr>
      <td><code>disabled</code></td>
      <td>
       Điều này chỉ định xem job có được thực thi hay không.
      </td>
    </tr>
  </tbody>
</table>

```typescript
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class NotificationService {
  @Cron('* * 0 * * *', {
    name: 'notifications',
    timeZone: 'Europe/Paris',
  })
  triggerNotifications() {}
}
```

Bạn có thể truy cập và kiểm soát một cron job sau khi nó được khai báo, hoặc tạo động một cron job (nơi mẫu cron của nó được định nghĩa tại runtime) với <a href="/techniques/task-scheduling#dynamic-schedule-module-api">Dynamic API</a>. Để truy cập một cron job khai báo thông qua API, bạn phải liên kết job với một tên bằng cách truyền thuộc tính `name` trong một đối tượng tùy chọn tùy chọn làm đối số thứ hai của decorator.

#### Intervals khai báo

Để khai báo rằng một phương thức nên chạy tại một khoảng (lặp lại) được chỉ định, đặt tiền tố định nghĩa phương thức với decorator `@Interval()`. Truyền giá trị khoảng, như một số tính bằng mili giây, đến decorator như được hiển thị dưới đây:

```typescript
@Interval(10000)
handleInterval() {
  this.logger.debug('Called every 10 seconds');
}
```

> info **Gợi ý** Cơ chế này sử dụng hàm JavaScript `setInterval()` dưới bề mặt. Bạn cũng có thể sử dụng một cron job để lên lịch các jobs lặp lại.

Nếu bạn muốn kiểm soát interval khai báo của mình từ bên ngoài class khai báo thông qua <a href="/techniques/task-scheduling#dynamic-schedule-module-api">Dynamic API</a>, liên kết interval với một tên bằng cách sử dụng cấu trúc sau:

```typescript
@Interval('notifications', 2500)
handleInterval() {}
```

Nếu một exception xảy ra, nó sẽ được ghi vào bảng điều khiển, vì mọi phương thức được chú thích với `@Interval()` tự động được bọc trong một khối `try-catch`.

<a href="techniques/task-scheduling#dynamic-intervals">Dynamic API</a> cũng cho phép **tạo** các intervals động, trong đó các thuộc tính của interval được định nghĩa tại runtime, và **liệt kê và xóa** chúng.

<app-banner-enterprise></app-banner-enterprise>

#### Timeouts khai báo

Để khai báo rằng một phương thức nên chạy (một lần) tại một timeout được chỉ định, đặt tiền tố định nghĩa phương thức với decorator `@Timeout()`. Truyền độ lệch thời gian tương đối (tính bằng mili giây), từ khi khởi động ứng dụng, đến decorator như được hiển thị dưới đây:

```typescript
@Timeout(5000)
handleTimeout() {
  this.logger.debug('Called once after 5 seconds');
}
```

> info **Gợi ý** Cơ chế này sử dụng hàm JavaScript `setTimeout()` dưới bề mặt.

Nếu một exception xảy ra, nó sẽ được ghi vào bảng điều khiển, vì mọi phương thức được chú thích với `@Timeout()` tự động được bọc trong một khối `try-catch`.

Nếu bạn muốn kiểm soát timeout khai báo của mình từ bên ngoài class khai báo thông qua <a href="/techniques/task-scheduling#dynamic-schedule-module-api">Dynamic API</a>, liên kết timeout với một tên bằng cách sử dụng cấu trúc sau:

```typescript
@Timeout('notifications', 2500)
handleTimeout() {}
```

<a href="techniques/task-scheduling#dynamic-timeouts">Dynamic API</a> cũng cho phép **tạo** các timeouts động, trong đó các thuộc tính của timeout được định nghĩa tại runtime, và **liệt kê và xóa** chúng.

#### Dynamic schedule module API

Module `@nestjs/schedule` cung cấp một API động cho phép quản lý các <a href="techniques/task-scheduling#declarative-cron-jobs">cron jobs</a>, <a href="techniques/task-scheduling#declarative-timeouts">timeouts</a> và <a href="techniques/task-scheduling#declarative-intervals">intervals</a> khai báo. API cũng cho phép tạo và quản lý các cron jobs, timeouts và intervals **động**, trong đó các thuộc tính được định nghĩa tại runtime.

#### Cron jobs động

Thu được một tham chiếu đến một instance `CronJob` theo tên từ bất cứ đâu trong mã của bạn bằng cách sử dụng API `SchedulerRegistry`. Đầu tiên, inject `SchedulerRegistry` sử dụng injection constructor tiêu chuẩn:

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

> info **Gợi ý** Nhập `SchedulerRegistry` từ package `@nestjs/schedule`.

Sau đó sử dụng nó trong một class như sau. Giả sử một cron job được tạo với khai báo sau:

```typescript
@Cron('* * 8 * * *', {
  name: 'notifications',
})
triggerNotifications() {}
```

Truy cập job này bằng cách sử dụng:

```typescript
const job = this.schedulerRegistry.getCronJob('notifications');

job.stop();
console.log(job.lastDate());
```

Phương thức `getCronJob()` trả về cron job được đặt tên. Đối tượng `CronJob` được trả về có các phương thức sau:

- `stop()` - dừng một job được lên lịch để chạy.
- `start()` - khởi động lại một job đã bị dừng.
- `setTime(time: CronTime)` - dừng một job, đặt thời gian mới cho nó, sau đó khởi động nó
- `lastDate()` - trả về một biểu diễn `DateTime` của ngày mà lần thực hiện cuối cùng của một job xảy ra.
- `nextDate()` - trả về một biểu diễn `DateTime` của ngày khi lần thực hiện tiếp theo của một job được lên lịch.
- `nextDates(count: number)` - Cung cấp một mảng (kích thước `count`) các biểu diễn `DateTime` cho tập hợp ngày tiếp theo sẽ kích hoạt thực thi job. `count` mặc định là 0, trả về một mảng trống.

> info **Gợi ý** Sử dụng `toJSDate()` trên các đối tượng `DateTime` để hiển thị chúng như một Date JavaScript tương đương với DateTime này.

**Tạo** một cron job mới động bằng cách sử dụng phương thức `SchedulerRegistry#addCronJob`, như sau:

```typescript
addCronJob(name: string, seconds: string) {
  const job = new CronJob(`${seconds} * * * * *`, () => {
    this.logger.warn(`time (${seconds}) for job ${name} to run!`);
  });

  this.schedulerRegistry.addCronJob(name, job);
  job.start();

  this.logger.warn(
    `job ${name} added for each minute at ${seconds} seconds!`,
  );
}
```

Trong mã này, chúng ta sử dụng đối tượng `CronJob` từ package `cron` để tạo cron job. Constructor `CronJob` nhận một mẫu cron (giống như decorator `@Cron()` <a href="techniques/task-scheduling#declarative-cron-jobs">decorator</a>) làm đối số đầu tiên của nó, và một callback để được thực thi khi bộ đếm thời gian cron kích hoạt làm đối số thứ hai của nó. Phương thức `SchedulerRegistry#addCronJob` nhận hai đối số: một tên cho `CronJob`, và đối tượng `CronJob` chính nó.

> warning **Cảnh báo** Nhớ inject `SchedulerRegistry` trước khi truy cập nó. Nhập `CronJob` từ package `cron`.

**Xóa** một cron job được đặt tên bằng cách sử dụng phương thức `SchedulerRegistry#deleteCronJob`, như sau:

```typescript
deleteCron(name: string) {
  this.schedulerRegistry.deleteCronJob(name);
  this.logger.warn(`job ${name} deleted!`);
}
```

**Liệt kê** tất cả các cron jobs bằng cách sử dụng phương thức `SchedulerRegistry#getCronJobs` như sau:

```typescript
getCrons() {
  const jobs = this.schedulerRegistry.getCronJobs();
  jobs.forEach((value, key, map) => {
    let next;
    try {
      next = value.nextDate().toJSDate();
    } catch (e) {
      next = 'error: next fire date is in the past!';
    }
    this.logger.log(`job: ${key} -> next: ${next}`);
  });
}
```

Phương thức `getCronJobs()` trả về một `map`. Trong mã này, chúng ta lặp qua map và cố gắng truy cập phương thức `nextDate()` của mỗi `CronJob`. Trong API `CronJob`, nếu một job đã kích hoạt và không có ngày kích hoạt trong tương lai, nó ném một exception.

#### Intervals động

Thu được một tham chiếu đến một interval với phương thức `SchedulerRegistry#getInterval`. Như ở trên, inject `SchedulerRegistry` sử dụng injection constructor tiêu chuẩn:

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

Và sử dụng nó như sau:

```typescript
const interval = this.schedulerRegistry.getInterval('notifications');
clearInterval(interval);
```

**Tạo** một interval mới động bằng cách sử dụng phương thức `SchedulerRegistry#addInterval`, như sau:

```typescript
addInterval(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Interval ${name} executing at time (${milliseconds})!`);
  };

  const interval = setInterval(callback, milliseconds);
  this.schedulerRegistry.addInterval(name, interval);
}
```

Trong mã này, chúng ta tạo một interval JavaScript tiêu chuẩn, sau đó truyền nó đến phương thức `SchedulerRegistry#addInterval`.
Phương thức đó nhận hai đối số: một tên cho interval, và interval chính nó.

**Xóa** một interval được đặt tên bằng cách sử dụng phương thức `SchedulerRegistry#deleteInterval`, như sau:

```typescript
deleteInterval(name: string) {
  this.schedulerRegistry.deleteInterval(name);
  this.logger.warn(`Interval ${name} deleted!`);
}
```

**Liệt kê** tất cả các intervals bằng cách sử dụng phương thức `SchedulerRegistry#getIntervals` như sau:

```typescript
getIntervals() {
  const intervals = this.schedulerRegistry.getIntervals();
  intervals.forEach(key => this.logger.log(`Interval: ${key}`));
}
```

#### Timeouts động

Thu được một tham chiếu đến một timeout với phương thức `SchedulerRegistry#getTimeout`. Như ở trên, inject `SchedulerRegistry` sử dụng injection constructor tiêu chuẩn:

```typescript
constructor(private readonly schedulerRegistry: SchedulerRegistry) {}
```

Và sử dụng nó như sau:

```typescript
const timeout = this.schedulerRegistry.getTimeout('notifications');
clearTimeout(timeout);
```

**Tạo** một timeout mới động bằng cách sử dụng phương thức `SchedulerRegistry#addTimeout`, như sau:

```typescript
addTimeout(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Timeout ${name} executing after (${milliseconds})!`);
  };

  const timeout = setTimeout(callback, milliseconds);
  this.schedulerRegistry.addTimeout(name, timeout);
}
```

Trong mã này, chúng ta tạo một timeout JavaScript tiêu chuẩn, sau đó truyền nó đến phương thức `SchedulerRegistry#addTimeout`.
Phương thức đó nhận hai đối số: một tên cho timeout, và timeout chính nó.

**Xóa** một timeout được đặt tên bằng cách sử dụng phương thức `SchedulerRegistry#deleteTimeout`, như sau:

```typescript
deleteTimeout(name: string) {
  this.schedulerRegistry.deleteTimeout(name);
  this.logger.warn(`Timeout ${name} deleted!`);
}
```

**Liệt kê** tất cả các timeouts bằng cách sử dụng phương thức `SchedulerRegistry#getTimeouts` như sau:

```typescript
getTimeouts() {
  const timeouts = this.schedulerRegistry.getTimeouts();
  timeouts.forEach(key => this.logger.log(`Timeout: ${key}`));
}
```

#### Ví dụ

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/27-scheduling).