### Subscriptions

Ngoài việc lấy dữ liệu sử dụng queries và sửa đổi dữ liệu sử dụng mutations, đặc tả GraphQL hỗ trợ một loại hoạt động thứ ba, gọi là `subscription`. Subscriptions GraphQL là một cách để đẩy dữ liệu từ server đến các client chọn lắng nghe các tin nhắn thời gian thực từ server. Subscriptions tương tự như queries ở chỗ chúng chỉ định một tập hợp các trường được giao cho client, nhưng thay vì trả về ngay lập tức một câu trả lời duy nhất, một kênh được mở và một kết quả được gửi cho client mỗi khi một sự kiện cụ thể xảy ra trên server.

Một trường hợp sử dụng phổ biến cho subscriptions là thông báo cho phía client về các sự kiện cụ thể, ví dụ việc tạo một đối tượng mới, các trường được cập nhật, v.v. (đọc thêm [ở đây](https://www.apollographql.com/docs/react/data/subscriptions)).

#### Kích hoạt subscriptions với driver Apollo

Để kích hoạt subscriptions, đặt thuộc tính `installSubscriptionHandlers` thành `true`.

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  installSubscriptionHandlers: true,
}),
```

> warning **Cảnh báo** Tùy chọn cấu hình `installSubscriptionHandlers` đã bị loại bỏ từ phiên bản mới nhất của Apollo server và sẽ sớm bị deprecated trong gói này cũng vậy. Theo mặc định, `installSubscriptionHandlers` sẽ fallback để sử dụng `subscriptions-transport-ws` ([đọc thêm](https://github.com/apollographql/subscriptions-transport-ws)) nhưng chúng tôi khuyên mạnh sử dụng thư viện `graphql-ws`([đọc thêm](https://github.com/enisdenjo/graphql-ws)) thay thế.

Để chuyển sang sử dụng gói `graphql-ws` thay thế, sử dụng cấu hình sau:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'graphql-ws': true
  },
}),
```

> info **Gợi ý** Bạn cũng có thể sử dụng cả hai gói (`subscriptions-transport-ws` và `graphql-ws`) cùng một lúc, ví dụ, để tương thích ngược.

#### Code first

Để tạo một subscription sử dụng phương pháp code first, chúng ta sử dụng decorator `@Subscription()` (được xuất từ gói `@nestjs/graphql`) và lớp `PubSub` từ gói `graphql-subscriptions`, cung cấp một API **publish/subscribe** đơn giản.

Bộ xử lý subscription sau chăm sóc việc **đăng ký** vào một sự kiện bằng cách gọi `PubSub#asyncIterableIterator`. Phương thức này nhận một đối số duy nhất, `triggerName`, tương ứng với tên chủ đề sự kiện.

```typescript
const pubSub = new PubSub();

@Resolver(() => Author)
export class AuthorResolver {
  // ...
  @Subscription(() => Comment)
  commentAdded() {
    return pubSub.asyncIterableIterator('commentAdded');
  }
}
```

> info **Gợi ý** Tất cả các decorator được xuất từ gói `@nestjs/graphql`, trong khi lớp `PubSub` được xuất từ gói `graphql-subscriptions`.

