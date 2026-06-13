### Decorators

Tất cả các decorators OpenAPI có sẵn đều có tiền tố `Api` để phân biệt chúng với các decorators core. Dưới đây là danh sách đầy đủ các decorators được export cùng với chỉ định cấp độ mà decorator có thể được áp dụng.

||                           |                     |
|| ------------------------- | ------------------- |
|| `@ApiBasicAuth()`         | Method / Controller |
|| `@ApiBearerAuth()`        | Method / Controller |
|| `@ApiBody()`              | Method              |
|| `@ApiConsumes()`          | Method / Controller |
|| `@ApiCookieAuth()`        | Method / Controller |
|| `@ApiExcludeController()` | Controller          |
|| `@ApiExcludeEndpoint()`   | Method              |
|| `@ApiExtension()`         | Method              |
|| `@ApiExtraModels()`       | Method / Controller |
|| `@ApiHeader()`            | Method / Controller |
|| `@ApiHideProperty()`      | Model               |
|| `@ApiOAuth2()`            | Method / Controller |
|| `@ApiOperation()`         | Method              |
|| `@ApiParam()`             | Method / Controller |
|| `@ApiProduces()`          | Method / Controller |
|| `@ApiSchema()`            | Model               |
|| `@ApiProperty()`          | Model               |
|| `@ApiPropertyOptional()`  | Model               |
|| `@ApiQuery()`             | Method / Controller |
|| `@ApiResponse()`          | Method / Controller |
|| `@ApiSecurity()`          | Method / Controller |
|| `@ApiTags()`              | Method / Controller |
|| `@ApiCallbacks()`         | Method / Controller |