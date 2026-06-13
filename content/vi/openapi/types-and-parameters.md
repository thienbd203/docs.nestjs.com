### Types và parameters

`SwaggerModule` tìm kiếm tất cả các decorators `@Body()`, `@Query()`, và `@Param()` trong các route handlers để tạo tài liệu API. Nó cũng tạo ra các định nghĩa model tương ứng bằng cách tận dụng reflection. Xem xét mã sau:

```typescript
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **Gợi ý** Để đặt rõ ràng định nghĩa body sử dụng decorator `@ApiBody()` (được import từ package `@nestjs/swagger`).

Dựa trên `CreateCatDto`, định nghĩa model Swagger UI sau sẽ được tạo:

<figure><img src="/assets/swagger-dto.png" /></figure>

Như bạn có thể thấy, định nghĩa là trống mặc dù class có một vài thuộc tính được khai báo. Để làm cho các thuộc tính class có thể nhìn thấy được bởi `SwaggerModule`, chúng ta phải hoặc annotate chúng với decorator `@ApiProperty()` hoặc sử dụng CLI plugin (đọc thêm trong phần **Plugin**) sẽ làm điều này tự động:

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

> info **Gợi ý** Thay vì annotate thủ công từng thuộc tính, hãy cân nhắc sử dụng Swagger plugin (xem phần [Plugin](/openapi/cli-plugin)) sẽ tự động cung cấp điều này cho bạn.

Hãy mở trình duyệt và xác minh model `CreateCatDto` được tạo:

<figure><img src="/assets/swagger-dto2.png" /></figure>

Ngoài ra, decorator `@ApiProperty()` cho phép đặt các thuộc tính [Schema Object](https://swagger.io/specification/#schemaObject) khác nhau:

```typescript
@ApiProperty({
  description: 'The age of a cat',
  minimum: 1,
  default: 1,
})
age: number;
```

> info **Gợi ý** Thay vì typing rõ ràng `{{"@ApiProperty({ required: false })"}}` bạn có thể sử dụng decorator ngắn gọn `@ApiPropertyOptional()`.

Để đặt rõ ràng kiểu của thuộc tính, sử dụng khóa `type`:

```typescript
@ApiProperty({
  type: Number,
})
age: number;
```

#### Arrays

Khi thuộc tính là một mảng, chúng ta phải chỉ định thủ công kiểu mảng như được hiển thị dưới đây:

```typescript
@ApiProperty({ type: [String] })
names: string[];
```

> info **Gợi ý** Cân nhắc sử dụng Swagger plugin (xem phần [Plugin](/openapi/cli-plugin)) sẽ tự động phát hiện các mảng.

Hoặc bao gồm kiểu làm phần tử đầu tiên của một mảng (như được hiển thị ở trên) hoặc đặt thuộc tính `isArray` thành `true`.

<app-banner-enterprise></app-banner-enterprise>

#### Circular dependencies

Khi bạn có các circular dependencies giữa các classes, sử dụng một hàm lazy để cung cấp thông tin kiểu cho `SwaggerModule`:

```typescript
@ApiProperty({ type: () => Node })
node: Node;
```

> info **Gợi ý** Cân nhắc sử dụng Swagger plugin (xem phần [Plugin](/openapi/cli-plugin)) sẽ tự động phát hiện các circular dependencies.

#### Generics và interfaces

Vì TypeScript không lưu trữ metadata về generics hoặc interfaces, khi bạn sử dụng chúng trong DTOs của mình, `SwaggerModule` có thể không thể tạo đúng các định nghĩa model tại runtime. Ví dụ, mã sau sẽ không được kiểm tra chính xác bởi module Swagger:

```typescript
createBulk(@Body() usersDto: CreateUserDto[])
```

Để vượt qua giới hạn này, bạn có thể đặt kiểu rõ ràng:

```typescript
@ApiBody({ type: [CreateUserDto] })
createBulk(@Body() usersDto: CreateUserDto[])
```

#### Enums

Để xác định một `enum`, chúng ta phải đặt thủ công thuộc tính `enum` trên `@ApiProperty` với một mảng các giá trị.

```typescript
@ApiProperty({ enum: ['Admin', 'Moderator', 'User']})
role: UserRole;
```

Ngoài ra, định nghĩa một TypeScript enum thực tế như sau:

```typescript
export enum UserRole {
  Admin = 'Admin',
  Moderator = 'Moderator',
  User = 'User',
}
```

Sau đó bạn có thể sử dụng enum trực tiếp với decorator parameter `@Query()` kết hợp với decorator `@ApiQuery()`.

```typescript
@ApiQuery({ name: 'role', enum: UserRole })
async filterByRole(@Query('role') role: UserRole = UserRole.User) {}
```

<figure><img src="/assets/enum_query.gif" /></figure>

Với `isArray` được đặt thành **true**, `enum` có thể được chọn như một **multi-select**:

<figure><img src="/assets/enum_query_array.gif" /></figure>

#### Enums schema

Theo mặc định, thuộc tính `enum` sẽ thêm một định nghĩa raw của [Enum](https://swagger.io/docs/specification/data-models/enums/) trên `parameter`.

```yaml
- breed:
    type: 'string'
    enum:
      - Persian
      - Tabby
      - Siamese
