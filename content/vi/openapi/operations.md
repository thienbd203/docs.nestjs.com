### Operations

Theo thuật ngữ OpenAPI, paths là các endpoints (tài nguyên), chẳng hạn như `/users` hoặc `/reports/summary`, mà API của bạn expose, và operations là các phương thức HTTP được sử dụng để thao tác với các paths này, chẳng hạn như `GET`, `POST` hoặc `DELETE`.

#### Tags

Để gắn một controller vào một tag cụ thể, sử dụng decorator `@ApiTags(...tags)`.

```typescript
@ApiTags('cats')
@Controller('cats')
export class CatsController {}
```


OpenAPI 3.2 mở rộng Tag Object để tags có thể được tổ chức thành một phân cấp và được annotate với một gợi ý về cách chúng nên được trình bày. Để khai báo các mối quan hệ này, định nghĩa tags trước với `DocumentBuilder` và truyền các tùy chọn `parent` và `kind` đến `addTag()`:

```typescript
const config = new DocumentBuilder()
  .setOpenAPIVersion('3.2.0')
  .addTag('Animals', 'Everything about animals', undefined, { kind: 'nav' })
  .addTag('Cats', 'Cat operations', undefined, { parent: 'Animals' })
  .addTag('Dogs', 'Dog operations', undefined, { parent: 'Animals' })
  .build();
```

Tùy chọn `parent` tham chiếu đến một tag khác theo tên, và `kind` là một chuỗi dạng tự do, có thể đọc được bằng máy tính gợi ý cách tag nên được sử dụng — thường là `nav`, `badge`, hoặc `audience`.

> warning **Cảnh báo** Các trường `parent` và `kind` thuộc về OpenAPI 3.2 Tag Object. Bạn phải gọi `setOpenAPIVersion('3.2.0')`, nếu không tài liệu được tạo vẫn khai báo `openapi: 3.0.0` và các validators nghiêm ngặt sẽ từ chối các trường này. Các trường phân cấp chỉ có thể được định nghĩa thông qua `DocumentBuilder.addTag()`; đặt chúng trên decorator `@ApiTags()` không có hiệu lực.

#### Headers

Để định nghĩa các headers tùy chỉnh được mong đợi như một phần của request, sử dụng `@ApiHeader()`.

```typescript
@ApiHeader({
  name: 'X-MyHeader',
  description: 'Custom header',
})
@Controller('cats')
export class CatsController {}
```

#### Responses

Để định nghĩa một HTTP response tùy chỉnh, sử dụng decorator `@ApiResponse()`.

```typescript
@Post()
@ApiResponse({ status: 201, description: 'The record has been successfully created.'})
@ApiResponse({ status: 403, description: 'Forbidden.'})
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Nest cung cấp một tập hợp các decorators **API response** ngắn gọn kế thừa từ decorator `@ApiResponse`:

- `@ApiOkResponse()`
- `@ApiCreatedResponse()`
- `@ApiAcceptedResponse()`
- `@ApiNoContentResponse()`
- `@ApiMovedPermanentlyResponse()`
- `@ApiFoundResponse()`
- `@ApiBadRequestResponse()`
- `@ApiUnauthorizedResponse()`
- `@ApiNotFoundResponse()`
- `@ApiForbiddenResponse()`
- `@ApiMethodNotAllowedResponse()`
- `@ApiNotAcceptableResponse()`
- `@ApiRequestTimeoutResponse()`
- `@ApiConflictResponse()`
- `@ApiPreconditionFailedResponse()`
- `@ApiTooManyRequestsResponse()`
- `@ApiGoneResponse()`
- `@ApiPayloadTooLargeResponse()`
- `@ApiUnsupportedMediaTypeResponse()`
- `@ApiUnprocessableEntityResponse()`
- `@ApiInternalServerErrorResponse()`
- `@ApiNotImplementedResponse()`
- `@ApiBadGatewayResponse()`
- `@ApiServiceUnavailableResponse()`
- `@ApiGatewayTimeoutResponse()`
- `@ApiDefaultResponse()`

```typescript
@Post()
@ApiCreatedResponse({ description: 'The record has been successfully created.'})
@ApiForbiddenResponse({ description: 'Forbidden.'})
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Để chỉ định một model trả về cho một request, chúng ta phải tạo một class và annotate tất cả các thuộc tính với decorator `@ApiProperty()`.

