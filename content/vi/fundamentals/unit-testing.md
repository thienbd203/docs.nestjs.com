### Testing

Testing tự động được coi là một phần thiết yếu của bất kỳ nỗ lực phát triển phần mềm nghiêm túc nào. Automation giúp việc lặp lại các bài kiểm tra riêng lẻ hoặc bộ test nhanh chóng và dễ dàng trong quá trình phát triển. Điều này giúp đảm bảo rằng các bản phát hành đáp ứng các mục tiêu về chất lượng và hiệu suất. Automation giúp tăng độ phủ và cung cấp một vòng lặp phản hồi nhanh hơn cho các nhà phát triển. Automation cả tăng năng suất của các nhà phát triển cá nhân và đảm bảo rằng các bài kiểm tra được chạy tại các mốc quan trọng của lifecycle phát triển, chẳng hạn như check-in kiểm soát mã nguồn, tích hợp tính năng và phát hành phiên bản.

Các bài kiểm tra như vậy thường kéo dài nhiều loại khác nhau, bao gồm unit tests, end-to-end (e2e) tests, integration tests, v.v. Mặc dù các lợi ích là không thể chối cãi, việc thiết lập chúng có thể tẻ nhạt. Nest cố gắng thúc đẩy các thực hành phát triển tốt nhất, bao gồm testing hiệu quả, vì vậy nó bao gồm các tính năng như sau để giúp các nhà phát triển và nhóm xây dựng và tự động hóa các bài kiểm tra. Nest:

