### Interceptors

Interceptor là một class được annotate với decorator `@Injectable()` và implements interface `NestInterceptor`.

<figure><img class="illustrative-image" src="/assets/Interceptors_1.png" /></figure>

Interceptors có một tập hợp các khả năng hữu ích được lấy cảm hứng từ kỹ thuật Aspect Oriented Programming (AOP). Chúng cho phép:

- bind logic thêm trước / sau thực thi method
- chuyển đổi kết quả trả về từ một function
- chuyển đổi exception được throw từ một function
- mở rộng hành vi function cơ bản
- hoàn toàn ghi đè một function tùy thuộc vào các điều kiện cụ thể (ví dụ, cho mục đích caching)

#### Basics

Mỗi interceptor implements method `intercept()`, nhận hai đối số. Đối số đầu tiên là instance `ExecutionContext` (chính xác cùng đối tượng như cho guards). `ExecutionContext` kế thừa từ `ArgumentsHost`. Chúng ta đã thấy `ArgumentsHost` trước đó trong chapter exception filters. Ở đó, chúng ta đã thấy rằng nó là một wrapper xung quanh các arguments đã được truyền đến handler gốc, và chứa các mảng arguments khác nhau dựa trên loại ứng dụng. Bạn có thể tham khảo lại exception filters để biết thêm về chủ đề này.

#### Execution context

Bằng cách mở rộng `ArgumentsHost`, `ExecutionContext` cũng thêm một số helper methods mới cung cấp chi tiết bổ sung về quy trình thực thi hiện tại. Các chi tiết này có thể hữu ích trong việc xây dựng các interceptors generic hơn có thể hoạt động trên một tập hợp rộng các controllers, methods, và execution contexts. Tìm hiểu thêm về `ExecutionContext` ở đây.

#### Call handler

Đối số thứ hai là một `CallHandler`. Interface `CallHandler` implements method `handle()`, mà bạn có thể sử dụng để invoke method route handler tại một số điểm trong interceptor của bạn. Nếu bạn không gọi method `handle()` trong implementation của method `intercept()`, method route handler sẽ không được thực thi tại tất cả.

Cách tiếp cận này có nghĩa là method `intercept()` hiệu quả **wraps** luồng request/response. Kết quả là, bạn có thể implement logic tùy chỉnh **cả trước và sau** thực thi của route handler cuối cùng. Rõ ràng là bạn có thể viết code trong method `intercept()` của bạn thực thi **trước** khi gọi `handle()`, nhưng làm thế nào bạn ảnh hưởng những gì xảy ra sau đó? Vì method `handle()` trả về một `Observable`, chúng ta có thể sử dụng các operators RxJS mạnh mẽ để thao tác thêm response. Sử dụng terminology Aspect Oriented Programming, việc invoke route handler (tức là gọi `handle()`) được gọi là một Pointcut, chỉ định rằng nó là điểm tại đó logic bổ sung của chúng ta được chèn.

Hãy xem xét, ví dụ, một yêu cầu `POST /cats` đến. Yêu cầu này định đến handler `create()` được định nghĩa bên trong `CatsController`. Nếu một interceptor không gọi method `handle()` được gọi ở bất kỳ đâu dọc đường, method `create()` sẽ không được thực thi. Một khi `handle()` được gọi (và `Observable` của nó đã được trả về), handler `create()` sẽ được kích hoạt. Và một khi luồng response được nhận qua `Observable`, các operations bổ sung có thể được thực hiện trên luồng, và một kết quả cuối cùng được trả về cho caller.

<app-banner-devtools></app-banner-devtools>

#### Aspect interception

Trường hợp sử dụng đầu tiên chúng ta sẽ xem xét là sử dụng một interceptor để log tương tác người dùng (ví dụ, lưu trữ các cuộc gọi người dùng, asynchronously dispatching events hoặc tính toán một timestamp). Chúng ta cho thấy một `LoggingInterceptor` đơn giản dưới đây:

```typescript
@@filename(logging.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor {
  intercept(context, next) {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```

> info **Gợi ý** `NestInterceptor<T, R>` là một generic interface trong đó `T` chỉ định loại của một `Observable<T>` (hỗ trợ luồng response), và `R` là loại của giá trị được wrap bởi `Observable<R>`.

> warning **Lưu ý** Interceptors, giống như controllers, providers, guards, v.v., có thể **inject dependencies** thông qua `constructor` của chúng.

Vì `handle()` trả về một RxJS `Observable`, chúng ta có một lựa chọn rộng các operators chúng ta có thể sử dụng để thao tác luồng. Trong ví dụ trên, chúng ta đã sử dụng operator `tap()`, invoke function logging ẩn danh của chúng ta khi kết thúc graceful hoặc exceptional của luồng observable, nhưng không can thiệp vào chu kỳ response.

#### Binding interceptors

Để thiết lập interceptor, chúng ta sử dụng decorator `@UseInterceptors()` được import từ package `@nestjs/common`. Giống như pipes và guards, interceptors có thể là controller-scoped, method-scoped, hoặc global-scoped.

```typescript
@@filename(cats.controller)
@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```

> info **Gợi ý** Decorator `@UseInterceptors()` được import từ package `@nestjs/common`.

Sử dụng cấu trúc trên, mỗi route handler được định nghĩa trong `CatsController` sẽ sử dụng `LoggingInterceptor`. Khi ai đó gọi endpoint `GET /cats`, bạn sẽ thấy output sau trong standard output của bạn:

```typescript
Before...
After... 1ms
```

Lưu ý rằng chúng ta truyền class `LoggingInterceptor` (thay vì một instance), để lại trách nhiệm instantiation cho framework và cho phép dependency injection. Như với pipes, guards, và exception filters, chúng ta cũng có thể truyền một instance in-place:

```typescript
@@filename(cats.controller)
@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```

Như đã đề cập, cấu trúc trên attach interceptor đến mọi handler được khai báo bởi controller này. Nếu chúng ta muốn giới hạn scope của interceptor đến một method đơn lẻ, chúng ta chỉ cần áp dụng decorator ở **mức method**.

Để thiết lập một global interceptor, chúng ta sử dụng method `useGlobalInterceptors()` của instance ứng dụng Nest:

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

Global interceptors được sử dụng trên toàn bộ ứng dụng, cho mọi controller và mọi route handler. Về mặt dependency injection, global interceptors được đăng ký từ bên ngoài bất kỳ module nào (với `useGlobalInterceptors()`, như trong ví dụ trên) không thể inject dependencies vì điều này được thực hiện bên ngoài context của bất kỳ module nào. Để giải quyết vấn đề này, bạn có thể thiết lập một interceptor **trực tiếp từ bất kỳ module nào** sử dụng cấu trúc sau:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

> info **Gợi ý** Khi sử dụng cách tiếp cận này để thực hiện dependency injection cho interceptor, lưu ý rằng bất kể module nơi cấu trúc này được sử dụng, interceptor thực tế là global. Nên làm điều này ở đâu? Chọn module nơi interceptor (`LoggingInterceptor` trong ví dụ trên) được định nghĩa. Ngoài ra, `useClass` không phải là cách duy nhất để xử lý đăng ký custom provider. Tìm hiểu thêm ở đây.

#### Response mapping

Chúng ta đã biết rằng `handle()` trả về một `Observable`. Luồng chứa giá trị **được trả về** từ route handler, và do đó chúng ta có thể dễ dàng mutate nó sử dụng operator `map()` của RxJS.

> warning **Cảnh báo** Tính năng response mapping không hoạt động với chiến lược response library-specific (sử dụng đối tượng `@Res()` trực tiếp bị cấm).

