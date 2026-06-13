### Pipes

Pipe là một class được annotate với decorator `@Injectable()`, implements interface `PipeTransform`.

<figure>
  <img class="illustrative-image" src="/assets/Pipe_1.png" />
</figure>

Pipes có hai trường hợp sử dụng điển hình:

- **transformation**: chuyển đổi dữ liệu đầu vào thành dạng mong muốn (ví dụ, từ string sang integer)
- **validation**: đánh giá dữ liệu đầu vào và nếu hợp lệ, chỉ cần để nó đi qua không thay đổi; nếu không, throw một exception

Trong cả hai trường hợp, pipes hoạt động trên các `arguments` đang được xử lý bởi một controller route handler. Nest chèn một pipe ngay trước khi một method được gọi, và pipe nhận các arguments định sẵn cho method và hoạt động trên chúng. Bất kỳ hoạt động chuyển đổi hoặc validation nào diễn ra tại thời điểm đó, sau đó route handler được gọi với bất kỳ (có thể) các arguments đã chuyển đổi.

Nest đi kèm với một số built-in pipes mà bạn có thể sử dụng out-of-the-box. Bạn cũng có thể xây dựng các custom pipes của riêng mình. Trong chương này, chúng ta sẽ giới thiệu các built-in pipes và cho thấy cách bind chúng đến các route handlers. Sau đó chúng ta sẽ xem xét một số custom-built pipes để cho thấy cách bạn có thể xây dựng một cái từ đầu.

> info **Gợi ý** Pipes chạy bên trong vùng exceptions. Điều này có nghĩa là khi một Pipe throw một exception nó được xử lý bởi lớp exceptions (global exceptions filter và bất kỳ exceptions filters nào được áp dụng cho context hiện tại). Cho thấy ở trên, nên rõ ràng rằng khi một exception được throw trong một Pipe, không có controller method nào được thực thi sau đó. Điều này cho bạn một kỹ thuật best-practice để validate dữ liệu đến ứng dụng từ các nguồn bên ngoài tại ranh giới hệ thống.

#### Built-in pipes

Nest đi kèm với một số pipes có sẵn out-of-the-box:

- `ValidationPipe`
- `ParseIntPipe`
- `ParseFloatPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `ParseEnumPipe`
- `DefaultValuePipe`
- `ParseFilePipe`
- `ParseDatePipe`

Chúng được export từ package `@nestjs/common`.

Hãy xem nhanh việc sử dụng `ParseIntPipe`. Đây là một ví dụ của trường hợp sử dụng **transformation**, trong đó pipe đảm bảo rằng một tham số method handler được chuyển đổi thành một JavaScript integer (hoặc throw một exception nếu chuyển đổi thất bại). Sau trong chương này, chúng ta sẽ cho thấy một simple custom implementation cho một `ParseIntPipe`. Các kỹ thuật ví dụ dưới đây cũng áp dụng cho các built-in transformation pipes khác (`ParseBoolPipe`, `ParseFloatPipe`, `ParseEnumPipe`, `ParseArrayPipe`, `ParseDatePipe`, và `ParseUUIDPipe`, mà chúng ta sẽ gọi là các pipes `Parse*` trong chương này).

#### Binding pipes

Để sử dụng một pipe, chúng ta cần bind một instance của class pipe đến context phù hợp. Trong ví dụ `ParseIntPipe` của chúng ta, chúng ta muốn liên kết pipe với một method handler route cụ thể, và đảm bảo nó chạy trước khi method được gọi. Chúng ta làm như vậy với cấu trúc sau, mà chúng ta sẽ gọi là binding pipe ở mức tham số method:

```typescript
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

Điều này đảm bảo rằng một trong hai điều kiện sau là đúng: hoặc tham số chúng ta nhận trong method `findOne()` là một số (như mong đợi trong cuộc gọi của chúng ta đến `this.catsService.findOne()`), hoặc một exception được throw trước khi route handler được gọi.

