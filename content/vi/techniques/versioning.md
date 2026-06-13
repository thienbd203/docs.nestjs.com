### Versioning

> info **Gợi ý** Chương này chỉ liên quan đến các ứng dụng dựa trên HTTP.

Versioning cho phép bạn có **các phiên bản khác nhau** của controllers hoặc các route riêng lẻ chạy trong cùng một ứng dụng. Các ứng dụng thay đổi rất thường xuyên và không có gì lạ khi có những thay đổi phá vỡ mà bạn cần thực hiện trong khi vẫn cần hỗ trợ phiên bản trước của ứng dụng.

Có 4 loại versioning được hỗ trợ:

<table>
  <tr>
    <td><a href='techniques/versioning#uri-versioning-type'><code>URI Versioning</code></a></td>
    <td>Phiên bản sẽ được truyền trong URI của yêu cầu (mặc định)</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#header-versioning-type'><code>Header Versioning</code></a></td>
    <td>Một header yêu cầu tùy chỉnh sẽ chỉ định phiên bản</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#media-type-versioning-type'><code>Media Type Versioning</code></a></td>
    <td>Header <code>Accept</code> của yêu cầu sẽ chỉ định phiên bản</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#custom-versioning-type'><code>Custom Versioning</code></a></td>
    <td>Bất kỳ khía cạnh nào của yêu cầu có thể được sử dụng để chỉ định phiên bản (các). Một hàm tùy chỉnh được cung cấp để trích xuất các phiên bản nói trên.</td>
  </tr>
</table>

#### Loại URI Versioning

URI Versioning sử dụng phiên bản được truyền trong URI của yêu cầu, chẳng hạn như `https://example.com/v1/route` và `https://example.com/v2/route`.

> warning **Lưu ý** Với URI Versioning, phiên bản sẽ tự động được thêm vào URI sau <a href="faq/global-prefix">tiền tố đường dẫn toàn cục</a> (nếu có), và trước bất kỳ đường dẫn controller hoặc route nào.

Để bật URI Versioning cho ứng dụng của bạn, hãy làm như sau:

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
// hoặc "app.enableVersioning()"
app.enableVersioning({
  type: VersioningType.URI,
});
await app.listen(process.env.PORT ?? 3000);
```

> warning **Lưu ý** Phiên bản trong URI sẽ tự động được thêm tiền tố `v` theo mặc định, tuy nhiên giá trị tiền tố có thể được cấu hình bằng cách đặt khóa `prefix` thành tiền tố mong muốn của bạn hoặc `false` nếu bạn muốn vô hiệu hóa nó.

> info **Gợi ý** Enum `VersioningType` có sẵn để sử dụng cho thuộc tính `type` và được nhập từ package `@nestjs/common`.

#### Loại Header Versioning

Header Versioning sử dụng một header yêu cầu tùy chỉnh, do người dùng chỉ định, để chỉ định phiên bản trong đó giá trị của header sẽ là phiên bản để sử dụng cho yêu cầu.

Ví dụ Yêu cầu HTTP cho Header Versioning:

Để bật **Header Versioning** cho ứng dụng của bạn, hãy làm như sau:

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.HEADER,
  header: 'Custom-Header',
});
await app.listen(process.env.PORT ?? 3000);
```

Thuộc tính `header` nên là tên của header sẽ chứa phiên bản của yêu cầu.

> info **Gợi ý** Enum `VersioningType` có sẵn để sử dụng cho thuộc tính `type` và được nhập từ package `@nestjs/common`.

#### Loại Media Type Versioning

Media Type Versioning sử dụng header `Accept` của yêu cầu để chỉ định phiên bản.

Trong header `Accept`, phiên bản sẽ được phân tách khỏi loại phương tiện bằng dấu chấm phẩy, `;`. Sau đó nó nên chứa một cặp khóa-giá trị đại diện cho phiên bản để sử dụng cho yêu cầu, chẳng hạn như `Accept: application/json;v=2`. Khóa được coi nhiều hơn như một tiền tố khi xác định phiên bản sẽ được cấu hình để bao gồm khóa và dấu phân cách.

