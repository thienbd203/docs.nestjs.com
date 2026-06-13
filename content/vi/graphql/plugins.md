### Plugins với Apollo

Plugins cho phép bạn mở rộng chức năng cốt lõi của Apollo Server bằng cách thực hiện các thao tác tùy chỉnh để phản hồi các sự kiện nhất định. Hiện tại, các sự kiện này tương ứng với các giai đoạn riêng lẻ của vòng đời yêu cầu GraphQL, và việc khởi động chính Apollo Server (đọc thêm [ở đây](https://www.apollographql.com/docs/apollo-server/integrations/plugins/)). Ví dụ, một plugin logging cơ bản có thể ghi lại chuỗi query GraphQL liên quan đến mỗi yêu cầu được gửi đến Apollo Server.

#### Plugins tùy chỉnh

Để tạo một plugin, khai báo một lớp được chú thích với decorator `@Plugin` được xuất từ gói `@nestjs/apollo`. Ngoài ra, để có autocompletion code tốt hơn, triển khai interface `ApolloServerPlugin` từ gói `@apollo/server`.

```typescript
import { ApolloServerPlugin, GraphQLRequestListener } from '@apollo/server';
import { Plugin } from '@nestjs/apollo';

@Plugin()
export class LoggingPlugin implements ApolloServerPlugin {
  async requestDidStart(): Promise<GraphQLRequestListener<any>> {
    console.log('Request started');
    return {
      async willSendResponse() {
        console.log('Will send response');
      },
    };
  }
}
```

Với điều này đã có sẵn, chúng ta có thể đăng ký `LoggingPlugin` dưới dạng một nhà cung cấp.

```typescript
@Module({
  providers: [LoggingPlugin],
})
export class CommonModule {}
```

Nest sẽ tự động khởi tạo một plugin và áp dụng nó cho Apollo Server.

#### Sử dụng plugins bên ngoài

Có một số plugin được cung cấp out-of-the-box. Để sử dụng một plugin hiện có, chỉ cần nhập nó và thêm nó vào mảng `plugins`:

```typescript
GraphQLModule.forRoot({
  // ...
  plugins: [ApolloServerOperationRegistry({ /* options */})]
}),
```

> info **Gợi ý** Plugin `ApolloServerOperationRegistry` được xuất từ gói `@apollo/server-plugin-operation-registry`.

#### Plugins với Mercurius

Một số plugin Fastify cụ thể của mercurius hiện có phải được tải sau plugin mercurius (đọc thêm [ở đây](https://mercurius.dev/#/docs/plugins)) trên cây plugin.

> warning **Cảnh báo** [mercurius-upload](https://github.com/mercurius-js/mercurius-upload) là một ngoại lệ và nên được đăng ký trong tệp chính.

Để làm điều này, `MercuriusDriver` phơi bày một tùy chọn cấu hình `plugins` tùy chọn. Nó đại diện cho một mảng các đối tượng bao gồm hai thuộc tính: `plugin` và `options` của nó. Do đó, đăng ký [cache plugin](https://github.com/mercurius-js/cache) sẽ trông như sau:

```typescript
GraphQLModule.forRoot({
  driver: MercuriusDriver,
  // ...
  plugins: [
    {
      plugin: cache,
      options: {
        ttl: 10,
        policy: {
          Query: {
            add: true
          }
        }
      },
    }
  ]
}),
```