### Complexity

> warning **Cảnh báo** Chương này chỉ áp dụng cho phương pháp code first.

Query complexity cho phép bạn định nghĩa mức độ phức tạp của các trường nhất định, và hạn chế các query với **mức độ phức tạp tối đa**. Ý tưởng là định nghĩa mức độ phức tạp của mỗi trường bằng cách sử dụng một số đơn giản. Một mặc định phổ biến là cho mỗi trường một độ phức tạp của `1`. Ngoài ra, tính toán độ phức tạp của một query GraphQL có thể được tùy chỉnh với các bộ ước tính độ phức tạp (complexity estimators). Một bộ ước tính độ phức tạp là một hàm đơn giản tính toán độ phức tạp cho một trường. Bạn có thể thêm bất kỳ số lượng bộ ước tính độ phức tạp nào vào quy tắc, sau đó được thực hiện một lần một. Bộ ước tính đầu tiên trả về giá trị độ phức tạp số xác định độ phức tạp cho trường đó.

Gói `@nestjs/graphql` tích hợp rất tốt với các công cụ như [graphql-query-complexity](https://github.com/slicknode/graphql-query-complexity) cung cấp giải pháp dựa trên phân tích chi phí. Với thư viện này, bạn có thể từ chối các query đến server GraphQL của bạn được coi là quá tốn kém để thực thi.

#### Cài đặt

Để bắt đầu sử dụng nó, chúng ta trước tiên cài đặt dependency cần thiết.

```bash
$ npm install --save graphql-query-complexity
```

#### Bắt đầu

Sau khi quá trình cài đặt hoàn tất, chúng ta có thể định nghĩa lớp `ComplexityPlugin`:

```typescript
import { GraphQLSchemaHost } from '@nestjs/graphql';
import { Plugin } from '@nestjs/apollo';
import {
  ApolloServerPlugin,
  BaseContext,
  GraphQLRequestListener,
} from '@apollo/server';
import { GraphQLError } from 'graphql';
import {
  fieldExtensionsEstimator,
  getComplexity,
  simpleEstimator,
} from 'graphql-query-complexity';

@Plugin()
export class ComplexityPlugin implements ApolloServerPlugin {
  constructor(private gqlSchemaHost: GraphQLSchemaHost) {}

  async requestDidStart(): Promise<GraphQLRequestListener<BaseContext>> {
    const maxComplexity = 20;
    const { schema } = this.gqlSchemaHost;

    return {
      async didResolveOperation({ request, document }) {
        const complexity = getComplexity({
          schema,
          operationName: request.operationName,
          query: document,
          variables: request.variables,
          estimators: [
            fieldExtensionsEstimator(),
            simpleEstimator({ defaultComplexity: 1 }),
          ],
        });
        if (complexity > maxComplexity) {
          throw new GraphQLError(
            `Query is too complex: ${complexity}. Maximum allowed complexity: ${maxComplexity}`,
          );
        }
        console.log('Query Complexity:', complexity);
      },
    };
  }
}
```

Để mục đích minh họa, chúng ta đã chỉ định độ phức tạp tối đa cho phép là `20`. Trong ví dụ trên, chúng ta đã sử dụng 2 bộ ước tính, `simpleEstimator` và `fieldExtensionsEstimator`.

- `simpleEstimator`: bộ ước tính đơn giản trả về một độ phức tạp cố định cho mỗi trường
- `fieldExtensionsEstimator`: bộ ước tính mở rộng trường trích xuất giá trị độ phức tạp cho mỗi trường của schema của bạn

> info **Gợi ý** Nhớ thêm lớp này vào mảng providers trong bất kỳ module nào.

#### Độ phức tạp cấp trường

Với plugin này đã có sẵn, bây giờ chúng ta có thể định nghĩa độ phức tạp cho bất kỳ trường nào bằng cách chỉ định thuộc tính `complexity` trong đối tượng options được chuyển vào decorator `@Field()`, như sau:

```typescript
@Field({ complexity: 3 })
title: string;
```

Ngoài ra, bạn có thể định nghĩa hàm bộ ước tính:

```typescript
@Field({ complexity: (options: ComplexityEstimatorArgs) => ... })
title: string;
```

#### Độ phức tạp cấp Query/Mutation

Ngoài ra, các decorator `@Query()` và `@Mutation()` có thể có thuộc tính `complexity` được chỉ định như sau:

```typescript
@Query({ complexity: (options: ComplexityEstimatorArgs) => options.args.count * options.childComplexity })
items(@Args('count') count: number) {
  return this.itemsService.getItems({ count });
}
```