Để bật **Media Type Versioning** cho ứng dụng của bạn, hãy làm như sau:

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key: 'v=',
});
await app.listen(process.env.PORT ?? 3000);
```

Thuộc tính `key` nên là khóa và dấu phân cách của cặp khóa-giá trị chứa phiên bản. Đối với ví dụ `Accept: application/json;v=2`, thuộc tính `key` sẽ được đặt thành `v=`.

> info **Gợi ý** Enum `VersioningType` có sẵn để sử dụng cho thuộc tính `type` và được nhập từ package `@nestjs/common`.

#### Loại Custom Versioning

Custom Versioning sử dụng bất kỳ khía cạnh nào của yêu cầu để chỉ định phiên bản (hoặc các phiên bản). Yêu cầu đến được phân tích bằng cách sử dụng một hàm `extractor` trả về một chuỗi hoặc mảng chuỗi.

Nếu nhiều phiên bản được cung cấp bởi người yêu cầu, hàm extractor có thể trả về một mảng chuỗi, được sắp xếp theo thứ tự từ phiên bản cao nhất/nhất đến phiên bản thấp nhất/thấp nhất. Các phiên bản được khớp với các routes theo thứ tự từ cao nhất đến thấp nhất.

Nếu một chuỗi rỗng hoặc mảng được trả về từ `extractor`, không có route nào được khớp và 404 được trả về.

Ví dụ, nếu một yêu cầu đến chỉ định nó hỗ trợ các phiên bản `1`, `2`, và `3`, hàm `extractor` **PHẢI** trả về `[3, 2, 1]`. Điều này đảm bảo rằng phiên bản route cao nhất có thể được chọn đầu tiên.

Nếu các phiên bản `[3, 2, 1]` được trích xuất, nhưng các routes chỉ tồn tại cho phiên bản `2` và `1`, route khớp với phiên bản `2` được chọn (phiên bản `3` tự động bị bỏ qua).

> warning **Lưu ý** Chọn phiên bản khớp cao nhất dựa trên mảng được trả về từ `extractor` > **không hoạt động đáng tin cậy** với adapter Express do các hạn chế thiết kế. Một phiên bản duy nhất (hoặc là chuỗi hoặc mảng 1 phần tử) hoạt động tốt trong Express. Fastify hỗ trợ đúng cả việc chọn phiên bản khớp cao nhất và chọn phiên bản đơn.

Để bật **Custom Versioning** cho ứng dụng của bạn, tạo một hàm `extractor` và truyền nó vào ứng dụng của bạn như sau:

```typescript
@@filename(main)
// Ví dụ extractor trích xuất danh sách các phiên bản từ một header tùy chỉnh và biến nó thành một mảng được sắp xếp.
// Ví dụ này sử dụng Fastify, nhưng các yêu cầu Express có thể được xử lý theo cách tương tự.
const extractor = (request: FastifyRequest): string | string[] =>
  [request.headers['custom-versioning-field'] ?? '']
     .flatMap(v => v.split(','))
     .filter(v => !!v)
     .sort()
     .reverse()

