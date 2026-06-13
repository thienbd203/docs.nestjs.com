### Mapped types

Khi bạn xây dựng các tính năng như **CRUD** (Create/Read/Update/Delete), việc xây dựng các biến thể trên một kiểu thực thể cơ bản thường rất hữu ích. Nest cung cấp một số hàm tiện ích thực hiện các biến đổi kiểu để làm cho nhiệm vụ này thuận tiện hơn.

#### Partial

Khi xây dựng các kiểu validation input (còn gọi là DTOs), việc xây dựng các biến thể **create** và **update** trên cùng một kiểu thường rất hữu ích. Ví dụ, biến thể **create** có thể yêu cầu tất cả các trường, trong khi biến thể **update** có thể làm cho tất cả các trường trở thành optional.

Nest cung cấp hàm tiện ích `PartialType()` để làm nhiệm vụ này dễ dàng hơn và giảm thiểu boilerplate.

Hàm `PartialType()` trả về một kiểu (class) với tất cả các thuộc tính của kiểu input được đặt thành optional. Ví dụ, giả sử chúng ta có một kiểu **create** như sau:

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

Theo mặc định, tất cả các trường này đều required. Để tạo một kiểu với cùng các trường, nhưng mỗi trường là optional, sử dụng `PartialType()` truyền class reference (`CreateCatDto`) làm đối số:

```typescript
export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

> info **Gợi ý** Hàm `PartialType()` được import từ package `@nestjs/swagger`.

#### Pick

Hàm `PickType()` xây dựng một kiểu mới (class) bằng cách chọn một tập hợp các thuộc tính từ một kiểu input. Ví dụ, giả sử chúng ta bắt đầu với một kiểu như:

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

Chúng ta có thể chọn một tập hợp các thuộc tính từ class này sử dụng hàm tiện ích `PickType()`:

```typescript
export class UpdateCatAgeDto extends PickType(CreateCatDto, ['age'] as const) {}
```

> info **Gợi ý** Hàm `PickType()` được import từ package `@nestjs/swagger`.

#### Omit

Hàm `OmitType()` xây dựng một kiểu bằng cách chọn tất cả các thuộc tính từ một kiểu input và sau đó xóa một tập hợp các khóa cụ thể. Ví dụ, giả sử chúng ta bắt đầu với một kiểu như:

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

Chúng ta có thể tạo một kiểu dẫn xuất có mọi thuộc tính **trừ** `name` như được hiển thị dưới đây. Trong cấu trúc này, đối số thứ hai cho `OmitType` là một mảng các tên thuộc tính.

```typescript
export class UpdateCatDto extends OmitType(CreateCatDto, ['name'] as const) {}
```

> info **Gợi ý** Hàm `OmitType()` được import từ package `@nestjs/swagger`.

#### Intersection

Hàm `IntersectionType()` kết hợp hai kiểu thành một kiểu mới (class). Ví dụ, giả sử chúng ta bắt đầu với hai kiểu như:

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  breed: string;
}

export class AdditionalCatInfo {
  @ApiProperty()
  color: string;
}
```

Chúng ta có thể tạo một kiểu mới kết hợp tất cả các thuộc tính trong cả hai kiểu.

```typescript
export class UpdateCatDto extends IntersectionType(
  CreateCatDto,
  AdditionalCatInfo,
) {}
```

> info **Gợi ý** Hàm `IntersectionType()` được import từ package `@nestjs/swagger`.

#### Composition

Các hàm tiện ích ánh xạ kiểu có thể được kết hợp. Ví dụ, sau đây sẽ tạo ra một kiểu (class) có tất cả các thuộc tính của kiểu `CreateCatDto` ngoại trừ `name`, và các thuộc tính đó sẽ được đặt thành optional:

```typescript
export class UpdateCatDto extends PartialType(
  OmitType(CreateCatDto, ['name'] as const),
) {}
```