### Directives

Một directive có thể được đính kèm vào một trường hoặc bao gồm fragment, và có thể ảnh hưởng đến việc thực thi query theo bất kỳ cách nào server mong muốn (đọc thêm [ở đây](https://graphql.org/learn/queries/#directives)). Đặc tả GraphQL cung cấp một số directive mặc định:

- `@include(if: Boolean)` - chỉ bao gồm trường này trong kết quả nếu đối số là true
- `@skip(if: Boolean)` - bỏ qua trường này nếu đối số là true
- `@deprecated(reason: String)` - đánh dấu trường là deprecated với thông báo

Một directive là một định danh được đặt trước bởi ký tự `@`, tùy chọn theo sau bởi một danh sách các đối số được đặt tên, có thể xuất hiện sau hầu hết mọi phần tử trong ngôn ngữ query và schema GraphQL.

#### Directives tùy chỉnh

Để chỉ định những gì nên xảy ra khi Apollo/Mercurius gặp directive của bạn, bạn có thể tạo một hàm transformer. Hàm này sử dụng hàm `mapSchema` để lặp qua các vị trí trong schema của bạn (định nghĩa trường, định nghĩa loại, v.v.) và thực hiện các chuyển đổi tương ứng.

```typescript
import { getDirective, MapperKind, mapSchema } from '@graphql-tools/utils';
import { defaultFieldResolver, GraphQLSchema } from 'graphql';

export function upperDirectiveTransformer(
  schema: GraphQLSchema,
  directiveName: string,
) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const upperDirective = getDirective(
        schema,
        fieldConfig,
        directiveName,
      )?.[0];

      if (upperDirective) {
        const { resolve = defaultFieldResolver } = fieldConfig;

        // Thay thế resolver gốc bằng một hàm *đầu tiên* gọi
        // resolver gốc, sau đó chuyển đổi kết quả của nó thành chữ hoa
        fieldConfig.resolve = async function (source, args, context, info) {
          const result = await resolve(source, args, context, info);
          if (typeof result === 'string') {
            return result.toUpperCase();
          }
          return result;
        };
        return fieldConfig;
      }
    },
  });
}
```

Bây giờ, áp dụng hàm chuyển đổi `upperDirectiveTransformer` trong phương thức `GraphQLModule#forRoot` sử dụng hàm `transformSchema`:

```typescript
GraphQLModule.forRoot({
  // ...
  transformSchema: (schema) => upperDirectiveTransformer(schema, 'upper'),
});
```

Sau khi được đăng ký, directive `@upper` có thể được sử dụng trong schema của chúng ta. Tuy nhiên, cách bạn áp dụng directive sẽ thay đổi tùy thuộc vào phương pháp bạn sử dụng (code first hoặc schema first).

#### Code first

Trong phương pháp code first, sử dụng decorator `@Directive()` để áp dụng directive.

```typescript
@Directive('@upper')
@Field()
title: string;
```

> info **Gợi ý** Decorator `@Directive()` được xuất từ gói `@nestjs/graphql`.

Directives có thể được áp dụng trên các trường, resolver trường, input và object types, cũng như queries, mutations, và subscriptions. Dưới đây là một ví dụ về directive được áp dụng ở cấp bộ xử lý query:

```typescript
@Directive('@deprecated(reason: "This query will be removed in the next version")')
@Query(() => Author, { name: 'author' })
async getAuthor(@Args({ name: 'id', type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

> warn **Cảnh báo** Các directive được áp dụng thông qua decorator `@Directive()` sẽ không được phản ánh trong tệp định nghĩa schema được tạo.

Cuối cùng, đảm bảo khai báo directives trong `GraphQLModule`, như sau:

```typescript
GraphQLModule.forRoot({
  // ...,
  transformSchema: schema => upperDirectiveTransformer(schema, 'upper'),
  buildSchemaOptions: {
    directives: [
      new GraphQLDirective({
        name: 'upper',
        locations: [DirectiveLocation.FIELD_DEFINITION],
      }),
    ],
  },
}),
```

> info **Gợi ý** Cả `GraphQLDirective` và `DirectiveLocation` đều được xuất từ gói `graphql`.

#### Schema first

Trong phương pháp schema first, áp dụng directives trực tiếp trong SDL.

```graphql
directive @upper on FIELD_DEFINITION

type Post {
  id: Int!
  title: String! @upper
  votes: Int
}
```