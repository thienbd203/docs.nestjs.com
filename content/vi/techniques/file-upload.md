### File upload

Để xử lý việc tải lên file, Nest cung cấp một module tích hợp dựa trên package middleware [multer](https://github.com/expressjs/multer) cho Express. Multer xử lý dữ liệu được đăng trong định dạng `multipart/form-data`, được sử dụng chủ yếu để tải lên các file thông qua yêu cầu HTTP `POST`. Module này hoàn toàn có thể cấu hình và bạn có thể điều chỉnh hành vi của nó theo yêu cầu ứng dụng của bạn.

> warning **Cảnh báo** Multer không thể xử lý dữ liệu không ở định dạng multipart được hỗ trợ (`multipart/form-data`). Ngoài ra, lưu ý rằng package này không tương thích với `FastifyAdapter`.

Để có an toàn kiểu tốt hơn, hãy cài đặt package gõ Multer:

```shell
$ npm i -D @types/multer
```

Với package này được cài đặt, chúng ta bây giờ có thể sử dụng kiểu `Express.Multer.File` (bạn có thể nhập kiểu này như sau: `import {{ '{' }} Express {{ '}' }} from 'express'`).

#### Ví dụ cơ bản

Để tải lên một file đơn, chỉ cần gắn interceptor `FileInterceptor()` vào route handler và trích xuất `file` từ `request` bằng decorator `@UploadedFile()`.

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
uploadFile(@UploadedFile() file: Express.Multer.File) {
  console.log(file);
}
@@switch
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
@Bind(UploadedFile())
uploadFile(file) {
  console.log(file);
}
```

> info **Gợi ý** Decorator `FileInterceptor()` được xuất từ package `@nestjs/platform-express`. Decorator `@UploadedFile()` được xuất từ `@nestjs/common`.

Decorator `FileInterceptor()` nhận hai đối số:

- `fieldName`: chuỗi cung cấp tên của trường từ biểu mẫu HTML giữ một file
- `options`: đối tượng tùy chọn kiểu `MulterOptions`. Đây là đối tượng tương tự được sử dụng bởi constructor multer (chi tiết thêm [tại đây](https://github.com/expressjs/multer#multeropts)).

> warning **Cảnh báo** `FileInterceptor()` có thể không tương thích với các nhà cung cấp đám mây bên thứ ba như Google Firebase hoặc khác.

#### Xác thực file

Thường thì có thể hữu ích để xác thực metadata file incoming, như kích thước file hoặc mime-type của file. Để làm điều này, bạn có thể tạo [Pipe](https://docs.nestjs.com/pipes) của riêng mình và bind nó vào tham số được chú thích với decorator `UploadedFile`. Ví dụ dưới đây minh họa cách một pipe xác thực kích thước file cơ bản có thể được implement:

```typescript
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class FileSizeValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    // "value" is an object containing the file's attributes and metadata
    const oneKb = 1000;
    return value.size < oneKb;
  }
}
```

Điều này có thể được sử dụng kết hợp với `FileInterceptor` như sau:

```typescript
@Post('file')
@UseInterceptors(FileInterceptor('file'))
uploadFileAndValidate(@UploadedFile(
  new FileSizeValidationPipe(),
  // other pipes can be added here
) file: Express.Multer.File, ) {
  return file;
}
```

Nest cung cấp một pipe tích hợp để xử lý các use case phổ biến và tạo điều kiện/chuẩn hóa việc thêm các mới. Pipe này được gọi là `ParseFilePipe`, và bạn có thể sử dụng nó như sau:

```typescript
@Post('file')
uploadFileAndPassValidation(
  @Body() body: SampleDto,
  @UploadedFile(
    new ParseFilePipe({
      validators: [
        // ... Set of file validator instances here
      ]
    })
  )
  file: Express.Multer.File,
) {
  return {
    body,
    file: file.buffer.toString(),
  };
}
```

Như bạn có thể thấy, cần phải chỉ định một mảng các file validator sẽ được thực thi bởi `ParseFilePipe`. Chúng ta sẽ thảo luận về interface của một validator, nhưng đáng nhắc là pipe này cũng có hai tùy chọn **tùy chọn** bổ sung:

<table>
  <tr>
    <td><code>errorHttpStatusCode</code></td>
    <td>Mã trạng thái HTTP sẽ được throw trong trường hợp <b>bất kỳ</b> validator nào thất bại. Mặc định là <code>400</code> (BAD REQUEST)</td>
  </tr>
  <tr>
    <td><code>exceptionFactory</code></td>
    <td>Một factory nhận thông báo lỗi và trả về một lỗi.</td>
  </tr>
</table>

Bây giờ, quay lại interface `FileValidator`. Để tích hợp các validator với pipe này, bạn phải sử dụng các implementation tích hợp sẵn hoặc cung cấp `FileValidator` tùy chỉnh của riêng bạn. Xem ví dụ dưới đây:

```typescript
export abstract class FileValidator<TValidationOptions = Record<string, any>> {
  constructor(protected readonly validationOptions: TValidationOptions) {}