> warning **Lưu ý** `PubSub` là một lớp phơi bày một API `publish` và `subscribe` đơn giản. Đọc thêm về nó [ở đây](https://www.apollographql.com/docs/graphql-subscriptions/setup.html). Lưu ý rằng tài liệu Apollo cảnh báo rằng triển khai mặc định không phù hợp cho production (đọc thêm [ở đây](https://github.com/apollographql/graphql-subscriptions#getting-started-with-your-first-subscription)). Các ứng dụng production nên sử dụng một triển khai `PubSub` được hỗ trợ bởi một kho lưu trữ bên ngoài (đọc thêm [ở đây](https://github.com/apollographql/graphql-subscriptions#pubsub-implementations)).

Điều này sẽ dẫn đến việc tạo phần sau của schema GraphQL trong SDL:

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

Lưu ý rằng subscriptions, theo định nghĩa, trả về một đối tượng với một thuộc tính cấp cao nhất duy nhất có khóa là tên của subscription. Tên này được kế thừa từ tên phương thức bộ xử lý subscription (tức là `commentAdded` ở trên), hoặc được cung cấp rõ ràng bằng cách chuyển một tùy chọn với khóa `name` làm đối số thứ hai cho decorator `@Subscription()`, như được hiển thị dưới đây.

```typescript
@Subscription(() => Comment, {
  name: 'commentAdded',
})
subscribeToCommentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

Cấu trúc này tạo ra cùng SDL với mẫu code trước, nhưng cho phép chúng ta tách rời tên phương thức khỏi subscription.

#### Publishing

Bây giờ, để xuất bản sự kiện, chúng ta sử dụng phương thức `PubSub#publish`. Điều này thường được sử dụng trong một mutation để kích hoạt cập nhật phía client khi một phần của đồ thị đối tượng đã thay đổi. Ví dụ:

```typescript
@@filename(posts/posts.resolver)
@Mutation(() => Comment)
async addComment(
  @Args('postId', { type: () => Int }) postId: number,
  @Args('comment', { type: () => Comment }) comment: CommentInput,
) {
  const newComment = this.commentsService.addComment({ id: postId, comment });
  pubSub.publish('commentAdded', { commentAdded: newComment });
  return newComment;
}
```

Phương thức `PubSub#publish` nhận một `triggerName` (một lần nữa, hãy nghĩ về điều này như một tên chủ đề sự kiện) làm tham số đầu tiên, và một payload sự kiện làm tham số thứ hai. Như đã đề cập, subscription, theo định nghĩa, trả về một giá trị và giá trị đó có một hình dạng. Hãy nhìn lại SDL được tạo cho subscription `commentAdded` của chúng ta:

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

Điều này cho chúng ta biết rằng subscription phải trả về một đối tượng với tên thuộc tính cấp cao nhất của `commentAdded` có một giá trị là một đối tượng `Comment`. Điểm quan trọng cần lưu ý là hình dạng của payload sự kiện được phát ra bởi phương thức `PubSub#publish` phải tương ứng với hình dạng của giá trị mong đợi trả về từ subscription. Vì vậy, trong ví dụ của chúng ta ở trên, câu lệnh `pubSub.publish('commentAdded', {{ '{' }} commentAdded: newComment {{ '}' }})` xuất bản một sự kiện `commentAdded` với payload có hình dạng phù hợp. Nếu các hình dạng này không khớp, subscription của bạn sẽ thất bại trong giai đoạn xác thực GraphQL.

#### Lọc subscriptions

Để lọc ra các sự kiện cụ thể, đặt thuộc tính `filter` thành một hàm lọc. Hàm này hoạt động tương tự như hàm được chuyển cho một mảng `filter`. Nó nhận hai đối số: `payload` chứa payload sự kiện (như được gửi bởi nhà xuất bản sự kiện), và `variables` nhận bất kỳ đối số nào được chuyển trong trong yêu cầu subscription. Nó trả về một boolean xác định liệu sự kiện này có nên được xuất bản cho người nghe client hay không.

```typescript
@Subscription(() => Comment, {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded(@Args('title') title: string) {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

#### Thay đổi payload subscription

Để thay đổi payload sự kiện được xuất bản, đặt thuộc tính `resolve` thành một hàm. Hàm nhận payload sự kiện (như được gửi bởi nhà xuất bản sự kiện) và trả về giá trị phù hợp.

```typescript
@Subscription(() => Comment, {
  resolve: value => value,
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

> warning **Lưu ý** Nếu bạn sử dụng tùy chọn `resolve`, bạn nên trả về payload chưa được gói (ví dụ, với ví dụ của chúng ta, trả về một đối tượng `newComment` trực tiếp, không phải một đối tượng `{{ '{' }} commentAdded: newComment {{ '}' }}`).

Nếu bạn cần truy cập các nhà cung cấp được inject (ví dụ, sử dụng một dịch vụ bên ngoài để xác thực dữ liệu), sử dụng cấu trúc sau.

```typescript
@Subscription(() => Comment, {
  resolve(this: AuthorResolver, value) {
    // "this" refers to an instance of "AuthorResolver"
    return value;
  }
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

Cấu trúc tương tự hoạt động với các bộ lọc:

```typescript
@Subscription(() => Comment, {
  filter(this: AuthorResolver, payload, variables) {
    // "this" refers to an instance of "AuthorResolver"
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

#### Schema first

Để tạo một subscription tương đương trong Nest, chúng ta sẽ sử dụng decorator `@Subscription()`.

```typescript
const pubSub = new PubSub();

@Resolver('Author')
export class AuthorResolver {
  // ...
  @Subscription()
  commentAdded() {
    return pubSub.asyncIterableIterator('commentAdded');
  }
}
```

Để lọc ra các sự kiện cụ thể dựa trên context và đối số, đặt thuộc tính `filter`.

```typescript
@Subscription('commentAdded', {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

Để thay đổi payload được xuất bản, chúng ta có thể sử dụng một hàm `resolve`.

```typescript
@Subscription('commentAdded', {
  resolve: value => value,
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

Nếu bạn cần truy cập các nhà cung cấp được inject (ví dụ, sử dụng một dịch vụ bên ngoài để xác thực dữ liệu), sử dụng cấu trúc sau.

```typescript
@Subscription('commentAdded', {
  resolve(this: AuthorResolver, value) {
    // "this" refers to an instance of "AuthorResolver"
    return value;
  }
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

Cấu trúc tương tự hoạt động với các bộ lọc:

```typescript
@Subscription('commentAdded', {
  filter(this: AuthorResolver, payload, variables) {
    // "this" refers to an instance of "AuthorResolver"
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

Bước cuối cùng là cập nhật tệp định nghĩa loại.

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

type Comment {
  id: String
  content: String
}

type Subscription {
  commentAdded(title: String!): Comment
}
```

Với điều này, chúng ta đã tạo một subscription `commentAdded(title: String!): Comment` duy nhất. Bạn có thể tìm thấy một triển khai mẫu đầy đủ [ở đây](https://github.com/nestjs/nest/blob/master/sample/12-graphql-schema-first).

#### PubSub

Chúng ta đã khởi tạo một thể hiện `PubSub` cục bộ ở trên. Cách tiếp cận ưu tiên là định nghĩa `PubSub` dưới dạng một [nhà cung cấp](/fundamentals/custom-providers) và inject nó thông qua constructor (sử dụng decorator `@Inject()`). Điều này cho phép chúng ta tái sử dụng thể hiện trên toàn bộ ứng dụng. Ví dụ, định nghĩa một nhà cung cấp như sau, sau đó inject `'PUB_SUB'` nơi cần thiết.

```typescript
{
  provide: 'PUB_SUB',
  useValue: new PubSub(),
}
```

#### Tùy chỉnh server subscriptions

Để tùy chỉnh server subscriptions (ví dụ, thay đổi đường dẫn), sử dụng thuộc tính options `subscriptions`.

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'subscriptions-transport-ws': {
      path: '/graphql'
    },
  }
}),
```

Nếu bạn đang sử dụng gói `graphql-ws` cho subscriptions, thay thế khóa `subscriptions-transport-ws` bằng `graphql-ws`, như sau:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'graphql-ws': {
      path: '/graphql'
    },
  }
}),
```

#### Xác thực qua WebSockets

Việc kiểm tra xem người dùng có được xác thực hay không có thể được thực hiện bên trong hàm callback `onConnect` mà bạn có thể chỉ định trong options `subscriptions`.

`onConnect` sẽ nhận làm đối số đầu tiên `connectionParams` được chuyển cho `SubscriptionClient` (đọc [thêm](https://www.apollographql.com/docs/react/data/subscriptions/#5-authenticate-over-websocket-optional)).

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'subscriptions-transport-ws': {
      onConnect: (connectionParams) => {
        const authToken = connectionParams.authToken;
        if (!isValid(authToken)) {
          throw new Error('Token is not valid');
        }
        // extract user information from token
        const user = parseToken(authToken);
        // return user info to add them to the context later
        return { user };
      },
    }
  },
  context: ({ connection }) => {
    // connection.context will be equal to what was returned by the "onConnect" callback
  },
}),
```

`authToken` trong ví dụ này chỉ được gửi một lần bởi client, khi kết nối được thiết lập lần đầu tiên.
Tất cả các subscriptions được thực hiện với kết nối này sẽ có cùng `authToken`, và do đó cùng thông tin người dùng.

> warning **Lưu ý** Có một lỗi trong `subscriptions-transport-ws` cho phép các kết nối bỏ qua giai đoạn `onConnect` (đọc [thêm](https://github.com/apollographql/subscriptions-transport-ws/issues/349)). Bạn không nên giả định rằng `onConnect` được gọi khi người dùng bắt đầu một subscription, và luôn kiểm tra rằng `context` được điền.

Nếu bạn đang sử dụng gói `graphql-ws`, chữ ký của callback `onConnect` sẽ hơi khác một chút:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'graphql-ws': {
      onConnect: (context: Context<any>) => {
        const { connectionParams, extra } = context;
        // user validation will remain the same as in the example above
        // when using with graphql-ws, additional context value should be stored in the extra field
        extra.user = { user: {} };
      },
    },
  },
  context: ({ extra }) => {
    // you can now access your additional context value through the extra field
  },
});
```

#### Kích hoạt subscriptions với driver Mercurius

Để kích hoạt subscriptions, đặt thuộc tính `subscription` thành `true`.

```typescript
GraphQLModule.forRoot<MercuriusDriverConfig>({
  driver: MercuriusDriver,
  subscription: true,
}),
```

> info **Gợi ý** Bạn cũng có thể chuyển đối tượng options để thiết lập một emitter tùy chỉnh, xác thực các kết nối inbound, v.v. Đọc thêm [ở đây](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md#plugin-options) (xem `subscription`).

#### Code first

Để tạo một subscription sử dụng phương pháp code first, chúng ta sử dụng decorator `@Subscription()` (được xuất từ gói `@nestjs/graphql`) và lớp `PubSub` từ gói `mercurius`, cung cấp một API **publish/subscribe** đơn giản.

Bộ xử lý subscription sau chăm sóc việc **đăng ký** vào một sự kiện bằng cách gọi `PubSub#asyncIterableIterator`. Phương thức này nhận một đối số duy nhất, `triggerName`, tương ứng với tên chủ đề sự kiện.

```typescript
@Resolver(() => Author)
export class AuthorResolver {
  // ...
  @Subscription(() => Comment)
  commentAdded(@Context('pubsub') pubSub: PubSub) {
    return pubSub.subscribe('commentAdded');
  }
}
```

> info **Gợi ý** Tất cả các decorator được sử dụng trong ví dụ trên được xuất từ gói `@nestjs/graphql`, trong khi lớp `PubSub` được xuất từ gói `mercurius`.

> warning **Lưu ý** `PubSub` là một lớp phơi bày một API `publish` và `subscribe` đơn giản. Kiểm tra [phần này](https://github.com/mercurius-js/mercurius/blob/master/docs/subscriptions.md#subscriptions-with-custom-pubsub) về cách đăng ký một lớp `PubSub` tùy chỉnh.

Điều này sẽ dẫn đến việc tạo phần sau của schema GraphQL trong SDL:

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

Lưu ý rằng subscriptions, theo định nghĩa, trả về một đối tượng với một thuộc tính cấp cao nhất duy nhất có khóa là tên của subscription. Tên này được kế thừa từ tên phương thức bộ xử lý subscription (tức là `commentAdded` ở trên), hoặc được cung cấp rõ ràng bằng cách chuyển một tùy chọn với khóa `name` làm đối số thứ hai cho decorator `@Subscription()`, như được hiển thị dưới đây.

```typescript
@Subscription(() => Comment, {
  name: 'commentAdded',
})
subscribeToCommentAdded(@Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

Cấu trúc này tạo ra cùng SDL với mẫu code trước, nhưng cho phép chúng ta tách rời tên phương thức khỏi subscription.

#### Publishing

Bây giờ, để xuất bản sự kiện, chúng ta sử dụng phương thức `PubSub#publish`. Điều này thường được sử dụng trong một mutation để kích hoạt cập nhật phía client khi một phần của đồ thị đối tượng đã thay đổi. Ví dụ:

```typescript
@@filename(posts/posts.resolver)
@Mutation(() => Comment)
async addComment(
  @Args('postId', { type: () => Int }) postId: number,
  @Args('comment', { type: () => Comment }) comment: CommentInput,
  @Context('pubsub') pubSub: PubSub,
) {
  const newComment = this.commentsService.addComment({ id: postId, comment });
  await pubSub.publish({
    topic: 'commentAdded',
    payload: {
      commentAdded: newComment
    }
  });
  return newComment;
}
```

Như đã đề cập, subscription, theo định nghĩa, trả về một giá trị và giá trị đó có một hình dạng. Hãy nhìn lại SDL được tạo cho subscription `commentAdded` của chúng ta:

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

Điều này cho chúng ta biết rằng subscription phải trả về một đối tượng với tên thuộc tính cấp cao nhất của `commentAdded` có một giá trị là một đối tượng `Comment`. Điểm quan trọng cần lưu ý là hình dạng của payload sự kiện được phát ra bởi phương thức `PubSub#publish` phải tương ứng với hình dạng của giá trị mong đợi trả về từ subscription. Vì vậy, trong ví dụ của chúng ta ở trên, câu lệnh `pubSub.publish({{ '{' }} topic: 'commentAdded', payload: {{ '{' }} commentAdded: newComment {{ '}' }} {{ '}' }})` xuất bản một sự kiện `commentAdded` với payload có hình dạng phù hợp. Nếu các hình dạng này không khớp, subscription của bạn sẽ thất bại trong giai đoạn xác thực GraphQL.

#### Lọc subscriptions

Để lọc ra các sự kiện cụ thể, đặt thuộc tính `filter` thành một hàm lọc. Hàm này hoạt động tương tự như hàm được chuyển cho một mảng `filter`. Nó nhận hai đối số: `payload` chứa payload sự kiện (như được gửi bởi nhà xuất bản sự kiện), và `variables` nhận bất kỳ đối số nào được chuyển trong trong yêu cầu subscription. Nó trả về một boolean xác định liệu sự kiện này có nên được xuất bản cho người nghe client hay không.

```typescript
@Subscription(() => Comment, {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded(@Args('title') title: string, @Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

Nếu bạn cần truy cập các nhà cung cấp được inject (ví dụ, sử dụng một dịch vụ bên ngoài để xác thực dữ liệu), sử dụng cấu trúc sau.

```typescript
@Subscription(() => Comment, {
  filter(this: AuthorResolver, payload, variables) {
    // "this" refers to an instance of "AuthorResolver"
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded(@Args('title') title: string, @Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

#### Schema first

Để tạo một subscription tương đương trong Nest, chúng ta sẽ sử dụng decorator `@Subscription()`.

```typescript
const pubSub = new PubSub();

@Resolver('Author')
export class AuthorResolver {
  // ...
  @Subscription()
  commentAdded(@Context('pubsub') pubSub: PubSub) {
    return pubSub.subscribe('commentAdded');
  }
}
```

Để lọc ra các sự kiện cụ thể dựa trên context và đối số, đặt thuộc tính `filter`.

```typescript
@Subscription('commentAdded', {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded(@Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

Nếu bạn cần truy cập các nhà cung cấp được inject (ví dụ, sử dụng một dịch vụ bên ngoài để xác thực dữ liệu), sử dụng cấu trúc sau.

```typescript
@Subscription('commentAdded', {
  filter(this: AuthorResolver, payload, variables) {
    // "this" refers to an instance of "AuthorResolver"
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded(@Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

Bước cuối cùng là cập nhật tệp định nghĩa loại.

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

type Comment {
  id: String
  content: String
}

type Subscription {
  commentAdded(title: String!): Comment
}
```

Với điều này, chúng ta đã tạo một subscription `commentAdded(title: String!): Comment` duy nhất.

#### PubSub

Trong các ví dụ trên, chúng ta đã sử dụng emitter `PubSub` mặc định ([mqemitter](https://github.com/mcollina/mqemitter))
Cách tiếp cận ưu tiên (cho production) là sử dụng `mqemitter-redis`. Ngoài ra, một triển khai `PubSub` tùy chỉnh có thể được cung cấp (đọc thêm [ở đây](https://github.com/mercurius-js/mercurius/blob/master/docs/subscriptions.md))

```typescript
GraphQLModule.forRoot<MercuriusDriverConfig>({
  driver: MercuriusDriver,
  subscription: {
    emitter: require('mqemitter-redis')({
      port: 6579,
      host: '127.0.0.1',
    }),
  },
});
```

#### Xác thực qua WebSockets

Việc kiểm tra xem người dùng có được xác thực hay không có thể được thực hiện bên trong hàm callback `verifyClient` mà bạn có thể chỉ định trong options `subscription`.

`verifyClient` sẽ nhận đối tượng `info` làm đối số đầu tiên mà bạn có thể sử dụng để truy xuất các tiêu đề của yêu cầu.

```typescript
GraphQLModule.forRoot<MercuriusDriverConfig>({
  driver: MercuriusDriver,
  subscription: {
    verifyClient: (info, next) => {
      const authorization = info.req.headers?.authorization as string;
      if (!authorization?.startsWith('Bearer ')) {
        return next(false);
      }
      next(true);
    },
  }
}),
```