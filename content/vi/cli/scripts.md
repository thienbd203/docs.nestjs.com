### Nest CLI và scripts

Phần này cung cấp thêm thông tin nền về cách lệnh `nest` tương tác với các trình biên dịch và scripts để giúp nhân viên DevOps quản lý môi trường phát triển.

Một ứng dụng Nest là một ứng dụng TypeScript **tiêu chuẩn** cần được biên dịch sang JavaScript trước khi có thể thực thi. Có nhiều cách để thực hiện bước biên dịch, và các nhà phát triển/nhóm được tự do chọn cách hoạt động tốt nhất cho họ. Với điều này trong tâm, Nest cung cấp một bộ công cụ out-of-the-box tìm cách thực hiện những điều sau:

- Cung cấp quy trình build/thực thi tiêu chuẩn, có sẵn tại dòng lệnh, "chỉ hoạt động" với các mặc định hợp lý.
- Đảm bảo rằng quy trình build/thực thi là **mở**, để các nhà phát triển có thể truy cập trực tiếp các công cụ cơ bản để tùy chỉnh chúng bằng các tính năng và tùy chọn gốc.
- Giữ nguyên là một framework TypeScript/Node.js hoàn toàn tiêu chuẩn, để toàn bộ pipeline biên dịch/triển khai/thực thi có thể được quản lý bởi bất kỳ công cụ bên ngoài nào mà nhóm phát triển chọn sử dụng.

Mục tiêu này được thực hiện thông qua sự kết hợp của lệnh `nest`, một trình biên dịch TypeScript được cài đặt cục bộ, và các scripts `package.json`. Chúng tôi mô tả cách các công nghệ này hoạt động cùng nhau dưới đây. Điều này nên giúp bạn hiểu những gì đang xảy ra ở mỗi bước của quy trình build/thực thi, và cách tùy chỉnh hành vi đó nếu cần.

#### Binary nest

