## Tận dụng sức mạnh của TypeScript & GraphQL

[GraphQL](https://graphql.org/) là một ngôn ngữ truy vấn mạnh mẽ cho API và một runtime để thực hiện các truy vấn đó với dữ liệu hiện có của bạn. Nó là một phương pháp elegan giải quyết nhiều vấn đề thường thấy với REST API. Để có bối cảnh, chúng tôi đề xuất đọc bài viết so sánh [này](https://www.apollographql.com/blog/graphql-vs-rest) giữa GraphQL và REST. GraphQL kết hợp với [TypeScript](https://www.typescriptlang.org/) giúp bạn phát triển tính an toàn kiểu tốt hơn với các truy vấn GraphQL của mình, cung cấp cho bạn typing từ đầu đến cuối.

Trong chương này, chúng tôi giả định bạn có hiểu biết cơ bản về GraphQL, và tập trung vào cách làm việc với module `@nestjs/graphql` tích hợp sẵn. `GraphQLModule` có thể được cấu hình để sử dụng [Apollo](https://www.apollographql.com/) server (với driver `@nestjs/apollo`) và [Mercurius](https://github.com/mercurius-js/mercurius) (với driver `@nestjs/mercurius`). Chúng tôi cung cấp các tích hợp chính thức cho các gói GraphQL đã được chứng minh này để cung cấp một cách đơn giản để sử dụng GraphQL với Nest (xem thêm các tích hợp [ở đây](https://docs.nestjs.com/graphql/quick-start#third-party-integrations)).

Bạn cũng có thể xây dựng driver chuyên dụng của riêng mình (đọc thêm về điều đó [ở đây](/graphql/other-features#creating-a-custom-driver)).

#### Cài đặt

Bắt đầu bằng cách cài đặt các gói cần thiết:

```bash
# Cho Express và Apollo (mặc định)
$ npm i @nestjs/graphql @nestjs/apollo @apollo/server @as-integrations/express5 graphql

# Cho Fastify và Apollo
# npm i @nestjs/graphql @nestjs/apollo @apollo/server @as-integrations/fastify graphql

# Cho Fastify và Mercurius
# npm i @nestjs/graphql @nestjs/mercurius graphql mercurius
```

> warning **Cảnh báo** Các gói `@nestjs/graphql@>=9` và `@nestjs/apollo^10` tương thích với **Apollo v3** (kiểm tra hướng dẫn di chuyển Apollo Server 3 [migration guide](https://www.apollographql.com/docs/apollo-server/migration/) để biết thêm chi tiết), trong khi `@nestjs/graphql@^8` chỉ hỗ trợ **Apollo v2** (ví dụ, gói `apollo-server-express@2.x.x`).

#### Tổng quan

Nest cung cấp hai cách để xây dựng ứng dụng GraphQL, phương pháp **code first** và **schema first**. Bạn nên chọn phương pháp phù hợp nhất với mình. Hầu hết các chương trong phần GraphQL này được chia thành hai phần chính: một phần bạn nên theo nếu bạn chọn **code first**, và phần kia để sử dụng nếu bạn chọn **schema first**.

Trong phương pháp **code first**, bạn sử dụng decorators và các lớp TypeScript để tạo schema GraphQL tương ứng. Phương pháp này hữu ích nếu bạn thích làm việc độc quyền với TypeScript và tránh việc chuyển đổi giữa các cú pháp ngôn ngữ.

Trong phương pháp **schema first**, nguồn sự thật là các tệp SDL (Schema Definition Language) của GraphQL. SDL là một cách không phụ thuộc ngôn ngữ để chia sẻ các tệp schema giữa các nền tảng khác nhau. Nest tự động tạo các định nghĩa TypeScript của bạn (sử dụng cả lớp hoặc interface) dựa trên các schema GraphQL để giảm nhu cầu viết code boilerplate dư thừa.

<app-banner-courses-graphql-cf></app-banner-courses-graphql-cf>

#### Bắt đầu với GraphQL & TypeScript

> info **Gợi ý** Trong các chương tiếp theo, chúng tôi sẽ tích hợp gói `@nestjs/apollo`. Nếu bạn muốn sử dụng gói `mercurius` thay thế, hãy điều hướng đến [phần này](/graphql/quick-start#mercurius-integration).

Sau khi các gói được cài đặt, chúng ta có thể import `GraphQLModule` và cấu hình nó với phương thức tĩnh `forRoot()`.

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
    }),
  ],
})
export class AppModule {}
```

> info **Gợi ý** Để tích hợp `mercurius`, bạn nên sử dụng `MercuriusDriver` và `MercuriusDriverConfig` thay thế. Cả hai đều được xuất từ gói `@nestjs/mercurius`.

Phương thức `forRoot()` nhận một đối tượng options làm đối số. Các tùy chọn này được chuyển đến instance driver bên dưới (đọc thêm về các cài đặt có sẵn ở đây: [Apollo](https://www.apollographql.com/docs/apollo-server/api/apollo-server) và [Mercurius](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md#plugin-options)). Ví dụ, nếu bạn muốn tắt `playground` và tắt chế độ `debug` (đối với Apollo), hãy chuyển các tùy chọn sau:

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      playground: false,
    }),
  ],
})
export class AppModule {}
```

Trong trường hợp này, các tùy chọn này sẽ được chuyển đến constructor `ApolloServer`.

#### GraphQL playground

Playground là một IDE GraphQL đồ họa, tương tác, trên trình duyệt, có sẵn theo mặc định trên cùng URL với server GraphQL. Để truy cập playground, bạn cần một server GraphQL cơ bản được cấu hình và đang chạy. Để xem ngay bây giờ, bạn có thể cài đặt và xây dựng [ví dụ hoạt động ở đây](https://github.com/nestjs/nest/tree/master/sample/23-graphql-code-first). Ngoài ra, nếu bạn đang theo dõi các mẫu code này, sau khi bạn hoàn thành các bước trong [chương Resolvers](/graphql/resolvers-map), bạn có thể truy cập playground.

Với điều đó đã có sẵn, và với ứng dụng của bạn đang chạy trong nền, sau đó bạn có thể mở trình duyệt web của mình và điều hướng đến `http://localhost:3000/graphql` (host và port có thể thay đổi tùy thuộc vào cấu hình của bạn). Sau đó bạn sẽ thấy GraphQL playground, như hiển thị dưới đây.

<figure>
  <img src="/assets/playground.png" alt="" />
</figure>

> info **Lưu ý** Tích hợp `@nestjs/mercurius` không đi kèm với tích hợp GraphQL Playground tích hợp. Thay vào đó, bạn có thể sử dụng [GraphiQL](https://github.com/graphql/graphiql) (đặt `graphiql: true`).

> warning **Cảnh báo** Cập nhật (14/04/2025): Playground Apollo mặc định đã bị deprecated và sẽ bị loại bỏ trong bản phát hành chính tiếp theo. Thay vào đó, bạn có thể sử dụng [GraphiQL](https://github.com/graphql/graphiql), chỉ cần đặt `graphiql: true` trong cấu hình `GraphQLModule`, như được hiển thị dưới đây:
>
> ```typescript
> GraphQLModule.forRoot<ApolloDriverConfig>({
>   driver: ApolloDriver,
>   graphiql: true,
> }),
> ```
>
> Nếu ứng dụng của bạn sử dụng [subscriptions](/graphql/subscriptions), hãy đảm bảo sử dụng `graphql-ws`, vì `subscriptions-transport-ws` không được hỗ trợ bởi GraphiQL.

#### Code first

Trong phương pháp **code first**, bạn sử dụng decorators và các lớp TypeScript để tạo schema GraphQL tương ứng.

Để sử dụng phương pháp code first, bắt đầu bằng cách thêm thuộc tính `autoSchemaFile` vào đối tượng options:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
}),
```

Giá trị thuộc tính `autoSchemaFile` là đường dẫn nơi schema được tạo tự động của bạn sẽ được tạo. Ngoài ra, schema có thể được tạo ngay lập tức trong bộ nhớ. Để kích hoạt điều này, đặt thuộc tính `autoSchemaFile` thành `true`:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: true,
}),
```

Theo mặc định, các loại trong schema được tạo sẽ theo thứ tự chúng được định nghĩa trong các module được bao gồm. Để sắp xếp schema theo từ vựng, đặt thuộc tính `sortSchema` thành `true`:

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
  sortSchema: true,
}),
```

#### Ví dụ

Một mẫu code first hoạt động đầy đủ có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/23-graphql-code-first).

#### Schema first

Để sử dụng phương pháp schema first, bắt đầu bằng cách thêm thuộc tính `typePaths` vào đối tượng options. Thuộc tính `typePaths` chỉ ra nơi `GraphQLModule` nên tìm các tệp định nghĩa schema GraphQL SDL mà bạn sẽ viết. Các tệp này sẽ được kết hợp trong bộ nhớ; điều này cho phép bạn chia nhỏ schema của mình thành một số tệp và định vị chúng gần resolvers của chúng.

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
}),
```

Bạn thường cũng sẽ cần có các định nghĩa TypeScript (lớp và interface) tương ứng với các loại SDL GraphQL. Tạo các định nghĩa TypeScript tương ứng bằng tay là dư thừa và tẻ nhạt. Nó khiến chúng ta không có một nguồn sự thật duy nhất -- mỗi thay đổi được thực hiện trong SDL buộc chúng ta cũng phải điều chỉnh các định nghĩa TypeScript. Để giải quyết vấn đề này, gói `@nestjs/graphql` có thể **tự động tạo** các định nghĩa TypeScript từ cây cú pháp trừu tượng ([AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree)). Để kích hoạt tính năng này, thêm thuộc tính options `definitions` khi cấu hình `GraphQLModule`.

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
  },
}),
```

Thuộc tính path của đối tượng `definitions` chỉ ra nơi lưu đầu ra TypeScript được tạo. Theo mặc định, tất cả các loại TypeScript được tạo được tạo dưới dạng interfaces. Để tạo các lớp thay thế, chỉ định thuộc tính `outputAs` với giá trị `'class'`.

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
    outputAs: 'class',
  },
}),
```

