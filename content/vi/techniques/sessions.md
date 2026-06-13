### Session

**HTTP sessions** cung cấp một cách để lưu trữ thông tin về người dùng trên nhiều yêu cầu, điều này đặc biệt hữu ích cho các ứng dụng [MVC](/techniques/mvc).

#### Sử dụng với Express (mặc định)

Đầu tiên cài đặt [package cần thiết](https://github.com/expressjs/session) (và các loại của nó cho người dùng TypeScript):

```shell
$ npm i express-session
$ npm i -D @types/express-session
```

Sau khi cài đặt hoàn tất, áp dụng middleware `express-session` làm middleware toàn cục (ví dụ, trong file `main.ts` của bạn).

```typescript
import * as session from 'express-session';
// somewhere in your initialization file
app.use(
  session({
    secret: 'my-secret',
    resave: false,
    saveUninitialized: false,
  }),
);
```

> warning **Lưu ý** Bộ lưu trữ session phía server mặc định được thiết kế có chủ đích không dành cho môi trường production. Nó sẽ rò rỉ bộ nhớ trong hầu hết các điều kiện, không mở rộng quy mô qua một quy trình đơn, và có nghĩa là để gỡ lỗi và phát triển. Đọc thêm trong [repository chính thức](https://github.com/expressjs/session).

`secret` được sử dụng để ký cookie ID session. Điều này có thể là một chuỗi cho một secret đơn, hoặc một mảng nhiều secret. Nếu một mảng secret được cung cấp, chỉ phần tử đầu tiên sẽ được sử dụng để ký cookie ID session, trong khi tất cả các phần tử sẽ được xem xét khi xác minh chữ ký trong các yêu cầu. Secret bản thân không nên dễ dàng được phân tích cú pháp bởi con người và tốt nhất là một tập hợp các ký tự ngẫu nhiên.

Bật tùy chọn `resave` buộc session phải được lưu lại vào session store, ngay cả khi session không bao giờ được sửa đổi trong yêu cầu. Giá trị mặc định là `true`, nhưng sử dụng mặc định đã bị phản đối, vì mặc định sẽ thay đổi trong tương lai.

Tương tự, bật tùy chọn `saveUninitialized` Buộc một session "chưa được khởi tạo" phải được lưu vào store. Một session chưa được khởi tạo khi nó mới nhưng không được sửa đổi. Chọn `false` hữu ích để implement các session đăng nhập, giảm việc sử dụng lưu trữ server, hoặc tuân thủ các luật yêu cầu quyền phép trước khi đặt cookie. Chọn `false` cũng sẽ giúp với các điều kiện race nơi một client thực hiện nhiều yêu cầu song song mà không có session ([nguồn](https://github.com/expressjs/session#saveuninitialized)).

Bạn có thể truyền một số tùy chọn khác cho middleware `session`, đọc thêm về chúng trong [tài liệu API](https://github.com/expressjs/session#options).

> info **Gợi ý** Vui lòng lưu ý rằng `secure: true` là một tùy chọn được khuyến nghị. Tuy nhiên, nó yêu cầu một website có https, tức là HTTPS là cần thiết cho các cookie an toàn. Nếu secure được đặt và bạn truy cập site của mình qua HTTP, cookie sẽ không được đặt. Nếu bạn có node.js của bạn phía sau một proxy và sử dụng `secure: true`, bạn cần đặt `"trust proxy"` trong express.

Với điều này, bạn bây giờ có thể đặt và đọc các giá trị session từ bên trong các route handler, như sau:

```typescript
@Get()
findAll(@Req() request: Request) {
  request.session.visits = request.session.visits ? request.session.visits + 1 : 1;
}
```

> info **Gợi ý** Decorator `@Req()` được nhập từ `@nestjs/common`, trong khi `Request` từ package `express`.

Ngoài ra, bạn có thể sử dụng decorator `@Session()` để trích xuất một đối tượng session từ yêu cầu, như sau:

```typescript
@Get()
findAll(@Session() session: Record<string, any>) {
  session.visits = session.visits ? session.visits + 1 : 1;
}
```

> info **Gợi ý** Decorator `@Session()` được nhập từ package `@nestjs/common`.

#### Sử dụng với Fastify

Đầu tiên cài đặt package cần thiết:

```shell
$ npm i @fastify/secure-session
```

Sau khi cài đặt hoàn tất, đăng ký plugin `fastify-secure-session`:

```typescript
import secureSession from '@fastify/secure-session';

// somewhere in your initialization file
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
await app.register(secureSession, {
  secret: 'averylogphrasebiggerthanthirtytwochars',
  salt: 'mq9hDxBVDbspDR6n',
});
```

> info **Gợi ý** Bạn cũng có thể tạo trước một key ([xem hướng dẫn](https://github.com/fastify/fastify-secure-session)) hoặc sử dụng [keys rotation](https://github.com/fastify/fastify-secure-session#using-keys-with-key-rotation).

Đọc thêm về các tùy chọn có sẵn trong [repository chính thức](https://github.com/fastify/fastify-secure-session).

Với điều này, bạn bây giờ có thể đặt và đọc các giá trị session từ bên trong các route handler, như sau:

```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  const visits = request.session.get('visits');
  request.session.set('visits', visits ? visits + 1 : 1);
}
```

Ngoài ra, bạn có thể sử dụng decorator `@Session()` để trích xuất một đối tượng session từ yêu cầu, như sau:

```typescript
@Get()
findAll(@Session() session: secureSession.Session) {
  const visits = session.get('visits');
  session.set('visits', visits ? visits + 1 : 1);
}
```

> info **Gợi ý** Decorator `@Session()` được nhập từ `@nestjs/common`, trong khi `secureSession.Session` từ package `@fastify/secure-session` (câu lệnh import: `import * as secureSession from '@fastify/secure-session'`).