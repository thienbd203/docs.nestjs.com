### Validation

Đây là thực hành tốt nhất để xác nhận tính đúng đắn của bất kỳ dữ liệu nào được gửi vào một ứng dụng web. Để tự động xác nhận các yêu cầu đến, Nest cung cấp một số pipes có sẵn ngay lập tức:

- `ValidationPipe`
- `ParseIntPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`

`ValidationPipe` tận dụng package [class-validator](https://github.com/typestack/class-validator) mạnh mẽ và các decorators xác nhận khai báo của nó. `ValidationPipe` cung cấp một cách tiếp cận tiện lợi để thực thi các quy tắc xác nhận cho tất cả các payloads client đến, trong đó các quy tắc cụ thể được khai báo với các chú thích đơn giản trong các khai báo class/DTO cục bộ trong mỗi module.

#### Tổng quan

Trong chương [Pipes](/pipes), chúng tôi đã đi qua quá trình xây dựng các pipes đơn giản và liên kết chúng với các controllers, phương thức hoặc ứng dụng toàn cục để chứng minh cách quá trình hoạt động. Hãy chắc chắn xem lại chương đó để hiểu tốt nhất các chủ đề của chương này. Ở đây, chúng tôi sẽ tập trung vào các trường hợp sử dụng **thực tế** khác nhau của `ValidationPipe`, và cho biết cách sử dụng một số tính năng tùy chỉnh nâng cao của nó.

#### Sử dụng ValidationPipe tích hợp

Để bắt đầu sử dụng nó, trước tiên chúng ta cài đặt dependency cần thiết.

```bash
$ npm i --save class-validator class-transformer
```

> info **Gợi ý** `ValidationPipe` được xuất từ package `@nestjs/common`.

Vì pipe này sử dụng các thư viện [`class-validator`](https://github.com/typestack/class-validator) và [`class-transformer`](https://github.com/typestack/class-transformer), có nhiều tùy chọn có sẵn. Bạn cấu hình các cài đặt này thông qua một đối tượng cấu hình được truyền đến pipe. Sau đây là các tùy chọn tích hợp:

```typescript
export interface ValidationPipeOptions extends ValidatorOptions {
  transform?: boolean;
  disableErrorMessages?: boolean;
  exceptionFactory?: (errors: ValidationError[]) => any;
  errorFormat?: 'list' | 'grouped';
}
```

Ngoài những điều này, tất cả các tùy chọn `class-validator` (kế thừa từ interface `ValidatorOptions`) đều có sẵn:

<table>
  <tr>
    <th>Tùy chọn</th>
    <th>Kiểu</th>
    <th>Mô tả</th>
  </tr>
  <tr>
    <td><code>enableDebugMessages</code></td>
    <td><code>boolean</code></td>
    <td>Nếu được đặt thành true, validator sẽ in các thông báo cảnh báo bổ sung vào bảng điều khiển khi có điều gì đó không đúng.</td>
  </tr>
  <tr>
    <td><code>skipUndefinedProperties</code></td>
    <td><code>boolean</code></td>
    <td>Nếu được đặt thành true thì validator sẽ bỏ qua xác nhận của tất cả các thuộc tính không được định nghĩa trong đối tượng xác nhận.</td>
  </tr>
  <tr>
    <td><code>skipNullProperties</code></td>
    <td><code>boolean</code></td>
    <td>Nếu được đặt thành true thì validator sẽ bỏ qua xác nhận của tất cả các thuộc tính là null trong đối tượng xác nhận.</td>
  </tr>
  <tr>
    <td><code>skipMissingProperties</code></td>
    <td><code>boolean</code></td>
    <td>Nếu được đặt thành true thì validator sẽ bỏ qua xác nhận của tất cả các thuộc tính là null hoặc không được định nghĩa trong đối tượng xác nhận.</td>
  </tr>
  <tr>
    <td><code>whitelist</code></td>
    <td><code>boolean</code></td>
    <td>Nếu được đặt thành true, validator sẽ cắt bỏ đối tượng được xác nhận (được trả về) của bất kỳ thuộc tính nào không sử dụng bất kỳ decorator xác nhận nào.</td>
  </tr>
  <tr>
    <td><code>forbidNonWhitelisted</code></td>
    <td><code>boolean</code></td>
    <td>Nếu được đặt thành true, thay vì cắt bỏ các thuộc tính không có trong whitelist, validator sẽ ném một exception.</td>
  </tr>
  <tr>
    <td><code>forbidUnknownValues</code></td>
    <td><code>boolean</code></td>
    <td>Nếu được đặt thành true, các nỗ lực xác nhận các đối tượng không xác định sẽ thất bại ngay lập tức.</td>
  </tr>
  <tr>
    <td><code>disableErrorMessages</code></td>
    <td><code>boolean</code></td>
    <td>Nếu được đặt thành true, các lỗi xác nhận sẽ không được trả về cho client.</td>
  </tr>
  <tr>
    <td><code>errorHttpStatusCode</code></td>
    <td><code>number</code></td>
    <td>Cài đặt này cho phép bạn chỉ định loại exception nào sẽ được sử dụng trong trường hợp lỗi. Theo mặc định nó ném <code>BadRequestException</code>.</td>
  </tr>
  <tr>
    <td><code>exceptionFactory</code></td>
    <td><code>Function</code></td>
    <td>Nhận một mảng các lỗi xác nhận và trả về một đối tượng exception để được ném.</td>
  </tr>
  <tr>
    <td><code>groups</code></td>
    <td><code>string[]</code></td>
    <td>Các nhóm được sử dụng trong quá trình xác nhận đối tượng.</td>
  </tr>
  <tr>
    <td><code>always</code></td>
    <td><code>boolean</code></td>
    <td>Đặt mặc định cho tùy chọn <code>always</code> của các decorators. Mặc định có thể được ghi đè trong các tùy chọn decorator.</td>
  </tr>

  <tr>
    <td><code>strictGroups</code></td>
    <td><code>boolean</code></td>
    <td>Nếu <code>groups</code> không được đưa ra hoặc trống, bỏ qua các decorators có ít nhất một nhóm.</td>
  </tr>
  <tr>
    <td><code>dismissDefaultMessages</code></td>
    <td><code>boolean</code></td>
    <td>Nếu được đặt thành true, xác nhận sẽ không sử dụng các thông báo mặc định. Thông báo lỗi luôn sẽ là <code>undefined</code> nếu nó không được đặt rõ ràng.</td>
  </tr>
  <tr>
    <td><code>validationError.target</code></td>
    <td><code>boolean</code></td>
    <td>Chỉ định xem target có nên được expose trong <code>ValidationError</code> hay không.</td>
  </tr>
  <tr>
    <td><code>validationError.value</code></td>
    <td><code>boolean</code></td>
    <td>Chỉ định xem giá trị được xác nhận có nên được expose trong <code>ValidationError</code> hay không.</td>
  </tr>
  <tr>
    <td><code>stopAtFirstError</code></td>
    <td><code>boolean</code></td>
    <td>Khi được đặt thành true, xác nhận của thuộc tính đã cho sẽ dừng sau khi gặp lỗi đầu tiên. Mặc định là false.</td>
  </tr>
  <tr>
    <td><code>errorFormat</code></td>
    <td><code>'list' | 'grouped'</code></td>
    <td>Chỉ định định dạng của các thông báo lỗi xác nhận. <code>'list'</code> (mặc định) trả về một mảng các chuỗi thông báo lỗi. <code>'grouped'</code> trả về một đối tượng với các đường dẫn thuộc tính làm khóa và các mảng các thông báo lỗi chưa được sửa đổi làm giá trị, bảo toàn các thông báo xác nhận tùy chỉnh mà không thêm tiền tố đường dẫn cha.</td>
  </tr>
</table>

> info **Lưu ý** Tìm thêm thông tin về package `class-validator` trong [repository](https://github.com/typestack/class-validator) của nó.

#### Tự động xác nhận

Chúng tôi sẽ bắt đầu bằng cách liên kết `ValidationPipe` ở cấp ứng dụng, do đó đảm bảo tất cả các endpoints được bảo vệ khỏi nhận dữ liệu không đúng.

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

Để kiểm thử pipe của chúng ta, hãy tạo một endpoint cơ bản.

```typescript
@Post()
create(@Body() createUserDto: CreateUserDto) {
  return 'This action adds a new user';
}
```

> info **Gợi ý** Vì TypeScript không lưu trữ metadata về **generics hoặc interfaces**, khi bạn sử dụng chúng trong các DTO của mình, `ValidationPipe` có thể không thể xác nhận đúng dữ liệu đến. Vì lý do này, hãy cân nhắc sử dụng các lớp cụ thể trong các DTO của bạn.

> info **Gợi ý** Khi nhập các DTO của bạn, bạn không thể sử dụng một nhập chỉ kiểu vì nó sẽ bị xóa tại runtime, tức là nhớ `import {{ '{' }} CreateUserDto {{ '}' }}` thay vì `import type {{ '{' }} CreateUserDto {{ '}' }}`.

Bây giờ chúng ta có thể thêm một vài quy tắc xác nhận trong `CreateUserDto` của chúng ta. Chúng tôi làm điều này bằng cách sử dụng các decorator được cung cấp bởi package `class-validator`, được mô tả chi tiết [ở đây](https://github.com/typestack/class-validator#validation-decorators). Theo cách này, bất kỳ route nào sử dụng `CreateUserDto` sẽ tự động thực thi các quy tắc xác nhận này.

```typescript
import { IsEmail, IsNotEmpty } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsNotEmpty()
  password: string;
}
```

Với các quy tắc này đã sẵn sàng, nếu một yêu cầu đánh vào endpoint của chúng ta với một thuộc tính `email` không hợp lệ trong body yêu cầu, ứng dụng sẽ tự động phản hồi với mã `400 Bad Request`, cùng với body phản hồi sau:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": ["email must be an email"]
}
```

Ngoài việc xác nhận các body yêu cầu, `ValidationPipe` cũng có thể được sử dụng với các thuộc tính đối tượng yêu cầu khác. Hãy tưởng tượng rằng chúng tôi muốn chấp nhận `:id` trong đường dẫn endpoint. Để đảm bảo rằng chỉ các số được chấp nhận cho tham số yêu cầu này, chúng ta có thể sử dụng cấu trúc sau:

```typescript
@Get(':id')
findOne(@Param() params: FindOneParams) {
  return 'This action returns a user';
}
```

`FindOneParams`, giống như một DTO, chỉ đơn giản là một class định nghĩa các quy tắc xác nhận sử dụng `class-validator`. Nó sẽ trông như sau:

```typescript
import { IsNumberString } from 'class-validator';

export class FindOneParams {
  @IsNumberString()
  id: string;
}
```

#### Vô hiệu hóa chi tiết lỗi

Thông báo lỗi có thể hữu ích để giải thích điều gì không chính xác trong một yêu cầu. Tuy nhiên, một số môi trường production thích vô hiệu hóa chi tiết lỗi. Làm điều này bằng cách truyền một đối tượng tùy chọn đến `ValidationPipe`:

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    disableErrorMessages: true,
  }),
);
```

Kết quả là, các thông báo lỗi chi tiết sẽ không được hiển thị trong body phản hồi.

#### Cắt bỏ thuộc tính

`ValidationPipe` của chúng ta cũng có thể lọc ra các thuộc tính không nên được nhận bởi phương thức handler. Trong trường hợp này, chúng ta có thể **whitelist** các thuộc tính có thể chấp nhận, và bất kỳ thuộc tính nào không được bao gồm trong whitelist được tự động cắt bỏ từ đối tượng kết quả. Ví dụ, nếu handler của chúng ta mong đợi các thuộc tính `email` và `password`, nhưng một yêu cầu cũng bao gồm một thuộc tính `age`, thuộc tính này có thể được tự động loại bỏ từ DTO kết quả. Để bật hành vi như vậy, đặt `whitelist` thành `true`.

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
  }),
);
```

Khi được đặt thành true, điều này sẽ tự động loại bỏ các thuộc tính không có trong whitelist (những thuộc tính không có decorator nào trong lớp xác nhận).

Ngoài ra, bạn có thể ngăn yêu cầu xử lý khi các thuộc tính không có trong whitelist hiện diện, và trả về một phản hồi lỗi cho người dùng. Để bật điều này, đặt thuộc tính tùy chọn `forbidNonWhitelisted` thành `true`, kết hợp với việc đặt `whitelist` thành `true`.

<app-banner-courses></app-banner-courses>

#### Chuyển đổi các đối tượng payload

Các payloads đến qua mạng là các đối tượng JavaScript thuần túy. `ValidationPipe` có thể tự động chuyển đổi các payloads thành các đối tượng được gõ theo các lớp DTO của chúng. Để bật chuyển đổi tự động, đặt `transform` thành `true`. Điều này có thể được thực hiện ở cấp phương thức:

```typescript
@@filename(cats.controller)
@Post()
@UsePipes(new ValidationPipe({ transform: true }))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Để bật hành vi này toàn cục, đặt tùy chọn trên một pipe toàn cục:

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,
  }),
);
```

Với tùy chọn chuyển đổi tự động được bật, `ValidationPipe` cũng sẽ thực hiện chuyển đổi của các kiểu nguyên thủy. Trong ví dụ sau, phương thức `findOne()` nhận một đối số đại diện cho một tham số đường dẫn `id` được trích xuất:

```typescript
@Get(':id')
findOne(@Param('id') id: number) {
  console.log(typeof id === 'number'); // true
  return 'This action returns a user';
}
```

Theo mặc định, mọi tham số đường dẫn và tham số query đến qua mạng như một `string`. Trong ví dụ trên, chúng ta đã chỉ định kiểu `id` như một `number` (trong chữ ký phương thức). Do đó, `ValidationPipe` sẽ cố gắng tự động chuyển đổi một định danh chuỗi thành một số.

#### Chuyển đổi rõ ràng

Trong phần trên, chúng tôi đã cho thấy cách `ValidationPipe` có thể chuyển đổi ngầm định các tham số query và đường dẫn dựa trên kiểu mong đợi. Tuy nhiên, tính năng này yêu cầu có chuyển đổi tự động được bật.

Ngoài ra (với chuyển đổi tự động bị vô hiệu hóa), bạn có thể ép kiểu rõ ràng các giá trị bằng cách sử dụng `ParseIntPipe` hoặc `ParseBoolPipe` (lưu ý rằng `ParseStringPipe` không cần thiết vì, như đã đề cập trước đó, mọi tham số đường dẫn và tham số query đến qua mạng như một `string` theo mặc định).

```typescript
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('sort', ParseBoolPipe) sort: boolean,
) {
  console.log(typeof id === 'number'); // true
  console.log(typeof sort === 'boolean'); // true
  return 'This action returns a user';
}
```

> info **Gợi ý** `ParseIntPipe` và `ParseBoolPipe` được xuất từ package `@nestjs/common`.

#### Các kiểu được ánh xạ

Khi bạn xây dựng các tính năng như **CRUD** (Create/Read/Update/Delete), việc xây dựng các biến thể trên một kiểu entity cơ bản thường hữu ích. Nest cung cấp một số hàm tiện ích thực hiện các chuyển đổi kiểu để làm cho nhiệm vụ này tiện lợi hơn.

> **Cảnh báo** Nếu ứng dụng của bạn sử dụng package `@nestjs/swagger`, xem [chương này](/openapi/mapped-types) để biết thêm thông tin về Mapped Types. Tương tự, nếu bạn sử dụng package `@nestjs/graphql`, xem [chương này](/graphql/mapped-types). Cả hai package đều phụ thuộc nhiều vào các kiểu và do đó chúng yêu cầu một nhập khác để được sử dụng. Do đó, nếu bạn sử dụng `@nestjs/mapped-types` (thay vì một thích hợp, tùy thuộc vào loại ứng dụng của bạn, là `@nestjs/swagger` hoặc `@nestjs/graphql`), bạn có thể gặp các tác động phụ khác nhau, không được tài liệu hóa.

Khi xây dựng các kiểu xác nhận đầu vào (cũng được gọi là DTOs), việc xây dựng các biến thể **create** và **update** trên cùng một kiểu thường hữu ích. Ví dụ, biến thể **create** có thể yêu cầu tất cả các trường, trong khi biến thể **update** có thể làm cho tất cả các trường tùy chọn.

Nest cung cấp hàm tiện ích `PartialType()` để làm cho nhiệm vụ này dễ dàng hơn và giảm boilerplate.

Hàm `PartialType()` trả về một kiểu (class) với tất cả các thuộc tính của kiểu đầu vào được đặt thành tùy chọn. Ví dụ, giả sử chúng ta có một kiểu **create** như sau:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Theo mặc định, tất cả các trường này là bắt buộc. Để tạo một kiểu với cùng các trường, nhưng với mỗi trường tùy chọn, sử dụng `PartialType()` truyền tham chiếu class (`CreateCatDto`) làm đối số:

```typescript
export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

