### Federation

Federation cung cấp một phương thức để chia nhỏ server GraphQL monolithic của bạn thành các microservices độc lập. Nó bao gồm hai thành phần: một gateway và một hoặc nhiều microservices được liên kết. Mỗi microservice giữ một phần của schema và gateway hợp nhất các schema thành một schema duy nhất có thể được tiêu thụ bởi client.

Để trích dẫn từ [tài liệu Apollo](https://blog.apollographql.com/apollo-federation-f260cf525d21), Federation được thiết kế với các nguyên tắc cốt lõi sau:

- Xây dựng một graph nên là **khai báo.** Với federation, bạn soạn một graph một cách khai báo từ trong schema của bạn thay vì viết code ghép schema mệnh lệnh.
- Code nên được tách biệt theo **mối quan tâm**, không phải theo loại. Thường không có đội duy nhất nào kiểm soát mọi khía cạnh của một loại quan trọng như User hoặc Product, vì vậy định nghĩa của các loại này nên được phân phối trên các đội và codebases, thay vì được tập trung.
- Graph nên đơn giản để client tiêu thụ. Cùng nhau, các dịch vụ được liên kết có thể tạo thành một graph hoàn chỉnh, tập trung vào sản phẩm phản ánh chính xác cách nó đang được tiêu thụ trên client.
- Nó chỉ là **GraphQL**, sử dụng chỉ các tính năng tuân thủ đặc tả của ngôn ngữ. Bất kỳ ngôn ngữ nào, không chỉ JavaScript, có thể triển khai federation.

> warning **Cảnh báo** Federation hiện không hỗ trợ subscriptions.

Trong các phần sau, chúng ta sẽ thiết lập một ứng dụng demo bao gồm một gateway và hai endpoint được liên kết: Users service và Posts service.

#### Federation với Apollo

Bắt đầu bằng cách cài đặt các dependencies cần thiết:

```bash
$ npm install --save @apollo/subgraph
```

#### Schema first

"User service" cung cấp một schema đơn giản. Lưu ý directive `@key`: nó hướng dẫn trình lập kế hoạch query Apollo rằng một thể hiện cụ thể của `User` có thể được lấy nếu bạn chỉ định `id` của nó. Ngoài ra, lưu ý rằng chúng ta `extend` loại `Query`.

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

extend type Query {
  getUser(id: ID!): User
}
```

Resolver cung cấp một phương thức bổ sung tên là `resolveReference()`. Phương thức này được kích hoạt bởi Apollo Gateway bất cứ khi nào một tài nguyên liên quan yêu cầu một thể hiện User. Chúng ta sẽ thấy một ví dụ về điều này trong Posts service sau. Vui lòng lưu ý rằng phương thức phải được chú thích với decorator `@ResolveReference()`.

```typescript
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { UsersService } from './users.service';

@Resolver('User')
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query()
  getUser(@Args('id') id: string) {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: string }) {
    return this.usersService.findById(reference.id);
  }
}
```

Cuối cùng, chúng ta kết nối mọi thứ bằng cách đăng ký `GraphQLModule` chuyển driver `ApolloFederationDriver` trong đối tượng cấu hình:

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { UsersResolver } from './users.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [UsersResolver],
})
export class AppModule {}
```

#### Code first

Bắt đầu bằng cách thêm một số decorator bổ sung vào thực thể `User`.

```ts
import { Directive, Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field()
  name: string;
}
```

Resolver cung cấp một phương thức bổ sung tên là `resolveReference()`. Phương thức này được kích hoạt bởi Apollo Gateway bất cứ khi nào một tài nguyên liên quan yêu cầu một thể hiện User. Chúng ta sẽ thấy một ví dụ về điều này trong Posts service sau. Vui lòng lưu ý rằng phương thức phải được chú thích với decorator `@ResolveReference()`.