```typescript
export class Cat {
  @ApiProperty()
  id: number;

  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

Sau đó model `Cat` có thể được sử dụng kết hợp với thuộc tính `type` của decorator response.

```typescript
@ApiTags('cats')
@Controller('cats')
export class CatsController {
  @Post()
  @ApiCreatedResponse({
    description: 'The record has been successfully created.',
    type: Cat,
  })
  async create(@Body() createCatDto: CreateCatDto): Promise<Cat> {
    return this.catsService.create(createCatDto);
  }
}
```

Hãy mở trình duyệt và xác minh model `Cat` được tạo:

<figure><img src="/assets/swagger-response-type.png" /></figure>

Thay vì định nghĩa responses cho mỗi endpoint hoặc controller riêng lẻ, bạn có thể định nghĩa một response toàn cầu cho tất cả các endpoints sử dụng class `DocumentBuilder`. Cách tiếp cận này hữu ích khi bạn muốn định nghĩa một response toàn cầu cho tất cả các endpoints trong ứng dụng của bạn (ví dụ: cho các lỗi như `401 Unauthorized` hoặc `500 Internal Server Error`).

```typescript
const config = new DocumentBuilder()
  .addGlobalResponse({
    status: 500,
    description: 'Internal server error',
  })
  // các cấu hình khác
  .build();
```

#### File upload

Bạn có thể kích hoạt file upload cho một phương thức cụ thể với decorator `@ApiBody` cùng với `@ApiConsumes()`. Đây là một ví dụ đầy đủ sử dụng kỹ thuật [File Upload](/techniques/file-upload):

```typescript
@UseInterceptors(FileInterceptor('file'))
@ApiConsumes('multipart/form-data')
@ApiBody({
  description: 'List of cats',
  type: FileUploadDto,
})
uploadFile(@UploadedFile() file: Express.Multer.File) {}
```

Trong đó `FileUploadDto` được định nghĩa như sau:

```typescript
class FileUploadDto {
  @ApiProperty({ type: 'string', format: 'binary' })
  file: any;
}
```

Để xử lý việc tải lên nhiều files, bạn có thể định nghĩa `FilesUploadDto` như sau:

```typescript
class FilesUploadDto {
  @ApiProperty({ type: 'array', items: { type: 'string', format: 'binary' } })
  files: any[];
}
```

#### Extensions

Để thêm một Extension vào một request sử dụng decorator `@ApiExtension()`. Tên extension phải có tiền tố `x-`.

```typescript
@ApiExtension('x-foo', { hello: 'world' })
```

#### Nâng cao: Generic `ApiResponse`

Với khả năng cung cấp [Raw Definitions](/openapi/types-and-parameters#raw-definitions), chúng ta có thể định nghĩa Generic schema cho Swagger UI. Giả sử chúng ta có DTO sau:

```ts
export class PaginatedDto<TData> {
  @ApiProperty()
  total: number;

  @ApiProperty()
  limit: number;

  @ApiProperty()
  offset: number;

  results: TData[];
}
```

Chúng ta bỏ qua việc decorate `results` vì chúng ta sẽ cung cấp một raw definition cho nó sau này. Bây giờ, hãy định nghĩa một DTO khác và đặt tên, ví dụ, `CatDto`, như sau:

```ts
export class CatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

Với điều này, chúng ta có thể định nghĩa một response `PaginatedDto<CatDto>`, như sau:

```ts
@ApiOkResponse({
  schema: {
    allOf: [
      { $ref: getSchemaPath(PaginatedDto) },
      {
        properties: {
          results: {
            type: 'array',
            items: { $ref: getSchemaPath(CatDto) },
          },
        },
      },
    ],
  },
})
async findAll(): Promise<PaginatedDto<CatDto>> {}
```

