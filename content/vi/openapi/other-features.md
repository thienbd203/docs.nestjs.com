### Các tính năng khác

Trang này liệt kê tất cả các tính năng khác có sẵn mà bạn có thể thấy hữu ích.

#### Global prefix

Để bỏ qua một tiền tố toàn cầu cho các routes được đặt thông qua `setGlobalPrefix()`, sử dụng `ignoreGlobalPrefix`:

```typescript
const document = SwaggerModule.createDocument(app, options, {
  ignoreGlobalPrefix: true,
});
```

#### Global parameters

Bạn có thể định nghĩa các tham số cho tất cả các routes sử dụng `DocumentBuilder`, như được hiển thị dưới đây:

```typescript
const config = new DocumentBuilder()
  .addGlobalParameters({
    name: 'tenantId',
    in: 'header',
  })
  // các cấu hình khác
  .build();
```

#### Global responses

Bạn có thể định nghĩa các responses toàn cầu cho tất cả các routes sử dụng `DocumentBuilder`. Điều này hữu ích để thiết lập các responses nhất quán trên tất cả các endpoints trong ứng dụng của bạn, chẳng hạn như các mã lỗi như `401 Unauthorized` hoặc `500 Internal Server Error`.

```typescript
const config = new DocumentBuilder()
  .addGlobalResponse({
    status: 500,
    description: 'Internal server error',
  })
  // các cấu hình khác
  .build();
```

#### Multiple specifications

`SwaggerModule` cung cấp một cách để hỗ trợ nhiều specifications. Nói cách khác, bạn có thể phục vụ các tài liệu khác nhau, với các UI khác nhau, trên các endpoints khác nhau.

Để hỗ trợ nhiều specifications, ứng dụng của bạn phải được viết với cách tiếp cận module. Phương thức `createDocument()` nhận đối số thứ 3, `extraOptions`, là một đối tượng với một thuộc tính tên là `include`. Thuộc tính `include` nhận một giá trị là một mảng các modules.

Bạn có thể thiết lập hỗ trợ nhiều specifications như được hiển thị dưới đây:

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { CatsModule } from './cats/cats.module';
import { DogsModule } from './dogs/dogs.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  /**
   * createDocument(application, configurationOptions, extraOptions);
   *
   * phương thức createDocument nhận đối số thứ 3 tùy chọn "extraOptions"
   * là một đối tượng với thuộc tính "include" nơi bạn có thể truyền một Mảng
   * của các Modules mà bạn muốn bao gồm trong Swagger Specification đó
   * Ví dụ: CatsModule và DogsModule sẽ có hai Swagger Specifications riêng biệt
   * sẽ được expose trên hai SwaggerUI khác nhau với hai endpoints khác nhau.
   */

  const options = new DocumentBuilder()
    .setTitle('Cats example')
    .setDescription('The cats API description')
    .setVersion('1.0')
    .addTag('cats')
    .build();

  const catDocumentFactory = () =>
    SwaggerModule.createDocument(app, options, {
      include: [CatsModule],
    });
  SwaggerModule.setup('api/cats', app, catDocumentFactory);

  const secondOptions = new DocumentBuilder()
    .setTitle('Dogs example')
    .setDescription('The dogs API description')
    .setVersion('1.0')
    .addTag('dogs')
    .build();

  const dogDocumentFactory = () =>
    SwaggerModule.createDocument(app, secondOptions, {
      include: [DogsModule],
    });
  SwaggerModule.setup('api/dogs', app, dogDocumentFactory);

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

Bây giờ bạn có thể khởi động server của bạn với lệnh sau:

```bash
$ npm run start
```

Điều hướng đến `http://localhost:3000/api/cats` để xem Swagger UI cho cats:

<figure><img src="/assets/swagger-cats.png" /></figure>

Lần lượt, `http://localhost:3000/api/dogs` sẽ expose Swagger UI cho dogs:

<figure><img src="/assets/swagger-dogs.png" /></figure>

#### Dropdown trong thanh explorer

Để kích hoạt hỗ trợ cho nhiều specifications trong menu dropdown của thanh explorer, bạn cần đặt `explorer: true` và cấu hình `swaggerOptions.urls` trong `SwaggerCustomOptions` của bạn.

> info **Gợi ý** Đảm bảo rằng `swaggerOptions.urls` trỏ đến định dạng JSON của các tài liệu Swagger của bạn! Để chỉ định tài liệu JSON, sử dụng `jsonDocumentUrl` trong `SwaggerCustomOptions`. Để biết thêm các tùy chọn thiết lập, kiểm tra [ở đây](/openapi/introduction#setup-options).

Đây là cách thiết lập nhiều specifications từ một dropdown trong thanh explorer:

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { CatsModule } from './cats/cats.module';
import { DogsModule } from './dogs/dogs.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Tùy chọn API chính
  const options = new DocumentBuilder()
    .setTitle('Multiple Specifications Example')
    .setDescription('Description for multiple specifications')
    .setVersion('1.0')
    .build();

  // Tạo tài liệu API chính
  const document = SwaggerModule.createDocument(app, options);

  // Thiết lập Swagger UI API chính với hỗ trợ dropdown
  SwaggerModule.setup('api', app, document, {
    explorer: true,
    swaggerOptions: {
      urls: [
        {
          name: '1. API',
          url: 'api/swagger.json',
        },
        {
          name: '2. Cats API',
          url: 'api/cats/swagger.json',
        },
        {
          name: '3. Dogs API',
          url: 'api/dogs/swagger.json',
        },
      ],
    },
    jsonDocumentUrl: '/api/swagger.json',
  });

  // Tùy chọn Cats API
  const catOptions = new DocumentBuilder()
    .setTitle('Cats Example')
    .setDescription('Description for the Cats API')
    .setVersion('1.0')
    .addTag('cats')
    .build();

  // Tạo tài liệu Cats API
  const catDocument = SwaggerModule.createDocument(app, catOptions, {
    include: [CatsModule],
  });

  // Thiết lập Swagger UI Cats API
  SwaggerModule.setup('api/cats', app, catDocument, {
    jsonDocumentUrl: '/api/cats/swagger.json',
  });

  // Tùy chọn Dogs API
  const dogOptions = new DocumentBuilder()
    .setTitle('Dogs Example')
    .setDescription('Description for the Dogs API')
    .setVersion('1.0')
    .addTag('dogs')
    .build();

  // Tạo tài liệu Dogs API
  const dogDocument = SwaggerModule.createDocument(app, dogOptions, {
    include: [DogsModule],
  });

  // Thiết lập Swagger UI Dogs API
  SwaggerModule.setup('api/dogs', app, dogDocument, {
    jsonDocumentUrl: '/api/dogs/swagger.json',
  });

  await app.listen(3000);
}

bootstrap();
```

Trong ví dụ này, chúng ta thiết lập một API chính cùng với các specifications riêng biệt cho Cats và Dogs, mỗi cái có thể truy cập từ dropdown trong thanh explorer.