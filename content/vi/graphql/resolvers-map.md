### Resolvers

Resolvers cung cấp các hướng dẫn để biến một thao tác [GraphQL](https://graphql.org/) (một query, mutation, hoặc subscription) thành dữ liệu. Chúng trả về cùng hình dạng dữ liệu mà chúng ta chỉ định trong schema của mình -- một cách đồng bộ hoặc dưới dạng một promise giải quyết thành kết quả của hình dạng đó. Thông thường, bạn tạo một **resolver map** theo cách thủ công. Ngược lại, gói `@nestjs/graphql` tạo một resolver map tự động sử dụng siêu dữ liệu được cung cấp bởi các decorator bạn sử dụng để chú thích các lớp. Để chứng minh quá trình sử dụng các tính năng của gói để tạo một GraphQL API, chúng ta sẽ tạo một API tác giả đơn giản.

#### Code first

Trong phương pháp code-first, chúng ta không theo quy trình điển hình của việc tạo schema GraphQL của mình bằng cách viết GraphQL SDL theo cách thủ công. Thay vào đó, chúng ta sử dụng decorators TypeScript để tạo SDL từ các định nghĩa lớp TypeScript. Gói `@nestjs/graphql` đọc siêu dữ liệu được định nghĩa thông qua các decorator và tự động tạo schema cho bạn.

#### Object types

Hầu hết các định nghĩa trong một schema GraphQL là **object types**. Mỗi object type bạn định nghĩa nên đại diện cho một đối tượng domain mà ứng dụng client có thể cần tương tác. Ví dụ, API mẫu của chúng ta cần có thể lấy danh sách các tác giả và bài viết của họ, vì vậy chúng ta nên định nghĩa loại `Author` và loại `Post` để hỗ trợ chức năng này.

Nếu chúng ta đang sử dụng phương pháp schema-first, chúng ta sẽ định nghĩa một schema như vậy với SDL như sau:

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post!]!
}
```

Trong trường hợp này, sử dụng phương pháp code first, chúng ta định nghĩa schema sử dụng các lớp TypeScript và sử dụng decorators TypeScript để chú thích các trường của các lớp đó. Tương đương với SDL trên trong phương pháp code first là:

```typescript
@@filename(authors/models/author.model)
import { Field, Int, ObjectType } from '@nestjs/graphql';
import { Post } from './post';

@ObjectType()
export class Author {
  @Field(type => Int)
  id: number;

  @Field({ nullable: true })
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  @Field(type => [Post])
  posts: Post[];
}
```

> info **Gợi ý** Hệ thống phản ánh siêu dữ liệu của TypeScript có một số hạn chế khiến nó không thể, ví dụ, xác định các thuộc tính mà một lớp bao gồm hoặc nhận biết liệu một thuộc tính nhất định là tùy chọn hay bắt buộc. Vì những hạn chế này, chúng ta phải hoặc sử dụng decorator `@Field()` một cách rõ ràng trong các lớp định nghĩa schema của mình để cung cấp siêu dữ liệu về loại GraphQL và tính tùy chọn của mỗi trường, hoặc sử dụng [CLI plugin](/graphql/cli-plugin) để tạo chúng cho chúng ta.

Object type `Author`, như bất kỳ lớp nào, được tạo từ một tập hợp các trường, với mỗi trường khai báo một loại. Loại của một trường tương ứng với một [loại GraphQL](https://graphql.org/learn/schema/). Loại GraphQL của một trường có thể là một object type khác hoặc một scalar type. Loại scalar GraphQL là một kiểu nguyên thủy (như `ID`, `String`, `Boolean`, hoặc `Int`) giải quyết thành một giá trị đơn.

> info **Gợi ý** Ngoài các loại scalar tích hợp sẵn của GraphQL, bạn có thể định nghĩa các loại scalar tùy chỉnh (đọc [thêm](/graphql/scalars)).

Định nghĩa object type `Author` ở trên sẽ khiến Nest **tạo** SDL mà chúng ta đã hiển thị ở trên:

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post!]!
}
```

Decorator `@Field()` chấp nhận một hàm loại tùy chọn (ví dụ, `type => Int`), và tùy chọn một đối tượng options.