- tự động scaffold các unit tests mặc định cho các components và e2e tests cho các ứng dụng
- cung cấp công cụ mặc định (như một test runner xây dựng một loader module/ứng dụng được cô lập)
- cung cấp tích hợp với [Jest](https://github.com/facebook/jest) và [Supertest](https://github.com/visionmedia/supertest) out-of-the-box, trong khi vẫn agnostic với các công cụ testing
- làm cho hệ thống dependency injection của Nest có sẵn trong môi trường testing để dễ dàng mock các components

Như đã đề cập, bạn có thể sử dụng bất kỳ **testing framework** nào bạn thích, vì Nest không ép buộc bất kỳ công cụ cụ thể nào. Chỉ cần thay thế các yếu tố cần thiết (như test runner), và bạn vẫn sẽ tận hưởng được lợi ích của các cơ sở testing có sẵn của Nest.

#### Installation

Để bắt đầu, trước tiên cài đặt package cần thiết:

```bash
$ npm i --save-dev @nestjs/testing
```

#### Unit testing

Trong ví dụ sau, chúng ta test hai class: `CatsController` và `CatsService`. Như đã đề cập, [Jest](https://github.com/facebook/jest) được cung cấp như framework testing mặc định. Nó đóng vai trò như một test-runner và cũng cung cấp các hàm assert và các utility test-double giúp với mocking, spying, v.v. Trong bài kiểm tra cơ bản sau, chúng ta instantiate thủ công các class này, và đảm bảo rằng controller và service đáp ứng hợp đồng API của chúng.

```typescript
@@filename(cats.controller.spec)
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
@@switch
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController;
  let catsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

> info **Hint** Giữ các file test của bạn ở gần các class mà chúng test. Các file testing nên có hậu tố `.spec` hoặc `.test`.

Vì mẫu trên là tầm thường, chúng ta không thực sự test bất cứ thứ gì cụ thể của Nest. Thật vậy, chúng ta thậm chí không sử dụng dependency injection (lưu ý rằng chúng ta truyền một instance của `CatsService` vào `catsController` của chúng ta). Hình thức testing này - nơi chúng ta instantiate thủ công các class đang được test - thường được gọi là **isolated testing** vì nó độc lập với framework. Hãy giới thiệu một số khả năng nâng cao hơn giúp bạn test các ứng dụng sử dụng nhiều tính năng của Nest hơn.

#### Testing utilities

Package `@nestjs/testing` cung cấp một tập hợp các utility cho phép quá trình testing mạnh mẽ hơn. Hãy viết lại ví dụ trước sử dụng class tích hợp sẵn `Test`:

```typescript
@@filename(cats.controller.spec)
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get(CatsService);
    catsController = moduleRef.get(CatsController);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
@@switch
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController;
  let catsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get(CatsService);
    catsController = moduleRef.get(CatsController);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

Class `Test` hữu ích để cung cấp một ngữ cảnh thực thi ứng dụng về cơ bản mock toàn bộ runtime của Nest, nhưng cung cấp cho bạn các hooks giúp dễ dàng quản lý các instances của class, bao gồm mocking và ghi đè. Class `Test` có một phương thức `createTestingModule()` nhận một đối tượng metadata module làm đối số của nó (cùng đối tượng bạn truyền cho decorator `@Module()`). Phương thức này trả về một instance `TestingModule` mà lần lượt cung cấp một vài phương thức. Đối với unit tests, phương thức quan trọng là `compile()`. Phương thức này bootstrap một module với các dependencies của nó (tương tự như cách một ứng dụng được bootstrap trong file `main.ts` thông thường sử dụng `NestFactory.create()`), và trả về một module sẵn sàng để test.

> info **Hint** Phương thức `compile()` là **asynchronous** và do đó phải được awaited. Khi module đã được biên dịch, bạn có thể truy xuất bất kỳ instance **static** nào mà nó khai báo (controllers và providers) sử dụng phương thức `get()`.

`TestingModule` kế thừa từ class [module reference](/fundamentals/module-ref), và do đó khả năng của nó để giải quyết động các providers có scope (transient hoặc request-scoped). Làm điều này với phương thức `resolve()` (phương thức `get()` chỉ có thể truy xuất các instances static).

```typescript
const moduleRef = await Test.createTestingModule({
  controllers: [CatsController],
  providers: [CatsService],
}).compile();

catsService = await moduleRef.resolve(CatsService);
```

> warning **Warning** Phương thức `resolve()` trả về một instance duy nhất của provider, từ **DI container sub-tree** riêng của nó. Mỗi sub-tree có một định danh context duy nhất. Do đó, nếu bạn gọi phương thức này nhiều hơn một lần và so sánh các tham chiếu instance, bạn sẽ thấy rằng chúng không bằng nhau.

> info **Hint** Tìm hiểu thêm về các tính năng module reference [tại đây](/fundamentals/module-ref).

Thay vì sử dụng phiên bản production của bất kỳ provider nào, bạn có thể ghi đè nó bằng một [custom provider](/fundamentals/custom-providers) cho mục đích testing. Ví dụ, bạn có thể mock một database service thay vì kết nối với một database trực tiếp. Chúng tôi sẽ bao gồm các ghi đè trong phần tiếp theo, nhưng chúng cũng có sẵn cho unit tests.

<app-banner-courses></app-banner-courses>

#### Auto mocking

Nest cũng cho phép bạn định nghĩa một mock factory để áp dụng cho tất cả các dependencies bị thiếu của bạn. Điều này hữu ích cho các trường hợp bạn có một số lượng lớn dependencies trong một class và mocking tất cả chúng sẽ mất nhiều thời gian và rất nhiều thiết lập. Để sử dụng tính năng này, `createTestingModule()` sẽ cần được chained với phương thức `useMocker()`, truyền một factory cho các dependency mocks của bạn. Factory này có thể nhận một token tùy chọn, là một token instance, bất kỳ token nào hợp lệ cho một provider Nest, và trả về một implementation mock. Dưới đây là một ví dụ về việc tạo một mocker generic sử dụng [`jest-mock`](https://www.npmjs.com/package/jest-mock) và một mock cụ thể cho `CatsService` sử dụng `jest.fn()`.

```typescript
// ...
import { ModuleMocker, MockMetadata } from 'jest-mock';

const moduleMocker = new ModuleMocker(global);

describe('CatsController', () => {
  let controller: CatsController;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      controllers: [CatsController],
    })
      .useMocker((token) => {
        const results = ['test1', 'test2'];
        if (token === CatsService) {
          return { findAll: jest.fn().mockResolvedValue(results) };
        }
        if (typeof token === 'function') {
          const mockMetadata = moduleMocker.getMetadata(
            token,
          ) as MockMetadata<any, any>;
          const Mock = moduleMocker.generateFromMetadata(
            mockMetadata,
          ) as ObjectConstructor;
          return new Mock();
        }
      })
      .compile();

    controller = moduleRef.get(CatsController);
  });
});
```

Bạn cũng có thể truy xuất các mocks này ra khỏi container testing như bạn thường làm với custom providers, `moduleRef.get(CatsService)`.

> info **Hint** Một mock factory chung chung, như `createMock` từ [`@golevelup/ts-jest`](https://github.com/golevelup/nestjs/tree/master/packages/testing) cũng có thể được truyền trực tiếp.

> info **Hint** Các providers `REQUEST` và `INQUIRER` không thể được auto-mock vì chúng đã được định nghĩa trước trong context. Tuy nhiên, chúng có thể được _ghi đè_ sử dụng cú pháp custom provider hoặc bằng cách sử dụng phương thức `.overrideProvider`.

#### End-to-end testing

Khác với unit testing, tập trung vào các module và class riêng lẻ, end-to-end (e2e) testing bao gồm sự tương tác của các class và module ở mức độ tổng hợp hơn -- gần hơn với loại tương tác mà người dùng cuối sẽ có với hệ thống production. Khi một ứng dụng phát triển, nó trở nên khó để test thủ công hành vi end-to-end của mỗi API endpoint. Các bài kiểm tra end-to-end tự động giúp chúng ta đảm bảo rằng hành vi tổng thể của hệ thống là chính xác và đáp ứng các yêu cầu dự án. Để thực hiện các bài kiểm tra e2e, chúng ta sử dụng cấu hình tương tự như chúng tôi vừa bao gồm trong **unit testing**. Ngoài ra, Nest giúp việc sử dụng thư viện [Supertest](https://github.com/visionmedia/supertest) để mô phỏng các HTTP requests trở nên dễ dàng.

```typescript
@@filename(cats.e2e-spec)
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
@@switch
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

> info **Hint** Nếu bạn đang sử dụng [Fastify](/techniques/performance) làm HTTP adapter của bạn, nó yêu cầu một cấu hình hơi khác, và có các khả năng testing tích hợp sẵn:
>
> ```ts
> let app: NestFastifyApplication;
>
> beforeAll(async () => {
>   app = moduleRef.createNestApplication<NestFastifyApplication>(
>     new FastifyAdapter(),
>   );
>
>   await app.init();
>   await app.getHttpAdapter().getInstance().ready();
> });
>
> it(`/GET cats`, () => {
>   return app
>     .inject({
>       method: 'GET',
>       url: '/cats',
>     })
>     .then((result) => {
>       expect(result.statusCode).toEqual(200);
>       expect(result.payload).toEqual(/* expectedPayload */);
>     });
> });
>
> afterAll(async () => {
>   await app.close();
> });
> ```

Trong ví dụ này, chúng ta xây dựng trên một số khái niệm được mô tả trước đó. Ngoài phương thức `compile()` chúng tôi đã sử dụng trước đó, bây giờ chúng tôi sử dụng phương thức `createNestApplication()` để instantiate một môi trường runtime Nest đầy đủ.

Một điều cần lưu ý là khi ứng dụng của bạn được biên dịch sử dụng phương thức `compile()`, `HttpAdapterHost#httpAdapter` sẽ không được định nghĩa tại thời điểm đó. Điều này là vì chưa có HTTP adapter hoặc server được tạo trong giai đoạn biên dịch này. Nếu test của bạn yêu cầu `httpAdapter`, bạn nên sử dụng phương thức `createNestApplication()` để tạo instance ứng dụng, hoặc refactor dự án của bạn để tránh dependency này khi khởi tạo dependency graph.

Được rồi, hãy chia nhỏ ví dụ:

Chúng ta lưu một tham chiếu đến ứng dụng đang chạy trong biến `app` của chúng ta để chúng ta có thể sử dụng nó để mô phỏng các HTTP requests.

Chúng ta mô phỏng các bài kiểm tra HTTP sử dụng hàm `request()` từ Supertest. Chúng ta muốn các HTTP requests này route đến ứng dụng Nest đang chạy của chúng ta, vì vậy chúng ta truyền hàm `request()` một tham chiếu đến HTTP listener nằm dưới Nest (mà lần lượt có thể được cung cấp bởi nền tảng Express). Do đó cấu trúc `request(app.getHttpServer())`. Lời gọi đến `request()` trao cho chúng ta một HTTP Server được bao bọc, bây giờ được kết nối với ứng dụng Nest,暴露 các phương thức để mô phỏng một HTTP request thực tế. Ví dụ, sử dụng `request(...).get('/cats')` sẽ khởi tạo một request đến ứng dụng Nest giống hệt một HTTP request **thực tế** như `get '/cats'` đến qua mạng.

Trong ví dụ này, chúng ta cũng cung cấp một implementation thay thế (test-double) của `CatsService` đơn giản trả về một giá trị hard-coded mà chúng ta có thể test. Sử dụng `overrideProvider()` để cung cấp một implementation thay thế như vậy. Tương tự, Nest cung cấp các phương thức để ghi đè các module, guards, interceptors, filters và pipes với các phương thức `overrideModule()`, `overrideGuard()`, `overrideInterceptor()`, `overrideFilter()`, và `overridePipe()` tương ứng.

Mỗi phương thức ghi đè (ngoại trừ `overrideModule()`) trả về một object với 3 phương thức khác nhau phản ánh những được mô tả cho [custom providers](https://docs.nestjs.com/fundamentals/custom-providers):

- `useClass`: bạn cung cấp một class sẽ được instantiate để cung cấp instance để ghi đè object (provider, guard, v.v.).
- `useValue`: bạn cung cấp một instance sẽ ghi đè object.
- `useFactory`: bạn cung cấp một hàm trả về một instance sẽ ghi đè object.

Mặt khác, `overrideModule()` trả về một object với phương thức `useModule()`, mà bạn có thể sử dụng để cung cấp một module sẽ ghi đè module gốc, như sau:

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideModule(CatsModule)
  .useModule(AlternateCatsModule)
  .compile();
```

Mỗi loại phương thức ghi đè, lần lượt, trả về instance `TestingModule`, và do đó có thể được chained với các phương thức khác trong [fluent style](https://en.wikipedia.org/wiki/Fluent_interface). Bạn nên sử dụng `compile()` ở cuối chuỗi như vậy để khiến Nest instantiate và khởi tạo module.

Ngoài ra, đôi khi bạn có thể muốn cung cấp một logger tùy chỉnh ví dụ khi các bài kiểm tra được chạy (ví dụ, trên máy chủ CI). Sử dụng phương thức `setLogger()` và truyền một object đáp ứng interface `LoggerService` để hướng dẫn `TestModuleBuilder` cách log trong các bài kiểm tra (theo mặc định, chỉ logs "error" sẽ được logged vào console).

Module đã biên dịch có một số phương thức hữu ích, như được mô tả trong bảng sau:

<table>
  <tr>
    <td>
      <code>createNestApplication()</code>
    </td>
    <td>
      Tạo và trả về một ứng dụng Nest (instance <code>INestApplication</code>) dựa trên module đã cho.
      Lưu ý rằng bạn phải khởi tạo ứng dụng thủ công sử dụng phương thức <code>init()</code>.
    </td>
  </tr>
  <tr>
    <td>
      <code>createNestMicroservice()</code>
    </td>
    <td>
      Tạo và trả về một microservice Nest (instance <code>INestMicroservice</code>) dựa trên module đã cho.
    </td>
  </tr>
  <tr>
    <td>
      <code>get()</code>
    </td>
    <td>
      Truy xuất một instance static của một controller hoặc provider (bao gồm guards, filters, v.v.) có sẵn trong ngữ cảnh ứng dụng. Kế thừa từ class <a href="/fundamentals/module-ref">module reference</a>.
    </td>
  </tr>
  <tr>
     <td>
      <code>resolve()</code>
    </td>
    <td>
      Truy xuất một instance có scope được tạo động (request hoặc transient) của một controller hoặc provider (bao gồm guards, filters, v.v.) có sẵn trong ngữ cảnh ứng dụng. Kế thừa từ class <a href="/fundamentals/module-ref">module reference</a>.
    </td>
  </tr>
  <tr>
    <td>
      <code>select()</code>
    </td>
    <td>
      Điều hướng qua dependency graph của module; có thể được sử dụng để truy xuất một instance cụ thể từ module được chọn (sử dụng cùng với chế độ strict (<code>strict: true</code>) trong phương thức <code>get()</code>).
    </td>
  </tr>
</table>

> info **Hint** Giữ các file test e2e của bạn bên trong thư mục `test`. Các file testing nên có hậu tố `.e2e-spec`.

#### Overriding globally registered enhancers

Nếu bạn có một guard được đăng ký toàn cầu (hoặc pipe, interceptor, hoặc filter), bạn cần thực hiện thêm một vài bước để ghi đè enhancer đó. Để tóm tắt lại, đăng ký gốc trông như sau:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

Đây là đăng ký guard như một "multi"-provider thông qua token `APP_*`. Để có thể thay thế `JwtAuthGuard` ở đây, đăng ký cần sử dụng một provider hiện có trong slot này:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useExisting: JwtAuthGuard,
    // ^^^^^^^^ notice the use of 'useExisting' instead of 'useClass'
  },
  JwtAuthGuard,
],
```

> info **Hint** Thay đổi `useClass` thành `useExisting` để tham chiếu một provider đã đăng ký thay vì để Nest instantiate nó sau token.

Bây giờ `JwtAuthGuard` hiển thị với Nest như một provider thông thường có thể được ghi đè khi tạo `TestingModule`:

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideProvider(JwtAuthGuard)
  .useClass(MockAuthGuard)
  .compile();
```

Bây giờ tất cả các bài kiểm tra của bạn sẽ sử dụng `MockAuthGuard` trên mỗi request.

#### Testing request-scoped instances

Các providers [request-scoped](/fundamentals/injection-scopes) được tạo duy nhất cho mỗi incoming **request**. Instance được garbage-collected sau khi request đã hoàn thành xử lý. Điều này tạo ra một vấn đề, vì chúng ta không thể truy cập một dependency injection sub-tree được tạo riêng cho một request được test.

Chúng ta biết (dựa trên các phần trên) rằng phương thức `resolve()` có thể được sử dụng để truy xuất một class được instantiate động. Ngoài ra, như được mô tả <a href="https://docs.nestjs.com/fundamentals/module-ref#resolving-scoped-providers">tại đây</a>, chúng ta biết rằng chúng ta có thể truyền một định danh context duy nhất để kiểm soát lifecycle của một sub-tree DI container. Làm thế nào để chúng ta tận dụng điều này trong ngữ cảnh testing?

Chiến lược là tạo ra một định danh context trước và buộc Nest sử dụng ID cụ thể này để tạo một sub-tree cho tất cả các incoming requests. Bằng cách này chúng ta sẽ có thể truy xuất các instances được tạo cho một request được test.

Để thực hiện điều này, sử dụng `jest.spyOn()` trên `ContextIdFactory`:

```typescript
const contextId = ContextIdFactory.create();
jest
  .spyOn(ContextIdFactory, 'getByRequest')
  .mockImplementation(() => contextId);
```

Bây giờ chúng ta có thể sử dụng `contextId` để truy cập một sub-tree DI container được tạo duy nhất cho bất kỳ request tiếp theo nào.

```typescript
catsService = await moduleRef.resolve(CatsService, contextId);
```