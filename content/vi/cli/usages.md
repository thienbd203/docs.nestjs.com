### Tham khảo lệnh CLI

#### nest new

Tạo một dự án Nest mới (chế độ tiêu chuẩn).

```bash
$ nest new <name> [options]
$ nest n <name> [options]
```

##### Mô tả

Tạo và khởi tạo một dự án Nest mới. Nhắc nhập package manager.

- Tạo một thư mục với `<name>` đã cho
- Điền thư mục với các file cấu hình
- Tạo các thư mục con cho mã nguồn (`/src`) và kiểm tra end-to-end (`/test`)
- Điền các thư mục con với các file mặc định cho các thành phần ứng dụng và kiểm tra

##### Đối số

|| Đối số | Mô tả                 |
|| -------- | --------------------------- |
|| `<name>` | Tên của dự án mới |

##### Tùy chọn

|| Tùy chọn                                | Mô tả                                                                                                                                                                                          |
|| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|| `--dry-run`                           | Báo cáo các thay đổi sẽ được thực hiện, nhưng không thay đổi hệ thống file.<br/> Alias: `-d`                                                                                                             |
|| `--skip-git`                          | Bỏ qua khởi tạo repository git.<br/> Alias: `-g`                                                                                                                                                 |
|| `--skip-install`                      | Bỏ qua cài đặt package.<br/> Alias: `-s`                                                                                                                                                          |
|| `--package-manager [package-manager]` | Chỉ định package manager. Sử dụng `npm`, `yarn`, hoặc `pnpm`. Package manager phải được cài đặt toàn cục.<br/> Alias: `-p`                                                                                  |
|| `--language [language]`               | Chỉ định ngôn ngữ lập trình (`TS` hoặc `JS`).<br/> Alias: `-l`                                                                                                                                        |
|| `--collection [collectionName]`       | Chỉ định bộ sưu tập schematics. Sử dụng tên gói của gói npm đã cài đặt chứa schematic.<br/> Alias: `-c`                                                                                      |
|| `--strict`                            | Bắt đầu dự án với các cờ trình biên dịch TypeScript sau được bật: `strictNullChecks`, `noImplicitAny`, `strictBindCallApply`, `forceConsistentCasingInFileNames`, `noFallthroughCasesInSwitch` |

#### nest generate

Tạo và/hoặc sửa đổi các file dựa trên một schematic

```bash
$ nest generate <schematic> <name> [options]
$ nest g <schematic> <name> [options]
```

##### Đối số

|| Đối số      | Mô tả                                                                                              |
|| ------------- | -------------------------------------------------------------------------------------------------------- |
|| `<schematic>` | `schematic` hoặc `collection:schematic` để tạo. Xem bảng dưới đây cho các schematics có sẵn. |
|| `<name>`      | Tên của thành phần được tạo.                                                                     |

##### Schematics

|| Tên          | Alias | Mô tả                                                                                                            |
|| ------------- | ----- | ---------------------------------------------------------------------------------------------------------------------- |
|| `app`         |       | Tạo một ứng dụng mới trong một monorepo (chuyển sang monorepo nếu là cấu trúc tiêu chuẩn).                    |
|| `library`     | `lib` | Tạo một thư viện mới trong một monorepo (chuyển sang monorepo nếu là cấu trúc tiêu chuẩn).                        |
|| `class`       | `cl`  | Tạo một class mới.                                                                                                  |
|| `controller`  | `co`  | Tạo một khai báo controller.                                                                                     |
|| `decorator`   | `d`   | Tạo một decorator tùy chỉnh.                                                                                           |
|| `filter`      | `f`   | Tạo một khai báo filter.                                                                                         |
|| `gateway`     | `ga`  | Tạo một khai báo gateway.                                                                                        |
|| `guard`       | `gu`  | Tạo một khai báo guard.                                                                                          |
|| `interface`   | `itf` | Tạo một interface.                                                                                                 |
|| `interceptor` | `itc` | Tạo một khai báo interceptor.                                                                                   |
|| `middleware`  | `mi`  | Tạo một khai báo middleware.                                                                                     |
|| `module`      | `mo`  | Tạo một khai báo module.                                                                                         |
|| `pipe`        | `pi`  | Tạo một khai báo pipe.                                                                                           |
|| `provider`    | `pr`  | Tạo một khai báo provider.                                                                                       |
|| `resolver`    | `r`   | Tạo một khai báo resolver.                                                                                       |
|| `resource`    | `res` | Tạo một tài nguyên CRUD mới. Xem [CRUD (resource) generator](/recipes/crud-generator) để biết thêm chi tiết. (chỉ TS) |
|| `service`     | `s`   | Tạo một khai báo service.                                                                                        |

