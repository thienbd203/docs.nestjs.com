### Mutations

Hầu hết các cuộc thảo luận về GraphQL tập trung vào việc lấy dữ liệu, nhưng bất kỳ nền tảng dữ liệu hoàn chỉnh nào cũng cần một cách để sửa đổi dữ liệu phía máy chủ. Trong REST, bất kỳ yêu cầu nào có thể dẫn đến việc gây ra tác dụng phụ trên máy chủ, nhưng thực hành tốt nhất đề xuất chúng ta không nên sửa đổi dữ liệu trong các yêu cầu GET. GraphQL tương tự - về mặt kỹ thuật bất kỳ query nào có thể được triển khai để gây ra việc ghi dữ liệu. Tuy nhiên, giống như REST, được khuyến nghị để quan sát quy ước rằng bất kỳ hoạt động nào gây ra ghi nên được gửi rõ ràng thông qua một mutation (đọc thêm [ở đây](https://graphql.org/learn/queries/#mutations)).

Tài liệu chính thức [Apollo](https://www.apollographql.com/docs/graphql-tools/generate-schema.html) sử dụng một ví dụ mutation `upvotePost()`. Mutation này triển khai một phương thức để tăng giá trị thuộc tính `votes` của một bài viết. Để tạo một mutation tương đương trong Nest, chúng ta sẽ sử dụng decorator `@Mutation()`.

#### Code first

Hãy thêm một phương thức khác vào `AuthorResolver` được sử dụng trong phần trước (xem [resolvers](/graphql/resolvers)).

```typescript
@Mutation(() => Post)
async upvotePost(@Args({ name: 'postId', type: () => Int }) postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

> info **Gợi ý** Tất cả các decorator (ví dụ, `@Resolver`, `@ResolveField`, `@Args`, v.v.) đều được xuất từ gói `@nestjs/graphql`.

Điều này sẽ dẫn đến việc tạo phần sau của schema GraphQL trong SDL:

```graphql
type Mutation {
  upvotePost(postId: Int!): Post
}
```

Phương thức `upvotePost()` nhận `postId` (`Int`) làm đối số và trả về một thực thể `Post` đã cập nhật. Vì những lý do được giải thích trong phần [resolvers](/graphql/resolvers), chúng ta phải đặt rõ ràng loại mong đợi.

Nếu mutation cần nhận một đối tượng làm đối số, chúng ta có thể tạo một **input type**. Input type là một loại đối tượng đặc biệt có thể được chuyển vào làm đối số (đọc thêm [ở đây](https://graphql.org/learn/schema/#input-types)). Để khai báo một input type, sử dụng decorator `@InputType()`.

```typescript
import { InputType, Field } from '@nestjs/graphql';

@InputType()
export class UpvotePostInput {
  @Field()
  postId: number;
}
```

> info **Gợi ý** Decorator `@InputType()` nhận một đối tượng options làm đối số, vì vậy bạn có thể, ví dụ, chỉ định mô tả của input type. Lưu ý rằng, do các hạn chế của hệ thống phản ánh siêu dữ liệu TypeScript, bạn phải sử dụng decorator `@Field` để chỉ định thủ công một loại, hoặc sử dụng [CLI plugin](/graphql/cli-plugin).

Sau đó chúng ta có thể sử dụng loại này trong lớp resolver:

```typescript
@Mutation(() => Post)
async upvotePost(
  @Args('upvotePostData') upvotePostData: UpvotePostInput,
) {}
```

#### Schema first

Hãy mở rộng `AuthorResolver` của chúng ta được sử dụng trong phần trước (xem [resolvers](/graphql/resolvers)).

```typescript
@Mutation()
async upvotePost(@Args('postId') postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

Lưu ý rằng chúng ta đã giả định ở trên rằng logic kinh doanh đã được chuyển đến `PostsService` (truy vấn bài viết và tăng thuộc tính `votes` của nó). Logic bên trong lớp `PostsService` có thể đơn giản hoặc tinh vi tùy theo nhu cầu. Điểm chính của ví dụ này là để hiển thị cách resolvers có thể tương tác với các nhà cung cấp khác.

Bước cuối cùng là thêm mutation của chúng ta vào định nghĩa loại hiện có.

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String
  votes: Int
}

type Query {
  author(id: Int!): Author
}

type Mutation {
  upvotePost(postId: Int!): Post
}
```

Mutation `upvotePost(postId: Int!): Post` bây giờ có sẵn để được gọi như một phần của GraphQL API của ứng dụng của chúng ta.