### Authentication

Authentication là một phần **cần thiết** của hầu hết các ứng dụng. Có nhiều cách tiếp cận và chiến lược khác nhau để xử lý authentication. Cách tiếp cận được chọn cho bất kỳ dự án nào phụ thuộc vào các yêu cầu ứng dụng cụ thể của dự án đó. Chương này trình bày một số cách tiếp cận để authentication có thể được điều chỉnh theo nhiều yêu cầu khác nhau.

Hãy làm rõ các yêu cầu của chúng ta. Đối với use case này, clients sẽ bắt đầu bằng cách authenticate với username và password. Sau khi authenticated, server sẽ phát hành một JWT có thể được gửi như một [bearer token](https://tools.ietf.org/html/rfc6750) trong một authorization header trên các yêu cầu tiếp theo để chứng minh authentication. Chúng ta cũng sẽ tạo một protected route chỉ có thể truy cập được bởi các yêu cầu chứa JWT hợp lệ.

Chúng ta sẽ bắt đầu với yêu cầu đầu tiên: authenticate một user. Sau đó chúng ta sẽ mở rộng điều đó bằng cách phát hành một JWT. Cuối cùng, chúng ta sẽ tạo một protected route kiểm tra JWT hợp lệ trên yêu cầu.

#### Creating an authentication module

Chúng ta sẽ bắt đầu bằng cách tạo `AuthModule` và trong đó, một `AuthService` và một `AuthController`. Chúng ta sẽ sử dụng `AuthService` để triển khai logic authentication, và `AuthController` để expose các endpoint authentication.

```bash
$ nest g module auth
$ nest g controller auth
$ nest g service auth
```

Khi triển khai `AuthService`, chúng ta sẽ thấy hữu ích khi đóng gói các thao tác user trong một `UsersService`, vì vậy hãy tạo module và service đó ngay bây giờ:

```bash
$ nest g module users
$ nest g service users
```

Thay thế nội dung mặc định của các file được tạo này như dưới đây. Đối với ứng dụng mẫu của chúng ta, `UsersService` chỉ duy trì một danh sách user trong bộ nhớ được hard-coded, và một phương thức find để lấy một user theo username. Trong một ứng dụng thực, đây là nơi bạn sẽ xây dựng user model và persistence layer của mình, sử dụng thư viện lựa chọn của bạn (ví dụ: TypeORM, Sequelize, Mongoose, v.v.).

```typescript
@@filename(users/users.service)
import { Injectable } from '@nestjs/common';

// This should be a real class/interface representing a user entity
export type User = any;

@Injectable()
export class UsersService {
  private readonly users = [
    {
      userId: 1,
      username: 'john',
      password: 'changeme',
    },
    {
      userId: 2,
      username: 'maria',
      password: 'guess',
    },
  ];

  async findOne(username: string): Promise<User | undefined> {
    return this.users.find(user => user.username === username);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  constructor() {
    this.users = [
      {
        userId: 1,
        username: 'john',
        password: 'changeme',
      },
      {
        userId: 2,
        username: 'maria',
        password: 'guess',
      },
    ];
  }

  async findOne(username) {
    return this.users.find(user => user.username === username);
  }
}
```

Trong `UsersModule`, thay đổi duy nhất cần thiết là thêm `UsersService` vào mảng exports của decorator `@Module` để nó có thể nhìn thấy bên ngoài module này (chúng ta sẽ sớm sử dụng nó trong `AuthService` của mình).

```typescript
@@filename(users/users.module)
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
@@switch
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

#### Implementing the "Sign in" endpoint

`AuthService` của chúng ta có nhiệm vụ lấy một user và xác minh password. Chúng ta tạo một phương thức `signIn()` cho mục đích này. Trong code dưới đây, chúng ta sử dụng ES6 spread operator tiện dụng để loại bỏ thuộc tính password khỏi user object trước khi trả về. Đây là một thực tế phổ biến khi trả về user objects, vì bạn không muốn expose các trường nhạy cảm như passwords hoặc các security keys khác.

```typescript
@@filename(auth/auth.service)
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private readonly usersService: UsersService) {}

  async signIn(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const { password, ...result } = user;
    // TODO: Generate a JWT and return it here
    // instead of the user object
    return result;
  }
}
@@switch
import { Injectable, Dependencies, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
@Dependencies(UsersService)
export class AuthService {
  constructor(usersService) {
    this.usersService = usersService;
  }

  async signIn(username: string, pass: string) {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const { password, ...result } = user;
    // TODO: Generate a JWT and return it here
    // instead of the user object
    return result;
  }
}
```

> warning **Cảnh báo** Tất nhiên trong một ứng dụng thực, bạn sẽ không lưu trữ password trong plain text. Thay vào đó, bạn sẽ sử dụng một thư viện như [bcrypt](https://github.com/kelektiv/node.bcrypt.js#readme), với thuật toán hash một chiều có salt. Với cách tiếp cận đó, bạn chỉ lưu trữ hashed passwords, và sau đó so sánh password được lưu trữ với phiên bản hash của password **đến**, do đó không bao giờ lưu trữ hoặc expose user passwords trong plain text. Để giữ cho ứng dụng mẫu của chúng ta đơn giản, chúng ta vi phạm quy tắc tuyệt đối đó và sử dụng plain text. **Đừng làm điều này trong ứng dụng thực của bạn!**

Bây giờ, chúng ta cập nhật `AuthModule` để import `UsersModule`.

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}
```

Với điều này đã có, hãy mở `AuthController` và thêm một phương thức `signIn()` vào nó. Phương thức này sẽ được gọi bởi client để authenticate một user. Nó sẽ nhận username và password trong request body, và sẽ trả về một JWT token nếu user được authenticate.

```typescript
@@filename(auth/auth.controller)
import { Body, Controller, Post, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }
}
```

> info **Gợi ý** Lý tưởng nhất, thay vì sử dụng type `Record<string, any>`, chúng ta nên sử dụng một DTO class để định nghĩa hình dạng của request body. Xem chương [validation](/techniques/validation) để biết thêm thông tin.

<app-banner-courses-auth></app-banner-courses-auth>

#### JWT token

Chúng ta đã sẵn sàng chuyển sang phần JWT của hệ thống auth của mình. Hãy xem xét và tinh chỉnh các yêu cầu của chúng ta:

- Cho phép users authenticate với username/password, trả về một JWT để sử dụng trong các cuộc gọi tiếp theo đến các protected API endpoints. Chúng ta đang trên đường tốt để đáp ứng yêu cầu này. Để hoàn thành nó, chúng ta sẽ cần viết code phát hành JWT.
- Tạo các API routes được bảo vệ dựa trên sự hiện diện của JWT hợp lệ như một bearer token

Chúng ta sẽ cần cài đặt thêm một package để hỗ trợ các yêu cầu JWT của mình:

```bash
$ npm install --save @nestjs/jwt
```

> info **Gợi ý** Package `@nestjs/jwt` (xem thêm [ở đây](https://github.com/nestjs/jwt)) là một package tiện ích giúp với việc thao tác JWT. Điều này bao gồm tạo và xác minh JWT tokens.

Để giữ cho các service của mình được modular hóa sạch sẽ, chúng ta sẽ xử lý việc tạo JWT trong `authService`. Mở file `auth.service.ts` trong thư mục `auth`, inject `JwtService`, và cập nhật phương thức `signIn` để tạo một JWT token như dưới đây:

```typescript
@@filename(auth/auth.service)
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async signIn(
    username: string,
    pass: string,
  ): Promise<{ access_token: string }> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { sub: user.userId, username: user.username };
    return {
      // 💡 Here the JWT secret key that's used for signing the payload
      // is the key that was passed in the JwtModule
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
@@switch
import { Injectable, Dependencies, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Dependencies(UsersService, JwtService)
@Injectable()
export class AuthService {
  constructor(usersService, jwtService) {
    this.usersService = usersService;
    this.jwtService = jwtService;
  }

  async signIn(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { username: user.username, sub: user.userId };
    return {
      // 💡 Here the JWT secret key that's used for signing the payload
      // is the key that was passed in the JwtModule
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
```

Chúng ta đang sử dụng thư viện `@nestjs/jwt`, cung cấp một hàm `signAsync()` để tạo JWT của chúng ta từ một tập hợp con của các thuộc tính của đối tượng `user`, sau đó chúng ta trả về như một đối tượng đơn giản với một thuộc tính `access_token`. Lưu ý: chúng ta chọn tên thuộc tính là `sub` để giữ giá trị `userId` của chúng ta để phù hợp với tiêu chuẩn JWT.

Bây giờ chúng ta cần cập nhật `AuthModule` để import các dependencies mới và cấu hình `JwtModule`.

Trước tiên, tạo `constants.ts` trong thư mục `auth`, và thêm code sau:

```typescript
@@filename(auth/constants)
export const jwtConstants = {
  secret: 'DO NOT USE THIS VALUE. INSTEAD, CREATE A COMPLEX SECRET AND KEEP IT SAFE OUTSIDE OF THE SOURCE CODE.',
};
@@switch
export const jwtConstants = {
  secret: 'DO NOT USE THIS VALUE. INSTEAD, CREATE A COMPLEX SECRET AND KEEP IT SAFE OUTSIDE OF THE SOURCE CODE.',
};
```

Chúng ta sẽ sử dụng điều này để chia sẻ key giữa các bước ký và xác minh JWT.

> warning **Cảnh báo** **Đừng expose key này công khai**. Chúng ta đã làm như vậy ở đây để làm rõ code đang làm gì, nhưng trong hệ thống production **bạn phải bảo vệ key này** bằng các biện pháp thích hợp như secrets vault, environment variable, hoặc configuration service.

Bây giờ, mở `auth.module.ts` trong thư mục `auth` và cập nhật nó để trông như sau:

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { JwtModule } from '@nestjs/jwt';
import { AuthController } from './auth.controller';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { JwtModule } from '@nestjs/jwt';
import { AuthController } from './auth.controller';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

> info **Gợi ý** Chúng ta đang đăng ký `JwtModule` như global để làm cho mọi thứ dễ dàng hơn cho chúng ta. Điều này có nghĩa là chúng ta không cần import `JwtModule` ở bất kỳ đâu khác trong ứng dụng của mình.

Chúng ta cấu hình `JwtModule` bằng cách sử dụng `register()`, truyền vào một đối tượng cấu hình. Xem [ở đây](https://github.com/nestjs/jwt/blob/master/README.md) để biết thêm về Nest `JwtModule` và [ở đây](https://github.com/auth0/node-jsonwebtoken#usage) để biết thêm chi tiết về các tùy chọn cấu hình có sẵn.

Hãy tiếp tục và test các route của chúng ta bằng cách sử dụng cURL một lần nữa. Bạn có thể test với bất kỳ đối tượng `user` nào được hard-coded trong `UsersService`.

```bash
$ # POST to /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
$ # Note: above JWT truncated
```

#### Implementing the authentication guard

Chúng ta giờ có thể giải quyết yêu cầu cuối cùng của mình: bảo vệ các endpoints bằng cách yêu cầu một JWT hợp lệ có mặt trên yêu cầu. Chúng ta sẽ làm điều này bằng cách tạo một `AuthGuard` mà chúng ta có thể sử dụng để bảo vệ các route của mình.

```typescript
@@filename(auth/auth.guard)
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Request } from 'express';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      // 💡 Here the JWT secret key that's used for verifying the payload
      // is the key that was passed in the JwtModule
      const payload = await this.jwtService.verifyAsync(token);
      // 💡 We're assigning the payload to the request object here
      // so that we can access it in our route handlers
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

Chúng ta giờ có thể triển khai protected route của mình và đăng ký `AuthGuard` để bảo vệ nó.

Mở file `auth.controller.ts` và cập nhật nó như dưới đây:

```typescript
@@filename(auth.controller)
import {
  Body,
  Controller,
  Get,
  HttpCode,
  HttpStatus,
  Post,
  Request,
  UseGuards
} from '@nestjs/common';
import { AuthGuard } from './auth.guard';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }

  @UseGuards(AuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

Chúng ta đang áp dụng `AuthGuard` mà chúng ta vừa tạo cho route `GET /profile` để nó sẽ được bảo vệ.

Đảm bảo ứng dụng đang chạy, và test các route bằng cách sử dụng `cURL`.

```bash
$ # GET /profile
$ curl http://localhost:3000/auth/profile
{"statusCode":401,"message":"Unauthorized"}

$ # POST /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."}

$ # GET /profile using access_token returned from previous step as bearer code
$ curl http://localhost:3000/auth/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
{"sub":1,"username":"john","iat":...,"exp":...}
```

Lưu ý rằng trong `AuthModule`, chúng ta đã cấu hình JWT để có thời hạn là `60 giây`. Đây là thời hạn quá ngắn, và việc xử lý các chi tiết về hết hạn token và làm mới nằm ngoài phạm vi của bài viết này. Tuy nhiên, chúng ta đã chọn điều đó để chứng minh một chất lượng quan trọng của JWT. Nếu bạn đợi 60 giây sau khi authenticate trước khi cố gắng thực hiện yêu cầu `GET /auth/profile`, bạn sẽ nhận được phản hồi `401 Unauthorized`. Điều này là do `@nestjs/jwt` tự động kiểm tra JWT về thời hạn của nó, giúp bạn khỏi rắc rối phải làm như vậy trong ứng dụng của mình.

Chúng ta đã hoàn thành việc triển khai authentication JWT của mình. Các JavaScript clients (như Angular/React/Vue), và các ứng dụng JavaScript khác, giờ có thể authenticate và giao tiếp an toàn với API Server của chúng ta.

#### Enable authentication globally

Nếu phần lớn các endpoints của bạn nên được bảo vệ theo mặc định, bạn có thể đăng ký authentication guard như một [global guard](/guards#binding-guards) và thay vì sử dụng decorator `@UseGuards()` trên đầu mỗi controller, bạn có thể đơn giản đánh dấu các route nào nên là public.

Trước tiên, đăng ký `AuthGuard` như một global guard bằng cách sử dụng cấu trúc sau (trong bất kỳ module nào, ví dụ, trong `AuthModule`):

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: AuthGuard,
  },
],
```

Với điều này đã có, Nest sẽ tự động bind `AuthGuard` đến tất cả các endpoints.

Bây giờ chúng ta phải cung cấp một cơ chế để khai báo các route là public. Đối với điều này, chúng ta có thể tạo một custom decorator bằng cách sử dụng hàm factory decorator `SetMetadata`.

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

Trong file trên, chúng ta đã xuất hai hằng số. Một là key metadata của chúng ta có tên `IS_PUBLIC_KEY`, và cái còn lại là decorator mới của chúng ta mà chúng ta sẽ gọi là `Public` (bạn có thể thay thế tên nó bằng `SkipAuth` hoặc `AllowAnon`, bất cứ điều gì phù hợp với dự án của bạn).

Bây giờ chúng ta có một custom decorator `@Public()`, chúng ta có thể sử dụng nó để decorate bất kỳ phương thức nào, như sau:

```typescript
@Public()
@Get()
findAll() {
  return [];
}
```

Cuối cùng, chúng ta cần `AuthGuard` để trả về `true` khi metadata `"isPublic"` được tìm thấy. Đối với điều này, chúng ta sẽ sử dụng class `Reflector` (đọc thêm [ở đây](/guards#putting-it-all-together)).

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService, private reflector: Reflector) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      // 💡 See this condition
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      // 💡 Here the JWT secret key that's used for verifying the payload
      // is the key that was passed in the JwtModule
      const payload = await this.jwtService.verifyAsync(token);
      // 💡 We're assigning the payload to the request object here
      // so that we can access it in our route handlers
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

#### Passport integration

[Passport](https://github.com/jaredhanson/passport) là thư viện authentication node.js phổ biến nhất, được biết đến bởi cộng đồng và được sử dụng thành công trong nhiều ứng dụng production. Rất đơn giản để tích hợp thư viện này với ứng dụng **Nest** bằng cách sử dụng module `@nestjs/passport`.

Để tìm hiểu cách bạn có thể tích hợp Passport với NestJS, hãy kiểm tra chương [này](/recipes/passport).

#### Example

Bạn có thể tìm thấy phiên bản hoàn chỉnh của code trong chương này [ở đây](https://github.com/nestjs/nest/tree/master/sample/19-auth-jwt).