##### Tùy chọn

|| Tùy chọn                          | Mô tả                                                                                                     |
|| ------------------------------- | --------------------------------------------------------------------------------------------------------------- |
|| `--dry-run`                     | Báo cáo các thay đổi sẽ được thực hiện, nhưng không thay đổi hệ thống file.<br/> Alias: `-d`                        |
|| `--project [project]`           | Dự án mà thành phần nên được thêm vào.<br/> Alias: `-p`                                                       |
|| `--flat`                        | Không tạo thư mục cho thành phần.                                                                       |
|| `--collection [collectionName]` | Chỉ định bộ sưu tập schematics. Sử dụng tên gói của gói npm đã cài đặt chứa schematic.<br/> Alias: `-c` |
|| `--spec`                        | Thực thi tạo file spec (mặc định)                                                                         |
|| `--no-spec`                     | Vô hiệu hóa tạo file spec                                                                                   |

#### nest build

Biên dịch một ứng dụng hoặc workspace vào một thư mục đầu ra.

Ngoài ra, lệnh `build` chịu trách nhiệm cho:

- ánh xạ các đường dẫn (nếu sử dụng đường dẫn alias) thông qua `tsconfig-paths`
- chú thích DTOs với decorators OpenAPI (nếu plugin CLI `@nestjs/swagger` được bật)
- chú thích DTOs với decorators GraphQL (nếu plugin CLI `@nestjs/graphql` được bật)

```bash
$ nest build <name> [options]
```

##### Đối số

|| Đối số | Mô tả                       |
|| -------- | --------------------------------- |
|| `<name>` | Tên của dự án để build. |

##### Tùy chọn