  /**
   * Indicates if this file should be considered valid, according to the options passed in the constructor.
   * @param file the file from the request object
   */
  abstract isValid(file?: any): boolean | Promise<boolean>;

  /**
   * Builds an error message in case the validation fails.
   * @param file the file from the request object
   */
  abstract buildErrorMessage(file: any): string;
}
```

> info **Gợi ý** Interface `FileValidator` hỗ trợ xác thực async thông qua hàm `isValid` của nó. Để tận dụng tính an toàn kiểu, bạn cũng có thể gõ tham số `file` là `Express.Multer.File` trong trường hợp bạn đang sử dụng express (mặc định) làm driver.

`FileValidator` là một class thông thường có quyền truy cập vào đối tượng file và xác thực nó theo các tùy chọn được cung cấp bởi client. Nest có hai implementation `FileValidator` tích hợp sẵn mà bạn có thể sử dụng trong dự án của mình:

- `MaxFileSizeValidator` - Kiểm tra xem kích thước của một file đã cho có nhỏ hơn giá trị được cung cấp hay không (được đo bằng `bytes`)
- `FileTypeValidator` - Kiểm tra xem mime-type của một file đã cho có khớp với một chuỗi hoặc RegExp đã cho hay không. Theo mặc định, xác thực mime-type bằng cách sử dụng nội dung file [magic number](https://www.ibm.com/support/pages/what-magic-number)

Để hiểu cách các này có thể được sử dụng kết hợp với `ParseFilePipe` đã đề cập ở trên, chúng ta sẽ sử dụng một đoạn mã đã được thay đổi của ví dụ cuối cùng được trình bày:

```typescript
@UploadedFile(
  new ParseFilePipe({
    validators: [
      new MaxFileSizeValidator({ maxSize: 1000 }),
      new FileTypeValidator({ fileType: 'image/jpeg' }),
    ],
  }),
)
file: Express.Multer.File,
```

> info **Gợi ý** Nếu số lượng validator tăng lớn hoặc các tùy chọn của chúng đang làm lộn xộn file, bạn có thể định nghĩa mảng này trong một file riêng và nhập nó tại đây như một hằng số được đặt tên như `fileValidators`.

Cuối cùng, bạn có thể sử dụng class `ParseFilePipeBuilder` đặc biệt cho phép bạn compose & xây dựng các validator của mình. Bằng cách sử dụng nó như hiển thị bên dưới, bạn có thể tránh khởi tạo thủ công từng validator và chỉ truyền các tùy chọn của chúng trực tiếp:

```typescript
@UploadedFile(
  new ParseFilePipeBuilder()
    .addFileTypeValidator({
      fileType: 'jpeg',
    })
    .addMaxSizeValidator({
      maxSize: 1000
    })
    .build({
      errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY
    }),
)
file: Express.Multer.File,
```

> info **Gợi ý** Sự hiện diện của file được yêu cầu theo mặc định, nhưng bạn có thể làm cho nó tùy chọn bằng cách thêm tham số `fileIsRequired: false` bên trong các tùy chọn hàm `build` (ở cùng cấp với `errorHttpStatusCode`).

#### Mảng các file

Để tải lên một mảng các file (được xác định bằng một tên trường đơn), sử dụng decorator `FilesInterceptor()` (lưu ý số nhiều **Files** trong tên decorator). Decorator này nhận ba đối số:

- `fieldName`: như mô tả ở trên
- `maxCount`: số tùy chọn định nghĩa số lượng file tối đa để chấp nhận
- `options`: đối tượng `MulterOptions` tùy chọn, như mô tả ở trên

Khi sử dụng `FilesInterceptor()`, trích xuất các file từ `request` với decorator `@UploadedFiles()`.

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FilesInterceptor('files'))
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
@@switch
@Post('upload')
@UseInterceptors(FilesInterceptor('files'))
@Bind(UploadedFiles())
uploadFile(files) {
  console.log(files);
}
```

> info **Gợi ý** Decorator `FilesInterceptor()` được xuất từ package `@nestjs/platform-express`. Decorator `@UploadedFiles()` được xuất từ `@nestjs/common`.

#### Nhiều file

Để tải lên nhiều file (tất cả với các key tên trường khác nhau), sử dụng decorator `FileFieldsInterceptor()`. Decorator này nhận hai đối số:

- `uploadedFields`: một mảng các đối tượng, trong đó mỗi đối tượng chỉ định thuộc tính `name` bắt buộc với giá trị chuỗi chỉ định một tên trường, như mô tả ở trên, và thuộc tính `maxCount` tùy chọn, như mô tả ở trên
- `options`: đối tượng `MulterOptions` tùy chọn, như mô tả ở trên

