### Serialization

Serialization là một quá trình diễn ra trước khi các đối tượng được trả về trong một phản hồi mạng. Đây là một nơi thích hợp để cung cấp các quy tắc để chuyển đổi và làm sạch dữ liệu sẽ được trả về cho client. Ví dụ, dữ liệu nhạy cảm như mật khẩu luôn nên được loại khỏi phản hồi. Hoặc, một số thuộc tính có thể yêu cầu chuyển đổi bổ sung, chẳng hạn như chỉ gửi một tập hợp con của các thuộc tính của một entity. Thực hiện các chuyển đổi này theo cách thủ công có thể tẻ nhạt và dễ gây lỗi, và có thể khiến bạn không chắc chắn rằng tất cả các trường hợp đã được bao phủ.

#### Tổng quan

Nest cung cấp một khả năng tích hợp để giúp đảm bảo rằng các hoạt động này có thể được thực hiện theo cách đơn giản. Interceptor `ClassSerializerInterceptor` sử dụng package mạnh mẽ [class-transformer](https://github.com/typestack/class-transformer) để cung cấp một cách declarative và có thể mở rộng để chuyển đổi các đối tượng. Hoạt động cơ bản mà nó thực hiện là lấy giá trị trả về bởi một phương thức handler và áp dụng hàm `instanceToPlain()` từ [class-transformer](https://github.com/typestack/class-transformer). Khi làm như vậy, nó có thể áp dụng các quy tắc được thể hiện bởi các decorator `class-transformer` trên một class entity/DTO, như mô tả bên dưới.

> info **Gợi ý** Serialization không áp dụng cho các phản hồi [StreamableFile](https://docs.nestjs.com/techniques/streaming-files#streamable-file-class).

#### Loại trừ thuộc tính

Giả sử rằng chúng ta muốn tự động loại bỏ thuộc tính `password` khỏi một entity người dùng. Chúng ta chú thích entity như sau:

```typescript
import { Exclude } from 'class-transformer';

export class UserEntity {
  id: number;
  firstName: string;
  lastName: string;

  @Exclude()
  password: string;

  constructor(partial: Partial<UserEntity>) {
    Object.assign(this, partial);
  }
}
```

Bây giờ hãy xem xét một controller với một phương thức handler trả về một instance của class này.

```typescript
@UseInterceptors(ClassSerializerInterceptor)
@Get()
findOne(): UserEntity {
  return new UserEntity({
    id: 1,
    firstName: 'John',
    lastName: 'Doe',
    password: 'password',
  });
}
```

> **Cảnh báo** Lưu ý rằng chúng ta phải trả về một instance của class. Nếu bạn trả về một đối tượng JavaScript thuần, ví dụ, `{{ '{' }} user: new UserEntity() {{ '}' }}`, đối tượng sẽ không được serialize đúng cách.

> info **Gợi ý** `ClassSerializerInterceptor` được nhập từ `@nestjs/common`.

Khi endpoint này được yêu cầu, client nhận được phản hồi sau:

```json
{
  "id": 1,
  "firstName": "John",
  "lastName": "Doe"
}
```

Lưu ý rằng interceptor có thể được áp dụng trên phạm vi toàn ứng dụng (như được đề cập [tại đây](https://docs.nestjs.com/interceptors#binding-interceptors)). Sự kết hợp của interceptor và khai báo class entity đảm bảo rằng **bất kỳ** phương thức nào trả về một `UserEntity` sẽ chắc chắn loại bỏ thuộc tính `password`. Điều này mang lại cho bạn một biện pháp thực thi tập trung cho quy tắc kinh doanh này.

#### Expose thuộc tính

Bạn có thể sử dụng decorator `@Expose()` để cung cấp tên alias cho các thuộc tính, hoặc để thực hiện một hàm để tính toán giá trị thuộc tính (tương tự như các hàm **getter**), như hiển thị bên dưới.

```typescript
@Expose()
get fullName(): string {
  return `${this.firstName} ${this.lastName}`;
}
```

#### Transform

Bạn có thể thực hiện chuyển đổi dữ liệu bổ sung bằng cách sử dụng decorator `@Transform()`. Ví dụ, cấu trúc sau trả về thuộc tính tên của `RoleEntity` thay vì trả về toàn bộ đối tượng.

```typescript
@Transform(({ value }) => value.name)
role: RoleEntity;
```

#### Truyền tùy chọn

Bạn có thể muốn sửa đổi hành vi mặc định của các hàm chuyển đổi. Để ghi đè các cài đặt mặc định, truyền chúng trong một đối tượng `options` với decorator `@SerializeOptions()`.

```typescript
@SerializeOptions({
  excludePrefixes: ['_'],
})
@Get()
findOne(): UserEntity {
  return new UserEntity();
}
```

> info **Gợi ý** Decorator `@SerializeOptions()` được nhập từ `@nestjs/common`.

Các tùy chọn được truyền thông qua `@SerializeOptions()` được truyền làm đối số thứ hai của hàm `instanceToPlain()` cơ bản. Trong ví dụ này, chúng ta đang tự động loại bỏ tất cả các thuộc tính bắt đầu bằng tiền tố `_`.

#### Chuyển đổi các đối tượng thuần

Bạn có thể thực thi các chuyển đổi ở cấp controller bằng cách sử dụng decorator `@SerializeOptions`. Điều này đảm bảo rằng tất cả các phản hồi được chuyển đổi thành các instance của class được chỉ định, áp dụng bất kỳ decorator nào từ class-validator hoặc class-transformer, ngay cả khi các đối tượng thuần được trả về. Cách tiếp cận này dẫn đến code sạch hơn mà không cần phải khởi tạo class liên tục hoặc gọi `plainToInstance`.

Trong ví dụ dưới đây, mặc dù trả về các đối tượng JavaScript thuần trong cả hai nhánh điều kiện, chúng sẽ được tự động chuyển đổi thành các instance `UserEntity`, với các decorator liên quan được áp dụng:

```typescript
@UseInterceptors(ClassSerializerInterceptor)
@SerializeOptions({ type: UserEntity })
@Get()
findOne(@Query() { id }: { id: number }): UserEntity {
  if (id === 1) {
    return {
      id: 1,
      firstName: 'John',
      lastName: 'Doe',
      password: 'password',
    };
  }

  return {
    id: 2,
    firstName: 'Kamil',
    lastName: 'Mysliwiec',
    password: 'password2',
  };
}
```

> info **Gợi ý** Bằng cách chỉ định kiểu trả về mong đợi cho controller, bạn có thể tận dụng khả năng type-checking của TypeScript để đảm bảo rằng đối tượng thuần được trả về tuân theo hình dạng của DTO hoặc entity. Hàm `plainToInstance` không cung cấp mức độ type hinting này, điều này có thể dẫn đến các lỗi tiềm ẩn nếu đối tượng thuần không khớp với cấu trúc DTO hoặc entity mong đợi.

#### Ví dụ

Một ví dụ hoạt động có sẵn [tại đây](https://github.com/nestjs/nest/tree/master/sample/21-serializer).

#### WebSockets và Microservices

Mặc dù chương này hiển thị các ví dụ sử dụng các ứng dụng kiểu HTTP (ví dụ, Express hoặc Fastify), `ClassSerializerInterceptor` hoạt động giống nhau cho WebSockets và Microservices, bất kể phương thức transport được sử dụng.

#### Tìm hiểu thêm

Đọc thêm về các decorator và tùy chọn có sẵn được cung cấp bởi package `class-transformer` [tại đây](https://github.com/typestack/class-transformer).