const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.CUSTOM,
  extractor,
});
await app.listen(process.env.PORT ?? 3000);
```

#### Sử dụng

Versioning cho phép bạn version các controllers, các route riêng lẻ, và cũng cung cấp một cách để một số tài nguyên opt-out khỏi versioning. Việc sử dụng versioning là giống nhau bất kể Loại Versioning mà ứng dụng của bạn sử dụng.

> warning **Lưu ý** Nếu versioning được bật cho ứng dụng nhưng controller hoặc route không chỉ định phiên bản, bất kỳ yêu cầu nào đến controller/route đó sẽ được trả về trạng thái phản hồi `404`. Tương tự, nếu một yêu cầu được nhận chứa một phiên bản không có controller hoặc route tương ứng, nó cũng sẽ được trả về trạng thái phản hồi `404`.

#### Phiên bản controller

Một phiên bản có thể được áp dụng cho một controller, đặt phiên bản cho tất cả các routes trong controller.

Để thêm một phiên bản vào một controller, hãy làm như sau:

```typescript
@@filename(cats.controller)
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1';
  }
}
@@switch
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll() {
    return 'This action returns all cats for version 1';
  }
}
```

#### Phiên bản route

Một phiên bản có thể được áp dụng cho một route riêng lẻ. Phiên bản này sẽ ghi đè bất kỳ phiên bản nào khác sẽ ảnh hưởng đến route, chẳng hạn như Phiên bản Controller.

Để thêm một phiên bản vào một route riêng lẻ, hãy làm như sau:

```typescript
@@filename(cats.controller)
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1(): string {
    return 'This action returns all cats for version 1';
  }

  @Version('2')
  @Get('cats')
  findAllV2(): string {
    return 'This action returns all cats for version 2';
  }
}
@@switch
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1() {
    return 'This action returns all cats for version 1';
  }

  @Version('2')
  @Get('cats')
  findAllV2() {
    return 'This action returns all cats for version 2';
  }
}
```

#### Nhiều phiên bản

Nhiều phiên bản có thể được áp dụng cho một controller hoặc route. Để sử dụng nhiều phiên bản, bạn sẽ đặt phiên bản thành một Mảng.

Để thêm nhiều phiên bản, hãy làm như sau:

```typescript
@@filename(cats.controller)
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1 or 2';
  }
}
@@switch
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll() {
    return 'This action returns all cats for version 1 or 2';
  }
}
```

#### Phiên bản "Trung lập"

Một số controllers hoặc routes có thể không quan tâm đến phiên bản và sẽ có cùng chức năng bất kể phiên bản. Để phù hợp với điều này, phiên bản có thể được đặt thành symbol `VERSION_NEUTRAL`.

Một yêu cầu đến sẽ được ánh xạ đến một controller hoặc route `VERSION_NEUTRAL` bất kể phiên bản được gửi trong yêu cầu thêm vào nếu yêu cầu không chứa phiên bản nào cả.

> warning **Lưu ý** Đối với URI Versioning, một tài nguyên `VERSION_NEUTRAL` sẽ không có phiên bản hiện diện trong URI.

Để thêm một controller hoặc route trung lập phiên bản, hãy làm như sau:

```typescript
@@filename(cats.controller)
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats regardless of version';
  }
}
@@switch
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll() {
    return 'This action returns all cats regardless of version';
  }
}
```

#### Phiên bản mặc định toàn cục

Nếu bạn không muốn cung cấp một phiên bản cho mỗi controller/hoặc các route riêng lẻ, hoặc nếu bạn muốn có một phiên bản cụ thể được đặt làm phiên bản mặc định cho mọi controller/route không có phiên bản được chỉ định, bạn có thể đặt `defaultVersion` như sau:

```typescript
@@filename(main)
app.enableVersioning({
  // ...
  defaultVersion: '1'
  // hoặc
  defaultVersion: ['1', '2']
  // hoặc
  defaultVersion: VERSION_NEUTRAL
});
```

#### Versioning middleware

[Middleware](https://docs.nestjs.com/middleware) cũng có thể sử dụng metadata versioning để cấu hình middleware cho phiên bản cụ thể của một route. Để làm như vậy, cung cấp số phiên bản làm một trong các tham số cho phương thức `MiddlewareConsumer.forRoutes()`:

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET, version: '2' });
  }
}
```

Với mã ở trên, `LoggerMiddleware` sẽ chỉ được áp dụng cho phiên bản '2' của endpoint `/cats`.

> info **Lưu ý** Middleware hoạt động với bất kỳ loại versioning nào được mô tả trong phần này: `URI`, `Header`, `Media Type` hoặc `Custom`.