Hàm loại được yêu cầu khi có khả năng mơ hồ giữa hệ thống loại TypeScript và hệ thống loại GraphQL. Cụ thể: nó **không** được yêu cầu cho các loại `string` và `boolean`; nó **được** yêu cầu cho `number` (phải được ánh xạ đến một GraphQL `Int` hoặc `Float`). Hàm loại chỉ cần trả về loại GraphQL mong muốn (như được hiển thị trong các ví dụ khác nhau trong các chương này).

Đối tượng options có thể có bất kỳ cặp khóa/giá trị nào sau đây:

- `nullable`: để chỉ định liệu một trường có thể null hay không (trong `@nestjs/graphql`, mỗi trường không thể null theo mặc định); `boolean`
- `description`: để đặt mô tả trường; `string`
- `deprecationReason`: để đánh dấu một trường là deprecated; `string`

Ví dụ:

```typescript
@Field({ description: `Book title`, deprecationReason: 'Not useful in v2 schema' })
title: string;
```

> info **Gợi ý** Bạn cũng có thể thêm mô tả vào, hoặc deprecated, toàn bộ object type: `@ObjectType({{ '{' }} description: 'Author model' {{ '}' }})`.

Khi trường là một mảng, chúng ta phải chỉ định thủ công loại mảng trong hàm loại của decorator `Field()`, như được hiển thị dưới đây:

```typescript
@Field(type => [Post])
posts: Post[];
```

> info **Gợi ý** Sử dụng ký hiệu ngoặc vuông mảng (`[ ]`), chúng ta có thể chỉ định độ sâu của mảng. Ví dụ, sử dụng `[[Int]]` sẽ đại diện cho một ma trận số nguyên.

Để khai báo rằng các mục của một mảng (không phải mảng chính nó) có thể null, đặt thuộc tính `nullable` thành `'items'` như được hiển thị dưới đây:

```typescript
@Field(type => [Post], { nullable: 'items' })
posts: Post[];
```

> info **Gợi ý** Nếu cả mảng và các mục của nó đều có thể null, đặt `nullable` thành `'itemsAndList'` thay thế.

Bây giờ mà object type `Author` đã được tạo, hãy định nghĩa object type `Post`.

```typescript
@@filename(posts/models/post.model)
import { Field, Int, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Post {
  @Field(type => Int)
  id: number;

  @Field()
  title: string;

  @Field(type => Int, { nullable: true })
  votes?: number;
}
```

Object type `Post` sẽ dẫn đến việc tạo phần sau của schema GraphQL trong SDL:

```graphql
type Post {
  id: Int!
  title: String!
  votes: Int
}
```

#### Resolver code first

Tại thời điểm này, chúng ta đã định nghĩa các đối tượng (định nghĩa loại) có thể tồn tại trong đồ thị dữ liệu của chúng ta, nhưng các client chưa có cách để tương tác với các đối tượng đó. Để giải quyết vấn đề đó, chúng ta cần tạo một lớp resolver. Trong phương pháp code first, một lớp resolver vừa định nghĩa các hàm resolver **và** tạo **Query type**. Điều này sẽ rõ ràng khi chúng ta làm việc qua ví dụ dưới đây:

