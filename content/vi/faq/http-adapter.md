### HTTP adapter

Thỉnh thoảng, bạn có thể muốn truy cập máy chủ HTTP cơ bản, hoặc trong ngữ cảnh ứng dụng Nest hoặc từ bên ngoài.

Mọi máy chủ/thư viện HTTP gốc (cụ thể cho nền tảng) (ví dụ, Express và Fastify) đều được bọc trong một **adapter**. Adapter được đăng ký như một provider có sẵn toàn cầu có thể được truy xuất từ ngữ cảnh ứng dụng, cũng như được inject vào các provider khác.

#### Chiến lược ngoài ngữ cảnh ứng dụng

Để lấy tham chiếu đến `HttpAdapter` từ bên ngoài ngữ cảnh ứng dụng, gọi phương thức `getHttpAdapter()`.

```typescript
@@filename()
const app = await NestFactory.create(AppModule);
const httpAdapter = app.getHttpAdapter();
```

#### Là injectable

Để lấy tham chiếu đến `HttpAdapterHost` từ trong ngữ cảnh ứng dụng, inject nó sử dụng cùng kỹ thuật như bất kỳ provider hiện có nào khác (ví dụ, sử dụng constructor injection).

```typescript
@@filename()
export class CatsService {
  constructor(private adapterHost: HttpAdapterHost) {}
}
@@switch
@Dependencies(HttpAdapterHost)
export class CatsService {
  constructor(adapterHost) {
    this.adapterHost = adapterHost;
  }
}
```

> info **Gợi ý** `HttpAdapterHost` được nhập từ gói `@nestjs/core`.

`HttpAdapterHost` **không** là `HttpAdapter` thực tế. Để lấy instance `HttpAdapter` thực tế, chỉ cần truy cập thuộc tính `httpAdapter`.

```typescript
const adapterHost = app.get(HttpAdapterHost);
const httpAdapter = adapterHost.httpAdapter;
```

`httpAdapter` là instance thực tế của HTTP adapter được sử dụng bởi framework cơ bản. Nó là một instance của `ExpressAdapter` hoặc `FastifyAdapter` (cả hai class đều mở rộng `AbstractHttpAdapter`).

Đối tượng adapter expose một số phương thức hữu ích để tương tác với máy chủ HTTP. Tuy nhiên, nếu bạn muốn truy cập trực tiếp instance thư viện (ví dụ, instance Express), gọi phương thức `getInstance()`.

```typescript
const instance = httpAdapter.getInstance();
```

#### Sự kiện lắng nghe

Để thực thi một hành động khi máy chủ bắt đầu lắng nghe các yêu cầu đến, bạn có thể đăng ký vào stream `listen$`, như được minh họa dưới đây:

```typescript
this.httpAdapterHost.listen$.subscribe(() =>
  console.log('HTTP server is listening'),
);
```

Ngoài ra, `HttpAdapterHost` cung cấp một thuộc tính boolean `listening` chỉ định xem máy chủ có đang hoạt động và lắng nghe hay không:

```typescript
if (this.httpAdapterHost.listening) {
  console.log('HTTP server is listening');
}
```