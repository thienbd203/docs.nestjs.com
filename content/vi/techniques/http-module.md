### HTTP module

[Axios](https://github.com/axios/axios) là một package HTTP client có tính năng phong phú được sử dụng rộng rãi. Nest bao bọc Axios và expose nó thông qua module tích hợp `HttpModule`. `HttpModule` xuất class `HttpService`, expose các phương thức dựa trên Axios để thực hiện các yêu cầu HTTP. Thư viện cũng chuyển đổi các phản hồi HTTP kết quả thành `Observables`.

> info **Gợi ý** Bạn cũng có thể sử dụng bất kỳ thư viện HTTP client Node.js mục đích chung nào trực tiếp, bao gồm [got](https://github.com/sindresorhus/got) hoặc [undici](https://github.com/nodejs/undici).

#### Cài đặt

Để bắt đầu sử dụng nó, trước tiên chúng ta cài đặt các dependencies cần thiết.

```bash
$ npm i --save @nestjs/axios axios
```

#### Bắt đầu

Sau khi quá trình cài đặt hoàn tất, để sử dụng `HttpService`, trước tiên nhập `HttpModule`.

```typescript
@Module({
  imports: [HttpModule],
  providers: [CatsService],
})
export class CatsModule {}
```

Tiếp theo, inject `HttpService` bằng cách sử dụng constructor injection thông thường.

> info **Gợi ý** `HttpModule` và `HttpService` được nhập từ package `@nestjs/axios`.

```typescript
@@filename()
@Injectable()
export class CatsService {
  constructor(private readonly httpService: HttpService) {}

  findAll(): Observable<AxiosResponse<Cat[]>> {
    return this.httpService.get('http://localhost:3000/cats');
  }
}
@@switch
@Injectable()
@Dependencies(HttpService)
export class CatsService {
  constructor(httpService) {
    this.httpService = httpService;
  }

  findAll() {
    return this.httpService.get('http://localhost:3000/cats');
  }
}
```

> info **Gợi ý** `AxiosResponse` là một interface được xuất từ package `axios` (`$ npm i axios`).

Tất cả các phương thức `HttpService` trả về một `AxiosResponse` được bao bọc trong một đối tượng `Observable`.

#### Cấu hình

[Axios](https://github.com/axios/axios) có thể được cấu hình với một loạt các tùy chọn để tùy chỉnh hành vi của `HttpService`. Đọc thêm về chúng [tại đây](https://github.com/axios/axios#request-config). Để cấu hình instance Axios cơ bản, truyền một đối tượng tùy chọn tùy chỉnh vào phương thức `register()` của `HttpModule` khi nhập nó. Đối tượng tùy chọn này sẽ được truyền trực tiếp vào constructor Axios cơ bản.

```typescript
@Module({
  imports: [
    HttpModule.register({
      timeout: 5000,
      maxRedirects: 5,
    }),
  ],
  providers: [CatsService],
})
export class CatsModule {}
```

#### Cấu hình async

Khi bạn cần truyền tùy chọn module một cách async thay vì tĩnh, sử dụng phương thức `registerAsync()`. Như với hầu hết các module động, Nest cung cấp một số kỹ thuật để xử lý cấu hình async.

Một kỹ thuật là sử dụng factory function:

```typescript
HttpModule.registerAsync({
  useFactory: () => ({
    timeout: 5000,
    maxRedirects: 5,
  }),
});
```

Giống như các factory provider khác, factory function của chúng ta có thể là [async](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) và có thể inject dependencies thông qua `inject`.

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    timeout: configService.get('HTTP_TIMEOUT'),
    maxRedirects: configService.get('HTTP_MAX_REDIRECTS'),
  }),
  inject: [ConfigService],
});
```

Ngoài ra, bạn có thể cấu hình `HttpModule` bằng cách sử dụng một class thay vì một factory, như hiển thị bên dưới.

```typescript
HttpModule.registerAsync({
  useClass: HttpConfigService,
});
```

Cấu trúc trên khởi tạo `HttpConfigService` bên trong `HttpModule`, sử dụng nó để tạo một đối tượng tùy chọn. Lưu ý rằng trong ví dụ này, `HttpConfigService` phải implement interface `HttpModuleOptionsFactory` như hiển thị bên dưới. `HttpModule` sẽ gọi phương thức `createHttpOptions()` trên đối tượng được khởi tạo của class được cung cấp.

```typescript
@Injectable()
class HttpConfigService implements HttpModuleOptionsFactory {
  createHttpOptions(): HttpModuleOptions {
    return {
      timeout: 5000,
      maxRedirects: 5,
    };
  }
}
```

Nếu bạn muốn tái sử dụng một provider tùy chọn hiện có thay vì tạo một bản sao riêng bên trong `HttpModule`, sử dụng cú pháp `useExisting`.

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useExisting: HttpConfigService,
});
```

Bạn cũng có thể truyền các `extraProviders` được gọi vào phương thức `registerAsync()`. Các providers này sẽ được hợp nhất với các providers của module.

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useClass: HttpConfigService,
  extraProviders: [MyAdditionalProvider],
});
```

Điều này hữu ích khi bạn muốn cung cấp các dependencies bổ sung cho factory function hoặc constructor của class.

#### Sử dụng Axios trực tiếp

Nếu bạn nghĩ rằng các tùy chọn của `HttpModule.register` không đủ cho bạn, hoặc nếu bạn chỉ muốn truy cập instance Axios cơ bản được tạo bởi `@nestjs/axios`, bạn có thể truy cập nó thông qua `HttpService#axiosRef` như sau:

```typescript
@Injectable()
export class CatsService {
  constructor(private readonly httpService: HttpService) {}

  findAll(): Promise<AxiosResponse<Cat[]>> {
    return this.httpService.axiosRef.get('http://localhost:3000/cats');
    //                      ^ AxiosInstance interface
  }
}
```

#### Ví dụ đầy đủ

Vì giá trị trả về của các phương thức `HttpService` là một Observable, chúng ta có thể sử dụng `rxjs` - `firstValueFrom` hoặc `lastValueFrom` để truy xuất dữ liệu của yêu cầu dưới dạng một promise.

```typescript
import { catchError, firstValueFrom } from 'rxjs';

@Injectable()
export class CatsService {
  private readonly logger = new Logger(CatsService.name);
  constructor(private readonly httpService: HttpService) {}

  async findAll(): Promise<Cat[]> {
    const { data } = await firstValueFrom(
      this.httpService.get<Cat[]>('http://localhost:3000/cats').pipe(
        catchError((error: AxiosError) => {
          this.logger.error(error.response.data);
          throw 'An error happened!';
        }),
      ),
    );
    return data;
  }
}
```

> info **Gợi ý** Truy cập tài liệu RxJS về [`firstValueFrom`](https://rxjs.dev/api/index/function/firstValueFrom) và [`lastValueFrom`](https://rxjs.dev/api/index/function/lastValueFrom) để biết sự khác biệt giữa chúng.