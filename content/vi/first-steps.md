### Bước đầu tiên

Trong bộ bài viết này, bạn sẽ học **các nguyên tắc cốt lõi** của Nest. Để làm quen với các thành phần xây dựng thiết yếu của ứng dụng Nest, chúng ta sẽ xây dựng một ứng dụng CRUD cơ bản với các tính năng bao phủ nhiều khía cạnh ở mức độ giới thiệu.

#### Ngôn ngữ

Chúng tôi yêu thích TypeScript, nhưng trên hết - chúng tôi yêu thích Node.js. Đó là lý do Nest tương thích với cả TypeScript và JavaScript thuần. Nest tận dụng các tính năng ngôn ngữ mới nhất, vì vậy để sử dụng nó với JavaScript vanilla, chúng ta cần một trình biên dịch Babel.

Chúng tôi sẽ chủ yếu sử dụng TypeScript trong các ví dụ chúng tôi cung cấp, nhưng bạn luôn có thể **chuyển đổi các đoạn code** sang cú pháp JavaScript vanilla (chỉ cần nhấp để chuyển đổi nút ngôn ngữ ở góc trên bên phải của mỗi đoạn).

#### Điều kiện tiên quyết

Vui lòng đảm bảo rằng Node.js (phiên bản >= 20) đã được cài đặt trên hệ điều hành của bạn.

#### Thiết lập

Thiết lập một dự án mới khá đơn giản với Nest CLI. Với npm được cài đặt, bạn có thể tạo một dự án Nest mới với các lệnh sau trong terminal của hệ điều hành:

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

> info **Gợi ý** Để tạo một dự án mới với tập hợp tính năng nghiêm ngặt hơn của TypeScript, hãy truyền flag `--strict` vào lệnh `nest new`.

Thư mục `project-name` sẽ được tạo, các node modules và một số file boilerplate khác sẽ được cài đặt, và một thư mục `src/` sẽ được tạo và điền với một số file chính.

<div class="file-tree">
  <div class="item">src</div>
  <div class="children">
    <div class="item">app.controller.spec.ts</div>
    <div class="item">app.controller.ts</div>
    <div class="item">app.module.ts</div>
    <div class="item">app.service.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

Dưới đây là tổng quan ngắn gọn về các file chính:

||                          |                                                                                                                     |
|| ------------------------ | ------------------------------------------------------------------------------------------------------------------- |
|| `app.controller.ts`      | Một controller cơ bản với một route đơn.                                                                             |
|| `app.controller.spec.ts` | Các bài kiểm tra đơn vị cho controller.                                                                                  |
|| `app.module.ts`          | Module gốc của ứng dụng.                                                                                 |
|| `app.service.ts`         | Một service cơ bản với một phương thức đơn.                                                                               |
|| `main.ts`                | File entry của ứng dụng sử dụng hàm chính `NestFactory` để tạo một instance ứng dụng Nest. |

`main.ts` bao gồm một async function, sẽ **bootstrap** ứng dụng của chúng ta:

