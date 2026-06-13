### Workspaces

Nest có hai chế độ để tổ chức mã:

- **chế độ tiêu chuẩn**: hữu ích để xây dựng các ứng dụng tập trung dự án riêng lẻ có các phụ thuộc và cài đặt riêng của chúng, và không cần tối ưu hóa để chia sẻ các module, hoặc tối ưu hóa các bản build phức tạp. Đây là chế độ mặc định.
- **chế độ monorepo**: chế độ này coi các artifact mã là một phần của một **monorepo** nhẹ nhàng, và có thể phù hợp hơn cho các nhóm nhà phát triển và/hoặc môi trường đa dự án. Nó tự động hóa các phần của quy trình build để làm cho việc tạo và kết hợp các thành phần mô-đun trở nên dễ dàng, thúc đẩy tái sử dụng mã, làm cho kiểm tra tích hợp dễ dàng hơn, làm cho việc chia sẻ các artifact cấp dự án như các quy tắc `eslint` và các chính sách cấu hình khác trở nên dễ dàng, và dễ sử dụng hơn các giải pháp thay thế như Git submodules. Chế độ monorepo sử dụng khái niệm về một **workspace**, được đại diện trong file `nest-cli.json`, để điều phối mối quan hệ giữa các thành phần của monorepo.

Điều quan trọng cần lưu ý là hầu như tất cả các tính năng của Nest đều độc lập với chế độ tổ chức mã của bạn. Hiệu ứng **duy nhất** của lựa chọn này là cách các dự án của bạn được kết hợp và cách các artifact build được tạo ra. Tất cả các chức năng khác, từ CLI đến các module cốt lõi đến các module bổ sung hoạt động giống nhau trong cả hai chế độ.

Ngoài ra, bạn có thể dễ dàng chuyển từ **chế độ tiêu chuẩn** sang **chế độ monorepo** bất kỳ lúc nào, vì vậy bạn có thể hoãn quyết định này cho đến khi lợi ích của một trong hai cách tiếp cận trở nên rõ ràng hơn.

#### Chế độ tiêu chuẩn

Khi bạn chạy `nest new`, một **dự án** mới được tạo cho bạn bằng một schematic tích hợp sẵn. Nest thực hiện những điều sau:

1. Tạo một thư mục mới, tương ứng với đối số `name` bạn cung cấp cho `nest new`
2. Điền thư mục đó với các file mặc định tương ứng với một ứng dụng Nest cơ bản mức tối thiểu. Bạn có thể kiểm tra các file này tại repository [typescript-starter](https://github.com/nestjs/typescript-starter).
3. Cung cấp các file bổ sung như `nest-cli.json`, `package.json` và `tsconfig.json` cấu hình và bật các công cụ khác nhau để biên dịch, kiểm tra và serve ứng dụng của bạn.

Từ đó, bạn có thể sửa đổi các file khởi động, thêm các thành phần mới, thêm các phụ thuộc (ví dụ, `npm install`), và phát triển ứng dụng của bạn như được đề cập trong phần còn lại của tài liệu này.

#### Chế độ monorepo

Để bật chế độ monorepo, bạn bắt đầu với một cấu trúc _chế độ tiêu chuẩn_, và thêm các **dự án**. Một dự án có thể là một **ứng dụng** đầy đủ (bạn thêm vào workspace với lệnh `nest generate app`) hoặc một **thư viện** (bạn thêm vào workspace với lệnh `nest generate library`). Chúng tôi sẽ thảo luận chi tiết về các loại thành phần dự án cụ thể này dưới đây. Điểm chính cần lưu ý bây giờ là đó là **hành động thêm một dự án** vào một cấu trúc chế độ tiêu chuẩn hiện có mà **chuyển đổi nó** sang chế độ monorepo. Hãy xem một ví dụ.

Nếu chúng ta chạy:

```bash
$ nest new my-project
```

Chúng ta đã xây dựng một cấu trúc _chế độ tiêu chuẩn_, với cấu trúc thư mục trông như sau:

<div class="file-tree">
  <div class="item">node_modules</div>
  <div class="item">src</div>
  <div class="children">
    <div class="item">app.controller.ts</div>
    <div class="item">app.module.ts</div>
    <div class="item">app.service.ts</div>
    <div class="item">main.ts</div>
  </div>
  <div class="item">nest-cli.json</div>
  <div class="item">package.json</div>
  <div class="item">tsconfig.json</div>
  <div class="item">eslint.config.mjs</div>
</div>

Chúng ta có thể chuyển đổi này sang một cấu trúc chế độ monorepo như sau:

```bash
$ cd my-project
$ nest generate app my-app
```

Tại thời điểm này, `nest` chuyển đổi cấu trúc hiện có sang một cấu trúc **chế độ monorepo**. Điều này dẫn đến một vài thay đổi quan trọng. Cấu trúc thư mục bây giờ trông như sau:

<div class="file-tree">
  <div class="item">apps</div>
    <div class="children">
      <div class="item">my-app</div>
      <div class="children">
        <div class="item">src</div>
        <div class="children">
          <div class="item">app.controller.ts</div>
          <div class="item">app.module.ts</div>
          <div class="item">app.service.ts</div>
          <div class="item">main.ts</div>
        </div>
        <div class="item">tsconfig.app.json</div>
      </div>
      <div class="item">my-project</div>
      <div class="children">
        <div class="item">src</div>
        <div class="children">
          <div class="item">app.controller.ts</div>
          <div class="item">app.module.ts</div>
          <div class="item">app.service.ts</div>
          <div class="item">main.ts</div>
        </div>
        <div class="item">tsconfig.app.json</div>
      </div>
    </div>
  <div class="item">nest-cli.json</div>
  <div class="item">package.json</div>
  <div class="item">tsconfig.json</div>
  <div class="item">eslint.config.mjs</div>
</div>

Schematic `generate app` đã tổ chức lại mã - di chuyển mỗi dự án **ứng dụng** dưới thư mục `apps`, và thêm một file `tsconfig.app.json` cụ thể cho dự án trong thư mục gốc của mỗi dự án. Ứng dụng `my-project` gốc của chúng ta đã trở thành **dự án mặc định** cho monorepo, và bây giờ là một peer với `my-app` vừa được thêm, nằm dưới thư mục `apps`. Chúng tôi sẽ đề cập đến các dự án mặc định dưới đây.

> error **Cảnh báo** Việc chuyển đổi một cấu trúc chế độ tiêu chuẩn sang monorepo chỉ hoạt động cho các dự án đã tuân theo cấu trúc dự án Nest chuẩn. Cụ thể, trong quá trình chuyển đổi, schematic cố gắng di chuyển các thư mục `src` và `test` trong một thư mục dự án dưới thư mục `apps` trong gốc. Nếu một dự án không sử dụng cấu trúc này, việc chuyển đổi sẽ thất bại hoặc tạo ra kết quả không đáng tin cậy.

#### Dự án workspace

Một monorepo sử dụng khái niệm về một workspace để quản lý các thực thể thành viên của nó. Workspaces được bao gồm bởi **dự án**. Một dự án có thể là:

- một **ứng dụng**: một ứng dụng Nest đầy đủ bao gồm một file `main.ts` để bootstrap ứng dụng. Ngoài các cân nhắc biên dịch và build, một dự án kiểu ứng dụng trong một workspace chức năng giống hệt với một ứng dụng trong một cấu trúc _chế độ tiêu chuẩn_.
- một **thư viện**: một thư viện là một cách đóng gói một tập hợp tính năng mục đích chung (module, provider, controller, v.v.) có thể được sử dụng trong các dự án khác. Một thư viện không thể chạy độc lập, và không có file `main.ts`. Đọc thêm về thư viện [ở đây](/cli/libraries).

Tất cả các workspace đều có một **dự án mặc định** (nên là một dự án kiểu ứng dụng). Điều này được định nghĩa bởi thuộc tính `"root"` cấp cao nhất trong file `nest-cli.json`, trỏ đến gốc của dự án mặc định (xem [Thuộc tính CLI](/cli/monorepo#cli-properties) dưới đây để biết thêm chi tiết). Thường thì đây là ứng dụng **chế độ tiêu chuẩn** mà bạn bắt đầu, và sau đó chuyển đổi sang monorepo bằng `nest generate app`. Khi bạn làm theo các bước này, thuộc tính này được điền tự động.

Các dự án mặc định được sử dụng bởi các lệnh `nest` như `nest build` và `nest start` khi không có tên dự án được cung cấp.

Ví dụ, trong cấu trúc monorepo ở trên, chạy

```bash
$ nest start
```

sẽ khởi động ứng dụng `my-project`. Để khởi động `my-app`, chúng ta sẽ sử dụng:

```bash
$ nest start my-app
```

#### Ứng dụng

Các dự án kiểu ứng dụng, hoặc những gì chúng ta có thể gọi không chính thức là chỉ "ứng dụng", là các ứng dụng Nest hoàn chỉnh mà bạn có thể chạy và triển khai. Bạn tạo một dự án kiểu ứng dụng với `nest generate app`.

Lệnh này tự động tạo một khung dự án, bao gồm các thư mục `src` và `test` tiêu chuẩn từ [typescript starter](https://github.com/nestjs/typescript-starter). Không giống như chế độ tiêu chuẩn, một dự án ứng dụng trong một monorepo không có bất kỳ phụ thuộc package (`package.json`) hoặc các artifact cấu hình dự án khác như `.prettierrc` và `eslint.config.mjs`. Thay vào đó, các phụ thuộc và file cấu hình cấp monorepo được sử dụng.

Tuy nhiên, schematic tạo ra một file `tsconfig.app.json` cụ thể cho dự án trong thư mục gốc của dự án. File cấu hình này tự động đặt các tùy chọn build phù hợp, bao gồm đặt thư mục đầu ra biên dịch đúng cách. File mở rộng file `tsconfig.json` cấp cao nhất (monorepo), vì vậy bạn có thể quản lý các cài đặt toàn cầu cấp monorepo, nhưng ghi đè chúng nếu cần ở cấp dự án.

#### Thư viện

Như đã đề cập, các dự án kiểu thư viện, hoặc đơn giản là "thư viện", là các gói các thành phần Nest cần được kết hợp vào các ứng dụng để chạy. Bạn tạo một dự án kiểu thư viện với `nest generate library`. Quyết định những gì thuộc về một thư viện là một quyết định thiết kế kiến trúc. Chúng tôi thảo luận về thư viện sâu hơn trong chương [thư viện](/cli/libraries).

#### Thuộc tính CLI

Nest giữ metadata cần thiết để tổ chức, build và triển khai cả các dự án cấu trúc tiêu chuẩn và monorepo trong file `nest-cli.json`. Nest tự động thêm và cập nhật file này khi bạn thêm dự án, vì vậy bạn thường không cần phải nghĩ về nó hoặc chỉnh sửa nội dung của nó. Tuy nhiên, có một số cài đặt bạn có thể muốn thay đổi thủ công, vì vậy có ích khi có sự hiểu biết tổng quan về file.

Sau khi chạy các bước trên để tạo một monorepo, file `nest-cli.json` của chúng ta trông như sau:

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "apps/my-project/src",
  "monorepo": true,
  "root": "apps/my-project",
  "compilerOptions": {
    "webpack": true,
    "tsConfigPath": "apps/my-project/tsconfig.app.json"
  },
  "projects": {
    "my-project": {
      "type": "application",
      "root": "apps/my-project",
      "entryFile": "main",
      "sourceRoot": "apps/my-project/src",
      "compilerOptions": {
        "tsConfigPath": "apps/my-project/tsconfig.app.json"
      }
    },
    "my-app": {
      "type": "application",
      "root": "apps/my-app",
      "entryFile": "main",
      "sourceRoot": "apps/my-app/src",
      "compilerOptions": {
        "tsConfigPath": "apps/my-app/tsconfig.app.json"
      }
    }
  }
}
```

File được chia thành các phần:

- một phần toàn cầu với các thuộc tính cấp cao nhất kiểm soát các cài đặt tiêu chuẩn và cấp monorepo
- một thuộc tính cấp cao nhất (`"projects"`) với metadata về mỗi dự án. Phần này chỉ có cho các cấu trúc chế độ monorepo.

Các thuộc tính cấp cao nhất như sau:

- `"collection"`: trỏ đến bộ sưu tập các schematics được sử dụng để tạo các thành phần; bạn thường không nên thay đổi giá trị này
- `"sourceRoot"`: trỏ đến gốc của mã nguồn cho dự án đơn lẻ trong các cấu trúc chế độ tiêu chuẩn, hoặc _dự án mặc định_ trong các cấu trúc chế độ monorepo
- `"compilerOptions"`: một bản đồ với các khóa chỉ định các tùy chọn trình biên dịch và các giá trị chỉ định cài đặt tùy chọn; xem chi tiết dưới đây
- `"generateOptions"`: một bản đồ với các khóa chỉ định các tùy chọn tạo toàn cầu và các giá trị chỉ định cài đặt tùy chọn; xem chi tiết dưới đây
- `"monorepo"`: (chỉ monorepo) cho một cấu trúc chế độ monorepo, giá trị này luôn là `true`
- `"root"`: (chỉ monorepo) trỏ đến gốc dự án của _dự án mặc định_

#### Tùy chọn trình biên dịch toàn cầu

Các thuộc tính này chỉ định trình biên dịch để sử dụng cũng như các tùy chọn khác nhau ảnh hưởng đến **bất kỳ** bước biên dịch nào, dù là một phần của `nest build` hay `nest start`, và bất kể trình biên dịch, dù là `tsc` hay webpack.

|| Tên Thuộc tính       | Kiểu Giá Trị Thuộc Tính | Mô tả                                                                                                                                                                                                                                                               |
|| ------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|| `webpack`           | boolean             | Nếu `true`, sử dụng [trình biên dịch webpack](https://webpack.js.org/). Nếu `false` hoặc không có, sử dụng `tsc`. Trong chế độ monorepo, mặc định là `true` (sử dụng webpack), trong chế độ tiêu chuẩn, mặc định là `false` (sử dụng `tsc`). Xem chi tiết dưới đây. (đã không dùng nữa: sử dụng `builder` thay thế) |
|| `tsConfigPath`      | string              | (**chỉ monorepo**) Trỏ đến file chứa các cài đặt `tsconfig.json` sẽ được sử dụng khi `nest build` hoặc `nest start` được gọi mà không có tùy chọn `project` (ví dụ, khi dự án mặc định được build hoặc khởi động).                                             |
|| `webpackConfigPath` | string              | Trỏ đến một file tùy chọn webpack. Nếu không được chỉ định, Nest tìm file `webpack.config.js`. Xem chi tiết dưới đây để biết thêm chi tiết.                                                                                                                                              |
|| `deleteOutDir`      | boolean             | Nếu `true`, bất cứ khi nào trình biên dịch được gọi, nó sẽ trước tiên xóa thư mục đầu ra biên dịch (như được cấu hình trong `tsconfig.json`, trong đó mặc định là `./dist`).                                                                                                     |
|| `assets`            | array               | Bật tự động phân phối các tài sản không phải TypeScript bất cứ khi nào một bước biên dịch bắt đầu (phân phối tài sản **không** xảy ra trên các biên dịch tăng dần trong chế độ `--watch`). Xem chi tiết dưới đây để biết thêm chi tiết.                                                                    |
|| `watchAssets`       | boolean             | Nếu `true`, chạy ở chế độ watch, watch **tất cả** các tài sản không phải TypeScript. (Để kiểm soát chi tiết hơn về các tài sản để watch, xem phần [Tài sản](cli/monorepo#assets) dưới đây).                                                                                            |
|| `manualRestart`     | boolean             | Nếu `true`, bật phím tắt `rs` để khởi động lại thủ công máy chủ. Giá trị mặc định là `false`.                                                                                                                                                                            |
|| `builder`           | string/object       | Chỉ dẫn CLI về `builder` nào để sử dụng để biên dịch dự án (`tsc`, `swc`, hoặc `webpack`). Để tùy chỉnh hành vi của builder, bạn có thể truyền một đối tượng chứa hai thuộc tính: `type` (`tsc`, `swc`, hoặc `webpack`) và `options`.                                         |
|| `typeCheck`         | boolean             | Nếu `true`, bật kiểm tra kiểu cho các dự án dẫn động bởi SWC (khi `builder` là `swc`). Giá trị mặc định là `false`.                                                                                                                                                             |

#### Tùy chọn tạo toàn cầu

Các thuộc tính này chỉ định các tùy chọn tạo mặc định để được sử dụng bởi lệnh `nest generate`.

|| Tên Thuộc tính | Kiểu Giá Trị Thuộc Tính | Mô tả                                                                                                                                                                                                                                                                                                                                                                                                                                   |
|| ------------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|| `spec`        | boolean _hoặc_ object | Nếu giá trị là boolean, giá trị `true` bật tạo `spec` theo mặc định và giá trị `false` vô hiệu hóa nó. Một cờ được truyền trên dòng lệnh CLI ghi đè cài đặt này, cũng như cài đặt `generateOptions` cụ thể cho dự án (chi tiết hơn dưới đây). Nếu giá trị là một đối tượng, mỗi khóa đại diện cho một tên schematic, và giá trị boolean xác định xem việc tạo spec mặc định được bật / vô hiệu hóa cho schematic cụ thể đó. |
|| `flat`        | boolean             | Nếu true, tất cả các lệnh tạo sẽ tạo một cấu trúc phẳng                                                                                                                                                                                                                                                                                                                                                                                 |

Ví dụ sau sử dụng một giá trị boolean để chỉ định rằng việc tạo file spec nên được vô hiệu hóa theo mặc định cho tất cả các dự án:

```javascript
{
  "generateOptions": {
    "spec": false
  },
  ...
}
```

Ví dụ sau sử dụng một giá trị boolean để chỉ định rằng việc tạo file phẳng nên là mặc định cho tất cả các dự án:

```javascript
{
  "generateOptions": {
    "flat": true
  },
  ...
}
```

Trong ví dụ sau, việc tạo file `spec` chỉ bị vô hiệu hóa cho các schematics `service` (ví dụ, `nest generate service...`):

```javascript
{
  "generateOptions": {
    "spec": {
      "service": false
    }
  },
  ...
}
```

> warning **Cảnh báo** Khi chỉ định `spec` như một đối tượng, khóa cho schematic tạo hiện không hỗ trợ xử lý alias tự động. Điều này có nghĩa là chỉ định một khóa như ví dụ `service: false` và cố gắng tạo một service thông qua alias `s`, spec vẫn sẽ được tạo. Để đảm bảo cả tên schematic bình thường và alias hoạt động như dự định, chỉ định cả tên lệnh bình thường cũng như alias, như được thấy dưới đây.
>
> ```javascript
> {
>   "generateOptions": {
>     "spec": {
>       "service": false,
>       "s": false
>     }
>   },
>   ...
> }
> ```

#### Tùy chọn tạo cụ thể cho dự án

Ngoài việc cung cấp các tùy chọn tạo toàn cầu, bạn cũng có thể chỉ định các tùy chọn tạo cụ thể cho dự án. Các tùy chọn tạo cụ thể cho dự án tuân theo cùng định dạng chính xác với các tùy chọn tạo toàn cầu, nhưng được chỉ định trực tiếp trên mỗi dự án.

Các tùy chọn tạo cụ thể cho dự án ghi đè các tùy chọn tạo toàn cầu.

```javascript
{
  "projects": {
    "cats-project": {
      "generateOptions": {
        "spec": {
          "service": false
        }
      },
      ...
    }
  },
  ...
}
```

> warning **Cảnh báo** Thứ tự ưu tiên cho các tùy chọn tạo như sau. Các tùy chọn được chỉ định trên dòng lệnh CLI có ưu tiên hơn các tùy chọn cụ thể cho dự án. Các tùy chọn cụ thể cho dự án ghi đè các tùy chọn toàn cầu.

#### Trình biên dịch được chỉ định

Lý do cho các trình biên dịch mặc định khác nhau là đối với các dự án lớn hơn (ví dụ, điển hình hơn trong một monorepo) webpack có thể có những lợi ích đáng kể về thời gian build và trong việc tạo một file duy nhất bundle tất cả các thành phần dự án lại với nhau. Nếu bạn muốn tạo các file riêng lẻ, đặt `"webpack"` thành `false`, điều này sẽ khiến quá trình build sử dụng `tsc` (hoặc `swc`).

#### Tùy chọn Webpack

File tùy chọn webpack có thể chứa các [tùy chọn cấu hình webpack](https://webpack.js.org/configuration/) tiêu chuẩn. Ví dụ, để nói với webpack để bundle `node_modules` (được loại trừ theo mặc định), thêm sau vào `webpack.config.js`:

```javascript
module.exports = {
  externals: [],
};
```

Vì file cấu hình webpack là một file JavaScript, bạn thậm chí có thể expose một hàm nhận các tùy chọn mặc định và trả về một đối tượng đã sửa đổi:

```javascript
module.exports = function (options) {
  return {
    ...options,
    externals: [],
  };
};
```

#### Tài sản

Biên dịch TypeScript tự động phân phối đầu ra trình biên dịch (các file `.js` và `.d.ts`) đến thư mục đầu ra được chỉ định. Nó cũng có thể thuận tiện để phân phối các file không phải TypeScript, chẳng hạn như các file `.graphql`, `images`, các file `.html` và các tài sản khác. Điều này cho phép bạn coi `nest build` (và bất kỳ bước biên dịch ban đầu nào) như một bước **build phát triển** nhẹ nhàng, trong đó bạn có thể đang chỉnh sửa các file không phải TypeScript và biên dịch và kiểm tra lặp đi lặp lại.
Các tài sản nên nằm trong thư mục `src` nếu không chúng sẽ không được sao chép.

Giá trị của khóa `assets` nên là một mảng các phần tử chỉ định các file để được phân phối. Các phần tử có thể là các chuỗi đơn giản với thông số file giống `glob`, ví dụ:

```typescript
"assets": ["**/*.graphql"],
"watchAssets": true,
```

Để kiểm soát chi tiết hơn, các phần tử có thể là các đối tượng với các khóa sau:

- `"include"`: thông số file giống `glob` cho các tài sản để được phân phối
- `"exclude"`: thông số file giống `glob` cho các tài sản để được **loại trừ** khỏi danh sách `include`
- `"outDir"`: một chuỗi chỉ định đường dẫn (tương đối với thư mục gốc) nơi các tài sản nên được phân phối. Mặc định là cùng thư mục đầu ra được cấu hình cho đầu ra trình biên dịch.
- `"watchAssets"`: boolean; nếu `true`, chạy ở chế độ watch watch các tài sản được chỉ định

Ví dụ:

```typescript
"assets": [
  { "include": "**/*.graphql", "exclude": "**/omitted.graphql", "watchAssets": true },
]
```

> warning **Cảnh báo** Đặt `watchAssets` trong một thuộc tính `compilerOptions` cấp cao nhất ghi đè bất kỳ cài đặt `watchAssets` nào trong thuộc tính `assets`.

#### Thuộc tính dự án

Phần tử này chỉ tồn tại cho các cấu trúc chế độ monorepo. Bạn thường không nên chỉnh sửa các thuộc tính này, vì chúng được sử dụng bởi Nest để định vị các dự án và các tùy chọn cấu hình của chúng trong monorepo.