Ví dụ, giả sử route được gọi như:

```bash
GET localhost:3000/abc
```

Nest sẽ throw một exception như sau:

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

Exception sẽ ngăn phần thân của method `findOne()` thực thi.

Trong ví dụ trên, chúng ta truyền một class (`ParseIntPipe`), không phải một instance, để lại trách nhiệm instantiation cho framework và cho phép dependency injection. Như với pipes và guards, chúng ta có thể thay vào đó truyền một instance in-place. Truyền một instance in-place là hữu ích nếu chúng ta muốn tùy chỉnh hành vi của built-in pipe bằng cách truyền các options:

```typescript
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

Binding các transformation pipes khác (tất cả các pipes **Parse***) hoạt động tương tự. Tất cả các pipes này hoạt động trong context của validating các tham số route, tham số query string và các giá trị request body.

Ví dụ với một tham số query string:

```typescript
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

Đây là một ví dụ sử dụng `ParseUUIDPipe` để parse một tham số string và validate nếu nó là một UUID.

```typescript
@@filename()
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
@@switch
@Get(':uuid')
@Bind(Param('uuid', new ParseUUIDPipe()))
async findOne(uuid) {
  return this.catsService.findOne(uuid);
}
```

> info **Gợi ý** Khi sử dụng `ParseUUIDPipe()` bạn đang parse UUID trong phiên bản 3, 4 hoặc 5, nếu bạn chỉ yêu cầu một phiên bản cụ thể của UUID bạn có thể truyền một phiên bản trong các options của pipe.

Ở trên chúng ta đã thấy các ví dụ về binding các built-in pipes khác nhau của họ `Parse*`. Binding validation pipes thì hơi khác một chút; chúng ta sẽ thảo luận điều đó trong phần sau.

> info **Gợi ý** Ngoài ra, xem Validation techniques để có các ví dụ rộng rãi về validation pipes.

#### Custom pipes

Như đã đề cập, bạn có thể xây dựng các custom pipes của riêng bạn. Trong khi Nest cung cấp một built-in `ParseIntPipe` và `ValidationPipe` mạnh mẽ, hãy xây dựng các phiên bản custom đơn giản của mỗi cái từ đầu để xem các custom pipes được xây dựng như thế nào.

Chúng ta bắt đầu với một `ValidationPipe` đơn giản. Ban đầu, chúng ta sẽ để nó chỉ nhận một giá trị đầu vào và ngay lập tức trả về cùng một giá trị, hoạt động như một identity function.

```typescript
@@filename(validation.pipe)
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class ValidationPipe {
  transform(value, metadata) {
    return value;
  }
}
```

> info **Gợi ý** `PipeTransform<T, R>` là một generic interface phải được implement bởi bất kỳ pipe nào. Generic interface sử dụng `T` để chỉ định loại của giá trị đầu vào `value`, và `R` để chỉ định loại trả về của method `transform()`.

Mọi pipe phải implement method `transform()` để fulfill hợp đồng interface `PipeTransform`. Method này có hai tham số:

- `value`
- `metadata`

Tham số `value` là argument method hiện tại đang được xử lý (trước khi nó được nhận bởi method xử lý route), và `metadata` là metadata của argument method hiện tại đang được xử lý. Đối tượng metadata có các thuộc tính này:

```typescript
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

Các thuộc tính này mô tả argument hiện tại đang được xử lý.

<table>
  <tr>
    <td>
      <code>type</code>
    </td>
    <td>Chỉ định liệu argument là một body
      <code>@Body()</code>, query
      <code>@Query()</code>, param
      <code>@Param()</code>, hoặc một custom parameter (đọc thêm
      ở đây).</td>
  </tr>
  <tr>
    <td>
      <code>metatype</code>
    </td>
    <td>
      Cung cấp metatype của argument, ví dụ,
      <code>String</code>. Lưu ý: giá trị là
      <code>undefined</code> nếu bạn hoặc bỏ qua một khai báo loại trong signature method handler route, hoặc sử dụng vanilla JavaScript.
    </td>
  </tr>
  <tr>
    <td>
      <code>data</code>
    </td>
    <td>Chuỗi được truyền đến decorator, ví dụ
      <code>@Body('string')</code>. Nó là
      <code>undefined</code> nếu bạn để trống parenthesis của decorator.</td>
  </tr>
</table>

> warning **Cảnh báo** TypeScript interfaces biến mất trong quá trình transpilation. Do đó, nếu loại của tham số method được khai báo như một interface thay vì một class, giá trị `metatype` sẽ là `Object`.

#### Schema based validation

Hãy làm cho validation pipe của chúng ta hữu ích hơn một chút. Nhìn kỹ hơn vào method `create()` của `CatsController`, nơi chúng ta có thể muốn đảm bảo rằng đối tượng post body là hợp lệ trước khi cố gắng chạy method service của chúng ta.

```typescript
@@filename()
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
async create(@Body() createCatDto) {
  this.catsService.create(createCatDto);
}
```

Hãy tập trung vào tham số body `createCatDto`. Loại của nó là `CreateCatDto`:

```typescript
@@filename(create-cat.dto)
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Chúng ta muốn đảm bảo rằng bất kỳ yêu cầu đến nào đến method create chứa một body hợp lệ. Vì vậy chúng ta phải validate ba thành viên của đối tượng `createCatDto`. Chúng ta có thể làm điều này bên trong method handler route, nhưng làm như vậy không lý tưởng vì nó sẽ phá vỡ nguyên tắc trách nhiệm đơn lẻ (SRP).

Một cách tiếp cận khác có thể là tạo một **validator class** và ủy thác nhiệm vụ ở đó. Điều này có bất lợi là chúng ta sẽ phải nhớ gọi validator này ở đầu mỗi method.

Làm thế nào về việc tạo validation middleware? Điều này có thể hoạt động, nhưng đáng tiếc, không thể tạo **generic middleware** có thể được sử dụng trên tất cả các contexts trên toàn bộ ứng dụng. Điều này là do middleware không biết về **execution context**, bao gồm handler sẽ được gọi và bất kỳ tham số nào của nó.

Đây, tất nhiên, chính xác là trường hợp sử dụng mà pipes được thiết kế. Vì vậy hãy tiếp tục và tinh chỉnh validation pipe của chúng ta.

<app-banner-courses></app-banner-courses>

#### Object schema validation

Có một số cách tiếp cận có sẵn để thực hiện object validation theo cách sạch sẽ, DRY. Một cách tiếp cận phổ biến là sử dụng validation **dựa trên schema**. Hãy tiếp tục và thử cách tiếp cận đó.

Thư viện Zod cho phép bạn tạo các schemas theo cách straightforward, với một API readable. Hãy xây dựng một validation pipe sử dụng các schemas dựa trên Zod.

Bắt đầu bằng cách cài đặt package cần thiết:

```bash
$ npm install --save zod
```

Trong mẫu code dưới đây, chúng ta tạo một class đơn giản nhận một schema như một tham số `constructor`. Sau đó chúng ta áp dụng method `schema.parse()`, validate argument đến của chúng ta với schema được cung cấp.

Như đã lưu ý trước đó, một **validation pipe** hoặc trả về giá trị không thay đổi hoặc throw một exception.

Trong phần tiếp theo, bạn sẽ thấy cách chúng ta cung cấp schema phù hợp cho một method controller cụ thể sử dụng decorator `@UsePipes()`. Làm như vậy làm cho validation pipe của chúng ta có thể tái sử dụng trên các contexts, giống như chúng ta đã đặt ra để làm.

```typescript
@@filename()
import { PipeTransform, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ZodSchema  } from 'zod';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
@@switch
import { BadRequestException } from '@nestjs/common';

export class ZodValidationPipe {
  constructor(private schema) {}

  transform(value, metadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }

}
```

#### Binding validation pipes