```typescript
@@filename(main)

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

Để tạo một instance ứng dụng Nest, chúng ta sử dụng class chính `NestFactory`. `NestFactory` expose một số static methods cho phép tạo một instance ứng dụng. Method `create()` trả về một đối tượng ứng dụng, which fulfills interface `INestApplication`. Đối tượng này cung cấp một tập hợp các methods được mô tả trong các chương sắp tới. Trong ví dụ `main.ts` ở trên, chúng ta chỉ cần khởi động HTTP listener của mình, cho phép ứng dụng chờ các yêu cầu HTTP inbound.

Lưu ý rằng một dự án được scaffold với Nest CLI tạo ra cấu trúc dự án ban đầu khuyến khích các nhà phát triển tuân theo quy ước giữ mỗi module trong thư mục riêng biệt của nó.

> info **Gợi ý** Mặc định, nếu bất kỳ lỗi nào xảy ra trong khi tạo ứng dụng, ứng dụng của bạn sẽ thoát với code `1`. Nếu bạn muốn nó throw một lỗi thay vào đó, hãy tắt tùy chọn `abortOnError` (ví dụ, `NestFactory.create(AppModule, { abortOnError: false })`).

<app-banner-courses></app-banner-courses>

#### Nền tảng

Nest nhằm mục đích là một framework platform-agnostic. Sự độc lập nền tảng giúp có thể tạo ra các phần logic có thể tái sử dụng mà các nhà phát triển có thể tận dụng trên một số loại ứng dụng khác nhau. Về mặt kỹ thuật, Nest có thể làm việc với bất kỳ framework HTTP Node nào khi một adapter được tạo. Có hai nền tảng HTTP được hỗ trợ out-of-the-box: express và fastify. Bạn có thể chọn cái phù hợp nhất với nhu cầu của bạn.

||                    |                                                                                                                                                                                                                                                                                                                                    |
|| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|| `platform-express` | Express là một framework web tối giản nổi tiếng cho node. Nó là một thư viện đã được thử nghiệm trong chiến trường, sẵn sàng cho production với rất nhiều tài nguyên được triển khai bởi cộng đồng. Package `@nestjs/platform-express` được sử dụng theo mặc định. Nhiều người dùng được phục vụ tốt với Express, và không cần thực hiện hành động nào để kích hoạt nó. |
|| `platform-fastify` | Fastify là một framework hiệu suất cao và chi phí thấp tập trung mạnh vào việc cung cấp hiệu quả và tốc độ tối đa. Đọc cách sử dụng nó ở đây.                                                                                                                                  |

Bất kể nền tảng nào được sử dụng, nó expose interface ứng dụng riêng của nó. Chúng được nhìn thấy lần lượt là `NestExpressApplication` và `NestFastifyApplication`.

Khi bạn truyền một type vào method `NestFactory.create()`, như trong ví dụ dưới đây, đối tượng `app` sẽ có các methods chỉ có sẵn cho nền tảng cụ thể đó. Tuy nhiên, lưu ý rằng bạn không **cần** chỉ định một type **trừ khi** bạn thực sự muốn truy cập API nền tảng cơ bản.

```typescript
const app = await NestFactory.create<NestExpressApplication>(AppModule);
```

#### Chạy ứng dụng

Khi quá trình cài đặt hoàn tất, bạn có thể chạy lệnh sau tại nhắc lệnh hệ điều hành để bắt đầu ứng dụng lắng nghe các yêu cầu HTTP inbound:

```bash
$ npm run start
```

> info **Gợi ý** Để tăng tốc quá trình phát triển (build nhanh hơn 20 lần), bạn có thể sử dụng SWC builder bằng cách truyền flag `-b swc` vào script `start`, như sau `npm run start -- -b swc`.

Lệnh này khởi động ứng dụng với HTTP server lắng nghe trên port được định nghĩa trong file `src/main.ts`. Khi ứng dụng đang chạy, mở trình duyệt của bạn và điều hướng đến `http://localhost:3000/`. Bạn sẽ thấy thông báo `Hello World!`.

Để xem các thay đổi trong files của bạn, bạn có thể chạy lệnh sau để bắt đầu ứng dụng:

```bash
$ npm run start:dev
```

Lệnh này sẽ xem files của bạn, tự động biên dịch lại và tải lại server.

#### Linting và định dạng

CLI cung cấp nỗ lực tốt nhất để scaffold một quy trình phát triển đáng tin cậy ở quy mô lớn. Do đó, một dự án Nest được tạo ra đi kèm với cả code linter và formatter được cài đặt sẵn (lần lượt là eslint và prettier).

> info **Gợi ý** Không chắc về vai trò của formatters so với linters? Tìm hiểu sự khác biệt ở đây.

Để đảm bảo tính ổn định và khả năng mở rộng tối đa, chúng tôi sử dụng các gói cli cơ bản `eslint` và `prettier`. Thiết lập này cho phép tích hợp IDE gọn gàng với các tiện ích mở rộng chính thức theo thiết kế.

Đối với môi trường headless nơi IDE không liên quan (Tích hợp liên tục, Git hooks, v.v.), một dự án Nest đi kèm với các script npm sẵn sàng sử dụng.

```bash
# Lint và autofix với eslint
$ npm run lint

# Format với prettier
$ npm run format
```