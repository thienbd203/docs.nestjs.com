### Read-Eval-Print-Loop (REPL)

REPL là một môi trường tương tác đơn giản lấy đầu vào người dùng đơn lẻ, thực thi chúng, và trả về kết quả cho người dùng.
Tính năng REPL cho phép bạn kiểm tra đồ thị phụ thuộc của bạn và gọi các phương thức trên các provider (và controllers) của bạn trực tiếp từ terminal của bạn.

#### Sử dụng

Để chạy ứng dụng NestJS của bạn ở chế độ REPL, tạo một file `repl.ts` mới (cùng với file `main.ts` hiện có) và thêm mã sau vào bên trong:

```typescript
@@filename(repl)
import { repl } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  await repl(AppModule);
}
bootstrap();
@@switch
import { repl } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  await repl(AppModule);
}
bootstrap();
```

Bây giờ trong terminal của bạn, khởi động REPL với lệnh sau:

```bash
$ npm run start -- --entryFile repl
```

> info **Gợi ý** `repl` trả về một đối tượng [máy chủ REPL Node.js](https://nodejs.org/api/repl.html).

Khi nó đang chạy và hoạt động, bạn nên thấy thông báo sau trong console của bạn:

```bash
LOG [NestFactory] Starting Nest application...
LOG [InstanceLoader] AppModule dependencies initialized
LOG REPL initialized
```

Và bây giờ bạn có thể bắt đầu tương tác với đồ thị phụ thuộc của bạn. Ví dụ, bạn có thể truy xuất một `AppService` (chúng ta đang sử dụng dự án khởi động như một ví dụ ở đây) và gọi phương thức `getHello()`:

```typescript
> get(AppService).getHello()
'Hello World!'
```

Bạn có thể thực thi bất kỳ mã JavaScript nào từ trong terminal của bạn, ví dụ, gán một instance của `AppController` cho một biến cục bộ, và sử dụng `await` để gọi một phương thức bất đồng bộ:

```typescript
> appController = get(AppController)
AppController { appService: AppService {} }
> await appController.getHello()
'Hello World!'
```

Để hiển thị tất cả các phương thức công khai có sẵn trên một provider hoặc controller cụ thể, sử dụng hàm `methods()`, như sau:

```typescript
> methods(AppController)

Methods:
 ◻ getHello
```

Để in tất cả các module đã đăng ký như một danh sách cùng với các controller và provider của chúng, sử dụng `debug()`.

```typescript
> debug()

AppModule:
 - controllers:
  ◻ AppController
 - providers:
  ◻ AppService
```

Demo nhanh:

<figure><img src="/assets/repl.gif" alt="REPL example" /></figure>

Bạn có thể tìm thêm thông tin về các phương thức gốc đã định nghĩa sẵn trong phần dưới đây.

#### Hàm gốc

REPL NestJS tích hợp sẵn đi kèm với một vài hàm gốc có sẵn toàn cầu khi bạn bắt đầu REPL. Bạn có thể gọi `help()` để liệt kê chúng.

Nếu bạn không nhớ chữ ký (tức là các tham số mong đợi và kiểu trả về) của một hàm, bạn có thể gọi `<function_name>.help`.
Ví dụ:

```text
> $.help
Retrieves an instance of either injectable or controller, otherwise, throws exception.
Interface: $(token: InjectionToken) => any
```

> info **Gợi ý** Các interface hàm đó được viết trong [cú pháp biểu thức kiểu hàm TypeScript](https://www.typescriptlang.org/docs/handbook/2/functions.html#function-type-expressions).

|| Hàm     | Mô tả                                                                                                        | Chữ ký                                                             |
|| ------------ | ------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
|| `debug`      | In tất cả các module đã đăng ký như một danh sách cùng với các controller và provider của chúng.                              | `debug(moduleCls?: ClassRef \| string) => void`                       |
|| `get` hoặc `$` | Truy xuất một instance của injectable hoặc controller, nếu không, ném ngoại lệ.                             | `get(token: InjectionToken) => any`                                   |
|| `methods`    | Hiển thị tất cả các phương thức công khai có sẵn trên một provider hoặc controller cụ thể.                                            | `methods(token: ClassRef \| string) => void`                          |
|| `resolve`    | Giải quyết instance transient hoặc phạm vi yêu cầu của injectable hoặc controller, nếu không, ném ngoại lệ.     | `resolve(token: InjectionToken, contextId: any) => Promise<any>`      |
|| `select`     | Cho phép điều hướng qua cây module, ví dụ, để kéo ra một instance cụ thể từ module đã chọn. | `select(token: DynamicModule \| ClassRef) => INestApplicationContext` |

#### Chế độ watch

Trong quá trình phát triển, việc chạy REPL ở chế độ watch rất hữu ích để phản ánh tất cả các thay đổi mã tự động:

```bash
$ npm run start -- --watch --entryFile repl
```

Điều này có một khuyết điểm, lịch sử lệnh REPL bị loại bỏ sau mỗi lần tải lại có thể gây phiền toái.
May mắn thay, có một giải pháp rất đơn giản. Sửa đổi hàm `bootstrap` của bạn như sau:

```typescript
async function bootstrap() {
  const replServer = await repl(AppModule);
  replServer.setupHistory(".nestjs_repl_history", (err) => {
    if (err) {
      console.error(err);
    }
  });
}
```

Bây giờ lịch sử được bảo tồn giữa các lần chạy/tải lại.