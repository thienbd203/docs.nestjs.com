### Hướng dẫn chuyển đổi

Bài viết này cung cấp hướng dẫn toàn diện để chuyển đổi từ NestJS phiên bản 10 sang phiên bản 11. Để khám phá các tính năng mới được giới thiệu trong v11, hãy xem [bài viết này](https://trilon.io/blog/announcing-nestjs-11-whats-new). Mặc dù bản cập nhật bao gồm một vài thay đổi phá vỡ nhỏ, chúng ít có khả năng ảnh hưởng đến hầu hết người dùng. Bạn có thể xem lại danh sách đầy đủ các thay đổi [ở đây](https://github.com/nestjs/nest/releases/tag/v11.0.0).

#### Nâng cấp các packages

Mặc dù bạn có thể nâng cấp các packages của mình thủ công, chúng tôi khuyến nghị sử dụng [npm-check-updates (ncu)](https://npmjs.com/package/npm-check-updates) để có quy trình hợp lý hơn.

#### Express v5

Sau nhiều năm phát triển, Express v5 đã được phát hành chính thức vào năm 2024 và trở thành phiên bản ổn định vào năm 2025. Với NestJS 11, Express v5 hiện là phiên bản mặc định được tích hợp vào framework. Mặc dù bản cập nhật này liền mạch cho hầu hết người dùng, điều quan trọng là phải biết rằng Express v5 giới thiệu một số thay đổi phá vỡ. Để có hướng dẫn chi tiết, hãy tham khảo [hướng dẫn chuyển đổi Express v5](https://expressjs.com/en/guide/migrating-5.html).

Một trong những cập nhật đáng chú ý nhất trong Express v5 là thuật toán khớp đường dẫn đường dẫn đã được sửa đổi. Các thay đổi sau đã được giới thiệu về cách các chuỗi đường dẫn được khớp với các yêu cầu đến:

- Dấu hoa thị `*` phải có tên, khớp với hành vi của các tham số: sử dụng `/*splat` hoặc `/{{ '{' }}*splat&#125;` thay vì `/*`. `splat` chỉ đơn giản là tên của tham số wildcard và không có ý nghĩa đặc biệt. Bạn có thể đặt tên nó bất cứ gì bạn thích, ví dụ, `*wildcard`
- Ký tự tùy chọn `?` không còn được hỗ trợ, sử dụng dấu ngoặc nhọn thay thế: `/:file{{ '{' }}.:ext&#125;`.
- Các ký tự regexp không được hỗ trợ.
- Một số ký tự đã được dành để tránh nhầm lẫn trong quá trình nâng cấp `(()[]?+!)`, sử dụng `\` để thoát chúng.
- Tên tham số hiện hỗ trợ các định danh JavaScript hợp lệ, hoặc được trích dẫn như `:"this"`.

Điều đó nói rằng, các routes trước đó hoạt động trong Express v4 có thể không hoạt động trong Express v5. Ví dụ:

```typescript
@Get('users/*')
findAll() {
  // Trong NestJS 11, điều này sẽ tự động được chuyển đổi thành một route Express v5 hợp lệ.
  // Trong khi nó vẫn có thể hoạt động, không còn khuyến khích sử dụng cú pháp wildcard này trong Express v5.
  return 'This route should not work in Express v5';
}
```

Để sửa vấn đề này, bạn có thể cập nhật route để sử dụng một wildcard được đặt tên:

```typescript
@Get('users/*splat')
findAll() {
  return 'This route will work in Express v5';
}
```

> warning **Cảnh báo** Lưu ý rằng `*splat` là một wildcard được đặt tên khớp bất kỳ đường dẫn nào không bao gồm đường dẫn gốc. Nếu bạn cần khớp cả đường dẫn gốc (`/users`), bạn có thể sử dụng `/users/{{ '{' }}*splat&#125;`, bọc wildcard trong dấu ngoặc nhọn (nhóm tùy chọn). Lưu ý rằng `splat` chỉ đơn giản là tên của tham số wildcard và không có ý nghĩa đặc biệt. Bạn có thể đặt tên nó bất cứ gì bạn thích, ví dụ, `*wildcard`.

Tương tự, nếu bạn có một middleware chạy trên tất cả các routes, bạn có thể cần cập nhật đường dẫn để sử dụng một wildcard được đặt tên:

```typescript
// Trong NestJS 11, điều này sẽ tự động được chuyển đổi thành một route Express v5 hợp lệ.
// Trong khi nó vẫn có thể hoạt động, không còn khuyến khích sử dụng cú pháp wildcard này trong Express v5.
forRoutes('*'); // <-- Điều này không nên hoạt động trong Express v5
```

Thay vào đó, bạn có thể cập nhật đường dẫn để sử dụng một wildcard được đặt tên:

```typescript
forRoutes('{*splat}'); // <-- Điều này sẽ hoạt động trong Express v5
```

Lưu ý rằng `{{ '{' }}*splat&#125;` là một wildcard được đặt tên khớp bất kỳ đường dẫn nào bao gồm cả đường dẫn gốc. Dấu ngoặc nhọn bên ngoài làm cho đường dẫn tùy chọn.

#### Phân tích cú pháp tham số query

> info **Lưu ý** Thay đổi này chỉ áp dụng cho Express v5.

Trong Express v5, các tham số query không còn được phân tích cú pháp bằng thư viện `qs` theo mặc định. Thay vào đó, bộ phân tích cú pháp `simple` được sử dụng, không hỗ trợ các đối tượng lồng nhau hoặc mảng.

Kết quả là, các chuỗi query như sau:

```plaintext
?filter[where][name]=John&filter[where][age]=30
?item[]=1&item[]=2
```

sẽ không còn được phân tích cú pháp như mong đợi. Để quay lại hành vi trước đó, bạn có thể cấu hình Express để sử dụng bộ phân tích cú pháp `extended` (mặc định trong Express v4) bằng cách đặt tùy chọn `query parser` thành `extended`:

```typescript
import { NestFactory } from '@nestjs/core';
import { NestExpressApplication } from '@nestjs/platform-express';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule); // <-- Đảm bảo sử dụng <NestExpressApplication>
  app.set('query parser', 'extended'); // <-- Thêm dòng này
  await app.listen(3000);
}
bootstrap();
```

#### Fastify v5

`@nestjs/platform-fastify` v11 hiện cuối cùng hỗ trợ Fastify v5. Bản cập nhật này nên liền mạch cho hầu hết người dùng; tuy nhiên, Fastify v5 giới thiệu một vài thay đổi phá vỡ, mặc dù các thay đổi này ít có khả năng ảnh hưởng đến đa số người dùng NestJS. Để biết thông tin chi tiết hơn, hãy tham khảo [hướng dẫn chuyển đổi Fastify v5](https://fastify.dev/docs/v5.1.x/Guides/Migration-Guide-V5/).

> info **Gợi ý** Không có thay đổi nào đối với khớp đường dẫn trong Fastify v5 (ngoại trừ middleware, xem phần dưới đây), vì vậy bạn có thể tiếp tục sử dụng cú pháp wildcard như bạn đã làm trước đó. Hành vi vẫn giữ nguyên và các routes được định nghĩa với wildcards (như `*`) vẫn sẽ hoạt động như mong đợi.

#### CORS Fastify

Theo mặc định, chỉ các [methods được liệt kê an toàn CORS](https://fetch.spec.whatwg.org/#methods) được cho phép. Nếu bạn cần bật các methods bổ sung (như `PUT`, `PATCH`, hoặc `DELETE`), bạn phải định nghĩa chúng rõ ràng trong tùy chọn `methods`.

```typescript
const methods = ['GET', 'POST', 'PUT', 'PATCH', 'DELETE']; // HOẶC chuỗi được phân tách bằng dấu phẩy 'GET,POST,PUT,PATCH,DELETE'

const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
  { cors: { methods } },
);

// HOẶC thay vào đó, bạn có thể sử dụng phương thức `enableCors`
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
app.enableCors({ methods });
```

#### Đăng ký middleware Fastify

NestJS 11 hiện sử dụng phiên bản mới nhất của package [path-to-regexp](https://www.npmjs.com/package/path-to-regexp) để khớp **các đường dẫn middleware** trong `@nestjs/platform-fastify`. Kết quả là, cú pháp `(.*)` để khớp tất cả các đường dẫn không còn được hỗ trợ. Thay vào đó, bạn nên sử dụng các wildcards được đặt tên.

Ví dụ, nếu bạn có một middleware áp dụng cho tất cả các routes:

```typescript
// Trong NestJS 11, điều này sẽ tự động được chuyển đổi thành một route hợp lệ, ngay cả khi bạn không cập nhật nó.
.forRoutes('(.*)');
```

Bạn sẽ cần cập nhật nó để sử dụng một wildcard được đặt tên thay thế:

```typescript
.forRoutes('*splat');
```

Trong đó `splat` chỉ là một tên tùy ý cho tham số wildcard. Bạn có thể đặt tên nó bất cứ gì bạn thích.

#### Thuật toán giải quyết module

Bắt đầu với NestJS 11, thuật toán giải quyết module đã được cải thiện để nâng cao hiệu suất và giảm việc sử dụng bộ nhớ cho hầu hết các ứng dụng. Thay đổi này không yêu cầu bất kỳ sự can thiệp thủ công nào, nhưng có một số trường hợp đặc biệt trong đó hành vi có thể khác với các phiên bản trước đó.

Trong NestJS v10 và trước đó, các module động được gán một khóa duy nhất không rõ ràng được tạo ra từ metadata động của module. Khóa này được sử dụng để xác định module trong sổ đăng ký module. Ví dụ, nếu bạn bao gồm `TypeOrmModule.forFeature([User])` trong nhiều modules, NestJS sẽ loại bỏ các trùng lặp của các modules và coi chúng như một nút module duy nhất trong sổ đăng ký. Quá trình này được gọi là loại bỏ trùng lặp nút.

Với việc phát hành NestJS v11, chúng tôi không còn tạo các hashes có thể dự đoán cho các module động. Thay vào đó, các tham chiếu đối tượng hiện được sử dụng để xác định xem một module có tương đương với một module khác hay không. Để chia sẻ cùng một module động trên nhiều modules, chỉ cần gán nó cho một biến và nhập nó ở bất cứ nơi nào cần. Cách tiếp cận mới này cung cấp sự linh hoạt hơn và đảm bảo rằng các module động được xử lý hiệu quả hơn.

Thuật toán mới này có thể ảnh hưởng đến các bài kiểm thử tích hợp của bạn nếu bạn sử dụng nhiều module động, vì không có việc loại bỏ trùng lặp thủ công được đề cập ở trên, TestingModule của bạn có thể có nhiều instances của một dependency. Điều này làm cho việc giả lập một phương thức trở nên khó khăn hơn một chút, vì bạn sẽ cần nhắm đến instance đúng. Các tùy chọn của bạn là:

- Loại bỏ trùng lặp module động bạn muốn giả lập
- Sử dụng `module.select(ParentModule).get(Target)` để tìm instance đúng
- Giả lập tất cả các instances sử dụng `module.get(Target, {{ '{' }} each: true &#125;)`
- Hoặc chuyển bài kiểm thử của bạn trở lại thuật toán cũ sử dụng `Test.createTestingModule({{ '{' }}&#125;, {{ '{' }} moduleIdGeneratorAlgorithm: 'deep-hash' &#125;)`

#### Suy luận kiểu Reflector

NestJS 11 giới thiệu một số cải tiến cho class `Reflector`, nâng cao chức năng của nó và suy luận kiểu cho các giá trị metadata. Các cập nhật này cung cấp trải nghiệm trực quan và mạnh mẽ hơn khi làm việc với metadata.

1. `getAllAndMerge` hiện trả về một đối tượng thay vì một mảng chứa một phần tử duy nhất khi chỉ có một mục metadata, và `value` là kiểu `object`. Thay đổi này cải thiện tính nhất quán khi xử lý metadata dựa trên đối tượng.
2. Kiểu trả về của `getAllAndOverride` đã được cập nhật thành `T | undefined` thay vì `T`. Cập nhật này phản ánh tốt hơn khả năng không tìm thấy metadata và đảm bảo xử lý đúng các trường hợp không xác định.
3. Đối số kiểu đã chuyển đổi của `ReflectableDecorator` hiện được suy luận đúng trên tất cả các phương thức.

Các cải tiến này nâng cao trải nghiệm nhà phát triển tổng thể bằng cách cung cấp an toàn kiểu tốt hơn và xử lý metadata trong NestJS 11.

#### Thứ tự thực thi lifecycle hooks

Các lifecycle hooks chấm dứt hiện được thực thi theo thứ tự ngược lại với các đối tác khởi tạo của chúng. Điều đó nói rằng, các hooks như `OnModuleDestroy`, `BeforeApplicationShutdown` và `OnApplicationShutdown` hiện được thực thi theo thứ tự ngược lại.

Hãy tưởng tượng kịch bản sau:

```plaintext
// Trong đó A, B và C là các modules và "->" đại diện cho dependency module.
A -> B -> C
```

Trong trường hợp này, các hooks `OnModuleInit` được thực thi theo thứ tự sau:

```plaintext
C -> B -> A
```

Trong khi các hooks `OnModuleDestroy` được thực thi theo thứ tự ngược lại:

```plaintext
A -> B -> C
```

> info **Gợi ý** Các module toàn cục được coi như một dependency của tất cả các modules khác. Điều này có nghĩa là các module toàn cục được khởi tạo trước tiên và bị hủy cuối cùng.

#### Thứ tự đăng ký middleware

Trong NestJS v11, hành vi của đăng ký middleware đã được cập nhật. Trước đây, thứ tự đăng ký middleware được xác định bởi sắp xếp topo của đồ thị dependency module, trong đó khoảng cách từ module gốc định nghĩa thứ tự đăng ký middleware, bất kể middleware được đăng ký trong một module toàn cục hay một module thông thường. Các module toàn cục được coi như các module thông thường về mặt này, điều này dẫn đến hành vi không nhất quán, đặc biệt khi so sánh với các tính năng framework khác.

Từ v11 trở đi, middleware được đăng ký trong các module toàn cục hiện được **thực thi trước tiên**, bất kể vị trí của nó trong đồ thị dependency module. Thay đổi này đảm bảo rằng middleware toàn cục luôn chạy trước bất kỳ middleware nào từ các modules được nhập, duy trì một thứ tự nhất quán và có thể dự đoán.

#### Module Cache

`CacheModule` (từ package `@nestjs/cache-manager`) đã được cập nhật để hỗ trợ phiên bản mới nhất của package `cache-manager`. Bản cập nhật này mang lại một vài thay đổi phá vỡ, bao gồm việc chuyển đổi sang [Keyv](https://keyv.org/), cung cấp một giao diện thống nhất cho lưu trữ khóa-giá trị trên nhiều backend stores thông qua các bộ điều hợp lưu trữ.

Sự khác biệt chính giữa phiên bản trước và phiên bản mới nằm trong cấu hình của các stores bên ngoài. Trong phiên bản trước, để đăng ký một store Redis, bạn có thể đã cấu hình nó như sau:

```ts
// Phiên bản cũ - không còn được hỗ trợ
CacheModule.registerAsync({
  useFactory: async () => {
    const store = await redisStore({
      socket: {
        host: 'localhost',
        port: 6379,
      },
    });

    return {
      store,
    };
  },
}),
```

Trong phiên bản mới, bạn nên sử dụng bộ điều hợp `Keyv` để cấu hình store:

```ts
// Phiên bản mới - được hỗ trợ
CacheModule.registerAsync({
  useFactory: async () => {
    return {
      stores: [
        new KeyvRedis('redis://localhost:6379'),
      ],
    };
  },
}),
```

Trong đó `KeyvRedis` được nhập từ package `@keyv/redis`. Xem [tài liệu Caching](/techniques/caching) để tìm hiểu thêm.

> warning **Cảnh báo** Trong bản cập nhật này, dữ liệu được cache được xử lý bởi thư viện Keyv hiện được cấu trúc như một đối tượng chứa các trường `value` và `expires`, ví dụ: `{{ '{' }}"value": "yourData", "expires": 1678901234567{{ '}' }}`. Trong khi Keyv tự động truy xuất trường `value` khi truy cập dữ liệu thông qua API của nó, điều quan trọng là phải lưu ý thay đổi này nếu bạn tương tác trực tiếp với dữ liệu cache (ví dụ, bên ngoài API cache-manager) hoặc cần hỗ trợ dữ liệu được viết bằng phiên bản trước của `@nestjs/cache-manager`.

#### Module Config

Nếu bạn đang sử dụng `ConfigModule` từ package `@nestjs/config`, hãy lưu ý một số thay đổi phá vỡ được giới thiệu trong `@nestjs/config@4.0.0`. Đáng chú ý nhất, thứ tự trong đó các biến cấu hình được đọc bởi phương thức `ConfigService#get` đã được cập nhật. Thứ tự mới là:

- Cấu hình nội bộ (các namespace cấu hình và các tệp cấu hình tùy chỉnh)
- Các biến môi trường được xác nhận (nếu xác nhận được bật và một schema được cung cấp)
- Đối tượng `process.env`

Trước đây, các biến môi trường được xác nhận và đối tượng `process.env` được đọc trước, ngăn chúng bị ghi đè bởi cấu hình nội bộ. Với bản cập nhật này, cấu hình nội bộ sẽ luôn có quyền ưu tiên hơn các biến môi trường.

Ngoài ra, tùy chọn cấu hình `ignoreEnvVars`, trước đây cho phép vô hiệu hóa xác nhận của đối tượng `process.env`, đã bị phản đối. Thay vào đó, sử dụng tùy chọn `validatePredefined` (đặt thành `false` để vô hiệu hóa xác nhận của các biến môi trường được định nghĩa trước). Các biến môi trường được định nghĩa trước đề cập đến các biến `process.env` được đặt trước khi module được nhập. Ví dụ, nếu bạn bắt đầu ứng dụng của bạn với `PORT=3000 node main.js`, biến `PORT` được coi là được định nghĩa trước. Tuy nhiên, các biến được tải bởi `ConfigModule` từ một tệp `.env` không được phân loại là được định nghĩa trước.

Một tùy chọn mới `skipProcessEnv` cũng đã được giới thiệu. Tùy chọn này cho phép bạn ngăn phương thức `ConfigService#get` truy cập đối tượng `process.env` hoàn toàn, điều này có thể hữu ích khi bạn muốn hạn chế dịch vụ đọc trực tiếp các biến môi trường.

#### Module Terminus

Nếu bạn đang sử dụng `TerminusModule` và đã xây dựng chỉ số sức khỏe tùy chỉnh của riêng mình, một API mới đã được giới thiệu trong phiên bản 11. `HealthIndicatorService` mới được thiết kế để nâng cao khả năng đọc và kiểm thử các chỉ số sức khỏe tùy chỉnh.

Trước phiên bản 11, một chỉ số sức khỏe có thể trông như sau:

```typescript
@Injectable()
export class DogHealthIndicator extends HealthIndicator {
  constructor(private readonly httpService: HttpService) {
    super();
  }

  async isHealthy(key: string) {
    try {
      const badboys = await this.getBadboys();
      const isHealthy = badboys.length === 0;

      const result = this.getStatus(key, isHealthy, {
        badboys: badboys.length,
      });

      if (!isHealthy) {
        throw new HealthCheckError('Dog check failed', result);
      }

      return result;
    } catch (error) {
      const result = this.getStatus(key, isHealthy);
      throw new HealthCheckError('Dog check failed', result);
    }
  }

  private getBadboys() {
    return firstValueFrom(
      this.httpService.get<Dog[]>('https://example.com/dog').pipe(
        map((response) => response.data),
        map((dogs) => dogs.filter((dog) => dog.state === DogState.BAD_BOY)),
      ),
    );
  }
}
```

Bắt đầu với phiên bản 11, được khuyến nghị sử dụng API `HealthIndicatorService` mới, hợp lý hóa quá trình triển khai. Dưới đây là cách cùng một chỉ số sức khỏe hiện có thể được triển khai:

```typescript
@Injectable()
export class DogHealthIndicator {
  constructor(
    private readonly httpService: HttpService,
    //  Inject `HealthIndicatorService` được cung cấp bởi `TerminusModule`
    private readonly healthIndicatorService: HealthIndicatorService,
  ) {}

  async isHealthy(key: string) {
    // Bắt đầu kiểm tra chỉ số sức khỏe cho khóa đã cho
    const indicator = this.healthIndicatorService.check(key);

    try {
      const badboys = await this.getBadboys();
      const isHealthy = badboys.length === 0;

      if (!isHealthy) {
        // Đánh dấu chỉ số là "down" và thêm thông tin bổ sung vào phản hồi
        return indicator.down({ badboys: badboys.length });
      }

      // Đánh dấu chỉ số sức khỏe là up
      return indicator.up();
    } catch (error) {
      return indicator.down('Unable to retrieve dogs');
    }
  }

  private getBadboys() {
    // ...
  }
}
```

Các thay đổi chính:

- `HealthIndicatorService` thay thế các lớp `HealthIndicator` và `HealthCheckError` cũ, cung cấp một API sạch hơn cho các health checks.
- Phương thức `check` cho phép theo dõi trạng thái dễ dàng (`up` hoặc `down`) trong khi hỗ trợ bao gồm metadata bổ sung trong các phản hồi health check.

> info **Thông tin** Vui lòng lưu ý rằng các lớp `HealthIndicator` và `HealthCheckError` đã được đánh dấu là phản đối và được lên lịch để loại bỏ trong bản phát hành chính tiếp theo.

#### Node.js v16 và v18 không còn được hỗ trợ

Bắt đầu với NestJS 11, Node.js v16 không còn được hỗ trợ, vì nó đạt hết vòng đời (EOL) vào ngày 11 tháng 9 năm 2023. Tương tự, hỗ trợ bảo mật được lên lịch kết thúc vào ngày 30 tháng 4 năm 2025 cho Node.js v18, vì vậy chúng tôi đã tiến lên và ngừng hỗ trợ nó.

NestJS 11 hiện yêu cầu **Node.js v20 hoặc cao hơn**.

Để đảm bảo trải nghiệm tốt nhất, chúng tôi khuyến nghị mạnh việc sử dụng phiên bản LTS mới nhất của Node.js.

#### Nền tảng triển khai chính thức Mau

Trong trường hợp bạn bỏ lỡ thông báo, chúng tôi đã ra mắt nền tảng triển khai chính thức của mình, [Mau](https://www.mau.nestjs.com/), vào năm 2024.
Mau là một nền tảng được quản lý hoàn toàn đơn giản hóa quy trình triển khai cho các ứng dụng NestJS. Với Mau, bạn có thể triển khai các ứng dụng của mình lên đám mây (**AWS**; Amazon Web Services) với một lệnh duy nhất, quản lý các biến môi trường của bạn, và giám sát hiệu suất ứng dụng của bạn theo thời gian thực.

Mau làm cho việc cung cấp và duy trì cơ sở hạ tầng của bạn đơn giản như nhấp chỉ một vài nút. Mau được thiết kế để đơn giản và trực quan, vì vậy bạn có thể tập trung vào việc xây dựng các ứng dụng của mình và không phải lo lắng về cơ sở hạ tầng cơ bản. Dưới bề mặt, chúng tôi sử dụng Amazon Web Services để cung cấp cho bạn một nền tảng mạnh mẽ và đáng tin cậy, trong khi trừu tượng hóa tất cả sự phức tạp của AWS. Chúng tôi xử lý tất cả công việc nặng nhọc cho bạn, vì vậy bạn có thể tập trung vào việc xây dựng các ứng dụng của mình và phát triển doanh nghiệp của bạn.

```bash
$ npm install -g @nestjs/mau
$ mau deploy
```

Bạn có thể tìm hiểu thêm về Mau [trong chương này](/deployment#easy-deployment-with-mau).