Trước đó, chúng ta đã thấy cách bind transformation pipes (như `ParseIntPipe` và phần còn lại của các pipes `Parse*`).

Binding validation pipes cũng rất straightforward.

Trong trường hợp này, chúng ta muốn bind pipe ở mức cuộc gọi method. Trong ví dụ hiện tại của chúng ta, chúng ta cần làm như sau để sử dụng `ZodValidationPipe`:

1. Tạo một instance của `ZodValidationPipe`
2. Truyền Zod schema context-specific trong constructor class của pipe
3. Bind pipe đến method

Ví dụ schema Zod:

```typescript
import { z } from 'zod';

export const createCatSchema = z
  .object({
    name: z.string(),
    age: z.number(),
    breed: z.string(),
  })
  .required();

export type CreateCatDto = z.infer<typeof createCatSchema>;
```

Chúng ta làm như vậy sử dụng decorator `@UsePipes()` như được hiển thị dưới đây:

```typescript
@@filename(cats.controller)
@Post()
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Bind(Body())
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **Gợi ý** Decorator `@UsePipes()` được import từ package `@nestjs/common`.

`ZodValidationPipe` có thể được áp dụng cho các tham số cụ thể và cùng với các built-in pipes. Ví dụ, nơi chúng ta muốn validate tham số `id` đường dẫn route với `ParseIntPipe` riêng biệt từ request body:

```typescript
@Put('/:id')
async update(
  @Param('id', ParseIntPipe) id: number,
  @Body(new ZodValidationPipe(createCatSchema)) body: CreateCatDto
): void {
  this.catsService.update(id, body);
}
```

> warning **Cảnh báo** Thư viện `zod` yêu cầu cấu hình `strictNullChecks` được kích hoạt trong file `tsconfig.json` của bạn.

#### Class validator

> warning **Cảnh báo** Các kỹ thuật trong phần này yêu cầu TypeScript và không có sẵn nếu ứng dụng của bạn được viết sử dụng vanilla JavaScript.

Hãy xem một implementation thay thế cho kỹ thuật validation của chúng ta.

Nest hoạt động tốt với thư viện class-validator. Thư viện mạnh mẽ này cho phép bạn sử dụng validation dựa trên decorator. Validation dựa trên decorator cực kỳ mạnh mẽ, đặc biệt khi kết hợp với các khả năng **Pipe** của Nest vì chúng ta có quyền truy cập vào `metatype` của thuộc tính được xử lý. Trước khi bắt đầu, chúng ta cần cài đặt các packages cần thiết:

```bash
$ npm i --save class-validator class-transformer
```

Khi những cái này được cài đặt, chúng ta có thể thêm một số decorators đến class `CreateCatDto`. Ở đây chúng ta thấy một lợi ích đáng kể của kỹ thuật này: class `CreateCatDto` vẫn là nguồn sự thật duy nhất cho đối tượng Post body của chúng ta (thay vì phải tạo một class validation riêng biệt).

```typescript
@@filename(create-cat.dto)
import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

> info **Gợi ý** Đọc thêm về các decorators class-validator ở đây.

Bây giờ chúng ta có thể tạo một class `ValidationPipe` sử dụng các annotations này.