Trong ví dụ này, chúng ta chỉ định rằng response sẽ có allOf `PaginatedDto` và thuộc tính `results` sẽ là kiểu `Array<CatDto>`.

- Hàm `getSchemaPath()` trả về đường dẫn OpenAPI Schema từ trong File Spec OpenAPI cho một model đã cho.
- `allOf` là một khái niệm mà OAS 3 cung cấp để bao gồm các use-case liên quan đến Inheritance khác nhau.

Cuối cùng, vì `PaginatedDto` không được tham chiếu trực tiếp bởi bất kỳ controller nào, `SwaggerModule` sẽ không thể tạo một định nghĩa model tương ứng ngay bây giờ. Trong trường hợp này, chúng ta phải thêm nó như một [Extra Model](/openapi/types-and-parameters#extra-models). Ví dụ, chúng ta có thể sử dụng decorator `@ApiExtraModels()` ở cấp độ controller, như sau:

```ts
@Controller('cats')
@ApiExtraModels(PaginatedDto)
export class CatsController {}
```

Nếu bạn chạy Swagger bây giờ, `swagger.json` được tạo cho endpoint cụ thể này nên có response sau được định nghĩa:

```json
"responses": {
  "200": {
    "description": "",
    "content": {
      "application/json": {
        "schema": {
          "allOf": [
            {
              "$ref": "#/components/schemas/PaginatedDto"
            },
            {
              "properties": {
                "results": {
                  "$ref": "#/components/schemas/CatDto"
                }
              }
            }
          ]
        }
      }
    }
  }
}
```

Để làm cho nó có thể tái sử dụng, chúng ta có thể tạo một decorator tùy chỉnh cho `PaginatedDto`, như sau:

```ts
export const ApiPaginatedResponse = <TModel extends Type<any>>(
  model: TModel,
) => {
  return applyDecorators(
    ApiExtraModels(PaginatedDto, model),
    ApiOkResponse({
      schema: {
        allOf: [
          { $ref: getSchemaPath(PaginatedDto) },
          {
            properties: {
              results: {
                type: 'array',
                items: { $ref: getSchemaPath(model) },
              },
            },
          },
        ],
      },
    }),
  );
};
```

> info **Gợi ý** Interface `Type<any>` và hàm `applyDecorators` được import từ package `@nestjs/common`.

Để đảm bảo rằng `SwaggerModule` sẽ tạo một định nghĩa cho model của chúng ta, chúng ta phải thêm nó như một extra model, giống như chúng ta đã làm trước đó với `PaginatedDto` trong controller.

Với điều này, chúng ta có thể sử dụng decorator tùy chỉnh `@ApiPaginatedResponse()` trên endpoint của chúng ta:

```ts
@ApiPaginatedResponse(CatDto)
async findAll(): Promise<PaginatedDto<CatDto>> {}
```

Đối với các công cụ tạo client, cách tiếp cận này tạo ra sự mơ hồ trong cách `PaginatedResponse<TModel>` được tạo cho client. Đoạn mã sau là một ví dụ về kết quả của trình tạo client cho endpoint `GET /` ở trên.

```typescript
// Angular
findAll(): Observable<{ total: number, limit: number, offset: number, results: CatDto[] }>
```

Như bạn có thể thấy, **Return Type** ở đây mơ hồ. Để giải quyết vấn đề này, bạn có thể thêm một thuộc tính `title` vào `schema` cho `ApiPaginatedResponse`:

```typescript
export const ApiPaginatedResponse = <TModel extends Type<any>>(
  model: TModel,
) => {
  return applyDecorators(
    ApiOkResponse({
      schema: {
        title: `PaginatedResponseOf${model.name}`,
        allOf: [
          // ...
        ],
      },
    }),
  );
};
```

Bây giờ kết quả của công cụ tạo client sẽ trở thành:

```ts
// Angular
findAll(): Observable<PaginatedResponseOfCatDto>
```