```typescript
@@filename(authors/authors.resolver)
@Resolver(() => Author)
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query(() => Author)
  async author(@Args('id', { type: () => Int }) id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField()
  async posts(@Parent() author: Author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> info **Gợi ý** Tất cả các decorator (ví dụ, `@Resolver`, `@ResolveField`, `@Args`, v.v.) đều được xuất từ gói `@nestjs/graphql`.

Bạn có thể định nghĩa nhiều lớp resolver. Nest sẽ kết hợp các lớp này tại thời gian chạy. Xem phần [module](/graphql/resolvers#module) dưới đây để biết thêm về tổ chức code.

> warning **Lưu ý** Logic bên trong các lớp `AuthorsService` và `PostsService` có thể đơn giản hoặc phức tạp tùy theo nhu cầu. Điểm chính của ví dụ này là để hiển thị cách xây dựng resolvers và cách chúng có thể tương tác với các nhà cung cấp khác.

Trong ví dụ trên, chúng ta đã tạo `AuthorsResolver`, định nghĩa một hàm resolver query và một hàm resolver trường. Để tạo một resolver, chúng ta tạo một lớp với các hàm resolver dưới dạng phương thức, và chú thích lớp với decorator `@Resolver()`.

Trong ví dụ này, chúng ta đã định nghĩa một bộ xử lý query để lấy đối tượng tác giả dựa trên `id` được gửi trong yêu cầu. Để chỉ định rằng phương thức là một bộ xử lý query, sử dụng decorator `@Query()`.

Đối số được chuyển cho decorator `@Resolver()` là tùy chọn, nhưng có hiệu lực khi đồ thị của chúng ta trở nên không tầm thường. Nó được sử dụng để cung cấp một đối tượng cha được sử dụng bởi các hàm resolver trường khi chúng đi xuống qua một đồ thị đối tượng.

Trong ví dụ của chúng ta, vì lớp bao gồm một hàm **resolver trường** (cho thuộc tính `posts` của object type `Author`), chúng ta **phải** cung cấp decorator `@Resolver()` với một giá trị để chỉ định lớp nào là loại cha (tức là tên lớp `ObjectType` tương ứng) cho tất cả các resolver trường được định nghĩa trong lớp này. Như nên rõ từ ví dụ, khi viết một hàm resolver trường, cần thiết để truy cập đối tượng cha (đối tượng mà trường đang được giải quyết là thành viên của). Trong ví dụ này, chúng ta điền mảng bài viết của một tác giả với một resolver trường gọi một service nhận `id` của tác giả làm đối số. Do đó, nhu cầu xác định đối tượng cha trong decorator `@Resolver()`. Lưu ý việc sử dụng tương ứng của decorator tham số phương thức `@Parent()` để sau đó trích xuất tham chiếu đến đối tượng cha đó trong resolver trường.

Chúng ta có thể định nghĩa nhiều hàm resolver `@Query()` (cả trong lớp này và trong bất kỳ lớp resolver nào khác), và chúng sẽ được tổng hợp thành một định nghĩa **Query type** duy nhất trong SDL được tạo cùng với các mục phù hợp trong resolver map. Điều này cho phép bạn định nghĩa các query gần với các mô hình và services mà chúng sử dụng, và giữ chúng được tổ chức tốt trong các module.

> info **Gợi ý** Nest CLI cung cấp một generator (schematic) tự động tạo **tất cả code boilerplate** để giúp chúng ta tránh phải làm tất cả điều này, và làm cho trải nghiệm nhà phát triển đơn giản hơn nhiều. Đọc thêm về tính năng này [ở đây](/recipes/crud-generator).

#### Tên loại Query

Trong các ví dụ trên, decorator `@Query()` tạo tên loại query schema GraphQL dựa trên tên phương thức. Ví dụ, hãy xem xét cấu trúc sau từ ví dụ trên:

```typescript
@Query(() => Author)
async author(@Args('id', { type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

Điều này tạo mục sau cho query tác giả trong schema của chúng ta (loại query sử dụng cùng tên với tên phương thức):

```graphql
type Query {
  author(id: Int!): Author
}
```

> info **Gợi ý** Tìm hiểu thêm về các query GraphQL [ở đây](https://graphql.org/learn/queries/).

Theo quy ước, chúng ta thích tách rời các tên này; ví dụ, chúng ta thích sử dụng một tên như `getAuthor()` cho phương thức bộ xử lý query của chúng ta, nhưng vẫn sử dụng `author` cho tên loại query của chúng ta. Điều tương tự áp dụng cho các resolver trường của chúng ta. Chúng ta có thể dễ dàng làm điều này bằng cách chuyển tên ánh xạ làm đối số của các decorator `@Query()` và `@ResolveField()`, như được hiển thị dưới đây:

```typescript
@@filename(authors/authors.resolver)
@Resolver(() => Author)
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query(() => Author, { name: 'author' })
  async getAuthor(@Args('id', { type: () => Int }) id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField('posts', () => [Post])
  async getPosts(@Parent() author: Author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

Phương thức bộ xử lý `getAuthor` ở trên sẽ dẫn đến việc tạo phần sau của schema GraphQL trong SDL:

```graphql
type Query {
  author(id: Int!): Author
}
```

#### Tùy chọn decorator Query

Đối tượng options của decorator `@Query()` (nơi chúng ta chuyển `{{ '{' }}name: 'author'{{ '}' }}` ở trên) chấp nhận một số cặp khóa/giá trị:

- `name`: tên của query; một `string`
- `description`: một mô tả sẽ được sử dụng để tạo tài liệu schema GraphQL (ví dụ, trong GraphQL playground); một `string`
- `deprecationReason`: đặt siêu dữ liệu query để hiển thị query là deprecated (ví dụ, trong GraphQL playground); một `string`
- `nullable`: liệu query có thể trả về phản hồi dữ liệu null hay không; `boolean` hoặc `'items'` hoặc `'itemsAndList'` (xem ở trên để biết chi tiết về `'items'` và `'itemsAndList'`)

#### Tùy chọn decorator Args

Sử dụng decorator `@Args()` để trích xuất đối số từ một yêu cầu để sử dụng trong phương thức bộ xử lý. Điều này hoạt động theo cách rất giống với [trích xuất đối số tham số route REST](/controllers#route-parameters).

Thông thường, decorator `@Args()` của bạn sẽ đơn giản và không yêu cầu một đối số đối tượng, như được thấy với phương thức `getAuthor()` ở trên. Ví dụ, nếu loại của một định danh là chuỗi, cấu trúc sau là đủ, và chỉ đơn giản chọn trường được đặt tên từ yêu cầu GraphQL inbound để sử dụng làm đối số phương thức.

```typescript
@Args('id') id: string
```

Trong trường hợp `getAuthor()`, loại `number` được sử dụng, điều này trình bày một thách thức. Loại TypeScript `number` không cung cấp cho chúng ta đủ thông tin về biểu diễn GraphQL mong đợi (ví dụ, `Int` so với `Float`). Do đó, chúng ta phải **rõ ràng** chuyển tham chiếu loại. Chúng ta làm điều đó bằng cách chuyển đối số thứ hai cho decorator `Args()`, chứa các tùy chọn đối số, như được hiển thị dưới đây:

```typescript
@Query(() => Author, { name: 'author' })
async getAuthor(@Args('id', { type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

Đối tượng options cho phép chúng ta chỉ định các cặp khóa-giá trị tùy chọn sau:

- `type`: một hàm trả về loại GraphQL
- `defaultValue`: một giá trị mặc định; `any`
- `description`: siêu dữ liệu mô tả; `string`
- `deprecationReason`: để deprecated một trường và cung cấp siêu dữ liệu mô tả lý do; `string`
- `nullable`: liệu trường có thể null hay không

Các phương thức bộ xử lý query có thể nhận nhiều đối số. Hãy tưởng tượng rằng chúng ta muốn lấy một tác giả dựa trên `firstName` và `lastName` của nó. Trong trường hợp này, chúng ta có thể gọi `@Args` hai lần:

```typescript
getAuthor(
  @Args('firstName', { nullable: true }) firstName?: string,
  @Args('lastName', { defaultValue: '' }) lastName?: string,
) {}
```

> info **Gợi ý** Trong trường hợp `firstName`, là một trường GraphQL có thể null, không cần thiết phải thêm các loại không giá trị của `null` hoặc `undefined` vào loại của trường này. Chỉ cần lưu ý, bạn sẽ cần type guard cho các loại không giá trị có thể này trong resolvers của mình, vì một trường GraphQL có thể null sẽ cho phép các loại đó truyền qua resolver của bạn.

#### Lớp đối số chuyên dụng

Với các cuộc gọi `@Args()` inline, code như ví dụ trên trở nên phình to. Thay vào đó, bạn có thể tạo một lớp đối số `GetAuthorArgs` chuyên dụng và truy cập nó trong phương thức bộ xử lý như sau:

```typescript
@Args() args: GetAuthorArgs
```

Tạo lớp `GetAuthorArgs` sử dụng `@ArgsType()` như được hiển thị dưới đây:

```typescript
@@filename(authors/dto/get-author.args)
import { MinLength } from 'class-validator';
import { Field, ArgsType } from '@nestjs/graphql';

@ArgsType()
class GetAuthorArgs {
  @Field({ nullable: true })
  firstName?: string;

  @Field({ defaultValue: '' })
  @MinLength(3)
  lastName: string;
}
```

> info **Gợi ý** Một lần nữa, do các hạn chế của hệ thống phản ánh siêu dữ liệu của TypeScript, cần thiết phải sử dụng decorator `@Field` để chỉ định thủ công loại và tính tùy chọn, hoặc sử dụng [CLI plugin](/graphql/cli-plugin). Ngoài ra, trong trường hợp `firstName`, là một trường GraphQL có thể null, không cần thiết phải thêm các loại không giá trị của `null` hoặc `undefined` vào loại của trường này. Chỉ cần lưu ý, bạn sẽ cần type guard cho các loại không giá trị có thể này trong resolvers của mình, vì một trường GraphQL có thể null sẽ cho phép các loại đó truyền qua resolver của bạn. 

Điều này sẽ dẫn đến việc tạo phần sau của schema GraphQL trong SDL:

```graphql
type Query {
  author(firstName: String, lastName: String = ''): Author
}
```

> info **Gợi ý** Lưu ý rằng các lớp đối số như `GetAuthorArgs` hoạt động rất tốt với `ValidationPipe` (đọc [thêm](/techniques/validation)).

#### Kế thừa lớp

Bạn có thể sử dụng kế thừa lớp TypeScript tiêu chuẩn để tạo các lớp cơ sở với các tính năng loại tiện ích chung (trường và thuộc tính trường, xác thực, v.v.) có thể được mở rộng. Ví dụ, bạn có thể có một tập hợp các đối số liên quan đến phân trang luôn bao gồm các trường tiêu chuẩn `offset` và `limit`, nhưng cũng các trường index khác là loại cụ thể. Bạn có thể thiết lập một hệ thống lớp như được hiển thị dưới đây.

Lớp cơ sở `@ArgsType()`:

```typescript
@ArgsType()
class PaginationArgs {
  @Field(() => Int)
  offset: number = 0;

  @Field(() => Int)
  limit: number = 10;
}
```

Lớp con loại cụ thể của lớp cơ sở `@ArgsType()`:

```typescript
@ArgsType()
class GetAuthorArgs extends PaginationArgs {
  @Field({ nullable: true })
  firstName?: string;

  @Field({ defaultValue: '' })
  @MinLength(3)
  lastName: string;
}
```

Cách tiếp cận tương tự có thể được thực hiện với các đối tượng `@ObjectType()`. Định nghĩa các thuộc tính chung trên lớp cơ sở:

```typescript
@ObjectType()
class Character {
  @Field(() => Int)
  id: number;

  @Field()
  name: string;
}
```

Thêm các thuộc tính loại cụ thể trên các lớp con:

```typescript
@ObjectType()
class Warrior extends Character {
  @Field()
  level: number;
}
```

Bạn có thể sử dụng kế thừa với một resolver cũng. Bạn có thể đảm bảo an toàn kiểu bằng cách kết hợp kế thừa và generics TypeScript. Ví dụ, để tạo một lớp cơ sở với một query `findAll` chung, sử dụng một cấu trúc như sau:

```typescript
function BaseResolver<T extends Type<unknown>>(classRef: T): any {
  @Resolver({ isAbstract: true })
  abstract class BaseResolverHost {
    @Query(() => [classRef], { name: `findAll${classRef.name}` })
    async findAll(): Promise<T[]> {
      return [];
    }
  }
  return BaseResolverHost;
}
```

Lưu ý những điều sau:

- Một loại trả về rõ ràng (`any` ở trên) được yêu cầu; nếu không, TypeScript phàn nàn về việc sử dụng định nghĩa lớp riêng tư. Được khuyến nghị: định nghĩa một interface thay vì sử dụng `any`.
- `Type` được nhập từ gói `@nestjs/common`
- Thuộc tính `isAbstract: true` chỉ ra rằng SDL (các câu lệnh Schema Definition Language) không nên được tạo cho lớp này. Lưu ý, bạn có thể đặt thuộc tính này cho các loại khác cũng để ngăn chặn việc tạo SDL.

Dưới đây là cách bạn có thể tạo một lớp con cụ thể của `BaseResolver`:

```typescript
@Resolver(() => Recipe)
export class RecipesResolver extends BaseResolver(Recipe) {
  constructor(private recipesService: RecipesService) {
    super();
  }
}
```

Cấu trúc này sẽ tạo SDL sau:

```graphql
type Query {
  findAllRecipe: [Recipe!]!
}
```

#### Generics

Chúng ta đã thấy một sử dụng của generics ở trên. Tính năng TypeScript mạnh mẽ này có thể được sử dụng để tạo các trừu tượng hữu ích. Ví dụ, dưới đây là một mẫu triển khai phân trang dựa trên cursor dựa trên [tài liệu này](https://graphql.org/learn/pagination/#pagination-and-edges):

```typescript
import { Field, ObjectType, Int } from '@nestjs/graphql';
import { Type } from '@nestjs/common';

interface IEdgeType<T> {
  cursor: string;
  node: T;
}

export interface IPaginatedType<T> {
  edges: IEdgeType<T>[];
  nodes: T[];
  totalCount: number;
  hasNextPage: boolean;
}

export function Paginated<T>(classRef: Type<T>): Type<IPaginatedType<T>> {
  @ObjectType(`${classRef.name}Edge`)
  abstract class EdgeType {
    @Field(() => String)
    cursor: string;

    @Field(() => classRef)
    node: T;
  }

  @ObjectType({ isAbstract: true })
  abstract class PaginatedType implements IPaginatedType<T> {
    @Field(() => [EdgeType], { nullable: true })
    edges: EdgeType[];

    @Field(() => [classRef], { nullable: true })
    nodes: T[];

    @Field(() => Int)
    totalCount: number;

    @Field()
    hasNextPage: boolean;
  }
  return PaginatedType as Type<IPaginatedType<T>>;
}
```

Với lớp cơ sở trên được định nghĩa, bây giờ chúng ta có thể dễ dàng tạo các loại chuyên dụng kế thừa hành vi này. Ví dụ:

```typescript
@ObjectType()
class PaginatedAuthor extends Paginated(Author) {}
```

#### Schema first

Như đã đề cập trong chương [trước](/graphql/quick-start), trong phương pháp schema-first, chúng ta bắt đầu bằng cách định nghĩa thủ công các loại schema trong SDL (đọc [thêm](https://graphql.org/learn/schema/#type-language)). Hãy xem xét các định nghĩa loại SDL sau.

> info **Gợi ý** Để thuận tiện trong chương này, chúng ta đã tổng hợp tất cả SDL ở một vị trí (ví dụ, một tệp `.graphql`, như được hiển thị dưới đây). Trong thực tế, bạn có thể thấy phù hợp để tổ chức code của mình theo cách mô-đun. Ví dụ, có thể hữu ích để tạo các tệp SDL riêng lẻ với các định nghĩa loại đại diện cho mỗi thực thể domain, cùng với các dịch vụ liên quan, code resolver, và lớp định nghĩa module Nest, trong một thư mục chuyên dụng cho thực thể đó. Nest sẽ tổng hợp tất cả các định nghĩa loại schema riêng lẻ tại thời gian chạy.

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String!
  votes: Int
}

type Query {
  author(id: Int!): Author
}
```

#### Resolver schema first

Schema trên phơi bày một query duy nhất - `author(id: Int!): Author`.

> info **Gợi ý** Tìm hiểu thêm về các query GraphQL [ở đây](https://graphql.org/learn/queries/).

Bây giờ hãy tạo một lớp `AuthorsResolver` giải quyết các query tác giả:

```typescript
@@filename(authors/authors.resolver)
@Resolver('Author')
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query()
  async author(@Args('id') id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField()
  async posts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> info **Gợi ý** Tất cả các decorator (ví dụ, `@Resolver`, `@ResolveField`, `@Args`, v.v.) đều được xuất từ gói `@nestjs/graphql`.

> warning **Lưu ý** Logic bên trong các lớp `AuthorsService` và `PostsService` có thể đơn giản hoặc phức tạp tùy theo nhu cầu. Điểm chính của ví dụ này là để hiển thị cách xây dựng resolvers và cách chúng có thể tương tác với các nhà cung cấp khác.

Decorator `@Resolver()` được yêu cầu. Nó nhận một đối số chuỗi tùy chọn với tên của một lớp. Tên lớp này được yêu cầu bất cứ khi nào lớp bao gồm các decorator `@ResolveField()` để thông báo cho Nest rằng phương thức được trang trí được liên kết với một loại cha (loại `Author` trong ví dụ hiện tại của chúng ta). Ngoài ra, thay vì đặt `@Resolver()` ở đầu lớp, điều này có thể được thực hiện cho từng phương thức:

```typescript
@Resolver('Author')
@ResolveField()
async posts(@Parent() author) {
  const { id } = author;
  return this.postsService.findAll({ authorId: id });
}
```

Trong trường hợp này (decorator `@Resolver()` ở cấp phương thức), nếu bạn có nhiều decorator `@ResolveField()` bên trong một lớp, bạn phải thêm `@Resolver()` vào tất cả chúng. Điều này không được coi là thực hành tốt nhất (vì nó tạo ra chi phí thêm).

> info **Gợi ý** Bất kỳ đối số tên lớp nào được chuyển cho `@Resolver()` **không** ảnh hưởng đến các query (decorator `@Query()`) hoặc mutations (decorator `@Mutation()`).

> warning **Cảnh báo** Sử dụng decorator `@Resolver` ở cấp phương thức không được hỗ trợ với phương pháp **code first**.

Trong các ví dụ trên, các decorator `@Query()` và `@ResolveField()` được liên kết với các loại schema GraphQL dựa trên tên phương thức. Ví dụ, hãy xem xét cấu trúc sau từ ví dụ trên:

```typescript
@Query()
async author(@Args('id') id: number) {
  return this.authorsService.findOneById(id);
}
```

Điều này tạo mục sau cho query tác giả trong schema của chúng ta (loại query sử dụng cùng tên với tên phương thức):

```graphql
type Query {
  author(id: Int!): Author
}
```

Theo quy ước, chúng ta sẽ thích tách rời chúng, sử dụng các tên như `getAuthor()` hoặc `getPosts()` cho các phương thức resolver của chúng ta. Chúng ta có thể dễ dàng làm điều này bằng cách chuyển tên ánh xạ làm đối số cho decorator, như được hiển thị dưới đây:

```typescript
@@filename(authors/authors.resolver)
@Resolver('Author')
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query('author')
  async getAuthor(@Args('id') id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField('posts')
  async getPosts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> info **Gợi ý** Nest CLI cung cấp một generator (schematic) tự động tạo **tất cả code boilerplate** để giúp chúng ta tránh phải làm tất cả điều này, và làm cho trải nghiệm nhà phát triển đơn giản hơn nhiều. Đọc thêm về tính năng này [ở đây](/recipes/crud-generator).

#### Tạo types

Giả sử rằng chúng ta sử dụng phương pháp schema-first và đã kích hoạt tính năng tạo typings (với `outputAs: 'class'` như được hiển thị trong chương [trước](/graphql/quick-start)), sau khi bạn chạy ứng dụng, nó sẽ tạo tệp sau (tại vị trí bạn chỉ định trong phương thức `GraphQLModule.forRoot()`). Ví dụ, trong `src/graphql.ts`:

```typescript
@@filename(graphql)
export class Author {
  id: number;
  firstName?: string;
  lastName?: string;
  posts?: Post[];
}
export class Post {
  id: number;
  title: string;
  votes?: number;
}

export abstract class IQuery {
  abstract author(id: number): Author | Promise<Author>;
}
```

Bằng cách tạo các lớp (thay vì kỹ thuật mặc định của việc tạo interfaces), bạn có thể sử dụng các decorator xác thực khai báo kết hợp với phương pháp schema-first, là một kỹ thuật cực kỳ hữu ích (đọc [thêm](/techniques/validation)). Ví dụ, bạn có thể thêm các decorator `class-validator` vào lớp `CreatePostInput` được tạo như được hiển thị dưới đây để thực thi độ dài chuỗi tối thiểu và tối đa trên trường `title`:

```typescript
import { MinLength, MaxLength } from 'class-validator';

export class CreatePostInput {
  @MinLength(3)
  @MaxLength(50)
  title: string;
}
```

> warning **Lưu ý** Để kích hoạt tự động xác thực các đầu vào (và tham số) của bạn, sử dụng `ValidationPipe`. Đọc thêm về xác thực [ở đây](/techniques/validation) và cụ thể hơn về pipes [ở đây](/pipes).

Tuy nhiên, nếu bạn thêm các decorator trực tiếp vào tệp được tạo tự động, chúng sẽ bị **ghi đè** mỗi khi tệp được tạo. Thay vào đó, tạo một tệp riêng và chỉ cần mở rộng lớp được tạo.

```typescript
import { MinLength, MaxLength } from 'class-validator';
import { Post } from '../../graphql.ts';

export class CreatePostInput extends Post {
  @MinLength(3)
  @MaxLength(50)
  title: string;
}
```

#### Decorator đối số GraphQL

Chúng ta có thể truy cập các đối số resolver GraphQL tiêu chuẩn sử dụng các decorator chuyên dụng. Dưới đây là so sánh giữa các decorator Nest và các tham số Apollo đơn giản mà chúng đại diện.

<table>
  <tbody>
    <tr>
      <td><code>@Root()</code> và <code>@Parent()</code></td>
      <td><code>root</code>/<code>parent</code></td>
    </tr>
    <tr>
      <td><code>@Context(param?: string)</code></td>
      <td><code>context</code> / <code>context[param]</code></td>
    </tr>
    <tr>
      <td><code>@Info(param?: string)</code></td>
      <td><code>info</code> / <code>info[param]</code></td>
    </tr>
    <tr>
      <td><code>@Args(param?: string)</code></td>
      <td><code>args</code> / <code>args[param]</code></td>
    </tr>
  </tbody>
</table>

Các đối số này có các nghĩa sau:

- `root`: một đối tượng chứa kết quả được trả về từ resolver trên trường cha, hoặc, trong trường hợp của trường `Query` cấp cao nhất, `rootValue` được chuyển từ cấu hình server.
- `context`: một đối tượng được chia sẻ bởi tất cả resolvers trong một query cụ thể; thường được sử dụng để chứa trạng thái mỗi yêu cầu.
- `info`: một đối tượng chứa thông tin về trạng thái thực thi của query.
- `args`: một đối tượng với các đối số được chuyển vào trường trong query.

<app-banner-devtools></app-banner-devtools>

#### Module

Sau khi chúng ta hoàn thành các bước trên, chúng ta đã chỉ định theo cách khai báo tất cả thông tin cần thiết bởi `GraphQLModule` để tạo một resolver map. `GraphQLModule` sử dụng phản ánh để nội quan siêu dữ liệu được cung cấp thông qua các decorator, và chuyển đổi các lớp thành resolver map chính xác một cách tự động.

Điều duy nhất khác bạn cần chăm sóc là **cung cấp** (tức là liệt kê dưới dạng `provider` trong một số module) lớp (các) resolver (`AuthorsResolver`), và nhập module (`AuthorsModule`) ở đâu đó, để Nest có thể sử dụng nó.

Ví dụ, chúng ta có thể làm điều này trong một `AuthorsModule`, cũng có thể cung cấp các dịch vụ khác cần thiết trong bối cảnh này. Đảm bảo nhập `AuthorsModule` ở đâu đó (ví dụ, trong module gốc, hoặc một số module khác được nhập bởi module gốc).

```typescript
@@filename(authors/authors.module)
@Module({
  imports: [PostsModule],
  providers: [AuthorsService, AuthorsResolver],
})
export class AuthorsModule {}
```

> info **Gợi ý** Nó hữu ích để tổ chức code của bạn theo **mô hình domain** của bạn (tương tự như cách bạn sẽ tổ chức các điểm vào trong một REST API). Trong cách tiếp cận này, giữ các mô hình của bạn (các lớp `ObjectType`), resolvers và services cùng nhau trong một module Nest đại diện cho mô hình domain. Giữ tất cả các thành phần này trong một thư mục duy nhất cho mỗi module. Khi bạn làm điều này và sử dụng [Nest CLI](/cli/overview) để tạo từng phần tử, Nest sẽ kết nối tất cả các phần này lại với nhau (định vị tệp trong các thư mục phù hợp, tạo các mục trong mảng `provider` và `imports`, v.v.) tự động cho bạn.