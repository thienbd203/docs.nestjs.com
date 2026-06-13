### Scalars

Một loại đối tượng GraphQL có tên và các trường, nhưng tại một số điểm các trường đó phải giải quyết thành một số dữ liệu cụ thể. Đó là nơi các loại scalar xuất hiện: chúng đại diện cho các lá của query (đọc thêm [ở đây](https://graphql.org/learn/schema/#scalar-types)). GraphQL bao gồm các loại mặc định sau: `Int`, `Float`, `String`, `Boolean` và `ID`. Ngoài các loại tích hợp sẵn này, bạn có thể cần hỗ trợ các loại dữ liệu nguyên tử tùy chỉnh (ví dụ, `Date`).

#### Code first

Phương pháp code-first đi kèm với năm scalars trong đó ba trong số đó là các bí danh đơn giản cho các loại GraphQL hiện có.

- `ID` (bí danh cho `GraphQLID`) - đại diện cho một định danh duy nhất, thường được sử dụng để fetch lại một đối tượng hoặc làm khóa cho bộ nhớ cache
- `Int` (bí danh cho `GraphQLInt`) - một số nguyên 32-bit có dấu
- `Float` (bí danh cho `GraphQLFloat`) - một giá trị dấu phẩy động chính xác kép có dấu
- `GraphQLISODateTime` - một chuỗi ngày-giờ tại UTC (được sử dụng theo mặc định để đại diện cho loại `Date`)
- `GraphQLTimestamp` - một số nguyên có dấu đại diện cho ngày và giờ dưới dạng số mili-giây từ bắt đầu của kỷ nguyên UNIX

`GraphQLISODateTime` (ví dụ `2019-12-03T09:54:33Z`) được sử dụng theo mặc định để đại diện cho loại `Date`. Để sử dụng `GraphQLTimestamp` thay thế, đặt `dateScalarMode` của đối tượng `buildSchemaOptions` thành `'timestamp'` như sau:

```typescript
GraphQLModule.forRoot({
  buildSchemaOptions: {
    dateScalarMode: 'timestamp',
  }
}),
```

Tương tự, `GraphQLFloat` được sử dụng theo mặc định để đại diện cho loại `number`. Để sử dụng `GraphQLInt` thay thế, đặt `numberScalarMode` của đối tượng `buildSchemaOptions` thành `'integer'` như sau:

```typescript
GraphQLModule.forRoot({
  buildSchemaOptions: {
    numberScalarMode: 'integer',
  }
}),
```

Ngoài ra, bạn có thể tạo các scalars tùy chỉnh.

#### Ghi đè một scalar mặc định

Để tạo một triển khai tùy chỉnh cho scalar `Date`, chỉ cần tạo một lớp mới.

```typescript
import { Scalar, CustomScalar } from '@nestjs/graphql';
import { Kind, ValueNode } from 'graphql';

@Scalar('Date', () => Date)
export class DateScalar implements CustomScalar<number, Date> {
  description = 'Date custom scalar type';

  parseValue(value: number): Date {
    return new Date(value); // value from the client
  }

  serialize(value: Date): number {
    return value.getTime(); // value sent to the client
  }

  parseLiteral(ast: ValueNode): Date {
    if (ast.kind === Kind.INT) {
      return new Date(ast.value);
    }
    return null;
  }
}
```

Với điều này đã có sẵn, đăng ký `DateScalar` dưới dạng một nhà cung cấp.

```typescript
@Module({
  providers: [DateScalar],
})
export class CommonModule {}
```

Bây giờ chúng ta có thể sử dụng loại `Date` trong các lớp của chúng ta.

```typescript
@Field()
creationDate: Date;
```

#### Nhập một scalar tùy chỉnh

Để sử dụng một scalar tùy chỉnh, nhập và đăng ký nó dưới dạng một resolver. Chúng ta sẽ sử dụng gói `graphql-type-json` cho mục đích minh họa. Gói npm này định nghĩa một loại scalar GraphQL `JSON`.

Bắt đầu bằng cách cài đặt gói:

```bash
$ npm i --save graphql-type-json
```

Sau khi gói được cài đặt, chúng ta chuyển một resolver tùy chỉnh cho phương thức `forRoot()`:

```typescript
import GraphQLJSON from 'graphql-type-json';

@Module({
  imports: [
    GraphQLModule.forRoot({
      resolvers: { JSON: GraphQLJSON },
    }),
  ],
})
export class AppModule {}
```

Bây giờ chúng ta có thể sử dụng loại `JSON` trong các lớp của chúng ta.

```typescript
@Field(() => GraphQLJSON)
info: JSON;
```

Để một bộ các scalars hữu ích, hãy nhìn vào gói [graphql-scalars](https://www.npmjs.com/package/graphql-scalars).

#### Tạo một scalar tùy chỉnh

Để định nghĩa một scalar tùy chỉnh, tạo một thể hiện `GraphQLScalarType` mới. Chúng ta sẽ tạo một scalar `UUID` tùy chỉnh.

```typescript
const regex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

function validate(uuid: unknown): string | never {
  if (typeof uuid !== 'string' || !regex.test(uuid)) {
    throw new Error('invalid uuid');
  }
  return uuid;
}

export const CustomUuidScalar = new GraphQLScalarType({
  name: 'UUID',
  description: 'A simple UUID parser',
  serialize: (value) => validate(value),
  parseValue: (value) => validate(value),
  parseLiteral: (ast) => validate(ast.value),
});
```

Chúng ta chuyển một resolver tùy chỉnh cho phương thức `forRoot()`:

```typescript
@Module({
  imports: [
    GraphQLModule.forRoot({
      resolvers: { UUID: CustomUuidScalar },
    }),
  ],
})
export class AppModule {}
```

Bây giờ chúng ta có thể sử dụng loại `UUID` trong các lớp của chúng ta.

```typescript
@Field(() => CustomUuidScalar)
uuid: string;
```

#### Schema first

Để định nghĩa một scalar tùy chỉnh (đọc thêm về scalars [ở đây](https://www.apollographql.com/docs/graphql-tools/scalars.html)), tạo một định nghĩa loại và một resolver chuyên dụng. Ở đây (như trong tài liệu chính thức), chúng ta sẽ sử dụng gói `graphql-type-json` cho mục đích minh họa. Gói npm này định nghĩa một loại scalar GraphQL `JSON`.

Bắt đầu bằng cách cài đặt gói:

```bash
$ npm i --save graphql-type-json
```

Sau khi gói được cài đặt, chúng ta chuyển một resolver tùy chỉnh cho phương thức `forRoot()`:

```typescript
import GraphQLJSON from 'graphql-type-json';

@Module({
  imports: [
    GraphQLModule.forRoot({
      typePaths: ['./**/*.graphql'],
      resolvers: { JSON: GraphQLJSON },
    }),
  ],
})
export class AppModule {}
```

Bây giờ chúng ta có thể sử dụng scalar `JSON` trong các định nghĩa loại của chúng ta:

```graphql
scalar JSON

type Foo {
  field: JSON
}
```

Một phương thức khác để định nghĩa một loại scalar là tạo một lớp đơn giản. Giả sử chúng ta muốn nâng cao schema của chúng ta với loại `Date`.

```typescript
import { Scalar, CustomScalar } from '@nestjs/graphql';
import { Kind, ValueNode } from 'graphql';

@Scalar('Date')
export class DateScalar implements CustomScalar<number, Date> {
  description = 'Date custom scalar type';

  parseValue(value: number): Date {
    return new Date(value); // value from the client
  }

  serialize(value: Date): number {
    return value.getTime(); // value sent to the client
  }

  parseLiteral(ast: ValueNode): Date {
    if (ast.kind === Kind.INT) {
      return new Date(ast.value);
    }
    return null;
  }
}
```

Với điều này đã có sẵn, đăng ký `DateScalar` dưới dạng một nhà cung cấp.

```typescript
@Module({
  providers: [DateScalar],
})
export class CommonModule {}
```

Bây giờ chúng ta có thể sử dụng scalar `Date` trong các định nghĩa loại.

```graphql
scalar Date
```

Theo mặc định, định nghĩa TypeScript được tạo cho tất cả các scalars là `any` - điều này không đặc biệt an toàn về loại.
Nhưng, bạn có thể cấu hình cách Nest tạo typings cho các scalars tùy chỉnh của bạn khi bạn chỉ định cách tạo các loại:

```typescript
import { GraphQLDefinitionsFactory } from '@nestjs/graphql';
import { join } from 'path';

const definitionsFactory = new GraphQLDefinitionsFactory();

definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
  defaultScalarType: 'unknown',
  customScalarTypeMapping: {
    DateTime: 'Date',
    BigNumber: '_BigNumber',
  },
  additionalHeader: "import _BigNumber from 'bignumber.js'",
});
```

> info **Gợi ý** Ngoài ra, bạn có thể sử dụng một tham chiếu loại thay thế, ví dụ: `DateTime: Date`. Trong trường hợp này, `GraphQLDefinitionsFactory` sẽ trích xuất thuộc tính tên của loại được chỉ định (`Date.name`) để tạo các định nghĩa TS. Lưu ý: thêm một câu lệnh import cho các loại không tích hợp sẵn (các loại tùy chỉnh) là cần thiết.

Bây giờ, cho các loại scalar tùy chỉnh GraphQL sau:

```graphql
scalar DateTime
scalar BigNumber
scalar Payload
```

Chúng ta bây giờ sẽ thấy các định nghĩa TypeScript được tạo sau trong `src/graphql.ts`:

```typescript
import _BigNumber from 'bignumber.js';

export type DateTime = Date;
export type BigNumber = _BigNumber;
export type Payload = unknown;
```

Ở đây, chúng ta đã sử dụng thuộc tính `customScalarTypeMapping` để cung cấp một bản đồ của các loại chúng ta muốn khai báo cho các scalars tùy chỉnh của chúng ta. Chúng ta cũng đã cung cấp một thuộc tính `additionalHeader` để chúng ta có thể thêm bất kỳ imports nào cần thiết cho các định nghĩa loại này. Cuối cùng, chúng ta đã thêm một `defaultScalarType` của `'unknown'`, để bất kỳ scalars tùy chỉnh nào không được chỉ định trong `customScalarTypeMapping` sẽ được bí danh thành `unknown` thay vì `any` (mà [TypeScript khuyến nghị](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-0.html#new-unknown-top-type) sử dụng kể từ 3.0 để thêm an toàn về loại).

> info **Gợi ý** Lưu ý rằng chúng ta đã nhập `_BigNumber` từ `bignumber.js`; điều này là để tránh [các tham chiếu loại vòng tròn](https://github.com/Microsoft/TypeScript/issues/12525#issuecomment-263166239).