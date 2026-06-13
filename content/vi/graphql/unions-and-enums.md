### Unions

Các loại union rất giống với interfaces, nhưng chúng không được chỉ định bất kỳ trường chung nào giữa các loại (đọc thêm [ở đây](https://graphql.org/learn/schema/#union-types)). Unions hữu ích để trả về các loại dữ liệu không liên quan từ một trường duy nhất.

#### Code first

Để định nghĩa một loại union GraphQL, chúng ta phải định nghĩa các lớp mà union này sẽ được tạo thành. Theo [ví dụ](https://www.apollographql.com/docs/apollo-server/schema/unions-interfaces/#union-type) từ tài liệu Apollo, chúng ta sẽ tạo hai lớp. Đầu tiên, `Book`:

```typescript
import { Field, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Book {
  @Field()
  title: string;
}
```

Và sau đó `Author`:

```typescript
import { Field, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Author {
  @Field()
  name: string;
}
```

Với điều này đã có sẵn, đăng ký union `ResultUnion` sử dụng hàm `createUnionType` được xuất từ gói `@nestjs/graphql`:

```typescript
export const ResultUnion = createUnionType({
  name: 'ResultUnion',
  types: () => [Author, Book] as const,
});
```

> warning **Cảnh báo** Mảng được trả về bởi thuộc tính `types` của hàm `createUnionType` nên được đưa ra một const assertion. Nếu const assertion không được đưa ra, một tệp khai báo sai sẽ được tạo tại thời gian biên dịch, và một lỗi sẽ xảy ra khi sử dụng nó từ một dự án khác.

Bây giờ, chúng ta có thể tham chiếu `ResultUnion` trong query của chúng ta:

```typescript
@Query(() => [ResultUnion])
search(): Array<typeof ResultUnion> {
  return [new Author(), new Book()];
}
```

Điều này sẽ dẫn đến việc tạo phần sau của schema GraphQL trong SDL:

```graphql
type Author {
  name: String!
}

type Book {
  title: String!
}

union ResultUnion = Author | Book

type Query {
  search: [ResultUnion!]!
}
```

Hàm `resolveType()` mặc định được tạo bởi thư viện sẽ trích xuất loại dựa trên giá trị được trả về từ phương thức resolver. Điều đó có nghĩa là trả về các thể hiện lớp thay vì đối tượng JavaScript theo nghĩa đen là bắt buộc.

Để cung cấp một hàm `resolveType()` tùy chỉnh, chuyển thuộc tính `resolveType` cho đối tượng options được chuyển vào hàm `createUnionType()`, như sau:

```typescript
export const ResultUnion = createUnionType({
  name: 'ResultUnion',
  types: () => [Author, Book] as const,
  resolveType(value) {
    if (value.name) {
      return Author;
    }
    if (value.title) {
      return Book;
    }
    return null;
  },
});
```

#### Schema first

Để định nghĩa một union trong phương pháp schema first, chỉ cần tạo một union GraphQL với SDL.

```graphql
type Author {
  name: String!
}

type Book {
  title: String!
}

union ResultUnion = Author | Book
```

Sau đó, bạn có thể sử dụng tính năng tạo typings (như được hiển thị trong chương [bắt đầu nhanh](/graphql/quick-start)) để tạo các định nghĩa TypeScript tương ứng:

```typescript
export class Author {
  name: string;
}

export class Book {
  title: string;
}

export type ResultUnion = Author | Book;
```

Unions yêu cầu một trường `__resolveType` thêm trong resolver map để xác định loại nào union nên giải quyết đến. Ngoài ra, lưu ý rằng lớp `ResultUnionResolver` phải được đăng ký dưới dạng một nhà cung cấp trong bất kỳ module nào. Hãy tạo một lớp `ResultUnionResolver` và định nghĩa phương thức `__resolveType`.

```typescript
@Resolver('ResultUnion')
export class ResultUnionResolver {
  @ResolveField()
  __resolveType(value) {
    if (value.name) {
      return 'Author';
    }
    if (value.title) {
      return 'Book';
    }
    return null;
  }
}
```

> info **Gợi ý** Tất cả các decorator đều được xuất từ gói `@nestjs/graphql`.

### Enums

Các loại liệt kê là một loại scalar đặc biệt bị giới hạn ở một tập hợp các giá trị được phép cụ thể (đọc thêm [ở đây](https://graphql.org/learn/schema/#enumeration-types)). Điều này cho phép bạn:

- xác nhận rằng bất kỳ đối số nào của loại này là một trong các giá trị được phép
- giao tiếp thông qua hệ thống loại rằng một trường sẽ luôn là một trong một tập hợp hữu hạn các giá trị

#### Code first

Khi sử dụng phương pháp code first, bạn định nghĩa một loại enum GraphQL bằng cách đơn giản tạo một enum TypeScript.

```typescript
export enum AllowedColor {
  RED,
  GREEN,
  BLUE,
}
```

Với điều này đã có sẵn, đăng ký enum `AllowedColor` sử dụng hàm `registerEnumType` được xuất từ gói `@nestjs/graphql`:

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
});
```

Bây giờ bạn có thể tham chiếu `AllowedColor` trong các loại của chúng ta:

```typescript
@Field(type => AllowedColor)
favoriteColor: AllowedColor;
```

Điều này sẽ dẫn đến việc tạo phần sau của schema GraphQL trong SDL:

```graphql
enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

Để cung cấp một mô tả cho enum, chuyển thuộc tính `description` vào hàm `registerEnumType()`.

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
  description: 'The supported colors.',
});
```

Để cung cấp một mô tả cho các giá trị enum, hoặc để đánh dấu một giá trị là deprecated, chuyển thuộc tính `valuesMap`, như sau:

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
  description: 'The supported colors.',
  valuesMap: {
    RED: {
      description: 'The default color.',
    },
    BLUE: {
      deprecationReason: 'Too blue.',
    },
  },
});
```

Điều này sẽ tạo schema GraphQL sau trong SDL:

```graphql
"""
The supported colors.
"""
enum AllowedColor {
  """
  The default color.
  """
  RED
  GREEN
  BLUE @deprecated(reason: "Too blue.")
}
```

#### Schema first

Để định nghĩa một enumerator trong phương pháp schema first, chỉ cần tạo một enum GraphQL với SDL.

```graphql
enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

Sau đó bạn có thể sử dụng tính năng tạo typings (như được hiển thị trong chương [bắt đầu nhanh](/graphql/quick-start)) để tạo các định nghĩa TypeScript tương ứng:

```typescript
export enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

Đôi khi một backend buộc một giá trị khác cho một enum nội bộ so với trong API công khai. Trong ví dụ này API chứa `RED`, tuy nhiên trong resolvers chúng ta có thể sử dụng `#f00` thay thế (đọc thêm [ở đây](https://www.apollographql.com/docs/apollo-server/schema/scalars-enums/#internal-values)). Để thực hiện điều này, khai báo một đối tượng resolver cho enum `AllowedColor`:

```typescript
export const allowedColorResolver: Record<keyof typeof AllowedColor, any> = {
  RED: '#f00',
};
```

> info **Gợi ý** Tất cả các decorator đều được xuất từ gói `@nestjs/graphql`.

Sau đó sử dụng đối tượng resolver này cùng với thuộc tính `resolvers` của phương thức `GraphQLModule#forRoot()`, như sau:

```typescript
GraphQLModule.forRoot({
  resolvers: {
    AllowedColor: allowedColorResolver,
  },
});
```