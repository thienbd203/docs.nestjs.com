### Giới thiệu

Specification [OpenAPI](https://swagger.io/specification/) là định dạng định nghĩa không phụ thuộc ngôn ngữ được sử dụng để mô tả các API RESTful. Nest cung cấp một [module](https://github.com/nestjs/swagger) chuyên dụng cho phép tạo specification như vậy bằng cách tận dụng decorators.

#### Cài đặt

Để bắt đầu sử dụng, trước tiên chúng ta cài đặt dependency cần thiết.

```bash
$ npm install --save @nestjs/swagger
```

#### Bootstrap

Sau khi quá trình cài đặt hoàn tất, mở file `main.ts` và khởi tạo Swagger sử dụng class `SwaggerModule`:

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Cats example')
    .setDescription('The cats API description')
    .setVersion('1.0')
    .addTag('cats')
    .build();
  const documentFactory = () => SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, documentFactory);

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> info **Gợi ý** Phương thức factory `SwaggerModule.createDocument()` được sử dụng cụ thể để tạo tài liệu Swagger khi bạn yêu cầu nó. Cách tiếp cận này giúp tiết kiệm một số thời gian khởi tạo, và tài liệu kết quả là một đối tượng có thể serialize tuân theo specification [OpenAPI Document](https://swagger.io/specification/#openapi-document). Thay vì phục vụ tài liệu qua HTTP, bạn cũng có thể lưu nó dưới dạng file JSON hoặc YAML và sử dụng nó theo nhiều cách khác nhau.

`DocumentBuilder` giúp cấu trúc một tài liệu cơ bản tuân theo OpenAPI Specification. Nó cung cấp một số phương thức cho phép đặt các thuộc tính như title, description, version, v.v. Để tạo một tài liệu đầy đủ (với tất cả các HTTP routes được định nghĩa), chúng ta sử dụng phương thức `createDocument()` của class `SwaggerModule`. Phương thức này nhận hai đối số, một instance ứng dụng và một đối tượng tùy chọn Swagger. Ngoài ra, chúng ta có thể cung cấp một đối số thứ ba, nên là kiểu `SwaggerDocumentOptions`. Xem thêm về điều này trong [phần Document options](/openapi/introduction#document-options).

Khi chúng ta tạo một tài liệu, chúng ta có thể gọi phương thức `setup()`. Nó chấp nhận:

1. Đường dẫn để mount Swagger UI
2. Một instance ứng dụng
3. Đối tượng tài liệu được khởi tạo ở trên
4. Tham số cấu hình tùy chọn (đọc thêm [ở đây](/openapi/introduction#setup-options))

Bây giờ bạn có thể chạy lệnh sau để khởi động HTTP server:

```bash
$ npm run start
```

Trong khi ứng dụng đang chạy, mở trình duyệt của bạn và điều hướng đến `http://localhost:3000/api`. Bạn sẽ thấy Swagger UI.

<figure><img src="/assets/swagger1.png" /></figure>

Như bạn có thể thấy, `SwaggerModule` tự động phản ánh tất cả các endpoints của bạn.

> info **Gợi ý** Để tạo và tải xuống một file Swagger JSON, điều hướng đến `http://localhost:3000/api-json` (giả sử rằng tài liệu Swagger của bạn có sẵn tại `http://localhost:3000/api`).
> Nó cũng có thể được expose trên một route của lựa chọn của bạn chỉ sử dụng phương thức setup từ `@nestjs/swagger`, như sau:
>
> ```typescript
> SwaggerModule.setup('swagger', app, documentFactory, {
>   jsonDocumentUrl: 'swagger/json',
> });
> ```
>
> Điều này sẽ expose nó tại `http://localhost:3000/swagger/json`

> warning **Cảnh báo** Khi sử dụng `fastify` và `helmet`, có thể có vấn đề với [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP), để giải quyết xung đột này, cấu hình CSP như được hiển thị dưới đây:
>
> ```typescript
> app.register(helmet, {
>   contentSecurityPolicy: {
>     directives: {
>       defaultSrc: [`'self'`],
>       styleSrc: [`'self'`, `'unsafe-inline'`],
>       imgSrc: [`'self'`, 'data:', 'validator.swagger.io'],
>       scriptSrc: [`'self'`, `https:`, `'unsafe-inline'`],
>     },
>   },
> });
>
> // Nếu bạn không định sử dụng CSP, bạn có thể sử dụng điều này:
> app.register(helmet, {
>   contentSecurityPolicy: false,
> });
> ```

#### Document options

Khi tạo một tài liệu, có thể cung cấp một số tùy chọn bổ sung để tinh chỉnh hành vi của thư viện. Các tùy chọn này nên là kiểu `SwaggerDocumentOptions`, có thể như sau:

```TypeScript
export interface SwaggerDocumentOptions {
  /**
   * Danh sách các module để bao gồm trong specification
   */
  include?: Function[];

  /**
   * Các models bổ sung, thêm nên được kiểm tra và bao gồm trong specification
   */
  extraModels?: Function[];

  /**
   * Nếu `true`, swagger sẽ bỏ qua tiền tố toàn cầu được đặt thông qua phương thức `setGlobalPrefix()`
   */
  ignoreGlobalPrefix?: boolean;

  /**
   * Nếu `true`, swagger cũng sẽ tải các routes từ các modules được import bởi các modules `include`
   */
  deepScanRoutes?: boolean;

  /**
   * Custom operationIdFactory sẽ được sử dụng để tạo `operationId`
   * dựa trên `controllerKey`, `methodKey`, và phiên bản.
   * @default () => controllerKey_methodKey_version
   */
  operationIdFactory?: OperationIdFactory;

  /**
   * Custom linkNameFactory sẽ được sử dụng để tạo tên của các links
   * trong trường `links` của các responses
   *
   * @see [Link objects](https://swagger.io/docs/specification/links/)
   *
   * @default () => `${controllerKey}_${methodKey}_from_${fieldKey}`
   */
  linkNameFactory?: (
    controllerKey: string,
    methodKey: string,
    fieldKey: string
  ) => string;

  /*
   * Tạo tags tự động dựa trên tên controller.
   * Nếu `false`, bạn phải sử dụng decorator `@ApiTags()` để định nghĩa tags.
   * Nếu không, tên controller mà không có hậu tố `Controller` sẽ được sử dụng.
   * @default true
   */
  autoTagControllers?: boolean;
}
```

Ví dụ, nếu bạn muốn đảm bảo rằng thư viện tạo ra tên operation như `createUser` thay vì `UsersController_createUser`, bạn có thể đặt như sau:

```TypeScript
const options: SwaggerDocumentOptions =  {
  operationIdFactory: (
    controllerKey: string,
    methodKey: string
  ) => methodKey
};
const documentFactory = () => SwaggerModule.createDocument(app, config, options);
```

#### Setup options

Bạn có thể cấu hình Swagger UI bằng cách truyền đối tượng tùy chọn thoả mãn interface `SwaggerCustomOptions` làm đối số thứ tư của phương thức `SwaggerModule#setup`.

```TypeScript
export interface SwaggerCustomOptions {
  /**
   * Nếu `true`, các đường dẫn tài nguyên Swagger sẽ được tiền tố bởi tiền tố toàn cầu được đặt thông qua `setGlobalPrefix()`.
   * Mặc định: `false`.
   * @see https://docs.nestjs.com/faq/global-prefix
   */
  useGlobalPrefix?: boolean;

  /**
   * Nếu `false`, Swagger UI sẽ không được phục vụ. Chỉ các định nghĩa API (JSON và YAML)
   * sẽ có thể truy cập được (trên `/{path}-json` và `/{path}-yaml`). Để vô hiệu hóa hoàn toàn cả Swagger UI và định nghĩa API, sử dụng `raw: false`.
   * Mặc định: `true`.
   * @deprecated Sử dụng `ui` thay vào đó.
   */
  swaggerUiEnabled?: boolean;

  /**
   * Nếu `false`, Swagger UI sẽ không được phục vụ. Chỉ các định nghĩa API (JSON và YAML)
   * sẽ có thể truy cập được (trên `/{path}-json` và `/{path}-yaml`). Để vô hiệu hóa hoàn toàn cả Swagger UI và định nghĩa API, sử dụng `raw: false`.
   * Mặc định: `true`.
   */
  ui?: boolean;

  /**
   * Nếu `true`, các định nghĩa raw cho tất cả các định dạng sẽ được phục vụ.
   * Ngoài ra, bạn có thể truyền một mảng để chỉ định các định dạng được phục vụ, ví dụ: `raw: ['json']` để chỉ phục vụ các định nghĩa JSON.
   * Nếu bị bỏ qua hoặc đặt thành một mảng rỗng, không có định nghĩa nào (JSON hoặc YAML) sẽ được phục vụ.
   * Sử dụng tùy chọn này để kiểm soát tính sẵn có của các endpoints liên quan đến Swagger.
   * Mặc định: `true`.
   */
  raw?: boolean | Array<'json' | 'yaml'>;

  /**
   * Url chỉ định định nghĩa API để tải trong Swagger UI.
   */
  swaggerUrl?: string;

  /**
   * Đường dẫn của định nghĩa API JSON để phục vụ.
   * Mặc định: `<path>-json`.
   */
  jsonDocumentUrl?: string;

  /**
   * Đường dẫn của định nghĩa API YAML để phục vụ.
   * Mặc định: `<path>-yaml`.
   */
  yamlDocumentUrl?: string;

  /**
   * Hook cho phép thay đổi tài liệu OpenAPI trước khi được phục vụ.
   * Nó được gọi sau khi tài liệu được tạo và trước khi nó được phục vụ dưới dạng JSON & YAML.
   */
  patchDocumentOnRequest?: <TRequest = any, TResponse = any>(
    req: TRequest,
    res: TResponse,
    document: OpenAPIObject
  ) => OpenAPIObject;

  /**
   * Nếu `true`, selector của các định nghĩa OpenAPI được hiển thị trong giao diện Swagger UI.
   * Mặc định: `false`.
   */
  explorer?: boolean;

  /**
   * Các tùy chọn Swagger UI bổ sung
   */
  swaggerOptions?: SwaggerUiOptions;

  /**
   * Các kiểu CSS tùy chỉnh để inject trong trang Swagger UI.
   */
  customCss?: string;

  /**
   * URL(s) của một stylesheet CSS tùy chỉnh để tải trong trang Swagger UI.
   */
  customCssUrl?: string | string[];

  /**
   * URL(s) của các file JavaScript tùy chỉnh để tải trong trang Swagger UI.
   */
  customJs?: string | string[];

  /**
   * Các script JavaScript tùy chỉnh để tải trong trang Swagger UI.
   */
  customJsStr?: string | string[];

  /**
   * Favicon tùy chỉnh cho trang Swagger UI.
   */
  customfavIcon?: string;

  /**
   * Tiêu đề tùy chỉnh cho trang Swagger UI.
   */
  customSiteTitle?: string;

  /**
   * Đường dẫn hệ thống file (ví dụ: ./node_modules/swagger-ui-dist) chứa các tài sản tĩnh Swagger UI.
   */
  customSwaggerUiPath?: string;

  /**
   * @deprecated Thuộc tính này không có hiệu lực.
   */
  validatorUrl?: string;

  /**
   * @deprecated Thuộc tính này không có hiệu lực.
   */
  url?: string;

  /**
   * @deprecated Thuộc tính này không có hiệu lực.
   */
  urls?: Record<'url' | 'name', string>[];
}
```

> info **Gợi ý** `ui` và `raw` là các tùy chọn độc lập. Vô hiệu hóa Swagger UI (`ui: false`) không vô hiệu hóa định nghĩa API (JSON/YAML). Ngược lại, vô hiệu hóa định nghĩa API (`raw: []`) không vô hiệu hóa Swagger UI.
>
> Ví dụ, cấu hình sau sẽ vô hiệu hóa Swagger UI nhưng vẫn cho phép truy cập định nghĩa API:
>
> ```typescript
> const options: SwaggerCustomOptions = {
>   ui: false, // Swagger UI bị vô hiệu hóa
>   raw: ['json'], // Định nghĩa API JSON vẫn có thể truy cập được (YAML bị vô hiệu hóa)
> };
> SwaggerModule.setup('api', app, options);
> ```
>
> Trong trường hợp này, http://localhost:3000/api-json vẫn có thể truy cập được, nhưng http://localhost:3000/api (Swagger UI) sẽ không.

#### Ví dụ

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/11-swagger).