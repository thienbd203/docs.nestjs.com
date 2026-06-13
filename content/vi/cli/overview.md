### Tổng quan

[Nest CLI](https://github.com/nestjs/nest-cli) là công cụ giao diện dòng lệnh giúp bạn khởi tạo, phát triển và bảo trì các ứng dụng Nest của mình. Nó hỗ trợ theo nhiều cách, bao gồm scaffold dự án, serve trong chế độ development, và build và bundle ứng dụng để phân phối production. Nó thể hiện các pattern kiến trúc best-practice để khuyến khích các ứng dụng có cấu trúc tốt.

#### Cài đặt

**Lưu ý**: Trong hướng dẫn này, chúng tôi mô tả việc sử dụng [npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm) để cài đặt các package, bao gồm Nest CLI. Các package manager khác có thể được sử dụng theo quyết định của bạn. Với npm, bạn có một số tùy chọn có sẵn để quản lý cách dòng lệnh OS của bạn giải quyết vị trí của file binary CLI `nest`. Ở đây, chúng tôi mô tả việc cài đặt binary `nest` toàn cục bằng tùy chọn `-g`. Điều này mang lại sự tiện lợi, và là cách tiếp cận chúng tôi giả định trong suốt tài liệu. Lưu ý rằng cài đặt **bất kỳ** package `npm` nào toàn cục sẽ để lại trách nhiệm đảm bảo chúng đang chạy phiên bản chính xác cho người dùng. Điều này cũng có nghĩa là nếu bạn có các dự án khác nhau, mỗi dự án sẽ chạy **cùng** phiên bản của CLI. Một giải pháp thay thế hợp lý là sử dụng chương trình [npx](https://github.com/npm/cli/blob/latest/docs/lib/content/commands/npx.md), được tích hợp sẵn trong cli `npm` (hoặc các tính năng tương tự với các package manager khác) để đảm bảo rằng bạn chạy một **phiên bản được quản lý** của Nest CLI. Chúng tôi khuyến nghị bạn tham khảo [tài liệu npx](https://github.com/npm/cli/blob/latest/docs/lib/content/commands/npx.md) và/hoặc nhân viên hỗ trợ DevOps của bạn để biết thêm thông tin.

Cài đặt CLI toàn cục bằng lệnh `npm install -g` (xem **Lưu ý** ở trên để biết chi tiết về cài đặt toàn cục).

```bash
$ npm install -g @nestjs/cli
```

> info **Gợi ý** Ngoài ra, bạn có thể sử dụng lệnh này `npx @nestjs/cli@latest` mà không cần cài đặt cli toàn cục.

#### Quy trình làm việc cơ bản

Sau khi cài đặt, bạn có thể gọi các lệnh CLI trực tiếp từ dòng lệnh OS của mình thông qua thực thi `nest`. Xem các lệnh `nest` có sẵn bằng cách nhập lệnh sau:

```bash
$ nest --help
```

Nhận trợ giúp về một lệnh cụ thể bằng cách sử dụng cấu trúc sau. Thay thế bất kỳ lệnh nào, như `new`, `add`, v.v., nơi bạn thấy `generate` trong ví dụ dưới đây để nhận trợ giúp chi tiết về lệnh đó:

```bash
$ nest generate --help
```

Để tạo, build và chạy một dự án Nest cơ bản mới trong chế độ development, hãy chuyển đến thư mục nên là thư mục cha của dự án mới của bạn, và chạy các lệnh sau:

```bash
$ nest new my-nest-project
$ cd my-nest-project
$ npm run start:dev
```

Trong trình duyệt của bạn, mở [http://localhost:3000](http://localhost:3000) để xem ứng dụng mới đang chạy. Ứng dụng sẽ tự động biên dịch lại và tải lại khi bạn thay đổi bất kỳ file nguồn nào.

> info **Gợi ý** Chúng tôi khuyến nghị sử dụng [SWC builder](/recipes/swc) để build nhanh hơn (hiệu suất cao hơn 10 lần so với trình biên dịch TypeScript mặc định).

#### Cấu trúc dự án

Khi bạn chạy `nest new`, Nest tạo ra cấu trúc ứng dụng boilerplate bằng cách tạo một thư mục mới và điền một tập hợp file ban đầu. Bạn có thể tiếp tục làm việc trong cấu trúc mặc định này, thêm các thành phần mới, như được mô tả trong suốt tài liệu này. Chúng tôi gọi cấu trúc dự án được tạo bởi `nest new` là **chế độ tiêu chuẩn**. Nest cũng hỗ trợ một cấu trúc thay thế để quản lý nhiều dự án và thư viện được gọi là **chế độ monorepo**.

Ngoài một số cân nhắc cụ thể về cách quá trình **build** hoạt động (về cơ bản, chế độ monorepo đơn giản hóa các phức tạp về build có thể phát sinh từ cấu trúc dự án kiểu monorepo), và hỗ trợ [thư viện](/cli/libraries) tích hợp sẵn, phần còn lại của các tính năng Nest, và tài liệu này, áp dụng đồng cho cả cấu trúc dự án chế độ tiêu chuẩn và chế độ monorepo. Trên thực tế, bạn có thể dễ dàng chuyển từ chế độ tiêu chuẩn sang chế độ monorepo bất kỳ lúc nào trong tương lai, vì vậy bạn có thể hoãn quyết định này một cách an toàn trong khi bạn vẫn đang học về Nest.

Bạn có thể sử dụng một trong hai chế độ để quản lý nhiều dự án. Dưới đây là tóm tắt nhanh về sự khác biệt:

|| Tính năng                                                    | Chế độ Tiêu chuẩn                                                      | Chế độ Monorepo                                              |
|| ---------------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------- |
|| Nhiều dự án                                          | Cấu trúc hệ thống file riêng biệt                                     | Cấu trúc hệ thống file duy nhất                               |
|| `node_modules` & `package.json`                            | Các instance riêng biệt                                                 | Chia sẻ trong monorepo                                     |
|| Trình biên dịch mặc định                                           | `tsc`                                                              | webpack                                                    |
|| Cài đặt trình biên dịch                                          | Được chỉ định riêng biệt                                               | Mặc định monorepo có thể được ghi đè cho mỗi dự án       |
|| File cấu hình như `eslint.config.mjs`, `.prettierrc`, v.v. | Được chỉ định riêng biệt                                               | Chia sẻ trong monorepo                                     |
|| Các lệnh `nest build` và `nest start`                     | Mục tiêu mặc định tự động là dự án (duy nhất) trong ngữ cảnh | Mục tiêu mặc định là **dự án mặc định** trong monorepo |
|| Thư viện                                                  | Quản lý thủ công, thường qua đóng gói npm                        | Hỗ trợ tích hợp sẵn, bao gồm quản lý đường dẫn và bundling   |

Đọc các phần về [Workspaces](/cli/monorepo) và [Thư viện](/cli/libraries) để biết thêm thông tin chi tiết giúp bạn quyết định chế độ nào phù hợp nhất với bạn.

<app-banner-courses></app-banner-courses>

#### Cú pháp lệnh CLI

Tất cả các lệnh `nest` đều theo cùng định dạng:

```bash
nest commandOrAlias requiredArg [optionalArg] [options]
```

Ví dụ:

```bash
$ nest new my-nest-project --dry-run
```

Ở đây, `new` là _commandOrAlias_. Lệnh `new` có một alias là `n`. `my-nest-project` là _requiredArg_. Nếu _requiredArg_ không được cung cấp trên dòng lệnh, `nest` sẽ nhắc nhập. Ngoài ra, `--dry-run` có một dạng viết tắt tương đương là `-d`. Với điều này trong tâm, lệnh sau là tương đương với lệnh trên:

```bash
$ nest n my-nest-project -d
```

Hầu hết các lệnh, và một số tùy chọn, đều có alias. Thử chạy `nest new --help` để xem các tùy chọn và alias này, và để xác nhận sự hiểu biết của bạn về các cấu trúc trên.

#### Tổng quan lệnh

Chạy `nest <command> --help` cho bất kỳ lệnh nào sau đây để xem các tùy chọn cụ thể của lệnh đó.

Xem [sử dụng](/cli/usages) để biết mô tả chi tiết cho từng lệnh.

|| Lệnh    | Alias | Mô tả                                                                                    |
|| ---------- | ----- | ---------------------------------------------------------------------------------------------- |
|| `new`      | `n`   | Scaffold một ứng dụng _chế độ tiêu chuẩn_ mới với tất cả các file boilerplate cần thiết để chạy.          |
|| `generate` | `g`   | Tạo và/hoặc sửa đổi các file dựa trên một schematic.                                          |
|| `build`    |       | Biên dịch một ứng dụng hoặc workspace vào một thư mục đầu ra.                                    |
|| `start`    |       | Biên dịch và chạy một ứng dụng (hoặc dự án mặc định trong một workspace).                          |
|| `add`      |       | Nhập một thư viện đã được đóng gói như một **nest library**, chạy schematic cài đặt của nó. |
|| `info`     | `i`   | Hiển thị thông tin về các gói nest đã cài đặt và thông tin hệ thống hữu ích khác.              |

#### Yêu cầu

Nest CLI yêu cầu một binary Node.js được xây dựng với [hỗ trợ quốc tế hóa](https://nodejs.org/api/intl.html) (ICU), chẳng hạn như các binary chính thức từ [trang dự án Node.js](https://nodejs.org/en/download). Nếu bạn gặp lỗi liên quan đến ICU, hãy kiểm tra xem binary của bạn có đáp ứng yêu cầu này không.

```bash
node -p process.versions.icu
```

Nếu lệnh in ra `undefined`, binary Node.js của bạn không có hỗ trợ quốc tế hóa.