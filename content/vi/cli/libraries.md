### Thư viện

Nhiều ứng dụng cần giải quyết các vấn đề chung giống nhau, hoặc tái sử dụng một thành phần mô-đun trong nhiều ngữ cảnh khác nhau. Nest có một vài cách để giải quyết vấn đề này, nhưng mỗi cách hoạt động ở một mức độ khác nhau để giải quyết vấn đề theo cách giúp đáp ứng các mục tiêu kiến trúc và tổ chức khác nhau.

Các [module](/modules) của Nest hữu ích để cung cấp ngữ cảnh thực thi cho phép chia sẻ các thành phần trong một ứng dụng đơn lẻ. Các module cũng có thể được đóng gói với [npm](https://npmjs.com) để tạo một thư viện có thể tái sử dụng có thể được cài đặt trong các dự án khác nhau. Điều này có thể là một cách hiệu quả để phân phối các thư viện có thể cấu hình, có thể tái sử dụng có thể được sử dụng bởi các tổ chức khác nhau, được kết nối lỏng lẻo hoặc không liên quan (ví dụ, bằng cách phân phối/cài đặt các thư viện bên thứ ba).

Để chia sẻ mã trong các nhóm được tổ chức chặt chẽ (ví dụ, trong giới hạn công ty/dự án), có thể hữu ích khi có một cách tiếp cận nhẹ hơn để chia sẻ các thành phần. Monorepos đã xuất hiện như một cấu trúc để cho phép điều đó, và trong một monorepo, một **thư viện** cung cấp một cách để chia sẻ mã theo cách dễ dàng, nhẹ nhàng. Trong một monorepo Nest, sử dụng thư viện cho phép dễ dàng lắp ráp các ứng dụng chia sẻ các thành phần. Trên thực tế, điều này khuyến khích phân tách các ứng dụng monolithic và các quy trình phát triển để tập trung vào việc xây dựng và kết hợp các thành phần mô-đun.

#### Thư viện Nest

Một thư viện Nest là một dự án Nest khác với một ứng dụng ở chỗ nó không thể chạy độc lập. Một thư viện phải được nhập vào một ứng dụng chứa để mã của nó có thể thực thi. Hỗ trợ tích hợp sẵn cho thư viện được mô tả trong phần này chỉ có sẵn cho **monorepos** (các dự án chế độ tiêu chuẩn có thể đạt được chức năng tương tự bằng cách sử dụng các gói npm).

Ví dụ, một tổ chức có thể phát triển một `AuthModule` quản lý xác thực bằng cách thực thi các chính sách công ty chi phối tất cả các ứng dụng nội bộ. Thay vì xây dựng module đó riêng biệt cho từng ứng dụng, hoặc đóng gói vật lý mã với npm và yêu cầu mỗi dự án cài đặt nó, một monorepo có thể định nghĩa module này như một thư viện. Khi được tổ chức theo cách này, tất cả người tiêu dùng của module thư viện có thể thấy phiên bản cập nhật nhất của `AuthModule` khi nó được commit. Điều này có thể mang lại lợi ích đáng kể cho việc điều phối phát triển và lắp ráp thành phần, và đơn giản hóa kiểm tra end-to-end.

#### Tạo thư viện

Bất kỳ chức năng nào phù hợp để tái sử dụng đều là ứng viên để được quản lý như một thư viện. Quyết định những gì nên là một thư viện, và những gì nên là một phần của ứng dụng, là một quyết định thiết kế kiến trúc. Tạo thư viện liên quan đến nhiều hơn là chỉ sao chép mã từ một ứng dụng hiện có sang một thư viện mới. Khi được đóng gói như một thư viện, mã thư viện phải được tách rời khỏi ứng dụng. Điều này có thể yêu cầu **nhiều** thời gian hơn ban đầu và buộc một số quyết định thiết kế mà bạn có thể không gặp với mã được kết nối chặt chẽ hơn. Nhưng nỗ lực bổ sung này có thể mang lại lợi ích khi thư viện có thể được sử dụng để cho phép lắp ráp ứng dụng nhanh hơn trên nhiều ứng dụng.

Để bắt đầu tạo thư viện, hãy chạy lệnh sau:

```bash
$ nest g library my-library
```

Khi bạn chạy lệnh, schematic `library` sẽ nhắc bạn nhập một prefix (còn gọi là alias) cho thư viện:

```bash
What prefix would you like to use for the library (default: @app)?
```

Điều này tạo ra một dự án mới trong workspace của bạn gọi là `my-library`.
Một dự án kiểu thư viện, giống như dự án kiểu ứng dụng, được tạo ra trong một thư mục được đặt tên bằng một schematic. Các thư viện được quản lý trong thư mục `libs` của gốc monorepo. Nest tạo thư mục `libs` lần đầu tiên khi một thư viện được tạo.

Các file được tạo cho một thư viện hơi khác với các file được tạo cho một ứng dụng. Dưới đây là nội dung của thư mục `libs` sau khi thực thi lệnh trên:

<div class="file-tree">
  <div class="item">libs</div>
  <div class="children">
    <div class="item">my-library</div>
    <div class="children">
      <div class="item">src</div>
      <div class="children">
        <div class="item">index.ts</div>
        <div class="item">my-library.module.ts</div>
        <div class="item">my-library.service.ts</div>
      </div>
      <div class="item">tsconfig.lib.json</div>
    </div>
  </div>
</div>

File `nest-cli.json` sẽ có một mục mới cho thư viện dưới khóa `"projects"`:

```javascript
...
{
    "my-library": {
      "type": "library",
      "root": "libs/my-library",
      "entryFile": "index",
      "sourceRoot": "libs/my-library/src",
      "compilerOptions": {
        "tsConfigPath": "libs/my-library/tsconfig.lib.json"
      }
}
...
```

Có hai sự khác biệt trong metadata `nest-cli.json` giữa thư viện và ứng dụng:

- thuộc tính `"type"` được đặt thành `"library"` thay vì `"application"`
- thuộc tính `"entryFile"` được đặt thành `"index"` thay vì `"main"`

Những khác biệt này khóa quá trình build để xử lý thư viện một cách phù hợp. Ví dụ, một thư viện xuất các hàm của nó thông qua file `index.js`.

Giống như các dự án kiểu ứng dụng, mỗi thư viện có file `tsconfig.lib.json` riêng của nó mở rộng file `tsconfig.json` gốc (toàn monorepo). Bạn có thể sửa đổi file này, nếu cần, để cung cấp các tùy chọn trình biên dịch cụ thể cho thư viện.

Bạn có thể build thư viện bằng lệnh CLI:

```bash
$ nest build my-library
```

#### Sử dụng thư viện

Với các file cấu hình được tạo tự động đã có sẵn, sử dụng thư viện là đơn giản. Làm thế nào chúng ta sẽ nhập `MyLibraryService` từ thư viện `my-library` vào ứng dụng `my-project`?

Trước tiên, lưu ý rằng sử dụng các module thư viện giống như sử dụng bất kỳ module Nest nào khác. Những gì monorepo làm là quản lý các đường dẫn theo cách mà nhập thư viện và tạo các bản build giờ đây là minh bạch. Để sử dụng `MyLibraryService`, chúng ta cần nhập module khai báo của nó. Chúng ta có thể sửa đổi `my-project/src/app.module.ts` như sau để nhập `MyLibraryModule`.

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { MyLibraryModule } from '@app/my-library';

@Module({
  imports: [MyLibraryModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Lưu ý ở trên rằng chúng ta đã sử dụng một đường dẫn alias của `@app` trong dòng `import` module ES, đó là `prefix` chúng ta đã cung cấp với lệnh `nest g library` ở trên. Bên dưới, Nest xử lý điều này thông qua ánh xạ đường dẫn tsconfig. Khi thêm một thư viện, Nest cập nhật khóa `"paths"` của file `tsconfig.json` toàn cục (monorepo) như sau:

```javascript
"paths": {
    "@app/my-library": [
        "libs/my-library/src"
    ],
    "@app/my-library/*": [
        "libs/my-library/src/*"
    ]
}
```

Vì vậy, tóm lại, sự kết hợp của các tính năng monorepo và thư viện đã làm cho việc bao gồm các module thư viện vào các ứng dụng trở nên dễ dàng và trực quan.

Cơ chế tương tự này cho phép xây dựng và triển khai các ứng dụng kết hợp thư viện. Khi bạn đã nhập `MyLibraryModule`, chạy `nest build` xử lý tất cả việc giải quyết module tự động và bundle ứng dụng cùng với bất kỳ phụ thuộc thư viện nào, để triển khai. Trình biên dịch mặc định cho một monorepo là **webpack**, vì vậy file phân phối kết quả là một file duy nhất bundle tất cả các file JavaScript đã biên dịch lại thành một file duy nhất. Bạn cũng có thể chuyển sang `tsc` như được mô tả <a href="https://docs.nestjs.com/cli/monorepo#global-compiler-options">ở đây</a>.