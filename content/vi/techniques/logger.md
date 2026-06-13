### Logger

Nest đi kèm với một logger dựa trên văn bản tích hợp sẵn được sử dụng trong quá trình khởi tạo ứng dụng và một số trường hợp khác như hiển thị các exception đã bắt được (tức là hệ thống logging). Chức năng này được cung cấp thông qua class `Logger` trong package `@nestjs/common`. Bạn có thể kiểm soát hoàn toàn hành vi của hệ thống logging, bao gồm bất kỳ điều sau:

- vô hiệu hóa logging hoàn toàn
- chỉ định mức độ chi tiết của log (ví dụ: hiển thị lỗi, cảnh báo, thông tin debug, v.v.)
- cấu hình định dạng của thông báo log (thô, json, có màu, v.v.)
- ghi đè timestamp trong logger mặc định (ví dụ: sử dụng tiêu chuẩn ISO8601 làm định dạng ngày tháng)
- ghi đè hoàn toàn logger mặc định
- tùy chỉnh logger mặc định bằng cách mở rộng nó
- tận dụng dependency injection để đơn giản hóa việc kết hợp và kiểm thử ứng dụng của bạn

Bạn cũng có thể sử dụng logger tích hợp sẵn hoặc tạo triển khai tùy chỉnh của riêng mình để ghi log các sự kiện và thông báo cấp ứng dụng của riêng bạn.

Nếu ứng dụng của bạn yêu cầu tích hợp với các hệ thống logging bên ngoài, logging dựa trên file tự động, hoặc chuyển tiếp logs đến một dịch vụ logging tập trung, bạn có thể triển khai giải pháp logging tùy chỉnh hoàn toàn bằng cách sử dụng thư viện logging Node.js. Một lựa chọn phổ biến là [Pino](https://github.com/pinojs/pino), được biết đến với hiệu suất cao và tính linh hoạt.

#### Tùy chỉnh cơ bản

Để vô hiệu hóa logging, đặt thuộc tính `logger` thành `false` trong đối tượng tùy chọn ứng dụng Nest (tùy chọn) được truyền làm đối số thứ hai cho phương thức `NestFactory.create()`.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: false,
});
await app.listen(process.env.PORT ?? 3000);
```

Để bật các mức độ logging cụ thể, đặt thuộc tính `logger` thành một mảng các chuỗi chỉ định các mức độ log để hiển thị, như sau:

```typescript
const app = await NestFactory.create(AppModule, {
  logger: ['error', 'warn'],
});
await app.listen(process.env.PORT ?? 3000);
```

Các giá trị trong mảng có thể là bất kỳ kết hợp nào của `'log'`, `'fatal'`, `'error'`, `'warn'`, `'debug'`, và `'verbose'`.

> info **Gợi ý** Mức độ log trong Nest là xếp tầng (kế thừa). Điều này có nghĩa là cung cấp một mức độ log cụ thể (như `'log'`) sẽ tự động bao gồm tất cả các mức độ có mức độ nghiêm trọng cao hơn (ví dụ: `'warn'`, `'error'`, và `'fatal'`).

Để vô hiệu hóa đầu ra có màu, truyền đối tượng `ConsoleLogger` với thuộc tính `colors` được đặt thành `false` làm giá trị của thuộc tính `logger`.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({
    colors: false,
  }),
});
```

