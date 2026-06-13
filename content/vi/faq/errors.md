### Các lỗi phổ biến

Trong quá trình phát triển với NestJS, bạn có thể gặp phải các lỗi khác nhau khi bạn học framework.

#### Lỗi "Cannot resolve dependency"

> info **Gợi ý** Hãy xem [NestJS Devtools](/devtools/overview#investigating-the-cannot-resolve-dependency-error) có thể giúp bạn giải quyết lỗi "Cannot resolve dependency" một cách dễ dàng.

Có lẽ thông báo lỗi phổ biến nhất là về việc Nest không thể giải quyết các phụ thuộc của một provider. Thông báo lỗi thường trông giống như sau:

```bash
Nest can't resolve dependencies of the <provider> (?). Please make sure that the argument <unknown_token> at index [<index>] is available in the <module> context.

Potential solutions:
- Is <module> a valid NestJS module?
- If <unknown_token> is a provider, is it part of the current <module>?
- If <unknown_token> is exported from a separate @Module, is that module imported within <module>?
  @Module({
    imports: [ /* the Module containing <unknown_token> */ ]
  })
```

Nguyên nhân phổ biến nhất của lỗi, là không có `<provider>` trong mảng `providers` của module. Hãy đảm bảo rằng provider thực sự nằm trong mảng `providers` và tuân theo [thực tiễn provider tiêu chuẩn của NestJS](/fundamentals/custom-providers#di-fundamentals).

Có một vài cạm bẫy, khá phổ biến. Một là đặt một provider trong mảng `imports`. Nếu đây là trường hợp, lỗi sẽ có tên của provider nơi `<module>` nên có.

Nếu bạn gặp phải lỗi này trong khi phát triển, hãy xem module được đề cập trong thông báo lỗi và xem các `providers` của nó. Đối với mỗi provider trong mảng `providers`, hãy đảm bảo module có quyền truy cập vào tất cả các phụ thuộc. Thường thì, `providers` bị trùng lặp trong một "Feature Module" và một "Root Module" có nghĩa là Nest sẽ cố gắng khởi tạo provider hai lần. Rất có thể, module chứa `<provider>` bị trùng lặp nên được thêm vào mảng `imports` của "Root Module" thay thế.

Nếu `<unknown_token>` ở trên là `dependency`, bạn có thể có một import file vòng tròn. Điều này khác với [phụ thuộc vòng tròn](/faq/common-errors#circular-dependency-error) dưới đây vì thay vì có các provider phụ thuộc vào nhau trong các constructor của chúng, nó chỉ có nghĩa là hai file kết thúc việc nhập lẫn nhau. Một trường hợp phổ biến sẽ là một file module khai báo một token và nhập một provider, và provider nhập hằng số token từ file module. Nếu bạn đang sử dụng các file barrel, hãy đảm bảo rằng các import barrel của bạn cũng không tạo ra các import vòng tròn này.

Nếu `<unknown_token>` ở trên là `Object`, điều này có nghĩa là bạn đang inject sử dụng một kiểu/interface mà không có token provider thích hợp. Để khắc phục điều đó, hãy đảm bảo rằng:

1. bạn đang nhập tham chiếu class hoặc sử dụng một token tùy chỉnh với decorator `@Inject()`. Đọc [trang custom providers](/fundamentals/custom-providers), và
2. đối với các provider dựa trên class, bạn đang nhập các class cụ thể thay vì chỉ kiểu thông qua cú pháp [`import type ...`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-8.html#type-only-imports-and-export).

Ngoài ra, hãy đảm bảo bạn không kết thúc việc inject provider vào chính nó vì tự inject không được phép trong NestJS. Khi điều này xảy ra, `<unknown_token>` có thể sẽ bằng với `<provider>`.

<app-banner-devtools></app-banner-devtools>

Nếu bạn đang trong thiết lập **monorepo**, bạn có thể gặp cùng lỗi như trên nhưng cho provider cốt lõi gọi là `ModuleRef` như một `<unknown_token>`:

```bash
Nest can't resolve dependencies of the <provider> (?).
Please make sure that the argument ModuleRef at index [<index>] is available in the <module> context.
...
```

Điều này có thể xảy ra khi dự án của bạn kết thúc việc tải hai module Node của gói `@nestjs/core`, như sau:

```text
.
├── package.json
├── apps
│   └── api
│       └── node_modules
│           └── @nestjs/bull
│               └── node_modules
│                   └── @nestjs/core
└── node_modules
    ├── (other packages)
    └── @nestjs/core
```

Giải pháp:

- Đối với **Yarn** Workspaces, sử dụng [tính năng nohoist](https://classic.yarnpkg.com/blog/2018/02/15/nohoist) để ngăn việc nâng gói `@nestjs/core`.
- Đối với **pnpm** Workspaces, đặt `@nestjs/core` như một peerDependencies trong module khác của bạn và `"dependenciesMeta": { "other-module-name": { "injected": true } }` trong package.json của ứng dụng nơi module được nhập. xem: [dependenciesmetainjected](https://pnpm.io/package_json#dependenciesmetainjected)

#### Lỗi "Circular dependency"

Thỉnh thoảng bạn sẽ thấy khó tránh [phụ thuộc vòng tròn](https://docs.nestjs.com/fundamentals/circular-dependency) trong ứng dụng của mình. Bạn sẽ cần thực hiện một số bước để giúp Nest giải quyết các vấn đề này. Các lỗi phát sinh từ các phụ thuộc vòng tròn trông như sau:

```bash
Nest cannot create the <module> instance.
The module at index [<index>] of the <module> "imports" array is undefined.

Potential causes:
- A circular dependency between modules. Use forwardRef() to avoid it. Read more: https://docs.nestjs.com/fundamentals/circular-dependency
- The module at index [<index>] is of type "undefined". Check your import statements and the type of the module.

Scope [<module_import_chain>]
# example chain AppModule -> FooModule
```

Các phụ thuộc vòng tròn có thể phát sinh từ cả các provider phụ thuộc vào nhau, hoặc các file typescript phụ thuộc vào nhau cho các hằng số, chẳng hạn như xuất các hằng số từ một file module và nhập chúng trong một file service. Trong trường hợp sau, được khuyến nghị tạo một file riêng biệt cho các hằng số của bạn. Trong trường hợp trước, hãy theo hướng dẫn về các phụ thuộc vòng tròn và đảm bảo rằng cả các module **và** các provider đều được đánh dấu với `forwardRef`.

#### Gỡ lỗi các lỗi phụ thuộc

Ngoài việc chỉ xác minh thủ công các phụ thuộc của bạn là đúng, kể từ Nest 8.1.0 bạn có thể đặt biến môi trường `NEST_DEBUG` thành một chuỗi giải quyết là đúng, và nhận thêm thông tin ghi nhật ký trong khi Nest giải quyết tất cả các phụ thuộc cho ứng dụng.

<figure><img src="/assets/injector_logs.png" /></figure>

Trong hình trên, chuỗi màu vàng là class chủ của phụ thuộc được inject, chuỗi màu xanh là tên của phụ thuộc được inject, hoặc token inject của nó, và chuỗi màu tím là module trong đó phụ thuộc đang được tìm kiếm. Sử dụng điều này, bạn thường có thể truy ngược lại việc giải quyết phụ thuộc cho những gì đang xảy ra và tại sao bạn gặp các vấn đề về dependency injection.

#### Vòng lặp vô tận "File change detected"

Người dùng Windows đang sử dụng phiên bản TypeScript 4.9 trở lên có thể gặp phải vấn đề này.
Điều này xảy ra khi bạn đang cố gắng chạy ứng dụng của mình ở chế độ watch, ví dụ `npm run start:dev` và thấy một vòng lặp vô tận của các thông báo nhật ký:

```bash
XX:XX:XX AM - File change detected. Starting incremental compilation...
XX:XX:XX AM - Found 0 errors. Watching for file changes.
```

Khi bạn đang sử dụng NestJS CLI để khởi động ứng dụng của bạn ở chế độ watch, nó được thực hiện bằng cách gọi `tsc --watch`, và kể từ phiên bản 4.9 của TypeScript, một [chiến lược mới](https://devblogs.microsoft.com/typescript/announcing-typescript-4-9/#file-watching-now-uses-file-system-events) để phát hiện các thay đổi file được sử dụng có thể là nguyên nhân của vấn đề này.
Để khắc phục vấn đề này, bạn cần thêm một cài đặt vào file tsconfig.json của bạn sau tùy chọn `"compilerOptions"` như sau:

```bash
  "watchOptions": {
    "watchFile": "fixedPollingInterval"
  }
```

Điều này nói với TypeScript sử dụng phương thức polling để kiểm tra các thay đổi file thay vì các sự kiện hệ thống file (phương thức mặc định mới), có thể gây ra vấn đề trên một số máy.
Bạn có thể đọc thêm về tùy chọn `"watchFile"` trong [tài liệu TypeScript](https://www.typescriptlang.org/tsconfig#watch-watchDirectory).