> info **Gợi ý** Hàm `PartialType()` được nhập từ package `@nestjs/mapped-types`.

Hàm `PickType()` xây dựng một kiểu mới (class) bằng cách chọn một tập hợp các thuộc tính từ một kiểu đầu vào. Ví dụ, giả sử chúng ta bắt đầu với một kiểu như:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Chúng ta có thể chọn một tập hợp các thuộc tính từ class này bằng cách sử dụng hàm tiện ích `PickType()`:

```typescript
export class UpdateCatAgeDto extends PickType(CreateCatDto, ['age'] as const) {}
```

> info **Gợi ý** Hàm `PickType()` được nhập từ package `@nestjs/mapped-types`.

Hàm `OmitType()` xây dựng một kiểu bằng cách chọn tất cả các thuộc tính từ một kiểu đầu vào và sau đó loại bỏ một tập hợp khóa cụ thể. Ví dụ, giả sử chúng ta bắt đầu với một kiểu như:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Chúng ta có thể tạo một kiểu dẫn xuất có mọi thuộc tính **ngoại trừ** `name` như được hiển thị dưới đây. Trong cấu trúc này, đối số thứ hai cho `OmitType` là một mảng tên thuộc tính.

```typescript
export class UpdateCatDto extends OmitType(CreateCatDto, ['name'] as const) {}
```

