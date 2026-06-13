### Exception filters

Sự khác biệt duy nhất giữa lớp [exception filter](/exception-filters) HTTP và lớp web sockets tương ứng là thay vì throwing `HttpException`, bạn nên sử dụng `WsException`.

```typescript
throw new WsException('Invalid credentials.');
```

> info **Gợi ý** Lớp `WsException` được import từ gói `@nestjs/websockets`.

Với mẫu trên, Nest sẽ xử lý exception đã throw và emit tin nhắn `exception` với cấu trúc sau:

```typescript
{
  status: 'error',
  message: 'Invalid credentials.'
}
```

#### Filters

Web sockets exception filters hoạt động tương đương với HTTP exception filters. Ví dụ sau sử dụng một filter phạm vi phương thức được khởi tạo thủ công. Giống như với các ứng dụng dựa trên HTTP, bạn cũng có thể sử dụng filters phạm vi gateway (tức là tiền tố lớp gateway với một decorator `@UseFilters()`).

```typescript
@UseFilters(new WsExceptionFilter())
@SubscribeMessage('events')
onEvent(client, data: any): WsResponse<any> {
  const event = 'events';
  return { event, data };
}
```

#### Kế thừa

Thông thường, bạn sẽ tạo các exception filter tùy chỉnh hoàn toàn được chế tác để đáp ứng các yêu cầu ứng dụng của bạn. Tuy nhiên, có thể có các trường hợp sử dụng khi bạn muốn chỉ đơn giản mở rộng **core exception filter**, và override hành vi dựa trên một số yếu tố.

Để ủy quyền xử lý exception cho filter cơ sở, bạn cần mở rộng `BaseWsExceptionFilter` và gọi phương thức `catch()` được kế thừa.

```typescript
@@filename()
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseWsExceptionFilter } from '@nestjs/websockets';

@Catch()
export class AllExceptionsFilter extends BaseWsExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseWsExceptionFilter } from '@nestjs/websockets';

@Catch()
export class AllExceptionsFilter extends BaseWsExceptionFilter {
  catch(exception, host) {
    super.catch(exception, host);
  }
}
```

Triển khai trên chỉ là một shell thể hiện cách tiếp cận. Triển khai exception filter mở rộng của bạn sẽ bao gồm **business logic** tùy chỉnh của bạn (ví dụ, xử lý các điều kiện khác nhau).