Hãy tạo `TransformInterceptor`, sẽ sửa đổi mỗi response theo cách trivial để minh họa quá trình. Nó sẽ sử dụng operator `map()` của RxJS để gán đối tượng response đến thuộc tính `data` của một đối tượng mới được tạo, trả về đối tượng mới cho client.

```typescript
@@filename(transform.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class TransformInterceptor {
  intercept(context, next) {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

> info **Gợi ý** Nest interceptors hoạt động với cả methods `intercept()` synchronous và asynchronous. Bạn có thể chỉ cần chuyển method thành `async` nếu cần thiết.

Với cấu trúc trên, khi ai đó gọi endpoint `GET /cats`, response sẽ trông như sau (giả sử rằng route handler trả về một mảng trống `[]`):

```json
{
  "data": []
}
```

Interceptors có giá trị lớn trong việc tạo các giải pháp có thể tái sử dụng cho các yêu cầu xảy ra trên toàn bộ ứng dụng.
Ví dụ, hãy tưởng tượng chúng ta cần chuyển đổi mỗi lần xuất hiện của một giá trị `null` thành một chuỗi trống `''`. Chúng ta có thể làm điều đó sử dụng một dòng code và bind interceptor global để nó sẽ tự động được sử dụng bởi mỗi handler được đăng ký.

```typescript
@@filename()
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor {
  intercept(context, next) {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
```

#### Exception mapping

Một trường hợp sử dụng thú vị khác là tận dụng operator `catchError()` của RxJS để ghi đè các exceptions được throw:

```typescript
@@filename(errors.interceptor)
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
@@switch
import { Injectable, BadGatewayException } from '@nestjs/common';
import { throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor {
  intercept(context, next) {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
```

#### Stream overriding

Có một số lý do tại sao chúng ta đôi khi muốn hoàn toàn ngăn việc gọi handler và thay vào đó trả về một giá trị khác. Một ví dụ rõ ràng là để implement một cache để cải thiện thời gian response. Hãy xem một **cache interceptor** đơn giản trả về response của nó từ một cache. Trong một ví dụ thực tế, chúng ta sẽ muốn xem xét các yếu tố khác như TTL, cache invalidation, cache size, v.v., nhưng đó nằm ngoài phạm vi của thảo luận này. Ở đây chúng ta sẽ cung cấp một ví dụ cơ bản minh họa khái niệm chính.

```typescript
@@filename(cache.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { of } from 'rxjs';

@Injectable()
export class CacheInterceptor {
  intercept(context, next) {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

`CacheInterceptor` của chúng ta có một biến `isCached` hardcoded và một response hardcoded `[]` cũng vậy. Điểm chính cần lưu ý là chúng ta trả về một luồng mới ở đây, được tạo bởi operator `of()` của RxJS, do đó route handler **sẽ không được gọi** tại tất cả. Khi ai đó gọi một endpoint sử dụng `CacheInterceptor`, response (một mảng trống hardcoded) sẽ được trả về ngay lập tức. Để tạo một giải pháp generic, bạn có thể tận dụng `Reflector` và tạo một decorator tùy chỉnh. `Reflector` được mô tả tốt trong chapter guards.

#### More operators

Khả năng thao tác luồng sử dụng các operators RxJS cho chúng ta nhiều khả năng. Hãy xem xét một trường hợp sử dụng phổ biến khác. Hãy tưởng tượng bạn muốn xử lý **timeouts** trên các yêu cầu route. Khi endpoint của bạn không trả về bất kỳ thứ gì sau một khoảng thời gian, bạn muốn chấm dứt với một error response. Cấu trúc sau cho phép điều này:

```typescript
@@filename(timeout.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
@@switch
import { Injectable, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor {
  intercept(context, next) {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
}
```

Sau 5 giây, xử lý yêu cầu sẽ bị hủy. Bạn cũng có thể thêm logic tùy chỉnh trước khi throw `RequestTimeoutException` (ví dụ release resources).