```ts
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { User } from './user.entity';
import { UsersService } from './users.service';

@Resolver(() => User)
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query(() => User)
  getUser(@Args('id') id: number): User {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: number }): User {
    return this.usersService.findById(reference.id);
  }
}
```

Cuối cùng, chúng ta kết nối mọi thứ bằng cách đăng ký `GraphQLModule` chuyển driver `ApolloFederationDriver` trong đối tượng cấu hình:

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // Not included in this example

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: true,
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/users-application) trong chế độ code first và [ở đây](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/users-application) trong chế độ schema first.

#### Ví dụ được liên kết: Posts

Post service được cho là phục vụ các bài viết được tổng hợp thông qua query `getPosts`, nhưng cũng mở rộng loại `User` của chúng ta với trường `user.posts`.

#### Schema first

"Posts service" tham chiếu loại `User` trong schema của nó bằng cách đánh dấu nó với từ khóa `extend`. Nó cũng khai báo một thuộc tính bổ sung trên loại `User` (`posts`). Lưu ý directive `@key` được sử dụng để khớp các thể hiện của User, và directive `@external` chỉ ra rằng trường `id` được quản lý ở nơi khác.

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post]
}

extend type Query {
  getPosts: [Post]
}
```

Trong ví dụ sau, `PostsResolver` cung cấp phương thức `getUser()` trả về một tham chiếu chứa `__typename` và một số thuộc tính bổ sung mà ứng dụng của bạn có thể cần để giải quyết tham chiếu, trong trường hợp này là `id`. `__typename` được sử dụng bởi GraphQL Gateway để xác định chính xác microservice chịu trách nhiệm cho loại User và truy xuất thể hiện tương ứng. "Users service" được mô tả ở trên sẽ được yêu cầu khi thực thi phương thức `resolveReference()`.

```typescript
import { Query, Resolver, Parent, ResolveField } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './posts.interfaces';

@Resolver('Post')
export class PostsResolver {
  constructor(private postsService: PostsService) {}

  @Query('getPosts')
  getPosts() {
    return this.postsService.findAll();
  }

  @ResolveField('user')
  getUser(@Parent() post: Post) {
    return { __typename: 'User', id: post.userId };
  }
}
```

Cuối cùng, chúng ta phải đăng ký `GraphQLModule`, tương tự như những gì chúng ta đã làm trong phần "Users service".

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { PostsResolver } from './posts.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [PostsResolver],
})
export class AppModule {}
```

#### Code first

Đầu tiên, chúng ta sẽ phải khai báo một lớp đại diện cho thực thể `User`. Mặc dù thực thể sống trong một dịch vụ khác, chúng ta sẽ sử dụng nó (mở rộng định nghĩa của nó) ở đây. Lưu ý các directive `@extends` và `@external`.

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@extends')
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  @Directive('@external')
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

Bây giờ hãy tạo resolver tương ứng cho phần mở rộng của chúng ta trên thực thể `User`, như sau:

```ts
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly postsService: PostsService) {}

  @ResolveField(() => [Post])
  public posts(@Parent() user: User): Post[] {
    return this.postsService.forAuthor(user.id);
  }
}
```

Chúng ta cũng phải định nghĩa lớp thực thể `Post`:

```ts
import { Directive, Field, ID, Int, ObjectType } from '@nestjs/graphql';
import { User } from './user.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class Post {
  @Field(() => ID)
  id: number;

  @Field()
  title: string;

  @Field(() => Int)
  authorId: number;

  @Field(() => User)
  user?: User;
}
```

Và resolver của nó:

```ts
import { Query, Args, ResolveField, Resolver, Parent } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => Post)
export class PostsResolver {
  constructor(private readonly postsService: PostsService) {}

  @Query(() => Post)
  findPost(@Args('id') id: number): Post {
    return this.postsService.findOne(id);
  }

  @Query(() => [Post])
  getPosts(): Post[] {
    return this.postsService.all();
  }

  @ResolveField(() => User)
  user(@Parent() post: Post): any {
    return { __typename: 'User', id: post.authorId };
  }
}
```