Để cấu hình tiền tố cho mỗi thông báo log, truyền đối tượng `ConsoleLogger` với thuộc tính `prefix` được đặt:

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({
    prefix: 'MyApp', // Mặc định là "Nest"
  }),
});
```

Dưới đây là tất cả các tùy chọn có sẵn được liệt kê trong bảng dưới đây:

|| Tùy chọn            | Mô tả                                                                                                                                                                                                                                                                                                                                          | Mặc định                                        |
|| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
|| `logLevels`       | Các mức độ log được bật.                                                                                                                                                                                                                                                                                                                                  | `['log', 'fatal', 'error', 'warn', 'debug', 'verbose']` |
|| `timestamp`       | Nếu được bật, sẽ in timestamp (chênh lệch thời gian) giữa thông báo log hiện tại và trước đó. Lưu ý: Tùy chọn này không được sử dụng khi `json` được bật.                                                                                                                                                                                                   | `false`                                        |
|| `prefix`          | Một tiền tố sẽ được sử dụng cho mỗi thông báo log. Lưu ý: Tùy chọn này không được sử dụng khi `json` được bật.                                                                                                                                                                                                                                                      | `Nest`                                         |
|| `json`            | Nếu được bật, sẽ in thông báo log ở định dạng JSON.                                                                                                                                                                                                                                                                                               | `false`                                        |
|| `colors`          | Nếu được bật, sẽ in thông báo log có màu. Mặc định là true nếu json bị vô hiệu hóa, ngược lại là false.                                                                                                                                                                                                                                                  | `true`                                         |
|| `context`         | Bối cảnh của logger.                                                                                                                                                                                                                                                                                                                           | `undefined`                                    |
|| `compact`         | Nếu được bật, sẽ in thông báo log trong một dòng duy nhất, ngay cả khi đó là một đối tượng có nhiều thuộc tính. Nếu được đặt thành một số, n phần tử bên trong nhất sẽ được kết hợp trên một dòng duy nhất miễn là tất cả các thuộc tính phù hợp với breakLength. Các phần tử mảng ngắn cũng được nhóm lại cùng nhau.                                                                 | `true`                                         |
|| `maxArrayLength`  | Chỉ định số lượng tối đa các phần tử Array, TypedArray, Map, Set, WeakMap, và WeakSet để bao gồm khi định dạng. Đặt thành null hoặc Infinity để hiển thị tất cả các phần tử. Đặt thành 0 hoặc số âm để không hiển thị phần tử nào. Bị bỏ qua khi `json` được bật, màu bị vô hiệu hóa, và `compact` được đặt thành true vì nó tạo ra đầu ra JSON có thể phân tích cú pháp.             | `100`                                          |
|| `maxStringLength` | Chỉ định số lượng tối đa ký tự để bao gồm khi định dạng. Đặt thành null hoặc Infinity để hiển thị tất cả các phần tử. Đặt thành 0 hoặc số âm để không hiển thị ký tự nào. Bị bỏ qua khi `json` được bật, màu bị vô hiệu hóa, và `compact` được đặt thành true vì nó tạo ra đầu ra JSON có thể phân tích cú pháp.                                                           | `10000`                                        |
|| `sorted`          | Nếu được bật, sẽ sắp xếp các khóa trong khi định dạng đối tượng. Cũng có thể là một hàm sắp xếp tùy chỉnh. Bị bỏ qua khi `json` được bật, màu bị vô hiệu hóa, và `compact` được đặt thành true vì nó tạo ra đầu ra JSON có thể phân tích cú pháp.                                                                                                                                | `false`                                        |
|| `depth`           | Chỉ định số lần đệ quy trong khi định dạng đối tượng. Điều này hữu ích để kiểm tra các đối tượng lớn. Để đệ quy đến kích thước ngăn xếp gọi tối đa, truyền Infinity hoặc null. Bị bỏ qua khi `json` được bật, màu bị vô hiệu hóa, và `compact` được đặt thành true vì nó tạo ra đầu ra JSON có thể phân tích cú pháp.                                         | `5`                                            |
|| `showHidden`      | Nếu true, các ký hiệu và thuộc tính không thể liệt kê của đối tượng được bao gồm trong kết quả được định dạng. Các mục nhập WeakMap và WeakSet cũng được bao gồm cũng như các thuộc tính prototype do người dùng định nghĩa                                                                                                                                                             | `false`                                        |
|| `breakLength`     | Độ dài tại đó các giá trị đầu vào được chia thành nhiều dòng. Đặt thành Infinity để định dạng đầu vào dưới dạng một dòng duy nhất (kết hợp với "compact" được đặt thành true). Mặc định là Infinity khi "compact" là true, 80 nếu không. Bị bỏ qua khi `json` được bật, màu bị vô hiệu hóa, và `compact` được đặt thành true vì nó tạo ra đầu ra JSON có thể phân tích cú pháp. | `Infinity`                                     |

#### Logging JSON

Logging JSON là thiết yếu cho khả năng quan sát ứng dụng hiện đại và tích hợp với các hệ thống quản lý log. Để bật logging JSON trong ứng dụng NestJS của bạn, cấu hình đối tượng `ConsoleLogger` với thuộc tính `json` của nó được đặt thành `true`. Sau đó, cung cấp cấu hình logger này làm giá trị cho thuộc tính `logger` khi tạo instance ứng dụng.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({
    json: true,
  }),
});
```

