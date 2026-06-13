### Nest Commander

Mở rộng trên tài liệu [ứng dụng độc lập](/standalone-applications) cũng có gói [nest-commander](https://jmcdo29.github.io/nest-commander) để viết các ứng dụng dòng lệnh trong một cấu trúc tương tự như ứng dụng Nest điển hình của bạn.

> info **info** `nest-commander` là một gói bên thứ ba và không được quản lý bởi toàn bộ nhóm cốt lõi NestJS. Vui lòng báo cáo bất kỳ vấn đề nào được tìm thấy với thư viện trong [repository thích hợp](https://github.com/jmcdo29/nest-commander/issues/new/choose)

#### Cài đặt

Giống như bất kỳ gói nào khác, bạn phải cài đặt nó trước khi có thể sử dụng nó.

```bash
$ npm i nest-commander
```

#### Một file Command

`nest-commander` làm cho việc viết các ứng dụng dòng lệnh mới trở nên dễ dàng với [decorators](https://www.typescriptlang.org/docs/handbook/decorators.html) thông qua decorator `@Command()` cho các class và decorator `@Option()` cho các phương thức của class đó. Mỗi file lệnh nên triển khai class trừu tượng `CommandRunner` và nên được trang trí với decorator `@Command()`.

Mỗi lệnh được xem như một `@Injectable()` bởi Nest, vì vậy Dependency Injection bình thường của bạn vẫn hoạt động như bạn mong đợi. Điều duy nhất cần lưu ý là class trừu tượng `CommandRunner`, nên được triển khai bởi mỗi lệnh. Class trừu tượng `CommandRunner` đảm bảo rằng tất cả các lệnh có một phương thức `run` trả về `Promise<void>` và nhận các tham số `string[], Record<string, any>`. Lệnh `run` là nơi bạn có thể bắt đầu tất cả logic của mình, nó sẽ nhận bất kỳ tham số nào không khớp với các cờ tùy chọn và truyền chúng dưới dạng mảng, chỉ trong trường hợp bạn thực sự có ý định làm việc với nhiều tham số. Đối với các tùy chọn, `Record<string, any>`, tên của các thuộc tính này khớp với thuộc tính `name` được cung cấp cho các decorator `@Option()`, trong khi giá trị của chúng khớp với trả về của handler tùy chọn. Nếu bạn muốn type-safety tốt hơn, bạn được chào mừng để tạo một interface cho các tùy chọn của bạn cũng như.

#### Chạy Command

Tương tự như cách trong một ứng dụng NestJS chúng ta có thể sử dụng `NestFactory` để tạo một máy chủ cho chúng ta, và chạy nó sử dụng `listen`, gói `nest-commander` expose một API dễ sử dụng để chạy máy chủ của bạn. Nhập `CommandFactory` và sử dụng phương thức `static` `run` và chuyển vào module gốc của ứng dụng của bạn. Điều này có thể trông như dưới đây

```ts
import { CommandFactory } from 'nest-commander';
import { AppModule } from './app.module';

async function bootstrap() {
  await CommandFactory.run(AppModule);
}

bootstrap();
```

Theo mặc định, logger của Nest bị vô hiệu hóa khi sử dụng `CommandFactory`. Tuy nhiên, có thể cung cấp nó, như đối số thứ hai cho hàm `run`. Bạn có thể cung cấp một logger NestJS tùy chỉnh, hoặc một mảng các mức log bạn muốn giữ - có thể hữu ích để ít nhất cung cấp `['error']` ở đây, nếu bạn chỉ muốn in ra các log lỗi của Nest.

```ts
import { CommandFactory } from 'nest-commander';
import { AppModule } from './app.module';
import { LogService } from './log.service';

async function bootstrap() {
  await CommandFactory.run(AppModule, new LogService());

  // hoặc, nếu bạn chỉ muốn in các cảnh báo và lỗi của Nest
  await CommandFactory.run(AppModule, ['warn', 'error']);
}

bootstrap();
```

Và đó là tất cả. Bên dưới, `CommandFactory` sẽ lo việc gọi `NestFactory` cho bạn và gọi `app.close()` khi cần thiết, vì vậy bạn không cần lo về rò rỉ bộ nhớ ở đó. Nếu bạn cần thêm một số xử lý lỗi, luôn có `try/catch` bọc lệnh `run`, hoặc bạn có thể chuỗi một số phương thức `.catch()` vào lệnh gọi `bootstrap()`.

#### Kiểm tra

Vậy thì việc viết một script dòng lệnh siêu tuyệt vời có ích gì nếu bạn không thể kiểm tra nó siêu dễ dàng, đúng không? May mắn thay, `nest-commander` có một số tiện ích bạn có thể sử dụng phù hợp hoàn hảo với hệ sinh thái NestJS, nó sẽ cảm thấy như ở nhà đối với bất kỳ Nestlings nào. Thay vì sử dụng `CommandFactory` để xây dựng lệnh trong chế độ kiểm tra, bạn có thể sử dụng `CommandTestFactory` và chuyển vào metadata của bạn, rất tương tự như cách `Test.createTestingModule` từ `@nestjs/testing` hoạt động. Trên thực tế, nó sử dụng gói này bên dưới. Bạn cũng vẫn có thể chuỗi các phương thức `overrideProvider` trước khi gọi `compile()` để bạn có thể thay đổi các phần DI ngay trong bài kiểm tra.

#### Đặt tất cả lại với nhau

Class sau sẽ tương đương với việc có một lệnh CLI có thể nhận subcommand `basic` hoặc được gọi trực tiếp, với `-n`, `-s`, và `-b` (cùng với các cờ dài của chúng) đều được hỗ trợ và với các parser tùy chỉnh cho mỗi tùy chọn. Cờ `--help` cũng được hỗ trợ, như thường lệ với commander.

```ts
import { Command, CommandRunner, Option } from 'nest-commander';
import { LogService } from './log.service';

interface BasicCommandOptions {
  string?: string;
  boolean?: boolean;
  number?: number;
}

@Command({ name: 'basic', description: 'A parameter parse' })
export class BasicCommand extends CommandRunner {
  constructor(private readonly logService: LogService) {
    super()
  }

  async run(
    passedParam: string[],
    options?: BasicCommandOptions,
  ): Promise<void> {
    if (options?.boolean !== undefined && options?.boolean !== null) {
      this.runWithBoolean(passedParam, options.boolean);
    } else if (options?.number) {
      this.runWithNumber(passedParam, options.number);
    } else if (options?.string) {
      this.runWithString(passedParam, options.string);
    } else {
      this.runWithNone(passedParam);
    }
  }

  @Option({
    flags: '-n, --number [number]',
    description: 'A basic number parser',
  })
  parseNumber(val: string): number {
    return Number(val);
  }

  @Option({
    flags: '-s, --string [string]',
    description: 'A string return',
  })
  parseString(val: string): string {
    return val;
  }

  @Option({
    flags: '-b, --boolean [boolean]',
    description: 'A boolean parser',
  })
  parseBoolean(val: string): boolean {
    return JSON.parse(val);
  }

  runWithString(param: string[], option: string): void {
    this.logService.log({ param, string: option });
  }

  runWithNumber(param: string[], option: number): void {
    this.logService.log({ param, number: option });
  }

  runWithBoolean(param: string[], option: boolean): void {
    this.logService.log({ param, boolean: option });
  }

  runWithNone(param: string[]): void {
    this.logService.log({ param });
  }
}
```

Đảm bảo class lệnh được thêm vào một module

```ts
@Module({
  providers: [LogService, BasicCommand],
})
export class AppModule {}
```

Và bây giờ để có thể chạy CLI trong main.ts của bạn bạn có thể làm như sau

```ts
async function bootstrap() {
  await CommandFactory.run(AppModule);
}

bootstrap();
```

Và chỉ như vậy, bạn đã có một ứng dụng dòng lệnh.

#### Thêm thông tin

Truy cập trang tài liệu [nest-commander](https://jmcdo29.github.io/nest-commander) để biết thêm thông tin, ví dụ, và tài liệu API.