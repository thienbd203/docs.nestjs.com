### Streaming files

> info **Lưu ý** Chương này cho thấy cách bạn có thể stream file từ **ứng dụng HTTP** của mình. Các ví dụ được trình bày bên dưới không áp dụng cho các ứng dụng GraphQL hoặc Microservice.

Có thể có lúc bạn muốn gửi lại một file từ REST API của mình cho client. Để làm điều này với Nest, thông thường bạn sẽ làm như sau:

```ts
@Controller('file')
export class FileController {
  @Get()
  getFile(@Res() res: Response) {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    file.pipe(res);
  }
}
```

Nhưng khi làm như vậy, bạn sẽ mất quyền truy cập vào logic interceptor post-controller của mình. Để xử lý điều này, bạn có thể trả về một instance `StreamableFile` và dưới lớp vỏ, framework sẽ lo việc piping phản hồi.

#### Class Streamable File

`StreamableFile` là một class giữ luồng sẽ được trả về. Để tạo một `StreamableFile` mới, bạn có thể truyền một `Buffer` hoặc một `Stream` vào constructor `StreamableFile`.

> info **gợi ý** Class `StreamableFile` có thể được nhập từ `@nestjs/common`.

#### Hỗ trợ cross-platform

Fastify, theo mặc định, có thể hỗ trợ gửi file mà không cần gọi `stream.pipe(res)`, vì vậy bạn không cần sử dụng class `StreamableFile` tại tất cả. Tuy nhiên, Nest hỗ trợ việc sử dụng `StreamableFile` trong cả hai loại nền tảng, vì vậy nếu bạn chuyển đổi giữa Express và Fastify thì không cần lo lắng về tính tương thích giữa hai engine này.

#### Ví dụ

Bạn có thể tìm một ví dụ đơn giản về việc trả về `package.json` dưới dạng file thay vì JSON bên dưới, nhưng ý tưởng này mở rộng tự nhiên thành hình ảnh, tài liệu và bất kỳ loại file nào khác.

```ts
import { Controller, Get, StreamableFile } from '@nestjs/common';
import { createReadStream } from 'node:fs';
import { join } from 'node:path';

@Controller('file')
export class FileController {
  @Get()
  getFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }
}
```

Loại nội dung mặc định (giá trị cho header phản hồi HTTP `Content-Type`) là `application/octet-stream`. Nếu bạn cần tùy chỉnh giá trị này, bạn có thể sử dụng tùy chọn `type` từ `StreamableFile`, hoặc sử dụng phương thức `res.set` hoặc decorator [`@Header()`](/controllers#response-headers), như sau:

```ts
import { Controller, Get, StreamableFile, Res } from '@nestjs/common';
import { createReadStream } from 'node:fs';
import { join } from 'node:path';
import type { Response } from 'express'; // Assuming that we are using the ExpressJS HTTP Adapter

@Controller('file')
export class FileController {
  @Get()
  getFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file, {
      type: 'application/json',
      disposition: 'attachment; filename="package.json"',
      // If you want to define the Content-Length value to another value instead of file's length:
      // length: 123,
    });
  }

  // Or even:
  @Get()
  getFileChangingResponseObjDirectly(@Res({ passthrough: true }) res: Response): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    res.set({
      'Content-Type': 'application/json',
      'Content-Disposition': 'attachment; filename="package.json"',
    });
    return new StreamableFile(file);
  }

  // Or even:
  @Get()
  @Header('Content-Type', 'application/json')
  @Header('Content-Disposition', 'attachment; filename="package.json"')
  getFileUsingStaticValues(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }  
}
```