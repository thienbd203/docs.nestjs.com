### Tính năng khác

Trong thế giới GraphQL, có rất nhiều cuộc tranh luận về việc xử lý các vấn đề như **xác thực**, hoặc **tác dụng phụ** của các hoạt động. Chúng ta nên xử lý các thứ này trong logic kinh doanh? Chúng ta nên sử dụng một hàm higher-order để nâng cao queries và mutations với logic ủy quyền? Hay chúng ta nên sử dụng [schema directives](https://www.apollographql.com/docs/apollo-server/schema/directives/)? Không có một câu trả lời một kích cỡ phù hợp cho tất cả cho các câu hỏi này.

Nest giúp giải quyết các vấn đề này với các tính năng đa nền tảng của nó như [guards](/guards) và [interceptors](/interceptors). Triết lý là giảm thiểu sự dư thừa và cung cấp công cụ giúp tạo ra các ứng dụng có cấu trúc tốt, dễ đọc và nhất quán.

#### Tổng quan

Bạn có thể sử dụng [guards](/guards), [interceptors](/interceptors), [filters](/exception-filters) và [pipes](/pipes) tiêu chuẩn theo cùng một cách với GraphQL như với bất kỳ ứng dụng RESTful nào. Ngoài ra, bạn có thể dễ dàng tạo các decorator của riêng mình bằng cách tận dụng tính năng [custom decorators](/custom-decorators). Hãy xem một bộ xử lý query GraphQL mẫu.

```typescript
@Query('author')
@UseGuards(AuthGuard)
async getAuthor(@Args('id', ParseIntPipe) id: number) {
  return this.authorsService.findOneById(id);
}
```

Như bạn có thể thấy, GraphQL hoạt động với cả guards và pipes theo cùng một cách như bộ xử lý HTTP REST. Vì điều này, bạn có thể chuyển logic xác thực của mình sang một guard; bạn thậm chí có thể tái sử dụng cùng một lớp guard trên cả giao diện API REST và GraphQL. Tương tự, interceptors hoạt động trên cả hai loại ứng dụng theo cùng một cách:

```typescript
@Mutation()
@UseInterceptors(EventsInterceptor)
async upvotePost(@Args('postId') postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

#### Execution context

Vì GraphQL nhận một loại dữ liệu khác trong yêu cầu inbound, [execution context](https://docs.nestjs.com/fundamentals/execution-context) được nhận bởi cả guards và interceptors hơi khác với GraphQL so với REST. Resolvers GraphQL có một tập hợp đối số riêng biệt: `root`, `args`, `context`, và `info`. Do đó guards và interceptors phải chuyển đổi `ExecutionContext` chung thành `GqlExecutionContext`. Điều này là đơn giản:

```typescript
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const ctx = GqlExecutionContext.create(context);
    return true;
  }
}
```

Đối tượng context GraphQL được trả về bởi `GqlExecutionContext.create()` phơi bày một phương thức **get** cho mỗi đối số resolver GraphQL (ví dụ, `getArgs()`, `getContext()`, v.v). Sau khi được chuyển đổi, chúng ta có thể dễ dàng chọn ra bất kỳ đối số GraphQL nào cho yêu cầu hiện tại.

#### Exception filters

Các [exception filters](/exception-filters) tiêu chuẩn của Nest cũng tương thích với các ứng dụng GraphQL. Như với `ExecutionContext`, các ứng dụng GraphQL nên chuyển đổi đối tượng `ArgumentsHost` thành một đối tượng `GqlArgumentsHost`.

```typescript
@Catch(HttpException)
export class HttpExceptionFilter implements GqlExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const gqlHost = GqlArgumentsHost.create(host);
    return exception;
  }
}
```

> info **Gợi ý** Cả `GqlExceptionFilter` và `GqlArgumentsHost` đều được nhập từ gói `@nestjs/graphql`.

Lưu ý rằng không giống như trường hợp REST, bạn không sử dụng đối tượng `response` gốc để tạo phản hồi.

#### Custom decorators

Như đã đề cập, tính năng [custom decorators](/custom-decorators) hoạt động như mong đợi với resolvers GraphQL.

```typescript
export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) =>
    GqlExecutionContext.create(ctx).getContext().user,
);
```

Sử dụng decorator tùy chỉnh `@User()` như sau:

```typescript
@Mutation()
async upvotePost(
  @User() user: UserEntity,
  @Args('postId') postId: number,
) {}
```

> info **Gợi ý** Trong ví dụ trên, chúng ta đã giả định rằng đối tượng `user` được gán cho context của ứng dụng GraphQL của bạn.

#### Thực thi enhancers ở cấp resolver trường

Trong ngữ cảnh GraphQL, Nest không chạy **enhancers** (tên chung cho interceptors, guards và filters) ở cấp trường [xem vấn đề này](https://github.com/nestjs/graphql/issues/320#issuecomment-511193229): chúng chỉ chạy cho phương thức `@Query()`/`@Mutation()` cấp cao nhất. Bạn có thể nói với Nest để thực thi interceptors, guards hoặc filters cho các phương thức được chú thích với `@ResolveField()` bằng cách đặt tùy chọn `fieldResolverEnhancers` trong `GqlModuleOptions`. Chuyển cho nó một danh sách `'interceptors'`, `'guards'`, và/hoặc `'filters'` theo nhu cầu:

```typescript
GraphQLModule.forRoot({
  fieldResolverEnhancers: ['interceptors']
}),
```

> **Cảnh báo** Kích hoạt enhancers cho field resolvers có thể gây ra các vấn đề hiệu suất khi bạn trả về nhiều bản ghi và field resolver của bạn được thực hiện hàng nghìn lần. Vì lý do này, khi bạn kích hoạt `fieldResolverEnhancers`, chúng tôi khuyên bạn nên bỏ qua việc thực thi các enhancers không hoàn toàn cần thiết cho field resolvers của bạn. Bạn có thể làm điều này sử dụng hàm helper sau:

```typescript
export function isResolvingGraphQLField(context: ExecutionContext): boolean {
  if (context.getType<GqlContextType>() === 'graphql') {
    const gqlContext = GqlExecutionContext.create(context);
    const info = gqlContext.getInfo();
    const parentType = info.parentType.name;
    return parentType !== 'Query' && parentType !== 'Mutation';
  }
  return false;
}
```

#### Tạo một driver tùy chỉnh

Nest cung cấp hai driver chính thức out-of-the-box: `@nestjs/apollo` và `@nestjs/mercurius`, cũng như một API cho phép các nhà phát triển xây dựng **custom drivers** mới. Với một driver tùy chỉnh, bạn có thể tích hợp bất kỳ thư viện GraphQL nào hoặc mở rộng tích hợp hiện có, thêm các tính năng bổ sung lên trên.

Ví dụ, để tích hợp gói `express-graphql`, bạn có thể tạo lớp driver sau:

```typescript
import { AbstractGraphQLDriver, GqlModuleOptions } from '@nestjs/graphql';
import { graphqlHTTP } from 'express-graphql';

class ExpressGraphQLDriver extends AbstractGraphQLDriver {
  async start(options: GqlModuleOptions<any>): Promise<void> {
    options = await this.graphQlFactory.mergeWithSchema(options);

    const { httpAdapter } = this.httpAdapterHost;
    httpAdapter.use(
      '/graphql',
      graphqlHTTP({
        schema: options.schema,
        graphiql: true,
      }),
    );
  }

  async stop() {}
}
```

Và sau đó sử dụng nó như sau:

```typescript
GraphQLModule.forRoot({
  driver: ExpressGraphQLDriver,
});
```