```typescript
@@filename(validation.pipe)
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

> info **Gợi ý** Như một lời nhắc nhở, bạn không phải xây dựng một validation pipe generic của riêng bạn vì `ValidationPipe` được cung cấp bởi Nest out-of-the-box. Built-in `ValidationPipe` cung cấp nhiều options hơn mẫu chúng ta xây dựng trong chương này, đã được giữ cơ bản cho mục đích minh họa cơ chế của một custom-built pipe. Bạn có thể tìm thấy đầy đủ chi tiết, cùng với nhiều ví dụ ở đây.

> warning **Lưu ý** Chúng ta đã sử dụng thư viện class-transformer ở trên được tạo bởi cùng tác giả với thư viện **class-validator**, và kết quả là, chúng chơi rất tốt với nhau.

Hãy đi qua code này. Đầu tiên, lưu ý rằng method `transform()` được đánh dấu là `async`. Điều này có thể vì Nest hỗ trợ cả synchronous và **asynchronous** pipes. Chúng ta làm method này `async` vì một số validations class-validator có thể async (sử dụng Promises).

Tiếp theo lưu ý rằng chúng ta đang sử dụng destructuring để extract trường metatype (chỉ extract thành viên này từ một `ArgumentMetadata`) vào tham số `metatype` của chúng ta. Điều này chỉ là shorthand để lấy `ArgumentMetadata` đầy đủ và sau đó có một statement bổ sung để gán biến metatype.

Tiếp theo, lưu ý hàm helper `toValidate()`. Nó chịu trách nhiệm bỏ qua bước validation khi argument hiện tại đang được xử lý là một kiểu JavaScript native (những cái này không thể có validation decorators attached, vì vậy không có lý do để chạy chúng qua bước validation).

Tiếp theo, chúng ta sử dụng function `plainToInstance()` của class-transformer để chuyển đổi đối tượng argument JavaScript plain của chúng ta thành một đối tượng typed để chúng ta có thể áp dụng validation. Lý do chúng ta phải làm điều này là đối tượng post body đến, khi deserialized từ yêu cầu mạng, không **có bất kỳ thông tin loại nào** (đây là cách nền tảng cơ bản, chẳng hạn như Express, hoạt động). Class-validator cần sử dụng các validation decorators chúng ta định nghĩa cho DTO của chúng ta trước đó, vì vậy chúng ta cần thực hiện chuyển đổi này để xử lý body đến như một đối tượng được decorate thích hợp, không chỉ là một đối tượng vanilla plain.

Cuối cùng, như đã lưu ý trước đó, vì đây là một **validation pipe** nó hoặc trả về giá trị không thay đổi, hoặc throw một exception.

Bước cuối cùng là bind `ValidationPipe`. Pipes có thể là parameter-scoped, method-scoped, controller-scoped, hoặc global-scoped. Trước đó, với validation pipe dựa trên Zod của chúng ta, chúng ta đã thấy một ví dụ về binding pipe ở mức method.

Trong ví dụ dưới đây, chúng ta sẽ bind instance pipe đến decorator route handler `@Body()` để pipe của chúng ta được gọi để validate post body.

```typescript
@@filename(cats.controller)
@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

Parameter-scoped pipes hữu ích khi logic validation chỉ quan tâm đến một tham số được chỉ định.

#### Global scoped pipes

Vì `ValidationPipe` được tạo để càng generic càng tốt, chúng ta có thể thực hiện tiện ích đầy đủ của nó bằng cách thiết lập nó như một pipe **global-scoped** để nó được áp dụng cho mọi route handler trên toàn bộ ứng dụng.

