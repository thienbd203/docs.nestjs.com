### Standalone applications

Có một số cách để mount một ứng dụng Nest. Bạn có thể tạo một web app, một microservice hoặc chỉ là một Nest **standalone application** thuần túy (không có bất kỳ network listeners nào). Ứng dụng Nest standalone là một wrapper quanh Nest **IoC container**, chứa tất cả các classes đã được instantiated. Chúng ta có thể obtain một tham chiếu đến bất kỳ instance hiện có nào từ bên trong bất kỳ module đã import nào trực tiếp sử dụng standalone application object. Do đó, bạn có thể tận dụng Nest framework ở bất cứ đâu, bao gồm ví dụ, các **CRON** jobs được scripted. Bạn thậm chí có thể xây dựng một **CLI** trên nền của nó.

#### Getting started

Để tạo một ứng dụng Nest standalone, sử dụng cấu trúc sau:

```typescript
@@filename()
async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  // your application logic here ...
}
bootstrap();
```

#### Retrieving providers từ static modules

Standalone application object cho phép bạn obtain một tham chiếu đến bất kỳ instance nào được đăng ký trong ứng dụng Nest. Hãy tưởng tượng rằng chúng ta có một provider `TasksService` trong module `TasksModule` đã được import bởi module `AppModule` của chúng ta. Class này cung cấp một tập hợp các methods mà chúng ta muốn gọi từ bên trong một CRON job.

```typescript
@@filename()
const tasksService = app.get(TasksService);
```

Để truy cập instance `TasksService` chúng ta sử dụng method `get()`. Method `get()` hoạt động như một **query** tìm kiếm một instance trong mỗi module đã đăng ký. Bạn có thể truyền bất kỳ token của provider nào cho nó. Ngoài ra, cho strict context checking, truyền một options object với property `strict: true`. Với tùy chọn này có hiệu lực, bạn phải navigate qua các modules cụ thể để obtain một instance cụ thể từ context được chọn.

```typescript
@@filename()
const tasksService = app.select(TasksModule).get(TasksService, { strict: true });
```

Sau đây là tóm tắt các methods có sẵn để retrieving instance references từ standalone application object.

<table>
  <tr>
    <td>
      <code>get()</code>
    </td>
    <td>
      Retrieves một instance của một controller hoặc provider (bao gồm guards, filters, v.v.) có sẵn trong application context.
    </td>
  </tr>
  <tr>
    <td>
      <code>select()</code>
    </td>
    <td>
      Navigates qua module's graph để pull ra một instance cụ thể của module được chọn (sử dụng cùng với strict mode như được mô tả ở trên).
    </td>
  </tr>
</table>

> info **Gợi ý** Trong non-strict mode, root module được chọn theo mặc định. Để chọn bất kỳ module nào khác, bạn cần navigate modules graph thủ công, từng bước.

Hãy nhớ rằng một standalone application không có bất kỳ network listeners nào, vì vậy bất kỳ tính năng Nest nào liên quan đến HTTP (ví dụ, middleware, interceptors, pipes, guards, v.v.) không có sẵn trong context này.

Ví dụ, ngay cả khi bạn đăng ký một global interceptor trong ứng dụng của bạn và sau đó retrieve một instance của controller sử dụng method `app.get()`, interceptor sẽ không được thực thi.

#### Retrieving providers từ dynamic modules

Khi xử lý dynamic modules, chúng ta nên cung cấp cùng một object đại diện cho dynamic module đã đăng ký trong ứng dụng đến `app.select`. Ví dụ:

```typescript
@@filename()
export const dynamicConfigModule = ConfigModule.register({ folder: './config' });

@Module({
  imports: [dynamicConfigModule],
})
export class AppModule {}
```

Sau đó bạn có thể chọn module đó sau:

```typescript
@@filename()
const configService = app.select(dynamicConfigModule).get(ConfigService, { strict: true });
```

#### Terminating phase

Nếu bạn muốn ứng dụng Node đóng sau khi script kết thúc (ví dụ, cho một script chạy CRON jobs), bạn phải gọi method `app.close()` ở cuối function `bootstrap` của bạn như sau:

```typescript
@@filename()
async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  // application logic...
  await app.close();
}
bootstrap();
```

Và như được đề cập trong chương Lifecycle events, điều đó sẽ kích hoạt lifecycle hooks.

#### Example

Một ví dụ hoạt động có sẵn ở đây.