Lệnh `nest` là một binary cấp OS (tức là chạy từ dòng lệnh OS). Lệnh này thực sự bao gồm 3 lĩnh vực riêng biệt, được mô tả dưới đây. Chúng tôi khuyến nghị bạn chạy các lệnh con build (`nest build`) và thực thi (`nest start`) thông qua các scripts `package.json` được cung cấp tự động khi một dự án được scaffold (xem [typescript starter](https://github.com/nestjs/typescript-starter) nếu bạn muốn bắt đầu bằng cách clone một repo, thay vì chạy `nest new`).

#### Build

`nest build` là một wrapper trên trình biên dịch tiêu chuẩn `tsc` hoặc trình biên dịch `swc` (cho [các dự án tiêu chuẩn](https://docs.nestjs.com/cli/overview#project-structure)) hoặc bundler webpack sử dụng `ts-loader` (cho [các monorepos](https://docs.nestjs.com/cli/overview#project-structure)). Nó không thêm bất kỳ tính năng hoặc bước biên dịch nào khác ngoại trừ việc xử lý `tsconfig-paths` out-of-the-box. Lý do nó tồn tại là hầu hết các nhà phát triển, đặc biệt khi bắt đầu với Nest, không cần điều chỉnh các tùy chọn trình biên dịch (ví dụ, file `tsconfig.json`) mà đôi khi có thể phức tạp.

Xem tài liệu [nest build](https://docs.nestjs.com/cli/usages#nest-build) để biết thêm chi tiết.

#### Thực thi

`nest start` đơn giản đảm bảo rằng dự án đã được build (giống như `nest build`), sau đó gọi lệnh `node` theo cách di động, dễ dàng để thực thi ứng dụng đã biên dịch. Giống như build, bạn được tự do tùy chỉnh quy trình này nếu cần, bằng cách sử dụng lệnh `nest start` và các tùy chọn của nó, hoặc hoàn toàn thay thế nó. Toàn bộ quy trình là một pipeline build và thực thi ứng dụng TypeScript tiêu chuẩn, và bạn được tự do quản lý quy trình như vậy.

Xem tài liệu [nest start](https://docs.nestjs.com/cli/usages#nest-start) để biết thêm chi tiết.

#### Tạo

Các lệnh `nest generate`, như tên gọi, tạo các dự án Nest mới, hoặc các thành phần trong chúng.

#### Scripts package

Chạy các lệnh `nest` ở cấp dòng lệnh OS yêu cầu binary `nest` được cài đặt toàn cục. Đây là một tính năng tiêu chuẩn của npm, và nằm ngoài sự kiểm soát trực tiếp của Nest. Một hệ quả của điều này là binary `nest` được cài đặt toàn cục **không** được quản lý như một phụ thuộc dự án trong `package.json`. Ví dụ, hai nhà phát triển khác nhau có thể đang chạy hai phiên bản khác nhau của binary `nest`. Giải pháp tiêu chuẩn cho điều này là sử dụng các scripts package để bạn có thể coi các công cụ được sử dụng trong các bước build và thực thi là các phụ thuộc phát triển.

Khi bạn chạy `nest new`, hoặc clone [typescript starter](https://github.com/nestjs/typescript-starter), Nest điền các scripts `package.json` của dự án mới với các lệnh như `build` và `start`. Nó cũng cài đặt các công cụ trình biên dịch cơ bản (như `typescript`) như **phụ thuộc phát triển**.

Bạn chạy các scripts build và thực thi với các lệnh như:

```bash
$ npm run build
```

và

```bash
$ npm run start
```

Các lệnh này sử dụng khả năng chạy script của npm để thực thi `nest build` hoặc `nest start` sử dụng binary `nest` được cài đặt **cục bộ**. Bằng cách sử dụng các scripts package tích hợp sẵn này, bạn có quản lý phụ thuộc đầy đủ đối với các lệnh Nest CLI\*. Điều này có nghĩa là, bằng cách tuân theo việc sử dụng này **được khuyến nghị**, tất cả các thành viên trong tổ chức của bạn có thể được đảm bảo đang chạy cùng phiên bản của các lệnh.

\*Điều này áp dụng cho các lệnh `build` và `start`. Các lệnh `nest new` và `nest generate` không phải là một phần của pipeline build/thực thi, vì vậy chúng hoạt động trong một ngữ cảnh khác, và không đi kèm với các scripts `package.json` tích hợp sẵn.

Đối với hầu hết các nhà phát triển/nhóm, được khuyến nghị sử dụng các scripts package để xây dựng và thực thi các dự án Nest của họ. Bạn có thể tùy chỉnh đầy đủ hành vi của các scripts này thông qua các tùy chọn của chúng (`--path`, `--webpack`, `--webpackPath`) và/hoặc tùy chỉnh các file tùy chọn trình biên dịch `tsc` hoặc webpack (ví dụ, `tsconfig.json`) nếu cần. Bạn cũng được tự do chạy một quy trình build hoàn toàn tùy chỉnh để biên dịch TypeScript (hoặc thậm chí thực thi TypeScript trực tiếp với `ts-node`).

#### Tương thích ngược

Vì các ứng dụng Nest là các ứng dụng TypeScript thuần túy, các phiên bản trước của scripts build/thực thi Nest sẽ tiếp tục hoạt động. Bạn không được yêu cầu nâng cấp chúng. Bạn có thể chọn tận dụng các lệnh `nest build` và `nest start` mới khi bạn đã sẵn sàng, hoặc tiếp tục chạy các scripts trước đó hoặc tùy chỉnh.

#### Di chuyển

Mặc dù bạn không được yêu cầu thực hiện bất kỳ thay đổi nào, bạn có thể muốn di chuyển sang việc sử dụng các lệnh CLI mới thay vì sử dụng các công cụ như `tsc-watch` hoặc `ts-node`. Trong trường hợp này, chỉ cần cài đặt phiên bản mới nhất của `@nestjs/cli`, cả toàn cục và cục bộ:

```bash
$ npm install -g @nestjs/cli
$ cd  /some/project/root/folder
$ npm install -D @nestjs/cli
```

Sau đó bạn có thể thay thế các `scripts` được định nghĩa trong `package.json` bằng các script sau:

```typescript
"build": "nest build",
"start": "nest start",
"start:dev": "nest start --watch",
"start:debug": "nest start --debug --watch",
```