Cấu hình này xuất logs ở định dạng JSON có cấu trúc, giúp dễ dàng tích hợp với các hệ thống bên ngoài như các bộ tổng hợp log và nền tảng đám mây. Ví dụ, các nền tảng như **AWS ECS** (Elastic Container Service) hỗ trợ sẵn JSON logs, cho phép các tính năng nâng cao như:

- **Lọc Log**: Dễ dàng thu hẹp logs dựa trên các trường như mức độ log, timestamp, hoặc metadata tùy chỉnh.
- **Tìm kiếm và Phân tích**: Sử dụng các công cụ truy vấn để phân tích và theo dõi xu hướng trong hành vi của ứng dụng của bạn.

Ngoài ra, nếu bạn đang sử dụng [NestJS Mau](https://mau.nestjs.com), logging JSON đơn giản hóa quá trình xem logs ở định dạng có cấu trúc, được tổ chức tốt, điều này đặc biệt hữu ích cho debug và giám sát hiệu suất.

> info **Lưu ý** Khi `json` được đặt thành `true`, `ConsoleLogger` tự động vô hiệu hóa màu văn bản bằng cách đặt thuộc tính `colors` thành `false`. Điều này đảm bảo rằng đầu ra vẫn là JSON hợp lệ, không có các thành phần định dạng. Tuy nhiên, cho mục đích phát triển, bạn có thể ghi đè hành vi này bằng cách đặt rõ ràng `colors` thành `true`. Điều này thêm các log JSON có màu, điều này có thể làm cho các mục log dễ đọc hơn trong quá trình debug cục bộ.

Khi logging JSON được bật, đầu ra log sẽ trông như sau (trong một dòng duy nhất):

```json
{
  "level": "log",
  "pid": 19096,
  "timestamp": 1607370779834,
  "message": "Starting Nest application...",
  "context": "NestFactory"
}
```

Bạn có thể thấy các biến thể khác trong [Pull Request](https://github.com/nestjs/nest/pull/14121) này.

#### Sử dụng logger để logging ứng dụng

Chúng ta có thể kết hợp một số kỹ thuật ở trên để cung cấp hành vi và định dạng nhất quán trên cả logging hệ thống Nest và logging sự kiện/thông báo ứng dụng của riêng chúng ta.

Một thực hành tốt là khởi tạo class `Logger` từ `@nestjs/common` trong mỗi service của chúng ta. Chúng ta có thể cung cấp tên service của mình làm đối số `context` trong constructor của `Logger`, như sau:

```typescript
import { Logger, Injectable } from '@nestjs/common';

@Injectable()
class MyService {
  private readonly logger = new Logger(MyService.name);

  doSomething() {
    this.logger.log('Doing something...');
  }
}
```

Trong triển khai logger mặc định, `context` được in trong ngoặc vuông, như `NestFactory` trong ví dụ dưới đây:

```bash
[Nest] 19096   - 12/08/2019, 7:12:59 AM   [NestFactory] Starting Nest application...
```

Nếu chúng ta cung cấp một logger tùy chỉnh thông qua `app.useLogger()`, nó thực sự sẽ được sử dụng nội bộ bởi Nest. Điều đó có nghĩa là mã của chúng ta vẫn không phụ thuộc vào triển khai, trong khi chúng ta có thể dễ dàng thay thế logger mặc định bằng logger tùy chỉnh của mình bằng cách gọi `app.useLogger()`.

Cách này, nếu chúng ta làm theo các bước từ phần trước và gọi `app.useLogger(app.get(MyLogger))`, các cuộc gọi sau này đến `this.logger.log()` từ `MyService` sẽ dẫn đến các cuộc gọi đến phương thức `log` từ instance `MyLogger`.

Điều này sẽ phù hợp với hầu hết các trường hợp. Nhưng nếu bạn cần tùy chỉnh nhiều hơn (như thêm và gọi các phương thức tùy chỉnh), hãy chuyển đến phần tiếp theo.

#### Logs với timestamps

Để bật logging timestamp cho mọi thông báo được ghi log, bạn có thể sử dụng cài đặt tùy chọn `timestamp: true` khi tạo instance logger.

```typescript
import { Logger, Injectable } from '@nestjs/common';

@Injectable()
class MyService {
  private readonly logger = new Logger(MyService.name, { timestamp: true });

  doSomething() {
    this.logger.log('Doing something with timestamp here ->');
  }
}
```

Điều này sẽ tạo ra đầu ra ở định dạng sau:

```bash
[Nest] 19096   - 04/19/2024, 7:12:59 AM   [MyService] Doing something with timestamp here +5ms
```

Lưu ý `+5ms` ở cuối dòng. Đối với mỗi câu lệnh log, chênh lệch thời gian từ thông báo trước đó được tính toán và hiển thị ở cuối dòng.

#### Triển khai tùy chỉnh

Bạn có thể cung cấp một triển khai logger tùy chỉnh để được sử dụng bởi Nest cho hệ thống logging bằng cách đặt giá trị của thuộc tính `logger` thành một đối tượng thỏa mãn interface `LoggerService`. Ví dụ, bạn có thể nói với Nest sử dụng đối tượng `console` JavaScript toàn cầu tích hợp sẵn (triển khai interface `LoggerService`), như sau:

```typescript
const app = await NestFactory.create(AppModule, {
  logger: console,
});
await app.listen(process.env.PORT ?? 3000);
```

Triển khai logger tùy chỉnh của riêng bạn là đơn giản. Chỉ cần triển khai từng phương thức của interface `LoggerService` như được hiển thị dưới đây.

```typescript
import { LoggerService, Injectable } from '@nestjs/common';

@Injectable()
export class MyLogger implements LoggerService {
  /**
   * Viết một log mức độ 'log'.
   */
  log(message: any, ...optionalParams: any[]) {}

  /**
   * Viết một log mức độ 'fatal'.
   */
  fatal(message: any, ...optionalParams: any[]) {}

  /**
   * Viết một log mức độ 'error'.
   */
  error(message: any, ...optionalParams: any[]) {}

  /**
   * Viết một log mức độ 'warn'.
   */
  warn(message: any, ...optionalParams: any[]) {}

  /**
   * Viết một log mức độ 'debug'.
   */
  debug?(message: any, ...optionalParams: any[]) {}

  /**
   * Viết một log mức độ 'verbose'.
   */
  verbose?(message: any, ...optionalParams: any[]) {}
}
```

Sau đó bạn có thể cung cấp một instance của `MyLogger` thông qua thuộc tính `logger` của đối tượng tùy chọn ứng dụng Nest.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new MyLogger(),
});
await app.listen(process.env.PORT ?? 3000);
```

Kỹ thuật này, trong khi đơn giản, không sử dụng dependency injection cho class `MyLogger`. Điều này có thể gây ra một số thách thức, đặc biệt là cho kiểm thử, và giới hạn khả năng tái sử dụng của `MyLogger`. Để có giải pháp tốt hơn, xem phần <a href="techniques/logger#dependency-injection">Dependency Injection</a> dưới đây.

#### Mở rộng logger tích hợp sẵn

Thay vì viết một logger từ đầu, bạn có thể đáp ứng nhu cầu của mình bằng cách mở rộng class `ConsoleLogger` tích hợp sẵn và ghi đè hành vi được chọn của triển khai mặc định.

```typescript
import { ConsoleLogger } from '@nestjs/common';

