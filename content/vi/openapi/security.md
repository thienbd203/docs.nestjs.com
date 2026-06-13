### Security

Để định nghĩa cơ chế bảo mật nào nên được sử dụng cho một operation cụ thể, sử dụng decorator `@ApiSecurity()`.

```typescript
@ApiSecurity('basic')
@Controller('cats')
export class CatsController {}
```

Trước khi bạn chạy ứng dụng của mình, hãy nhớ thêm định nghĩa bảo mật vào tài liệu cơ sở của bạn sử dụng `DocumentBuilder`:

```typescript
const options = new DocumentBuilder().addSecurity('basic', {
  type: 'http',
  scheme: 'basic',
});
```

Một số kỹ thuật xác thực phổ biến nhất được tích hợp sẵn (ví dụ: `basic` và `bearer`) và do đó bạn không phải định nghĩa cơ chế bảo mật thủ công như được hiển thị ở trên.

#### Basic authentication

Để kích hoạt basic authentication, sử dụng `@ApiBasicAuth()`.

```typescript
@ApiBasicAuth()
@Controller('cats')
export class CatsController {}
```

Trước khi bạn chạy ứng dụng của mình, hãy nhớ thêm định nghĩa bảo mật vào tài liệu cơ sở của bạn sử dụng `DocumentBuilder`:

```typescript
const options = new DocumentBuilder().addBasicAuth();
```

#### Bearer authentication

Để kích hoạt bearer authentication, sử dụng `@ApiBearerAuth()`.

```typescript
@ApiBearerAuth()
@Controller('cats')
export class CatsController {}
```

Trước khi bạn chạy ứng dụng của mình, hãy nhớ thêm định nghĩa bảo mật vào tài liệu cơ sở của bạn sử dụng `DocumentBuilder`:

```typescript
const options = new DocumentBuilder().addBearerAuth();
```

#### OAuth2 authentication

Để kích hoạt OAuth2, sử dụng `@ApiOAuth2()`.

```typescript
@ApiOAuth2(['pets:write'])
@Controller('cats')
export class CatsController {}
```

Trước khi bạn chạy ứng dụng của mình, hãy nhớ thêm định nghĩa bảo mật vào tài liệu cơ sở của bạn sử dụng `DocumentBuilder`:

```typescript
const options = new DocumentBuilder().addOAuth2();
```

#### Cookie authentication

Để kích hoạt cookie authentication, sử dụng `@ApiCookieAuth()`.

```typescript
@ApiCookieAuth()
@Controller('cats')
export class CatsController {}
```

Trước khi bạn chạy ứng dụng của mình, hãy nhớ thêm định nghĩa bảo mật vào tài liệu cơ sở của bạn sử dụng `DocumentBuilder`:

```typescript
const options = new DocumentBuilder().addCookieAuth('optional-session-id');
```