Khi sử dụng `FileFieldsInterceptor()`, trích xuất các file từ `request` với decorator `@UploadedFiles()`.

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FileFieldsInterceptor([
  { name: 'avatar', maxCount: 1 },
  { name: 'background', maxCount: 1 },
]))
uploadFile(@UploadedFiles() files: { avatar?: Express.Multer.File[], background?: Express.Multer.File[] }) {
  console.log(files);
}
@@switch
@Post('upload')
@Bind(UploadedFiles())
@UseInterceptors(FileFieldsInterceptor([
  { name: 'avatar', maxCount: 1 },
  { name: 'background', maxCount: 1 },
]))
uploadFile(files) {
  console.log(files);
}
```

#### Bất kỳ file nào

Để tải lên tất cả các trường với các key tên trường tùy ý, sử dụng decorator `AnyFilesInterceptor()`. Decorator này có thể chấp nhận một đối tượng `options` tùy chọn như mô tả ở trên.

Khi sử dụng `AnyFilesInterceptor()`, trích xuất các file từ `request` với decorator `@UploadedFiles()`.

```typescript
@@filename()
@Post('upload')
@UseInterceptors(AnyFilesInterceptor())
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
@@switch
@Post('upload')
@Bind(UploadedFiles())
@UseInterceptors(AnyFilesInterceptor())
uploadFile(files) {
  console.log(files);
}
```

#### Không có file

Để chấp nhận `multipart/form-data` nhưng không cho phép bất kỳ file nào được tải lên, sử dụng `NoFilesInterceptor`. Điều này đặt dữ liệu multipart làm các thuộc tính trên phần thân yêu cầu. Bất kỳ file nào được gửi cùng với yêu cầu sẽ throw một `BadRequestException`.

```typescript
@Post('upload')
@UseInterceptors(NoFilesInterceptor())
handleMultiPartData(@Body() body) {
  console.log(body)
}
```

#### Tùy chọn mặc định

Bạn có thể chỉ định các tùy chọn multer trong các file interceptor như mô tả ở trên. Để đặt các tùy chọn mặc định, bạn có thể gọi phương thức tĩnh `register()` khi bạn nhập `MulterModule`, truyền các tùy chọn được hỗ trợ. Bạn có thể sử dụng tất cả các tùy chọn được liệt kê [tại đây](https://github.com/expressjs/multer#multeropts).

```typescript
MulterModule.register({
  dest: './upload',
});
```

> info **Gợi ý** Class `MulterModule` được xuất từ package `@nestjs/platform-express`.

#### Cấu hình async

Khi bạn cần đặt tùy chọn `MulterModule` một cách async thay vì tĩnh, sử dụng phương thức `registerAsync()`. Như với hầu hết các module động, Nest cung cấp một số kỹ thuật để xử lý cấu hình async.

Một kỹ thuật là sử dụng factory function:

```typescript
MulterModule.registerAsync({
  useFactory: () => ({
    dest: './upload',
  }),
});
```

Giống như các [factory provider](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) khác, factory function của chúng ta có thể là `async` và có thể inject dependencies thông qua `inject`.

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    dest: configService.get<string>('MULTER_DEST'),
  }),
  inject: [ConfigService],
});
```

Ngoài ra, bạn có thể cấu hình `MulterModule` bằng cách sử dụng một class thay vì một factory, như hiển thị bên dưới:

```typescript
MulterModule.registerAsync({
  useClass: MulterConfigService,
});
```

Cấu trúc trên khởi tạo `MulterConfigService` bên trong `MulterModule`, sử dụng nó để tạo đối tượng tùy chọn cần thiết. Lưu ý rằng trong ví dụ này, `MulterConfigService` phải implement interface `MulterOptionsFactory`, như hiển thị bên dưới. `MulterModule` sẽ gọi phương thức `createMulterOptions()` trên đối tượng được khởi tạo của class được cung cấp.

```typescript
@Injectable()
class MulterConfigService implements MulterOptionsFactory {
  createMulterOptions(): MulterModuleOptions {
    return {
      dest: './upload',
    };
  }
}
```

Nếu bạn muốn tái sử dụng một provider tùy chọn hiện có thay vì tạo một bản sao riêng bên trong `MulterModule`, sử dụng cú pháp `useExisting`.

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

Bạn cũng có thể truyền các `extraProviders` được gọi vào phương thức `registerAsync()`. Các providers này sẽ được hợp nhất với các providers của module.

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useClass: ConfigService,
  extraProviders: [MyAdditionalProvider],
});
```

Điều này hữu ích khi bạn muốn cung cấp các dependencies bổ sung cho factory function hoặc constructor của class.

#### Ví dụ

Một ví dụ hoạt động có sẵn [tại đây](https://github.com/nestjs/nest/tree/master/sample/29-file-upload).