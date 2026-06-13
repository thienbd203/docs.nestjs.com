### Interfaces

Giống như nhiều hệ thống loại, GraphQL hỗ trợ interfaces. Một **Interface** là một loại trừu tượng bao gồm một tập hợp các trường nhất định mà một loại phải bao gồm để triển khai interface (đọc thêm [ở đây](https://graphql.org/learn/schema/#interfaces)).

#### Code first

Khi sử dụng phương pháp code first, bạn định nghĩa một interface GraphQL bằng cách tạo một lớp trừu tượng được chú thích với decorator `@InterfaceType()` được xuất từ `@nestjs/graphql`.

```typescript
import { Field, ID, InterfaceType } from '@nestjs/graphql';

@InterfaceType()
export abstract class Character {
  @Field(() => ID)
  id: string;

  @Field()
  name: string;
}
```

> warning **Cảnh báo** Các interfaces TypeScript không thể được sử dụng để định nghĩa các interfaces GraphQL.

Điều này sẽ dẫn đến việc tạo phần sau của schema GraphQL trong SDL:

```graphql
interface Character {
  id: ID!
  name: String!
}
```

Bây giờ, để triển khai interface `Character`, sử dụng khóa `implements`:

```typescript
@ObjectType({
  implements: () => [Character],
})
export class Human implements Character {
  id: string;
  name: string;
}
```

> info **Gợi ý** Decorator `@ObjectType()` được xuất từ gói `@nestjs/graphql`.

Hàm `resolveType()` mặc định được tạo bởi thư viện trích xuất loại dựa trên giá trị được trả về từ phương thức resolver. Điều này có nghĩa là bạn phải trả về các thể hiện lớp (bạn không thể trả về các đối tượng JavaScript theo nghĩa đen).

Để cung cấp một hàm `resolveType()` tùy chỉnh, chuyển thuộc tính `resolveType` cho đối tượng options được chuyển vào decorator `@InterfaceType()`, như sau:

```typescript
@InterfaceType({
  resolveType(book) {
    if (book.colors) {
      return ColoringBook;
    }
    return TextBook;
  },
})
export abstract class Book {
  @Field(() => ID)
  id: string;

  @Field()
  title: string;
}
```

#### Interface resolvers

Cho đến nay, sử dụng interfaces, bạn chỉ có thể chia sẻ các định nghĩa trường với các đối tượng của mình. Nếu bạn cũng muốn chia sẻ việc triển khai resolver trường thực tế, bạn có thể tạo một resolver interface chuyên dụng, như sau:

```typescript
import { Resolver, ResolveField, Parent, Info } from '@nestjs/graphql';

@Resolver((type) => Character) // Nhắc nhở: Character là một interface
export class CharacterInterfaceResolver {
  @ResolveField(() => [Character])
  friends(
    @Parent() character, // Đối tượng được giải quyết triển khai Character
    @Info() { parentType }, // Loại của đối tượng triển khai Character
    @Args('search', { type: () => String }) searchTerm: string,
  ) {
    // Lấy bạn bè của nhân vật
    return [];
  }
}
```

Bây giờ resolver trường `friends` được tự động đăng ký cho tất cả các loại đối tượng triển khai interface `Character`.

> warning **Cảnh báo** Điều này yêu cầu thuộc tính `inheritResolversFromInterfaces` được đặt thành true trong cấu hình `GraphQLModule`.

#### Schema first

Để định nghĩa một interface trong phương pháp schema first, chỉ cần tạo một interface GraphQL với SDL.

```graphql
interface Character {
  id: ID!
  name: String!
}
```

Sau đó, bạn có thể sử dụng tính năng tạo typings (như được hiển thị trong chương [bắt đầu nhanh](/graphql/quick-start)) để tạo các định nghĩa TypeScript tương ứng:

```typescript
export interface Character {
  id: string;
  name: string;
}
```

Interfaces yêu cầu một trường `__resolveType` thêm trong resolver map để xác định loại nào interface nên giải quyết đến. Hãy tạo một lớp `CharactersResolver` và định nghĩa phương thức `__resolveType`:

```typescript
@Resolver('Character')
export class CharactersResolver {
  @ResolveField()
  __resolveType(value) {
    if ('age' in value) {
      return Person;
    }
    return null;
  }
}
```

> info **Gợi ý** Tất cả các decorator đều được xuất từ gói `@nestjs/graphql`.