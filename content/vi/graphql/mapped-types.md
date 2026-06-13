### Mapped types

> warning **Cảnh báo** Chương này chỉ áp dụng cho phương pháp code first.

Khi bạn xây dựng các tính năng như CRUD (Create/Read/Update/Delete), thường hữu ích để xây dựng các biến thể trên một loại thực thể cơ sở. Nest cung cấp một số hàm tiện ích thực hiện các chuyển đổi loại để làm cho nhiệm vụ này thuận tiện hơn.

#### Partial

Khi xây dựng các loại xác thực đầu vào (còn gọi là Data Transfer Objects hoặc DTOs), thường hữu ích để xây dựng các biến thể **create** và **update** trên cùng một loại. Ví dụ, biến thể **create** có thể yêu cầu tất cả các trường, trong khi biến thể **update** có thể làm cho tất cả các trường tùy chọn.

Nest cung cấp hàm tiện ích `PartialType()` để làm cho nhiệm vụ này dễ dàng hơn và giảm thiểu boilerplate.

Hàm `PartialType()` trả về một loại (lớp) với tất cả các thuộc tính của loại đầu vào được đặt thành tùy chọn. Ví dụ, giả sử chúng ta có một loại **create** như sau:

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

Theo mặc định, tất cả các trường này được yêu cầu. Để tạo một loại với cùng các trường, nhưng với mỗi trường tùy chọn, sử dụng `PartialType()` chuyển tham chiếu lớp (`CreateUserInput`) làm đối số:

```typescript
@InputType()
export class UpdateUserInput extends PartialType(CreateUserInput) {}
```

> info **Gợi ý** Hàm `PartialType()` được nhập từ gói `@nestjs/graphql`.

Hàm `PartialType()` nhận một đối số thứ hai tùy chọn là một tham chiếu đến một factory decorator. Đối số này có thể được sử dụng để thay đổi hàm decorator được áp dụng cho lớp (con) kết quả. Nếu không được chỉ định, lớp con hiệu quả sử dụng cùng decorator với lớp **cha** (lớp được tham chiếu trong đối số đầu tiên). Trong ví dụ trên, chúng ta đang mở rộng `CreateUserInput` được chú thích với decorator `@InputType()`. Vì chúng ta muốn `UpdateUserInput` cũng được xử lý như thể nó được trang trí với `@InputType()`, chúng ta không cần chuyển `InputType` làm đối số thứ hai. Nếu các loại cha và con khác nhau, (ví dụ, cha được trang trí với `@ObjectType`), chúng ta sẽ chuyển `InputType` làm đối số thứ hai. Ví dụ:

```typescript
@InputType()
export class UpdateUserInput extends PartialType(User, InputType) {}
```

#### Pick

Hàm `PickType()` xây dựng một loại mới (lớp) bằng cách chọn một tập hợp các thuộc tính từ một loại đầu vào. Ví dụ, giả sử chúng ta bắt đầu với một loại như:

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

Chúng ta có thể chọn một tập hợp các thuộc tính từ lớp này sử dụng hàm tiện ích `PickType()`:

```typescript
@InputType()
export class UpdateEmailInput extends PickType(CreateUserInput, [
  'email',
] as const) {}
```

> info **Gợi ý** Hàm `PickType()` được nhập từ gói `@nestjs/graphql`.

#### Omit

Hàm `OmitType()` xây dựng một loại bằng cách chọn tất cả các thuộc tính từ một loại đầu vào và sau đó loại bỏ một tập hợp các khóa cụ thể. Ví dụ, giả sử chúng ta bắt đầu với một loại như:

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

Chúng ta có thể tạo một loại dẫn xuất có mọi thuộc tính **trừ** `email` như được hiển thị dưới đây. Trong cấu trúc này, đối số thứ hai cho `OmitType` là một mảng tên thuộc tính.

```typescript
@InputType()
export class UpdateUserInput extends OmitType(CreateUserInput, [
  'email',
] as const) {}
```

> info **Gợi ý** Hàm `OmitType()` được nhập từ gói `@nestjs/graphql`.

#### Intersection

Hàm `IntersectionType()` kết hợp hai loại thành một loại mới (lớp). Ví dụ, giả sử chúng ta bắt đầu với hai loại như:

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;
}

@ObjectType()
export class AdditionalUserInfo {
  @Field()
  firstName: string;

  @Field()
  lastName: string;
}
```

Chúng ta có thể tạo một loại mới kết hợp tất cả các thuộc tính trong cả hai loại.

```typescript
@InputType()
export class UpdateUserInput extends IntersectionType(
  CreateUserInput,
  AdditionalUserInfo,
) {}
```

> info **Gợi ý** Hàm `IntersectionType()` được nhập từ gói `@nestjs/graphql`.

#### Composition

Các hàm tiện ích ánh xạ loại có thể được kết hợp. Ví dụ, sau đây sẽ tạo ra một loại (lớp) có tất cả các thuộc tính của loại `CreateUserInput` ngoại trừ `email`, và các thuộc tính đó sẽ được đặt thành tùy chọn:

```typescript
@InputType()
export class UpdateUserInput extends PartialType(
  OmitType(CreateUserInput, ['email'] as const),
) {}
```