export class MyLogger extends ConsoleLogger {
  error(message: any, stack?: string, context?: string) {
    // thêm logic tùy chỉnh của bạn ở đây
    super.error(...arguments);
  }
}
```

Bạn có thể sử dụng logger mở rộng như vậy trong các module tính năng của bạn như được mô tả trong phần <a href="techniques/logger#using-the-logger-for-application-logging">Sử dụng logger để logging ứng dụng</a> dưới đây.

Bạn có thể nói với Nest sử dụng logger mở rộng của bạn cho hệ thống logging bằng cách truyền một instance của nó thông qua thuộc tính `logger` của đối tượng tùy chọn ứng dụng (như được hiển thị trong phần <a href="techniques/logger#custom-logger-implementation">Triển khai tùy chỉnh</a> ở trên), hoặc bằng cách sử dụng kỹ thuật được hiển thị trong phần <a href="techniques/logger#dependency-injection">Dependency Injection</a> dưới đây. Nếu bạn làm vậy, bạn nên cẩn thận gọi `super`, như được hiển thị trong mã mẫu ở trên, để ủy quyền cuộc gọi phương thức log cụ thể cho class cha (tích hợp sẵn) để Nest có thể dựa vào các tính năng tích hợp sẵn mà nó mong đợi.

<app-banner-courses></app-banner-courses>

#### Dependency injection

Để có chức năng logging nâng cao hơn, bạn sẽ muốn tận dụng dependency injection. Ví dụ, bạn có thể muốn inject một `ConfigService` vào logger của bạn để tùy chỉnh nó, và lần lượt inject logger tùy chỉnh của bạn vào các controller và/hoặc provider khác. Để bật dependency injection cho logger tùy chỉnh của bạn, tạo một class triển khai `LoggerService` và đăng ký class đó làm provider trong một module. Ví dụ, bạn có thể

1. Định nghĩa một class `MyLogger` mở rộng class `ConsoleLogger` tích hợp sẵn hoặc ghi đè hoàn toàn nó, như được hiển thị trong các phần trước. Đảm bảo triển khai interface `LoggerService`.
2. Tạo một `LoggerModule` như được hiển thị dưới đây, và cung cấp `MyLogger` từ module đó.

```typescript
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```

Với cấu trúc này, bây giờ bạn đang cung cấp logger tùy chỉnh của mình để được sử dụng bởi bất kỳ module nào khác. Vì class `MyLogger` của bạn là một phần của một module, nó có thể sử dụng dependency injection (ví dụ, để inject một `ConfigService`). Còn một kỹ thuật nữa cần thiết để cung cấp logger tùy chỉnh này để được sử dụng bởi Nest cho hệ thống logging (ví dụ, cho khởi tạo và xử lý lỗi).

Vì khởi tạo ứng dụng (`NestFactory.create()`) diễn ra ngoài bối cảnh của bất kỳ module nào, nó không tham gia vào giai đoạn Dependency Injection bình thường của khởi tạo. Vì vậy chúng ta phải đảm bảo rằng ít nhất một module ứng dụng nhập `LoggerModule` để kích hoạt Nest khởi tạo một instance singleton của class `MyLogger` của chúng ta.

Sau đó chúng ta có thể hướng dẫn Nest sử dụng cùng một instance singleton của `MyLogger` với cấu trúc sau:

```typescript
const app = await NestFactory.create(AppModule, {
  bufferLogs: true,
});
app.useLogger(app.get(MyLogger));
await app.listen(process.env.PORT ?? 3000);
```

> info **Lưu ý** Trong ví dụ trên, chúng ta đặt `bufferLogs` thành `true` để đảm bảo tất cả logs sẽ được đệm cho đến khi một logger tùy chỉnh được đính kèm (`MyLogger` trong trường hợp này) và quá trình khởi tạo ứng dụng hoàn thành hoặc thất bại. Nếu quá trình khởi tạo thất bại, Nest sẽ quay lại `ConsoleLogger` gốc để in ra bất kỳ thông báo lỗi nào được báo cáo. Ngoài ra, bạn có thể đặt `autoFlushLogs` thành `false` (mặc định `true`) để xóa logs thủ công (sử dụng phương thức `Logger.flush()`).

Ở đây chúng ta sử dụng phương thức `get()` trên instance `NestApplication` để truy xuất instance singleton của đối tượng `MyLogger`. Kỹ thuật này về cơ bản là một cách để "inject" một instance của logger để được sử dụng bởi Nest. Cuộc gọi `app.get()` truy xuất instance singleton của `MyLogger`, và phụ thuộc vào instance đó được inject trước trong một module khác, như được mô tả ở trên.

Bạn cũng có thể inject provider `MyLogger` này trong các lớp tính năng của bạn, do đó đảm bảo hành vi logging nhất quán trên cả logging hệ thống Nest và logging ứng dụng. Xem <a href="techniques/logger#using-the-logger-for-application-logging">Sử dụng logger để logging ứng dụng</a> và <a href="techniques/logger#injecting-a-custom-logger">Inject logger tùy chỉnh</a> dưới đây để biết thêm thông tin.

#### Injecting a custom logger