```

Specification ở trên hoạt động tốt cho hầu hết các trường hợp. Tuy nhiên, nếu bạn đang sử dụng một công cụ lấy specification làm **input** và tạo **client-side** code, bạn có thể gặp vấn đề với mã được tạo chứa các `enums` bị trùng lặp. Xem xét đoạn mã sau:

```typescript
// mã client-side được tạo
export class CatDetail {
  breed: CatDetailEnum;
}

export class CatInformation {
  breed: CatInformationEnum;
}

export enum CatDetailEnum {
  Persian = 'Persian',
  Tabby = 'Tabby',
  Siamese = 'Siamese',
}

export enum CatInformationEnum {
  Persian = 'Persian',
  Tabby = 'Tabby',
  Siamese = 'Siamese',
}
```

> info **Gợi ý** Đoạn mã trên được tạo sử dụng một công cụ gọi là [NSwag](https://github.com/RicoSuter/NSwag).

Bạn có thể thấy rằng bây giờ bạn có hai `enums` hoàn toàn giống nhau.
Để giải quyết vấn đề này, bạn có thể truyền một `enumName` cùng với thuộc tính `enum` trong decorator của mình.

```typescript
export class CatDetail {
  @ApiProperty({ enum: CatBreed, enumName: 'CatBreed' })
  breed: CatBreed;
}
```

Thuộc tính `enumName` cho phép `@nestjs/swagger` biến `CatBreed` thành `schema` riêng của nó mà lần lượt làm cho enum `CatBreed` có thể tái sử dụng. Specification sẽ trông như sau:

```yaml
CatDetail:
  type: 'object'
  properties:
    ...
    - breed:
        schema:
          $ref: '#/components/schemas/CatBreed'
CatBreed:
  type: string
  enum:
    - Persian
    - Tabby
    - Siamese
