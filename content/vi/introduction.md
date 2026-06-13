### Giới thiệu

Nest (NestJS) là một framework để xây dựng các ứng dụng server-side Node.js hiệu quả và có khả năng mở rộng. Nó sử dụng JavaScript tiến bộ, được xây dựng với và hỗ trợ đầy đủ TypeScript (tuy nhiên vẫn cho phép các nhà phát triển viết code bằng JavaScript thuần) và kết hợp các yếu tố của OOP (Lập trình hướng đối tượng), FP (Lập trình chức năng), và FRP (Lập trình phản ứng chức năng).

Dưới lớp vỏ, Nest sử dụng các framework HTTP server mạnh mẽ như Express (mặc định) và có thể được cấu hình để sử dụng Fastify cũng được!

Nest cung cấp một mức độ trừu tượng cao hơn các framework Node.js phổ biến (Express/Fastify), nhưng cũng expose các API của chúng trực tiếp cho nhà phát triển. Điều này cho phép các nhà phát triển tự do sử dụng vô số module bên thứ ba có sẵn cho nền tảng cơ bản.

#### Triết lý

Trong những năm gần đây, nhờ vào Node.js, JavaScript đã trở thành "ngôn ngữ chung" của web cho cả ứng dụng front-end và back-end. Điều này đã dẫn đến sự ra đời của các dự án tuyệt vời như Angular, React và Vue, cải thiện năng suất của nhà phát triển và cho phép tạo ra các ứng dụng front-end nhanh, có thể kiểm thử và có thể mở rộng. Tuy nhiên, mặc dù có rất nhiều thư viện, helper và công cụ tuyệt vời cho Node (và JavaScript phía server), không cái nào giải quyết hiệu quả vấn đề chính về **kiến trúc**.

Nest cung cấp một kiến trúc ứng dụng out-of-the-box cho phép các nhà phát triển và nhóm tạo ra các ứng dụng có khả năng kiểm thử cao, có khả năng mở rộng, loosely coupled (lỏng lẻo) và dễ dàng bảo trì. Kiến trúc này được lấy cảm hứng mạnh mẽ từ Angular.

#### Cài đặt

Để bắt đầu, bạn có thể scaffold dự án với Nest CLI, hoặc clone một dự án khởi động (cả hai sẽ tạo ra cùng kết quả).

Để scaffold dự án với Nest CLI, chạy các lệnh sau. Điều này sẽ tạo một thư mục dự án mới, và điền thư mục với các file Nest chính và các module hỗ trợ ban đầu, tạo cấu trúc cơ sở thông thường cho dự án của bạn. Tạo dự án mới với Nest CLI được khuyến nghị cho người dùng lần đầu. Chúng tôi sẽ tiếp tục với cách tiếp cận này trong First Steps.

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

> info **Gợi ý** Để tạo một dự án TypeScript mới với tập hợp tính năng nghiêm ngặt hơn, hãy truyền flag `--strict` vào lệnh `nest new`.

#### Các phương án thay thế

Ngoài ra, để cài đặt dự án khởi động TypeScript với Git:

```bash
$ git clone https://github.com/nestjs/typescript-starter.git project
$ cd project
$ npm install
$ npm run start
```

> info **Gợi ý** Nếu bạn muốn clone repository mà không có lịch sử git, bạn có thể sử dụng degit.

Mở trình duyệt của bạn và điều hướng đến `http://localhost:3000/`.

Để cài đặt phiên bản JavaScript của dự án khởi động, sử dụng `javascript-starter.git` trong chuỗi lệnh ở trên.

Bạn cũng có thể bắt đầu một dự án mới từ đầu bằng cách cài đặt các package chính và hỗ trợ. Hãy nhớ rằng bạn sẽ cần thiết lập các file boilerplate của dự án theo cách của riêng bạn. Tối thiểu, bạn sẽ cần các dependencies này: `@nestjs/core`, `@nestjs/common`, `rxjs`, và `reflect-metadata`. Kiểm tra bài viết ngắn này về cách tạo một dự án hoàn chỉnh: 5 bước để tạo một ứng dụng NestJS tối thiểu từ đầu!