|| Tùy chọn                  | Mô tả                                                                                                                                                                                |
|| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
|| `--path [path]`         | Đường dẫn đến file `tsconfig`. <br/>Alias `-p`                                                                                                                                                   |
|| `--config [path]`       | Đường dẫn đến file cấu hình `nest-cli`. <br/>Alias `-c`                                                                                                                                     |
|| `--watch`               | Chạy ở chế độ watch (tải lại trực tiếp).<br /> Nếu bạn đang sử dụng `tsc` để biên dịch, bạn có thể nhập `rs` để khởi động lại ứng dụng (khi tùy chọn `manualRestart` được đặt thành `true`). <br/>Alias `-w` |
|| `--builder [name]`      | Chỉ định builder để sử dụng để biên dịch (`tsc`, `swc`, hoặc `webpack`). <br/>Alias `-b`                                                                                                   |
|| `--webpack`             | Sử dụng webpack để biên dịch (đã không dùng nữa: sử dụng `--builder webpack` thay thế).                                                                                                                 |
|| `--webpackPath`         | Đường dẫn đến cấu hình webpack.                                                                                                                                                             |
|| `--tsc`                 | Buộc sử dụng `tsc` để biên dịch.                                                                                                                                                           |
|| `--watchAssets`         | Watch các file không phải TS (tài sản như `.graphql` v.v.). Xem [Tài sản](cli/monorepo#assets) để biết thêm chi tiết.                                                                                      |
|| `--type-check`          | Bật kiểm tra kiểu (khi SWC được sử dụng).                                                                                                                                                   |
|| `--all`                 | Build tất cả các dự án trong một monorepo.                                                                                                                                                          |
|| `--preserveWatchOutput` | Giữ lại đầu ra console cũ trong chế độ watch thay vì xóa màn hình. (chỉ chế độ watch `tsc`)                                                                                         |

#### nest start

Biên dịch và chạy một ứng dụng (hoặc dự án mặc định trong một workspace).

```bash
$ nest start <name> [options]
```

##### Đối số

|| Đối số | Mô tả                     |
|| -------- | ------------------------------- |
|| `<name>` | Tên của dự án để chạy. |

##### Tùy chọn

|| Tùy chọn                  | Mô tả                                                                                                                        |
|| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
|| `--path [path]`         | Đường dẫn đến file `tsconfig`. <br/>Alias `-p`                                                                                           |
|| `--config [path]`       | Đường dẫn đến file cấu hình `nest-cli`. <br/>Alias `-c`                                                                             |
|| `--watch`               | Chạy ở chế độ watch (tải lại trực tiếp) <br/>Alias `-w`                                                                                    |
|| `--builder [name]`      | Chỉ định builder để sử dụng để biên dịch (`tsc`, `swc`, hoặc `webpack`). <br/>Alias `-b`                                           |
|| `--preserveWatchOutput` | Giữ lại đầu ra console cũ trong chế độ watch thay vì xóa màn hình. (chỉ chế độ watch `tsc`)                                 |
|| `--watchAssets`         | Chạy ở chế độ watch (tải lại trực tiếp), watch các file không phải TS (tài sản). Xem [Tài sản](cli/monorepo#assets) để biết thêm chi tiết.               |
|| `--debug [hostport]`    | Chạy ở chế độ debug (với cờ --inspect) <br/>Alias `-d`                                                                            |
|| `--webpack`             | Sử dụng webpack để biên dịch. (đã không dùng nữa: sử dụng `--builder webpack` thay thế)                                                         |
|| `--webpackPath`         | Đường dẫn đến cấu hình webpack.                                                                                                     |
|| `--tsc`                 | Buộc sử dụng `tsc` để biên dịch.                                                                                                   |
|| `--exec [binary]`       | Binary để chạy (mặc định: `node`). <br/>Alias `-e`                                                                                   |
|| `--no-shell`            | Không tạo các process con trong một shell (xem tài liệu phương thức `child_process.spawn()` của node).                                      |
|| `--env-file`            | Tải các biến môi trường từ một file tương đối với thư mục hiện tại, làm cho chúng có sẵn cho các ứng dụng trên `process.env`. |
|| `-- [key=value]`        | Đối số dòng lệnh có thể được tham chiếu với `process.argv`.                                                                 |

#### nest add

Nhập một thư viện đã được đóng gói như một **nest library**, chạy schematic cài đặt của nó.

```bash
$ nest add <name> [options]
```

##### Đối số

|| Đối số | Mô tả                        |
|| -------- | ---------------------------------- |
|| `<name>` | Tên của thư viện để nhập. |

#### nest info

Hiển thị thông tin về các gói nest đã cài đặt và thông tin hệ thống hữu ích khác. Ví dụ:

```bash
$ nest info
```

```bash
 _   _             _      ___  _____  _____  _     _____
| \ | |           | |    |_  |/  ___|/  __ \| |   |_   _|
|  \| |  ___  ___ | |_     | |\ `--. | /  \/| |     | |
| . ` | / _ \/ __|| __|    | | `--. \| |    | |     | |
| |\  ||  __/\__ \| |_ /\__/ //\__/ /| \__/\| |_____| |_
\_| \_/ \___||___/ \__|\____/ \____/  \____/\_____/\___/

[System Information]
OS Version : macOS High Sierra
NodeJS Version : v20.18.0
[Nest Information]
microservices version : 10.0.0
websockets version : 10.0.0
testing version : 10.0.0
common version : 10.0.0
core version : 10.0.0
```