> info **Gợi ý** Hàm `OmitType()` được nhập từ package `@nestjs/mapped-types`.

Hàm `IntersectionType()` kết hợp hai kiểu thành một kiểu mới (class). Ví dụ, giả sử chúng ta bắt đầu với hai kiểu như:

```typescript
export class CreateCatDto {
  name: string;
  breed: string;
}

export class AdditionalCatInfo {
  color: string;
}
```

Chúng ta có thể tạo một kiểu mới kết hợp tất cả các thuộc tính trong cả hai kiểu.

```typescript
export class UpdateCatDto extends IntersectionType(
  CreateCatDto,
  AdditionalCatInfo,
) {}
```

> info **Gợi ý** Hàm `IntersectionType()` được nhập từ package `@nestjs/mapped-types`.

Các hàm tiện ích ánh xạ kiểu có thể kết hợp. Ví dụ, sau đây sẽ tạo ra một kiểu (class) có tất cả các thuộc tính của kiểu `CreateCatDto` ngoại trừ `name`, và các thuộc tính đó sẽ được đặt thành tùy chọn:

```typescript
export class UpdateCatDto extends PartialType(
  OmitType(CreateCatDto, ['name'] as const),
) {}
```

#### Phân tích cú pháp và xác nhận mảng

TypeScript không lưu trữ metadata về generics hoặc interfaces, vì vậy khi bạn sử dụng chúng trong các DTO của mình, `ValidationPipe` có thể không thể xác nhận đúng dữ liệu đến. Ví dụ, trong mã sau, `createUserDtos` sẽ không được xác nhận đúng:

```typescript
@Post()
createBulk(@Body() createUserDtos: CreateUserDto[]) {
  return 'This action adds new users';
}
```

Để xác nhận mảng, tạo một class chuyên dụng chứa một thuộc tính bọc mảng, hoặc sử dụng `ParseArrayPipe`.

```typescript
@Post()
createBulk(
  @Body(new ParseArrayPipe({ items: CreateUserDto }))
  createUserDtos: CreateUserDto[],
) {
  return 'This action adds new users';
}
```

Ngoài ra, `ParseArrayPipe` có thể hữu ích khi phân tích cú pháp các tham số query. Hãy xem xét một phương thức `findByIds()` trả về người dùng dựa trên các định danh được truyền như tham số query.

```typescript
@Get()
findByIds(
  @Query('ids', new ParseArrayPipe({ items: Number, separator: ',' }))
  ids: number[],
) {
  return 'This action returns users by ids';
}
```

Cấu trúc này xác nhận các tham số query đến từ một yêu cầu HTTP `GET` như sau:

```bash
GET /?ids=1,2,3
```

#### WebSockets và Microservices

Trong khi chương này hiển thị các ví dụ sử dụng các ứng dụng kiểu HTTP (ví dụ, Express hoặc Fastify), `ValidationPipe` hoạt động theo cùng một cách cho WebSockets và microservices, bất kể phương thức vận chuyển được sử dụng.

#### Tìm hiểu thêm

Đọc thêm về các validators tùy chỉnh, thông báo lỗi và các decorators có sẵn được cung cấp bởi package `class-validator` [ở đây](https://github.com/typestack/class-validator).