Phương pháp trên tạo động các định nghĩa TypeScript mỗi khi ứng dụng khởi động. Ngoài ra, có thể ưu tiên xây dựng một script đơn giản để tạo các định nghĩa này theo yêu cầu. Ví dụ, giả sử chúng ta tạo script sau là `generate-typings.ts`:

```typescript
import { GraphQLDefinitionsFactory } from '@nestjs/graphql';
import { join } from 'node:path';

const definitionsFactory = new GraphQLDefinitionsFactory();
definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
});
```

Bây giờ bạn có thể chạy script này theo yêu cầu:

```bash
$ ts-node generate-typings
```

> info **Gợi ý** Bạn có thể biên dịch script trước đó (ví dụ, với `tsc`) và sử dụng `node` để thực thi nó.

Để kích hoạt chế độ watch cho script (để tự động tạo typings bất cứ khi nào bất kỳ tệp `.graphql` nào thay đổi), chuyển tùy chọn `watch` cho phương thức `generate()`.

```typescript
definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
  watch: true,
});
```

Để tự động tạo thêm trường `__typename` cho mỗi loại đối tượng, kích hoạt tùy chọn `emitTypenameField`:

```typescript
definitionsFactory.generate({
  // ...
  emitTypenameField: true,
});
```

Để tạo resolvers (queries, mutations, subscriptions) dưới dạng các trường đơn giản không có đối số, kích hoạt tùy chọn `skipResolverArgs`:

```typescript
definitionsFactory.generate({
  // ...
  skipResolverArgs: true,
});
```

Để tạo enums dưới dạng các loại union của TypeScript thay vì các enum TypeScript thông thường, đặt tùy chọn `enumsAsTypes` thành `true`:

```typescript
definitionsFactory.generate({
  // ...
  enumsAsTypes: true,
});
```

#### Apollo Sandbox

Để sử dụng [Apollo Sandbox](https://www.apollographql.com/blog/announcement/platform/apollo-sandbox-an-open-graphql-ide-for-local-development/) thay vì `graphql-playground` làm IDE GraphQL cho phát triển cục bộ, sử dụng cấu hình sau:

```typescript
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloServerPluginLandingPageLocalDefault } from '@apollo/server/plugin/landingPage/default';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      playground: false,
      plugins: [ApolloServerPluginLandingPageLocalDefault()],
    }),
  ],
})
export class AppModule {}
```

#### Ví dụ

Một mẫu schema first hoạt động đầy đủ có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/12-graphql-schema-first).

#### Truy cập schema được tạo

Trong một số trường hợp (ví dụ kiểm tra end-to-end), bạn có thể muốn lấy tham chiếu đến đối tượng schema được tạo. Trong kiểm tra end-to-end, sau đó bạn có thể chạy các truy vấn bằng đối tượng `graphql` mà không sử dụng bất kỳ trình nghe HTTP nào.