Và cuối cùng, kết nối mọi thứ trong một module. Lưu ý các tùy chọn xây dựng schema, nơi chúng ta chỉ định rằng `User` là một loại mồ côi (bên ngoài).

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolver } from './posts.resolvers';
import { UsersResolver } from './users.resolvers';
import { PostsService } from './posts.service'; // Not included in example

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: true,
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/posts-application) cho chế độ code first và [ở đây](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/posts-application) cho chế độ schema first.

#### Ví dụ được liên kết: Gateway

Bắt đầu bằng cách cài đặt dependency cần thiết:

```bash
$ npm install --save @apollo/gateway
```

Gateway yêu cầu một danh sách các endpoint được chỉ định và nó sẽ tự động phát hiện các schema tương ứng. Do đó việc triển khai dịch vụ gateway sẽ giữ nguyên cho cả hai phương pháp code và schema first.

```typescript
import { IntrospectAndCompose } from '@apollo/gateway';
import { ApolloGatewayDriver, ApolloGatewayDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloGatewayDriverConfig>({
      driver: ApolloGatewayDriver,
      server: {
        // ... Apollo server options
        cors: true,
      },
      gateway: {
        supergraphSdl: new IntrospectAndCompose({
          subgraphs: [
            { name: 'users', url: 'http://user-service/graphql' },
            { name: 'posts', url: 'http://post-service/graphql' },
          ],
        }),
      },
    }),
  ],
})
export class AppModule {}
```

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/gateway) cho chế độ code first và [ở đây](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/gateway) cho chế độ schema first.

#### Federation với Mercurius

Bắt đầu bằng cách cài đặt các dependencies cần thiết:

```bash
$ npm install --save @apollo/subgraph @nestjs/mercurius
```

> info **Lưu ý** Gói `@apollo/subgraph` được yêu cầu để xây dựng một schema subgraph (các hàm `buildSubgraphSchema`, `printSubgraphSchema`).

#### Schema first

"User service" cung cấp một schema đơn giản. Lưu ý directive `@key`: nó hướng dẫn trình lập kế hoạch query Mercurius rằng một thể hiện cụ thể của `User` có thể được lấy nếu bạn chỉ định `id` của nó. Ngoài ra, lưu ý rằng chúng ta `extend` loại `Query`.

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

extend type Query {
  getUser(id: ID!): User
}
```

Resolver cung cấp một phương thức bổ sung tên là `resolveReference()`. Phương thức này được kích hoạt bởi Mercurius Gateway bất cứ khi nào một tài nguyên liên quan yêu cầu một thể hiện User. Chúng ta sẽ thấy một ví dụ về điều này trong Posts service sau. Vui lòng lưu ý rằng phương thức phải được chú thích với decorator `@ResolveReference()`.

```typescript
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { UsersService } from './users.service';

@Resolver('User')
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query()
  getUser(@Args('id') id: string) {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: string }) {
    return this.usersService.findById(reference.id);
  }
}
```

Cuối cùng, chúng ta kết nối mọi thứ bằng cách đăng ký `GraphQLModule` chuyển driver `MercuriusFederationDriver` trong đối tượng cấu hình:

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { UsersResolver } from './users.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      typePaths: ['**/*.graphql'],
      federationMetadata: true,
    }),
  ],
  providers: [UsersResolver],
})
export class AppModule {}
```

#### Code first

Bắt đầu bằng cách thêm một số decorator bổ sung vào thực thể `User`.

```ts
import { Directive, Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field()
  name: string;
}
```

Resolver cung cấp một phương thức bổ sung tên là `resolveReference()`. Phương thức này được kích hoạt bởi Mercurius Gateway bất cứ khi nào một tài nguyên liên quan yêu cầu một thể hiện User. Chúng ta sẽ thấy một ví dụ về điều này trong Posts service sau. Vui lòng lưu ý rằng phương thức phải được chú thích với decorator `@ResolveReference()`.