```

> info **Gợi ý** Bất kỳ **decorator** nào nhận `enum` làm thuộc tính cũng sẽ nhận `enumName`.

#### Property value examples

Bạn có thể đặt một ví dụ đơn giản cho một thuộc tính bằng cách sử dụng khóa `example`, như sau:

```typescript
@ApiProperty({
  example: 'persian',
})
breed: string;
```

Nếu bạn muốn cung cấp nhiều ví dụ, bạn có thể sử dụng khóa `examples` bằng cách truyền một đối tượng được cấu trúc như sau:

```typescript
@ApiProperty({
  examples: {
    Persian: { value: 'persian' },
    Tabby: { value: 'tabby' },
    Siamese: { value: 'siamese' },
    'Scottish Fold': { value: 'scottish_fold' },
  },
})
breed: string;
```

#### Raw definitions

Trong một số trường hợp, chẳng hạn như các mảng lồng nhau sâu hoặc ma trận, bạn có thể cần định nghĩa thủ công kiểu của mình:

```typescript
@ApiProperty({
  type: 'array',
  items: {
    type: 'array',
    items: {
      type: 'number',
    },
  },
})
coords: number[][];
```

Bạn cũng có thể chỉ định các schemas object raw, như sau:

```typescript
@ApiProperty({
  type: 'object',
  properties: {
    name: {
      type: 'string',
      example: 'Error'
    },
    status: {
      type: 'number',
      example: 400
    }
  },
  required: ['name', 'status']
})
rawDefinition: Record<string, any>;
```

Để định nghĩa thủ công nội dung input/output trong các class controller, sử dụng thuộc tính `schema`:

```typescript
@ApiBody({
  schema: {
    type: 'array',
    items: {
      type: 'array',
      items: {
        type: 'number',
      },
    },
  },
})
async create(@Body() coords: number[][]) {}
```

#### Extra models

Để định nghĩa các models bổ sung không được tham chiếu trực tiếp trong controllers của bạn nhưng nên được kiểm tra bởi module Swagger, sử dụng decorator `@ApiExtraModels()`:

```typescript
@ApiExtraModels(ExtraModel)
export class CreateCatDto {}
```

> info **Gợi ý** Bạn chỉ cần sử dụng `@ApiExtraModels()` một lần cho một class model cụ thể.

Ngoài ra, bạn có thể truyền một đối tượng tùy chọn với thuộc tính `extraModels` được chỉ định đến phương thức `SwaggerModule.createDocument()`, như sau:

```typescript
const documentFactory = () =>
  SwaggerModule.createDocument(app, options, {
    extraModels: [ExtraModel],
  });
```

Để lấy một reference (`$ref`) đến model của bạn, sử dụng hàm `getSchemaPath(ExtraModel)`:

```typescript
'application/vnd.api+json': {
   schema: { $ref: getSchemaPath(ExtraModel) },
},
```

#### oneOf, anyOf, allOf

Để kết hợp các schemas, bạn có thể sử dụng các từ khóa `oneOf`, `anyOf` hoặc `allOf` ([đọc thêm](https://swagger.io/docs/specification/data-models/oneof-anyof-allof-not/)).

```typescript
@ApiProperty({
  oneOf: [
    { $ref: getSchemaPath(Cat) },
    { $ref: getSchemaPath(Dog) },
  ],
})
pet: Cat | Dog;
```

Nếu bạn muốn định nghĩa một mảng polymorphic (tức là một mảng có các thành viên trải dài trên nhiều schemas), bạn nên sử dụng một định nghĩa raw (xem ở trên) để định nghĩa kiểu của bạn bằng tay.

```typescript
type Pet = Cat | Dog;

@ApiProperty({
  type: 'array',
  items: {
    oneOf: [
      { $ref: getSchemaPath(Cat) },
      { $ref: getSchemaPath(Dog) },
    ],
  },
})
pets: Pet[];
```

> info **Gợi ý** Hàm `getSchemaPath()` được import từ `@nestjs/swagger`.

Cả `Cat` và `Dog` phải được định nghĩa như extra models sử dụng decorator `@ApiExtraModels()` (ở cấp độ class).

#### Schema name và description

Như bạn có thể đã nhận thấy, tên của schema được tạo dựa trên tên của class model gốc (ví dụ, model `CreateCatDto` tạo ra một schema `CreateCatDto`). Nếu bạn muốn thay đổi tên schema, bạn có thể sử dụng decorator `@ApiSchema()`.

Dưới đây là một ví dụ:

```typescript
@ApiSchema({ name: 'CreateCatRequest' })
class CreateCatDto {}
```

Model ở trên sẽ được dịch thành schema `CreateCatRequest`.

Theo mặc định, không có description nào được thêm vào schema được tạo. Bạn có thể thêm một sử dụng thuộc tính `description`:

```typescript
@ApiSchema({ description: 'Description of the CreateCatDto schema' })
class CreateCatDto {}
```

Bằng cách đó, description sẽ được bao gồm trong schema, như sau:

```yaml
schemas:
  CreateCatDto:
    type: object
    description: Description of the CreateCatDto schema
```