Bạn có thể truy cập schema được tạo (trong cả phương pháp code first hoặc schema first), sử dụng lớp `GraphQLSchemaHost`:

```typescript
const { schema } = app.get(GraphQLSchemaHost);
```

> info **Gợi ý** Bạn phải gọi getter `GraphQLSchemaHost#schema` sau khi ứng dụng đã được khởi tạo (sau khi hook `onModuleInit` được kích hoạt bởi phương thức `app.listen()` hoặc `app.init()`).

#### Cấu hình async

Khi bạn cần chuyển tùy chọn module một cách không đồng bộ thay vì tĩnh, sử dụng phương thức `forRootAsync()`. Như với hầu hết các module động, Nest cung cấp một số kỹ thuật để xử lý cấu hình async.

Một kỹ thuật là sử dụng hàm factory:

```typescript
 GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  useFactory: () => ({
    typePaths: ['./**/*.graphql'],
  }),
}),
```

Giống như các nhà cung cấp factory khác, hàm factory của chúng ta có thể <a href="https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory">async</a> và có thể inject dependencies thông qua `inject`.

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    typePaths: configService.get<string>('GRAPHQL_TYPE_PATHS'),
  }),
  inject: [ConfigService],
}),
```

Ngoài ra, bạn có thể cấu hình `GraphQLModule` sử dụng một lớp thay vì một factory, như được hiển thị dưới đây:

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  useClass: GqlConfigService,
}),
```

Cấu trúc trên khởi tạo `GqlConfigService` bên trong `GraphQLModule`, sử dụng nó để tạo đối tượng options. Lưu ý rằng trong ví dụ này, `GqlConfigService` phải triển khai interface `GqlOptionsFactory`, như được hiển thị dưới đây. `GraphQLModule` sẽ gọi phương thức `createGqlOptions()` trên đối tượng được khởi tạo của lớp được cung cấp.

```typescript
@Injectable()
class GqlConfigService implements GqlOptionsFactory {
  createGqlOptions(): ApolloDriverConfig {
    return {
      typePaths: ['./**/*.graphql'],
    };
  }
}
```

Nếu bạn muốn tái sử dụng một nhà cung cấp options hiện có thay vì tạo một bản sao riêng tư bên trong `GraphQLModule`, sử dụng cú pháp `useExisting`.

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  imports: [ConfigModule],
  useExisting: ConfigService,
}),
```

#### Tích hợp Mercurius

Thay vì sử dụng Apollo, người dùng Fastify (đọc thêm [ở đây](/techniques/performance)) có thể thay thế sử dụng driver `@nestjs/mercurius`.

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { MercuriusDriver, MercuriusDriverConfig } from '@nestjs/mercurius';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusDriverConfig>({
      driver: MercuriusDriver,
      graphiql: true,
    }),
  ],
})
export class AppModule {}
```

> info **Gợi ý** Sau khi ứng dụng đang chạy, mở trình duyệt của bạn và điều hướng đến `http://localhost:3000/graphiql`. Bạn sẽ thấy [GraphQL IDE](https://github.com/graphql/graphiql).

Phương thức `forRoot()` nhận một đối tượng options làm đối số. Các tùy chọn này được chuyển đến instance driver bên dưới. Đọc thêm về các cài đặt có sẵn [ở đây](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md#plugin-options).

#### Nhiều endpoint

Một tính năng hữu ích khác của module `@nestjs/graphql` là khả năng phục vụ nhiều endpoint cùng một lúc. Điều này cho phép bạn quyết định các module nào nên được bao gồm trong endpoint nào. Theo mặc định, `GraphQL` tìm kiếm resolvers trong toàn bộ ứng dụng. Để giới hạn quét này chỉ một tập hợp con của các module, sử dụng thuộc tính `include`.

```typescript
GraphQLModule.forRoot({
  include: [CatsModule],
}),
```

> warning **Cảnh báo** Nếu bạn sử dụng `@apollo/server` với gói `@as-integrations/fastify` với nhiều endpoint GraphQL trong một ứng dụng, hãy đảm bảo kích hoạt cài đặt `disableHealthCheck` trong cấu hình `GraphQLModule`.

#### Tích hợp bên thứ ba

- [GraphQL Yoga](https://github.com/dotansimha/graphql-yoga)

#### Ví dụ

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/33-graphql-mercurius).