```ts
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { User } from './user.entity';
import { UsersService } from './users.service';

@Resolver(() => User)
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query(() => User)
  getUser(@Args('id') id: number): User {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: number }): User {
    return this.usersService.findById(reference.id);
  }
}
```

Cuối cùng, chúng ta kết nối mọi thứ bằng cách đăng ký `GraphQLModule` chuyển driver `MercuriusFederationDriver` trong đối tượng cấu hình:

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // Not included in this example

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      autoSchemaFile: true,
      federationMetadata: true,
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

#### Ví dụ được liên kết: Posts

Post service được cho là phục vụ các bài viết được tổng hợp thông qua query `getPosts`, nhưng cũng mở rộng loại `User` của chúng ta với trường `user.posts`.

#### Schema first

"Posts service" tham chiếu loại `User` trong schema của nó bằng cách đánh dấu nó với từ khóa `extend`. Nó cũng khai báo một thuộc tính bổ sung trên loại `User` (`posts`). Lưu ý directive `@key` được sử dụng để khớp các thể hiện của User, và directive `@external` chỉ ra rằng trường `id` được quản lý ở nơi khác.

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post]
}

extend type Query {
  getPosts: [Post]
}
```

Trong ví dụ sau, `PostsResolver` cung cấp phương thức `getUser()` trả về một tham chiếu chứa `__typename` và một số thuộc tính bổ sung mà ứng dụng của bạn có thể cần để giải quyết tham chiếu, trong trường hợp này là `id`. `__typename` được sử dụng bởi GraphQL Gateway để xác định chính xác microservice chịu trách nhiệm cho loại User và truy xuất thể hiện tương ứng. "Users service" được mô tả ở trên sẽ được yêu cầu khi thực thi phương thức `resolveReference()`.

```typescript
import { Query, Resolver, Parent, ResolveField } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './posts.interfaces';

@Resolver('Post')
export class PostsResolver {
  constructor(private postsService: PostsService) {}

  @Query('getPosts')
  getPosts() {
    return this.postsService.findAll();
  }

  @ResolveField('user')
  getUser(@Parent() post: Post) {
    return { __typename: 'User', id: post.userId };
  }
}
```

Cuối cùng, chúng ta phải đăng ký `GraphQLModule`, tương tự như những gì chúng ta đã làm trong phần "Users service".

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { PostsResolver } from './posts.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      federationMetadata: true,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [PostsResolver],
})
export class AppModule {}
```

#### Code first

Đầu tiên, chúng ta sẽ phải khai báo một lớp đại diện cho thực thể `User`. Mặc dù thực thể sống trong một dịch vụ khác, chúng ta sẽ sử dụng nó (mở rộng định nghĩa của nó) ở đây. Lưu ý các directive `@extends` và `@external`.

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@extends')
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  @Directive('@external')
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

Bây giờ hãy tạo resolver tương ứng cho phần mở rộng của chúng ta trên thực thể `User`, như sau:

```ts
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly postsService: PostsService) {}

  @ResolveField(() => [Post])
  public posts(@Parent() user: User): Post[] {
    return this.postsService.forAuthor(user.id);
  }
}
```

Chúng ta cũng phải định nghĩa lớp thực thể `Post`:

```ts
import { Directive, Field, ID, Int, ObjectType } from '@nestjs/graphql';
import { User } from './user.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class Post {
  @Field(() => ID)
  id: number;

  @Field()
  title: string;

  @Field(() => Int)
  authorId: number;

  @Field(() => User)
  user?: User;
}
```

Và resolver của nó:

```ts
import { Query, Args, ResolveField, Resolver, Parent } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => Post)
export class PostsResolver {
  constructor(private readonly postsService: PostsService) {}

  @Query(() => Post)
  findPost(@Args('id') id: number): Post {
    return this.postsService.findOne(id);
  }

  @Query(() => [Post])
  getPosts(): Post[] {
    return this.postsService.all();
  }

  @ResolveField(() => User)
  user(@Parent() post: Post): any {
    return { __typename: 'User', id: post.authorId };
  }
}
```

Và cuối cùng, kết nối mọi thứ trong một module. Lưu ý các tùy chọn xây dựng schema, nơi chúng ta chỉ định rằng `User` là một loại mồ côi (bên ngoài).

```ts
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolver } from './posts.resolvers';
import { UsersResolver } from './users.resolvers';
import { PostsService } from './posts.service'; // Not included in example

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      autoSchemaFile: true,
      federationMetadata: true,
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```

#### Ví dụ được liên kết: Gateway

Gateway yêu cầu một danh sách các endpoint được chỉ định và nó sẽ tự động phát hiện các schema tương ứng. Do đó việc triển khai dịch vụ gateway sẽ giữ nguyên cho cả hai phương pháp code và schema first.

```typescript
import {
  MercuriusGatewayDriver,
  MercuriusGatewayDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusGatewayDriverConfig>({
      driver: MercuriusGatewayDriver,
      gateway: {
        services: [
          { name: 'users', url: 'http://user-service/graphql' },
          { name: 'posts', url: 'http://post-service/graphql' },
        ],
      },
    }),
  ],
})
export class AppModule {}
```

### Federation 2

Để trích dẫn từ [tài liệu Apollo](https://www.apollographql.com/docs/federation/federation-2/new-in-federation-2), Federation 2 cải thiện trải nghiệm nhà phát triển từ Apollo Federation gốc (được gọi là Federation 1 trong tài liệu này), tương thích ngược với hầu hết các supergraph gốc.

> warning **Cảnh báo** Mercurius không hỗ trợ đầy đủ Federation 2. Bạn có thể xem danh sách các thư viện hỗ trợ Federation 2 [ở đây](https://www.apollographql.com/docs/federation/supported-subgraphs#javascript--typescript).

Trong các phần sau, chúng ta sẽ nâng cấp ví dụ trước lên Federation 2.

#### Ví dụ được liên kết: Users

Một thay đổi trong Federation 2 là các thực thể không có subgraph gốc, vì vậy chúng ta không cần mở rộng `Query` nữa. Để biết thêm chi tiết vui lòng tham khảo [chủ đề thực thể](https://www.apollographql.com/docs/federation/federation-2/new-in-federation-2#entities) trong tài liệu Apollo Federation 2.

#### Schema first

Chúng ta có thể đơn giản loại bỏ từ khóa `extend` khỏi schema.

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

type Query {
  getUser(id: ID!): User
}
```

#### Code first

Để sử dụng Federation 2, chúng ta cần chỉ định phiên bản federation trong tùy chọn `autoSchemaFile`.

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // Not included in this example

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: {
        federation: 2,
      },
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

#### Ví dụ được liên kết: Posts

Với cùng lý do như trên, chúng ta không cần mở rộng `User` và `Query` nữa.

#### Schema first

Chúng ta có thể đơn giản loại bỏ các directive `extend` và `external` khỏi schema

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

type User @key(fields: "id") {
  id: ID!
  posts: [Post]
}

type Query {
  getPosts: [Post]
}
```

#### Code first

Vì chúng ta không mở rộng thực thể `User` nữa, chúng ta có thể đơn giản loại bỏ các directive `extends` và `external` khỏi `User`.

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

Ngoài ra, tương tự như User service, chúng ta cần chỉ định trong `GraphQLModule` để sử dụng Federation 2.

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolver } from './posts.resolvers';
import { UsersResolver } from './users.resolvers';
import { PostsService } from './posts.service'; // Not included in example

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: {
        federation: 2,
      },
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```