```typescript
@@filename(main)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> warning **Lưu ý** Trong trường hợp hybrid apps method `useGlobalPipes()` không thiết lập pipes cho gateways và microservices. Đối với các ứng dụng microservice "standard" (non-hybrid), `useGlobalPipes()` mount pipes global.

Global pipes được sử dụng trên toàn bộ ứng dụng, cho mọi controller và mọi route handler.

Lưu ý rằng về mặt dependency injection, global pipes được đăng ký từ bên ngoài bất kỳ module nào (với `useGlobalPipes()` như trong ví dụ trên) không thể inject dependencies vì binding đã được thực hiện bên ngoài context của bất kỳ module nào. Để giải quyết vấn đề này, bạn có thể thiết lập một global pipe **trực tiếp từ bất kỳ module nào** sử dụng cấu trúc sau:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

> info **Gợi ý** Khi sử dụng cách tiếp cận này để thực hiện dependency injection cho pipe, lưu ý rằng bất kể module nơi cấu trúc này được sử dụng, pipe thực tế là global. Nên làm điều này ở đâu? Chọn module nơi pipe (`ValidationPipe` trong ví dụ trên) được định nghĩa. Ngoài ra, `useClass` không phải là cách duy nhất để xử lý đăng ký custom provider. Tìm hiểu thêm ở đây.

#### The built-in ValidationPipe

Như một lời nhắc nhở, bạn không phải xây dựng một validation pipe generic của riêng bạn vì `ValidationPipe` được cung cấp bởi Nest out-of-the-box. Built-in `ValidationPipe` cung cấp nhiều options hơn mẫu chúng ta xây dựng trong chương này, đã được giữ cơ bản cho mục đích minh họa cơ chế của một custom-built pipe. Bạn có thể tìm thấy đầy đủ chi tiết, cùng với nhiều ví dụ ở đây.

#### Transformation use case

Validation không phải là trường hợp sử dụng duy nhất cho custom pipes. Ở đầu chương này, chúng ta đề cập rằng một pipe cũng có thể **transform** dữ liệu đầu vào thành định dạng mong muốn. Điều này có thể vì giá trị trả về từ function `transform` hoàn toàn ghi đè giá trị trước đó của argument.

Khi nào điều này hữu ích? Hãy xem xét rằng đôi khi dữ liệu được truyền từ client cần trải qua một số thay đổi - ví dụ chuyển đổi một string thành một integer - trước khi nó có thể được xử lý đúng bởi method handler route. Hơn nữa, một số trường dữ liệu cần thiết có thể bị thiếu, và chúng ta muốn áp dụng các giá trị mặc định. **Transformation pipes** có thể thực hiện các chức năng này bằng cách chèn một function xử lý giữa yêu cầu client và request handler.

Đây là một `ParseIntPipe` đơn giản chịu trách nhiệm parsing một string thành một giá trị integer. (Như đã lưu ý ở trên, Nest có một built-in `ParseIntPipe` tinh vi hơn; chúng ta bao gồm cái này như một ví dụ đơn giản của một custom transformation pipe).

```typescript
@@filename(parse-int.pipe)
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
@@switch
import { Injectable, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe {
  transform(value, metadata) {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

Sau đó chúng ta có thể bind pipe này đến tham số được chọn như được hiển thị dưới đây:

```typescript
@@filename()
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id) {
  return this.catsService.findOne(id);
}
@@switch
@Get(':id')
@Bind(Param('id', new ParseIntPipe()))
async findOne(id) {
  return this.catsService.findOne(id);
}
```

Một trường hợp transformation hữu ích khác sẽ là chọn một thực thể user **tồn tại** từ database sử dụng một id được cung cấp trong yêu cầu:

```typescript
@@filename()
@Get(':id')
findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
  return userEntity;
}
@@switch
@Get(':id')
@Bind(Param('id', UserByIdPipe))
findOne(userEntity) {
  return userEntity;
}
```

Chúng ta để lại implementation của pipe này cho người đọc, nhưng lưu ý rằng giống như tất cả các transformation pipes khác, nó nhận một giá trị đầu vào (một `id`) và trả về một giá trị đầu ra (một đối tượng `UserEntity`). Điều này có thể làm cho code của bạn mang tính khai báo hơn và DRY bằng cách trừu tượng hóa code boilerplate khỏi handler của bạn và vào một pipe chung.

#### Providing defaults

Pipes `Parse*` mong đợi giá trị của tham số được định nghĩa. Chúng throw một exception khi nhận các giá trị `null` hoặc `undefined`. Để cho phép một endpoint xử lý các giá trị tham số querystring bị thiếu, chúng ta phải cung cấp một giá trị mặc định để được inject trước khi các pipes `Parse*` hoạt động trên các giá trị này. `DefaultValuePipe` phục vụ mục đích đó. Chỉ cần instantiate một `DefaultValuePipe` trong decorator `@Query()` trước khi pipe `Parse*` liên quan, như được hiển thị dưới đây:

```typescript
@@filename()
@Get()
async findAll(
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
) {
  return this.catsService.findAll({ activeOnly, page });
}
```