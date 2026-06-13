### Exception filters

Sự khác biệt duy nhất giữa lớp [exception filter](/exception-filters) HTTP và lớp microservices tương ứng là thay vì throwing `HttpException`, bạn nên sử dụng `RpcException`.

```typescript
throw new RpcException('Invalid credentials.');
```

> info **Gợi ý** Lớp `RpcException` được import từ gói `@nestjs/microservices`.

Với mẫu trên, Nest sẽ xử lý exception đã throw và trả về đối tượng `error` với cấu trúc sau:

```json
{
  "status": "error",
  "message": "Invalid credentials."
}
```

#### Filters

Microservice exception filters hoạt động tương tự như HTTP exception filters, với một sự khác biệt nhỏ. Phương thức `catch()` phải trả về một `Observable`.

```typescript
@@filename(rpc-exception.filter)
import { Catch, RpcExceptionFilter, ArgumentsHost } from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { RpcException } from '@nestjs/microservices';

@Catch(RpcException)
export class ExceptionFilter implements RpcExceptionFilter<RpcException> {
  catch(exception: RpcException, host: ArgumentsHost): Observable<any> {
    return throwError(() => exception.getError());
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { throwError } from 'rxjs';

@Catch(RpcException)
export class ExceptionFilter {
  catch(exception, host) {
    return throwError(() => exception.getError());
  }
}
```

> warning **Cảnh báo** Microservice exception filters toàn cục không được kích hoạt theo mặc định khi sử dụng [hybrid application](/faq/hybrid-application).

Ví dụ sau sử dụng một filter phạm vi phương thức được khởi tạo thủ công. Giống như với các ứng dụng dựa trên HTTP, bạn cũng có thể sử dụng filters phạm vi controller (tức là tiền tố lớp controller với một decorator `@UseFilters()`).

```typescript
@@filename()
@UseFilters(new ExceptionFilter())
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseFilters(new ExceptionFilter())
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```

#### Kế thừa

Thông thường, bạn sẽ tạo các exception filter tùy chỉnh hoàn toàn được chế tác để đáp ứng các yêu cầu ứng dụng của bạn. Tuy nhiên, có thể có các trường hợp sử dụng khi bạn muốn chỉ đơn giản mở rộng **core exception filter**, và override hành vi dựa trên một số yếu tố nhất định.

Để ủy quyền xử lý exception cho filter cơ sở, bạn cần mở rộng `BaseExceptionFilter` và gọi phương thức `catch()` được kế thừa.

```typescript
@@filename()
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseRpcExceptionFilter } from '@nestjs/microservices';

@Catch()
export class AllExceptionsFilter extends BaseRpcExceptionFilter {
  catch(exception: any, host: ArgumentsHost) {
    return super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseRpcExceptionFilter } from '@nestjs/microservices';

@Catch()
export class AllExceptionsFilter extends BaseRpcExceptionFilter {
  catch(exception, host) {
    return super.catch(exception, host);
  }
}
```

Triển khai trên chỉ là một shell thể hiện cách tiếp cận. Triển khai exception filter mở rộng của bạn sẽ bao gồm **business logic** tùy chỉnh của bạn (ví dụ, xử lý các điều kiện khác nhau).