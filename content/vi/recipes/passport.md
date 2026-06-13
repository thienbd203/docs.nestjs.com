### Passport (authentication)

[Passport](https://github.com/jaredhanson/passport) là thư viện authentication node.js phổ biến nhất, được cộng đồng biết đến rộng rãi và sử dụng thành công trong nhiều ứng dụng production. Việc tích hợp thư viện này với ứng dụng **Nest** rất đơn giản sử dụng module `@nestjs/passport`. Ở mức độ cao, Passport thực hiện một loạt các bước để:

- Xác thực người dùng bằng cách xác minh "credentials" của họ (như username/password, JSON Web Token ([JWT](https://jwt.io/)), hoặc token định danh từ một Identity Provider)
- Quản lý trạng thái đã xác thực (bằng cách phát hành một token di động, chẳng hạn như JWT, hoặc tạo một [Express session](https://github.com/expressjs/session))
- Gắn thông tin về người dùng đã xác thực vào đối tượng `Request` để sử dụng thêm trong các route handlers

Passport có hệ sinh thái [strategies](http://www.passportjs.org/) phong phú thực hiện các cơ chế xác thực khác nhau. Mặc dù đơn giản về khái niệm, tập hợp các Passport strategies bạn có thể chọn rất lớn và mang lại nhiều sự đa dạng. Passport trừu tượng hóa các bước đa dạng này thành một pattern chuẩn, và module `@nestjs/passport` bao bọc và chuẩn hóa pattern này thành các cấu trúc Nest quen thuộc.

Trong chương này, chúng ta sẽ triển khai một giải pháp xác thực end-to-end hoàn chỉnh cho một máy chủ RESTful API sử dụng các module mạnh mẽ và linh hoạt này. Bạn có thể sử dụng các khái niệm được mô tả ở đây để triển khai bất kỳ Passport strategy nào để tùy chỉnh sơ đồ xác thực của mình. Bạn có thể làm theo các bước trong chương này để xây dựng ví dụ hoàn chỉnh này.

#### Yêu cầu xác thực

Hãy làm rõ các yêu cầu của chúng ta. Đối với trường hợp sử dụng này, clients sẽ bắt đầu bằng cách xác thực với username và password. Sau khi được xác thực, server sẽ phát hành một JWT có thể được gửi như một [bearer token trong header authorization](https://tools.ietf.org/html/rfc6750) trên các yêu cầu tiếp theo để chứng minh xác thực. Chúng ta cũng sẽ tạo một route được bảo vệ chỉ có thể truy cập bởi các yêu cầu chứa JWT hợp lệ.

Chúng ta sẽ bắt đầu với yêu cầu đầu tiên: xác thực người dùng. Sau đó chúng ta sẽ mở rộng điều đó bằng cách phát hành JWT. Cuối cùng, chúng ta sẽ tạo một route được bảo vệ kiểm tra JWT hợp lệ trên yêu cầu.

Trước tiên chúng ta cần cài đặt các packages cần thiết. Passport cung cấp một strategy được gọi là [passport-local](https://github.com/jaredhanson/passport-local) thực hiện cơ chế xác thực username/password, phù hợp với nhu cầu của chúng ta cho phần trường hợp sử dụng này.

```bash
$ npm install --save @nestjs/passport passport passport-local
$ npm install --save-dev @types/passport-local
```

> warning **Lưu ý** Đối với **bất kỳ** Passport strategy nào bạn chọn, bạn luôn cần các packages `@nestjs/passport` và `passport`. Sau đó, bạn sẽ cần cài đặt package cụ thể cho strategy (ví dụ, `passport-jwt` hoặc `passport-local`) thực hiện chiến lược xác thực cụ thể mà bạn đang xây dựng. Ngoài ra, bạn cũng có thể cài đặt các định nghĩa type cho bất kỳ Passport strategy nào, như được hiển thị ở trên với `@types/passport-local`, cung cấp hỗ trợ trong khi viết code TypeScript.

#### Triển khai Passport strategies

Chúng ta bây giờ đã sẵn sàng để triển khai tính năng xác thực. Chúng ta sẽ bắt đầu với tổng quan về quy trình được sử dụng cho **bất kỳ** Passport strategy nào. Nó hữu ích khi nghĩ về Passport như một mini framework. Sự tinh tế của framework là nó trừu tượng hóa quá trình xác thực thành một vài bước cơ bản mà bạn tùy chỉnh dựa trên strategy bạn đang triển khai. Nó giống như một framework vì bạn cấu hình nó bằng cách cung cấp các tham số tùy chỉnh (như các đối tượng JSON đơn giản) và code tùy chỉnh dưới dạng các hàm callback, mà Passport gọi vào thời điểm thích hợp. Module `@nestjs/passport` bao bọc framework này trong một package kiểu Nest, giúp dễ dàng tích hợp vào ứng dụng Nest. Chúng ta sẽ sử dụng `@nestjs/passport` bên dưới, nhưng trước tiên hãy xem xét cách **Passport thuần** hoạt động.

Trong Passport thuần, bạn cấu hình một strategy bằng cách cung cấp hai thứ:

1. Một tập hợp các tùy chọn cụ thể cho strategy đó. Ví dụ, trong một JWT strategy, bạn có thể cung cấp một secret để ký tokens.
2. Một "verify callback", nơi bạn cho Passport biết cách tương tác với user store của bạn (nơi bạn quản lý tài khoản người dùng). Ở đây, bạn xác minh xem người dùng có tồn tại (và/hoặc tạo người dùng mới) hay không, và xem credentials của họ có hợp lệ hay không. Thư viện Passport mong đợi callback này trả về một user đầy đủ nếu xác thực thành công, hoặc null nếu thất bại (thất bại được định nghĩa là người dùng không được tìm thấy, hoặc trong trường hợp passport-local, password không khớp).

Với `@nestjs/passport`, bạn cấu hình một Passport strategy bằng cách mở rộng class `PassportStrategy`. Bạn truyền các tùy chọn strategy (mục 1 ở trên) bằng cách gọi phương thức `super()` trong subclass của bạn, tùy chọn truyền vào một đối tượng tùy chọn. Bạn cung cấp verify callback (mục 2 ở trên) bằng cách triển khai phương thức `validate()` trong subclass của bạn.

Chúng ta sẽ bắt đầu bằng cách tạo `AuthModule` và trong đó, một `AuthService`:

```bash
$ nest g module auth
$ nest g service auth
```

Khi chúng ta triển khai `AuthService`, chúng ta sẽ thấy hữu ích khi đóng gói các hoạt động người dùng trong một `UsersService`, vì vậy hãy tạo module và service đó ngay bây giờ:

```bash
$ nest g module users
$ nest g service users
```

Thay thế nội dung mặc định của các file được tạo này như được hiển thị bên dưới. Đối với ứng dụng mẫu của chúng ta, `UsersService` đơn giản là duy trì danh sách người dùng được mã hóa cứng trong bộ nhớ, và một phương thức find để truy xuất một người theo username. Trong một ứng dụng thực, đây là nơi bạn sẽ xây dựng mô hình người dùng và lớp persistence của mình, sử dụng thư viện lựa chọn của bạn (ví dụ, TypeORM, Sequelize, Mongoose, v.v.).

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

Trong `UsersModule`, thay đổi duy nhất cần thiết là thêm `UsersService` vào mảng exports của decorator `@Module` để nó hiển thị bên ngoài module này (chúng ta sẽ sớm sử dụng nó trong `AuthService` của mình).

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

`AuthService` của chúng ta có công việc truy xuất người dùng và xác minh password. Chúng ta tạo phương thức `validateUser()` cho mục đích này. Trong code bên dưới, chúng ta sử dụng toán tử spread ES6 tiện lợi để loại bỏ thuộc tính password khỏi đối tượng người dùng trước khi trả về nó. Chúng ta sẽ gọi vào phương thức `validateUser()` này từ Passport local strategy của mình trong một thời điểm.

```typescript
@@filename(auth/auth.service)
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
@Dependencies(UsersService)
export class AuthService {
  constructor(usersService) {
    this.usersService = usersService;
  }

  async validateUser(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }
}
```

> Warning **Cảnh báo** Tất nhiên trong một ứng dụng thực, bạn sẽ không lưu trữ password dưới dạng văn bản thuần. Thay vào đó, bạn sẽ sử dụng một thư viện như [bcrypt](https://github.com/kelektiv/node.bcrypt.js#readme), với thuật toán băm một chiều có salt. Với cách tiếp cận đó, bạn chỉ lưu trữ các password đã băm, sau đó so sánh password được lưu trữ với phiên bản đã băm của password **đến**, do đó không bao giờ lưu trữ hoặc tiết lộ passwords người dùng dưới dạng văn bản thuần. Để giữ ứng dụng mẫu của chúng ta đơn giản, chúng ta vi phạm mệnh lệnh tuyệt đối đó và sử dụng văn bản thuần. **Đừng làm điều này trong ứng dụng thực của bạn!**

Bây giờ, chúng ta cập nhật `AuthModule` để import `UsersModule`.

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
})
export class AuthModule {}
```

#### Triển khai Passport local

Bây giờ chúng ta có thể triển khai Passport **local authentication strategy** của mình. Tạo một file được gọi là `local.strategy.ts` trong thư mục `auth`, và thêm code sau:

```typescript
@@filename(auth/local.strategy)
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { AuthService } from './auth.service';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super();
  }

  async validate(username: string, password: string): Promise<any> {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
@@switch
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException, Dependencies } from '@nestjs/common';
import { AuthService } from './auth.service';

@Injectable()
@Dependencies(AuthService)
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(authService) {
    super();
    this.authService = authService;
  }

  async validate(username, password) {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

Chúng ta đã làm theo công thức được mô tả trước đó cho tất cả các Passport strategies. Trong trường hợp sử dụng của chúng ta với passport-local, không có tùy chọn cấu hình, vì vậy constructor của chúng ta chỉ gọi `super()`, không có đối tượng tùy chọn.

> info **Gợi ý** Chúng ta có thể truyền một đối tượng tùy chọn trong cuộc gọi đến `super()` để tùy chỉnh hành vi của passport strategy. Trong ví dụ này, passport-local strategy mặc định mong đợi các thuộc tính được gọi là `username` và `password` trong request body. Truyền một đối tượng tùy chọn để chỉ định các tên thuộc tính khác, ví dụ: `super({{ '{' }} usernameField: 'email' {{ '}' }})`. Xem [tài liệu Passport](http://www.passportjs.org/docs/configure/) để biết thêm thông tin.

Chúng ta cũng đã triển khai phương thức `validate()`. Đối với mỗi strategy, Passport sẽ gọi hàm verify (được triển khai với phương thức `validate()` trong `@nestjs/passport`) sử dụng một tập hợp tham số cụ thể cho strategy. Đối với local-strategy, Passport mong đợi một phương thức `validate()` với signature sau: `validate(username: string, password:string): any`.

Hầu hết công việc xác thực được thực hiện trong `AuthService` của chúng ta (với sự trợ giúp của `UsersService`), vì vậy phương thức này khá đơn giản. Phương thức `validate()` cho **bất kỳ** Passport strategy nào sẽ theo một pattern tương tự, chỉ thay đổi về chi tiết cách credentials được đại diện. Nếu người dùng được tìm thấy và credentials hợp lệ, người dùng được trả về để Passport có thể hoàn thành các nhiệm vụ của nó (ví dụ, tạo thuộc tính `user` trên đối tượng `Request`), và pipeline xử lý yêu cầu có thể tiếp tục. Nếu không được tìm thấy, chúng ta throw một exception và để [lớp exceptions](exception-filters) của chúng ta xử lý nó.

Thông thường, sự khác biệt đáng kể duy nhất trong phương thức `validate()` cho mỗi strategy là **cách** bạn xác định xem người dùng có tồn tại và hợp lệ hay không. Ví dụ, trong một JWT strategy, tùy thuộc vào yêu cầu, chúng ta có thể đánh giá xem `userId` được mang trong token đã giải mã có khớp với một bản ghi trong cơ sở dữ liệu người dùng của chúng ta hay không, hoặc khớp với danh sách các token đã bị thu hồi. Do đó, pattern của việc subclass và triển khai xác thực cụ thể cho strategy này nhất quán, tinh tế và có thể mở rộng.

Chúng ta cần cấu hình `AuthModule` của mình để sử dụng các tính năng Passport mà chúng ta vừa định nghĩa. Cập nhật `auth.module.ts` để trông như sau:

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { LocalStrategy } from './local.strategy';

@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { LocalStrategy } from './local.strategy';

@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
```

#### Built-in Passport Guards

Chương [Guards](guards) mô tả chức năng chính của Guards: để xác định xem một yêu cầu có được xử lý bởi route handler hay không. Điều đó vẫn đúng, và chúng ta sẽ sử dụng khả năng chuẩn đó sớm. Tuy nhiên, trong bối cảnh sử dụng module `@nestjs/passport`, chúng ta cũng sẽ giới thiệu một sự thay đổi nhỏ có thể gây nhầm lẫn lúc đầu, vì vậy hãy thảo luận về điều đó ngay bây giờ. Hãy xem xét rằng ứng dụng của bạn có thể tồn tại trong hai trạng thái, từ góc độ xác thực:

1. người dùng/client **không** đăng nhập (không được xác thực)
2. người dùng/client **đã** đăng nhập (được xác thực)

Trong trường hợp đầu tiên (người dùng chưa đăng nhập), chúng ta cần thực hiện hai chức năng riêng biệt:

- Hạn chế các routes mà người dùng chưa xác thực có thể truy cập (tức là từ chối truy cập vào các routes bị hạn chế). Chúng ta sẽ sử dụng Guards trong khả năng quen thuộc của chúng để xử lý chức năng này, bằng cách đặt một Guard trên các routes được bảo vệ. Như bạn có thể dự đoán, chúng ta sẽ kiểm tra sự hiện diện của JWT hợp lệ trong Guard này, vì vậy chúng ta sẽ làm việc trên Guard này sau, khi chúng ta đã phát hành JWT thành công.

- Khởi tạo **bước xác thực** chính nó khi người dùng chưa xác thực trước đó cố gắng đăng nhập. Đây là bước nơi chúng ta sẽ **phát hành** JWT cho người dùng hợp lệ. Suy nghĩ về điều này một lúc, chúng ta biết chúng ta sẽ cần `POST` credentials username/password để khởi tạo xác thực, vì vậy chúng ta sẽ thiết lập một route `POST /auth/login` để xử lý điều đó. Điều này đặt ra câu hỏi: làm thế nào chính xác chúng ta gọi passport-local strategy trong route đó?

Câu trả lời rất đơn giản: bằng cách sử dụng một loại Guard khác, hơi khác một chút. Module `@nestjs/passport` cung cấp cho chúng ta một Guard tích hợp sẵn làm điều đó cho chúng ta. Guard này gọi Passport strategy và khởi động các bước được mô tả ở trên (truy xuất credentials, chạy hàm verify, tạo thuộc tính `user`, v.v).

Trường hợp thứ hai được liệt kê ở trên (người dùng đã đăng nhập) đơn giản dựa vào loại Guard tiêu chuẩn mà chúng ta đã thảo luận để cho phép truy cập vào các routes được bảo vệ cho người dùng đã đăng nhập.

<app-banner-courses-auth></app-banner-courses-auth>

#### Login route

Với strategy đã có chỗ, chúng ta bây giờ có thể triển khai một route `/auth/login` tối giản, và áp dụng Guard tích hợp sẵn để khởi tạo luồng passport-local.

Mở file `app.controller.ts` và thay thế nội dung của nó bằng sau:

```typescript
@@filename(app.controller)
import { Controller, Request, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller()
export class AppController {
  @UseGuards(AuthGuard('local'))
  @Post('auth/login')
  async login(@Request() req) {
    return req.user;
  }
}
@@switch
import { Controller, Bind, Request, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller()
export class AppController {
  @UseGuards(AuthGuard('local'))
  @Post('auth/login')
  @Bind(Request())
  async login(req) {
    return req.user;
  }
}
```

Với `@UseGuards(AuthGuard('local'))`, chúng ta đang sử dụng một `AuthGuard` mà `@nestjs/passport` **tự động cung cấp** cho chúng ta khi chúng ta mở rộng passport-local strategy. Hãy phân tích điều đó. Passport local strategy của chúng ta có tên mặc định là `'local'`. Chúng ta tham chiếu tên đó trong decorator `@UseGuards()` để liên kết nó với code được cung cấp bởi package `passport-local`. Điều này được sử dụng để phân biệt strategy nào để gọi trong trường hợp chúng ta có nhiều Passport strategies trong ứng dụng của mình (mỗi cái có thể cung cấp một `AuthGuard` cụ thể cho strategy). Mặc dù chúng ta chỉ có một strategy như vậy cho đến nay, chúng ta sẽ sớm thêm một strategy thứ hai, vì vậy điều này cần thiết để phân biệt.

Để test route của chúng ta, chúng ta sẽ để route `/auth/login` đơn giản trả về người dùng hiện tại. Điều này cũng cho phép chúng ta chứng minh một tính năng Passport khác: Passport tự động tạo một đối tượng `user`, dựa trên giá trị chúng ta trả về từ phương thức `validate()`, và gán nó cho đối tượng `Request` như `req.user`. Sau này, chúng ta sẽ thay thế điều này bằng code để tạo và trả về JWT thay thế.

Vì đây là các routes API, chúng ta sẽ test chúng sử dụng thư viện [cURL](https://curl.haxx.se/) phổ biến. Bạn có thể test với bất kỳ đối tượng `user` nào được mã hóa cứng trong `UsersService`.

```bash
$ # POST to /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"userId":1,"username":"john"}
```

Mặc dù điều này hoạt động, truyền tên strategy trực tiếp đến `AuthGuard()` giới thiệu các chuỗi magic trong codebase. Thay vào đó, chúng tôi khuyến nghị tạo class riêng của bạn, như được hiển thị bên dưới:

```typescript
@@filename(auth/local-auth.guard)
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}
```

Bây giờ, chúng ta có thể cập nhật route handler `/auth/login` và sử dụng `LocalAuthGuard` thay thế:

```typescript
@UseGuards(LocalAuthGuard)
@Post('auth/login')
async login(@Request() req) {
  return req.user;
}
```

#### Logout route

Để đăng xuất, chúng ta có thể tạo một route bổ sung gọi `req.logout()` để xóa session của người dùng. Đây là cách tiếp cận điển hình được sử dụng trong xác thực dựa trên session, nhưng nó không áp dụng cho JWTs.

```typescript
@UseGuards(LocalAuthGuard)
@Post('auth/logout')
async logout(@Request() req) {
  return req.logout();
}
```

#### JWT functionality

Chúng ta đã sẵn sàng để chuyển sang phần JWT của hệ thống xác thực của chúng ta. Hãy xem xét và tinh chỉnh các yêu cầu của chúng ta:

- Cho phép người dùng xác thực với username/password, trả về JWT để sử dụng trong các cuộc gọi tiếp theo đến các endpoints API được bảo vệ. Chúng ta đang trên đường tốt để đáp ứng yêu cầu này. Để hoàn thành nó, chúng ta sẽ cần viết code phát hành JWT.
- Tạo các routes API được bảo vệ dựa trên sự hiện diện của JWT hợp lệ như một bearer token

Chúng ta sẽ cần cài đặt thêm một vài packages để hỗ trợ các yêu cầu JWT của chúng ta:

```bash
$ npm install --save @nestjs/jwt passport-jwt
$ npm install --save-dev @types/passport-jwt
```

Package `@nestjs/jwt` (xem thêm [tại đây](https://github.com/nestjs/jwt)) là một package tiện ích giúp với việc thao tác JWT. Package `passport-jwt` là package Passport thực hiện JWT strategy và `@types/passport-jwt` cung cấp các định nghĩa type TypeScript.

Hãy xem kỹ hơn về cách một yêu cầu `POST /auth/login` được xử lý. Chúng ta đã trang trí route sử dụng Guard tích hợp sẵn được cung cấp bởi passport-local strategy. Điều này có nghĩa là:

1. Route handler **sẽ chỉ được gọi nếu người dùng đã được xác thực**
2. Tham số `req` sẽ chứa thuộc tính `user` (được điền bởi Passport trong luồng xác thực passport-local)

Với điều này trong tâm trí, chúng ta bây giờ có thể cuối cùng tạo ra một JWT thực sự, và trả về nó trong route này. Để giữ cho các service của chúng ta được modular hóa sạch sẽ, chúng ta sẽ xử lý việc tạo JWT trong `authService`. Mở file `auth.service.ts` trong thư mục `auth`, và thêm phương thức `login()`, và import `JwtService` như được hiển thị:

```typescript
@@filename(auth/auth.service)
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user: any) {
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}

@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Dependencies(UsersService, JwtService)
@Injectable()
export class AuthService {
  constructor(usersService, jwtService) {
    this.usersService = usersService;
    this.jwtService = jwtService;
  }

  async validateUser(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user) {
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

Chúng ta đang sử dụng thư viện `@nestjs/jwt`, cung cấp một hàm `sign()` để tạo JWT của chúng ta từ một tập hợp con các thuộc tính đối tượng `user`, mà chúng ta sau đó trả về như một đối tượng đơn giản với một thuộc tính `access_token`. Lưu ý: chúng ta chọn tên thuộc tính là `sub` để giữ giá trị `userId` của chúng ta nhất quán với tiêu chuẩn JWT. Đừng quên inject provider JwtService vào `AuthService`.

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

Chúng ta sẽ sử dụng điều này để chia sẻ khóa của chúng ta giữa các bước ký và xác minh JWT.

> Warning **Cảnh báo** **Đừng công khai khóa này**. Chúng ta đã làm như vậy ở đây để làm rõ code đang làm gì, nhưng trong hệ thống production **bạn phải bảo vệ khóa này** sử dụng các biện pháp phù hợp như secrets vault, biến môi trường, hoặc service cấu hình.

Bây giờ, mở `auth.module.ts` trong thư mục `auth` và cập nhật nó để trông như sau:

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy],
  exports: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

Chúng ta cấu hình `JwtModule` sử dụng `register()`, truyền vào một đối tượng cấu hình. Xem [tại đây](https://github.com/nestjs/jwt/blob/master/README.md) để biết thêm về Nest `JwtModule` và [tại đây](https://github.com/auth0/node-jsonwebtoken#usage) để biết thêm chi tiết về các tùy chọn cấu hình có sẵn.

Bây giờ chúng ta có thể cập nhật route `/auth/login` để trả về JWT.

```typescript
@@filename(app.controller)
import { Controller, Request, Post, UseGuards } from '@nestjs/common';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }
}
@@switch
import { Controller, Bind, Request, Post, UseGuards } from '@nestjs/common';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  @Bind(Request())
  async login(req) {
    return this.authService.login(req.user);
  }
}
```

Hãy tiếp tục và test các routes của chúng ta sử dụng cURL một lần nữa. Bạn có thể test với bất kỳ đối tượng `user` nào được mã hóa cứng trong `UsersService`.

```bash
$ # POST to /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
$ # Note: above JWT truncated
```

#### Triển khai Passport JWT

Chúng ta bây giờ có thể giải quyết yêu cầu cuối cùng của mình: bảo vệ các endpoints bằng cách yêu cầu JWT hợp lệ hiện diện trên yêu cầu. Passport có thể giúp chúng ta ở đây. Nó cung cấp [passport-jwt](https://github.com/mikenicholson/passport-jwt) strategy để bảo vệ các endpoints RESTful với JSON Web Tokens. Bắt đầu bằng cách tạo một file được gọi là `jwt.strategy.ts` trong thư mục `auth`, và thêm code sau:

```typescript
@@filename(auth/jwt.strategy)
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { jwtConstants } from './constants';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.secret,
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, username: payload.username };
  }
}
@@switch
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { jwtConstants } from './constants';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.secret,
    });
  }

  async validate(payload) {
    return { userId: payload.sub, username: payload.username };
  }
}
```

Với `JwtStrategy` của chúng ta, chúng ta đã làm theo công thức tương tự được mô tả trước đó cho tất cả các Passport strategies. Strategy này yêu cầu một số khởi tạo, vì vậy chúng ta làm điều đó bằng cách truyền vào một đối tượng tùy chọn trong cuộc gọi `super()`. Bạn có thể đọc thêm về các tùy chọn có sẵn [tại đây](https://github.com/mikenicholson/passport-jwt#configure-strategy). Trong trường hợp của chúng ta, các tùy chọn này là:

- `jwtFromRequest`: cung cấp phương thức mà JWT sẽ được trích xuất từ `Request`. Chúng ta sẽ sử dụng cách tiếp cận tiêu chuẩn của việc cung cấp một bearer token trong header Authorization của các yêu cầu API của chúng ta. Các tùy chọn khác được mô tả [tại đây](https://github.com/mikenicholson/passport-jwt#extracting-the-jwt-from-the-request).
- `ignoreExpiration`: chỉ để rõ ràng, chúng ta chọn cài đặt mặc định `false`, ủy nhiệm trách nhiệm đảm bảo rằng JWT chưa hết hạn cho module Passport. Điều này có nghĩa là nếu route của chúng ta được cung cấp với JWT đã hết hạn, yêu cầu sẽ bị từ chối và phản hồi `401 Unauthorized` được gửi. Passport xử lý điều này tự động cho chúng ta một cách thuận tiện.
- `secretOrKey`: chúng ta đang sử dụng tùy chọn thuận tiện của việc cung cấp một secret đối xứng để ký token. Các tùy chọn khác, chẳng hạn như khóa công khai được mã hóa PEM, có thể phù hợp hơn cho các ứng dụng production (xem [tại đây](https://github.com/mikenicholson/passport-jwt#configure-strategy) để biết thêm thông tin). Trong mọi trường hợp, như đã cảnh báo trước đó, **đừng công khai secret này**.

Phương thức `validate()` xứng đáng được thảo luận một chút. Đối với jwt-strategy, Passport trước tiên xác minh chữ ký JWT và giải mã JSON. Sau đó nó gọi phương thức `validate()` của chúng ta truyền JSON đã giải mã như tham số duy nhất của nó. Dựa trên cách ký JWT hoạt động, **chúng ta được đảm bảo rằng chúng ta đang nhận một token hợp lệ** mà chúng ta đã ký trước đó và phát hành cho người dùng hợp lệ.

Kết quả của tất cả điều này, phản hồi của chúng ta cho callback `validate()` là tầm thường: chúng ta đơn giản trả về một đối tượng chứa các thuộc tính `userId` và `username`. Nhớ lại một lần nữa rằng Passport sẽ xây dựng một đối tượng `user` dựa trên giá trị trả về của phương thức `validate()` của chúng ta, và gắn nó như một thuộc tính trên đối tượng `Request`.

Ngoài ra, bạn có thể trả về một mảng, trong đó giá trị đầu tiên được sử dụng để tạo một đối tượng `user` và giá trị thứ hai được sử dụng để tạo một đối tượng `authInfo`.

Cũng đáng chỉ ra rằng cách tiếp cận này để lại cho chúng ta chỗ ('hooks' như vậy) để đưa logic kinh doanh khác vào quá trình. Ví dụ, chúng ta có thể thực hiện một tra cứu database trong phương thức `validate()` của mình để trích xuất thêm thông tin về người dùng, dẫn đến một đối tượng `user` phong phú hơn có sẵn trong Request của chúng ta. Đây cũng là nơi chúng ta có thể quyết định thực hiện xác thực token thêm, chẳng hạn như tra cứu `userId` trong danh sách các token đã bị thu hồi, cho phép chúng ta thực hiện thu hồi token. Mô hình chúng ta đã triển khai ở đây trong code mẫu của chúng ta là mô hình "JWT không trạng thái" nhanh, trong đó mỗi cuộc gọi API được ủy quyền ngay lập tức dựa trên sự hiện diện của JWT hợp lệ, và một chút thông tin về người yêu cầu (`userId` và `username` của nó) có sẵn trong pipeline Request của chúng ta.

Thêm `JwtStrategy` mới như một provider trong `AuthModule`:

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { JwtStrategy } from './jwt.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { JwtStrategy } from './jwt.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

Bằng cách import cùng secret được sử dụng khi chúng ta ký JWT, chúng ta đảm bảo rằng giai đoạn **xác minh** được thực hiện bởi Passport, và giai đoạn **ký** được thực hiện trong AuthService của chúng ta, sử dụng một secret chung.

Cuối cùng, chúng ta định nghĩa class `JwtAuthGuard` mở rộng `AuthGuard` tích hợp sẵn:

```typescript
@@filename(auth/jwt-auth.guard)
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

#### Triển khai route được bảo vệ và JWT strategy guards

Chúng ta bây giờ có thể triển khai route được bảo vệ của mình và Guard liên quan của nó.

Mở file `app.controller.ts` và cập nhật nó như được hiển thị bên dưới:

```typescript
@@filename(app.controller)
import { Controller, Get, Request, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from './auth/jwt-auth.guard';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }

  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
@@switch
import { Controller, Dependencies, Bind, Get, Request, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from './auth/jwt-auth.guard';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Dependencies(AuthService)
@Controller()
export class AppController {
  constructor(authService) {
    this.authService = authService;
  }

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  @Bind(Request())
  async login(req) {
    return this.authService.login(req.user);
  }

  @UseGuards(JwtAuthGuard)
  @Get('profile')
  @Bind(Request())
  getProfile(req) {
    return req.user;
  }
}
```

Một lần nữa, chúng ta đang áp dụng `AuthGuard` mà module `@nestjs/passport` đã tự động cung cấp cho chúng ta khi chúng ta cấu hình module passport-jwt. Guard này được tham chiếu theo tên mặc định của nó, `jwt`. Khi route `GET /profile` của chúng ta được truy cập, Guard sẽ tự động gọi passport-jwt strategy tùy chỉnh được cấu hình của chúng ta, xác thực JWT, và gán thuộc tính `user` cho đối tượng `Request`.

Đảm bảo ứng dụng đang chạy, và test các routes sử dụng `cURL`.

```bash
$ # GET /profile
$ curl http://localhost:3000/profile
$ # result -> {"statusCode":401,"message":"Unauthorized"}

$ # POST /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm... }

$ # GET /profile using access_token returned from previous step as bearer code
$ curl http://localhost:3000/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
$ # result -> {"userId":1,"username":"john"}
```

Lưu ý rằng trong `AuthModule`, chúng ta cấu hình JWT để có thời gian hết hạn là `60 giây`. Điều này có thể quá ngắn, và xử lý các chi tiết về hết hạn và làm mới token nằm ngoài phạm vi của bài viết này. Tuy nhiên, chúng ta chọn điều đó để chứng minh một phẩm chất quan trọng của JWTs và strategy passport-jwt. Nếu bạn đợi 60 giây sau khi xác thực trước khi cố gắng yêu cầu `GET /profile`, bạn sẽ nhận được phản hồi `401 Unauthorized`. Điều này là do Passport tự động kiểm tra JWT về thời gian hết hạn của nó, giúp bạn tránh phải làm như vậy trong ứng dụng của mình.

Chúng ta bây giờ đã hoàn thành việc triển khai xác thực JWT của mình. Các clients JavaScript (như Angular/React/Vue), và các ứng dụng JavaScript khác, bây giờ có thể xác thực và giao tiếp an toàn với API Server của chúng ta.

#### Mở rộng guards

Trong hầu hết các trường hợp, sử dụng class `AuthGuard` được cung cấp là đủ. Tuy nhiên, có thể có các trường hợp sử dụng khi bạn muốn đơn giản mở rộng xử lý lỗi mặc định hoặc logic xác thực. Để làm điều này, bạn có thể mở rộng class tích hợp sẵn và ghi đè các phương thức trong một subclass.

```typescript
import {
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    // Thêm logic xác thực tùy chỉnh của bạn ở đây
    // ví dụ, gọi super.logIn(request) để thiết lập một session.
    return super.canActivate(context);
  }

  handleRequest(err, user, info) {
    // Bạn có thể throw một exception dựa trên tham số "info" hoặc "err"
    if (err || !user) {
      throw err || new UnauthorizedException();
    }
    return user;
  }
}
```

Ngoài việc mở rộng xử lý lỗi mặc định và logic xác thực, chúng ta có thể cho phép xác thực đi qua một chuỗi các strategies. Strategy đầu tiên thành công, chuyển hướng, hoặc lỗi sẽ dừng chuỗi. Các thất bại xác thực sẽ tiếp tục qua từng strategy theo tuần tự, cuối cùng thất bại nếu tất cả các strategies thất bại.

```typescript
export class JwtAuthGuard extends AuthGuard(['strategy_jwt_1', 'strategy_jwt_2', '...']) { ... }
```

#### Kích hoạt xác thực toàn cục

Nếu phần lớn các endpoints của bạn nên được bảo vệ theo mặc định, bạn có thể đăng ký authentication guard như một [global guard](/guards#binding-guards) và thay vì sử dụng decorator `@UseGuards()` trên đầu mỗi controller, bạn có thể đơn giản đánh dấu các routes nào nên là public.

Trước tiên, đăng ký `JwtAuthGuard` như một global guard sử dụng cấu trúc sau (trong bất kỳ module nào):

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

Với điều này có chỗ, Nest sẽ tự động gắn `JwtAuthGuard` vào tất cả các endpoints.

Bây giờ chúng ta phải cung cấp một cơ chế để khai báo routes như public. Để làm điều này, chúng ta có thể tạo một decorator tùy chỉnh sử dụng hàm factory decorator `SetMetadata`.

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

Trong file trên, chúng ta đã xuất hai hằng số. Một là metadata key của chúng ta được gọi là `IS_PUBLIC_KEY`, và cái khác là decorator mới của chính chúng ta mà chúng ta sẽ gọi là `Public` (bạn có thể đặt tên thay thế là `SkipAuth` hoặc `AllowAnon`, bất cứ thứ gì phù hợp với dự án của bạn).

Bây giờ rằng chúng ta có decorator tùy chỉnh `@Public()`, chúng ta có thể sử dụng nó để trang trí bất kỳ phương thức nào, như sau:

```typescript
@Public()
@Get()
findAll() {
  return [];
}
```

Cuối cùng, chúng ta cần `JwtAuthGuard` để trả về `true` khi metadata `"isPublic"` được tìm thấy. Để làm điều này, chúng ta sẽ sử dụng class `Reflector` (đọc thêm [tại đây](/guards#putting-it-all-together)).

```typescript
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      return true;
    }
    return super.canActivate(context);
  }
}
```

#### Request-scoped strategies

API passport dựa trên việc đăng ký các strategies vào instance toàn cục của thư viện. Do đó các strategies không được thiết kế để có các tùy chọn phụ thuộc vào yêu cầu hoặc được khởi tạo động cho mỗi yêu cầu (đọc thêm về các providers [request-scoped](/fundamentals/injection-scopes)). Khi bạn cấu hình strategy của mình để là request-scoped, Nest sẽ không bao giờ khởi tạo nó vì nó không được gắn với bất kỳ route cụ thể nào. Không có cách vật lý để xác định các strategies "request-scoped" nào nên được thực thi cho mỗi yêu cầu.

Tuy nhiên, có các cách để giải quyết động các providers request-scoped trong strategy. Để làm điều này, chúng ta tận dụng tính năng [module reference](/fundamentals/module-ref).

Trước tiên, mở file `local.strategy.ts` và inject `ModuleRef` theo cách bình thường:

```typescript
constructor(private moduleRef: ModuleRef) {
  super({
    passReqToCallback: true,
  });
}
```

> info **Gợi ý** Class `ModuleRef` được import từ package `@nestjs/core`.

Hãy đảm bảo đặt thuộc tính cấu hình `passReqToCallback` thành `true`, như được hiển thị ở trên.

Trong bước tiếp theo, instance yêu cầu sẽ được sử dụng để obtain định danh ngữ cảnh hiện tại, thay vì tạo một cái mới (đọc thêm về ngữ cảnh yêu cầu [tại đây](/fundamentals/module-ref#getting-current-sub-tree)).

Bây giờ, trong phương thức `validate()` của class `LocalStrategy`, sử dụng phương thức `getByRequest()` của class `ContextIdFactory` để tạo một context id dựa trên đối tượng yêu cầu, và truyền điều này vào cuộc gọi `resolve()`:

```typescript
async validate(
  request: Request,
  username: string,
  password: string,
) {
  const contextId = ContextIdFactory.getByRequest(request);
  // "AuthService" là một provider request-scoped
  const authService = await this.moduleRef.resolve(AuthService, contextId);
  ...
}
```

Trong ví dụ trên, phương thức `resolve()` sẽ trả về bất đồng bộ instance request-scoped của provider `AuthService` (chúng ta giả định rằng `AuthService` được đánh dấu như một provider request-scoped).

#### Tùy chỉnh Passport

Bất kỳ tùy chọn tùy chỉnh Passport tiêu chuẩn nào có thể được truyền theo cùng cách, sử dụng phương thức `register()`. Các tùy chọn có sẵn phụ thuộc vào strategy đang được triển khai. Ví dụ:

```typescript
PassportModule.register({ session: true });
```

Bạn cũng có thể truyền strategies một đối tượng tùy chọn trong constructors của chúng để cấu hình chúng.
Đối với local strategy bạn có thể truyền ví dụ:

```typescript
constructor(private authService: AuthService) {
  super({
    usernameField: 'email',
    passwordField: 'password',
  });
}
```

Hãy xem [Trang web Passport chính thức](http://www.passportjs.org/docs/oauth/) để biết tên thuộc tính.

#### Named strategies

Khi triển khai một strategy, bạn có thể cung cấp một tên cho nó bằng cách truyền một đối số thứ hai cho hàm `PassportStrategy`. Nếu bạn không làm điều này, mỗi strategy sẽ có một tên mặc định (ví dụ, 'jwt' cho jwt-strategy):

```typescript
export class JwtStrategy extends PassportStrategy(Strategy, 'myjwt')
```

Sau đó, bạn tham chiếu điều này thông qua một decorator như `@UseGuards(AuthGuard('myjwt'))`.

#### GraphQL

Để sử dụng AuthGuard với [GraphQL](https://docs.nestjs.com/graphql/quick-start), hãy mở rộng class `AuthGuard` tích hợp sẵn và ghi đè phương thức `getRequest()`.

```typescript
@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  getRequest(context: ExecutionContext) {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req;
  }
}
```

Để lấy người dùng đã xác thực hiện tại trong graphql resolver của bạn, bạn có thể định nghĩa một decorator `@CurrentUser()`:

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

export const CurrentUser = createParamDecorator(
  (data: unknown, context: ExecutionContext) => {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req.user;
  },
);
```

Để sử dụng decorator ở trên trong resolver của bạn, hãy đảm bảo bao gồm nó như một tham số của query hoặc mutation của bạn:

```typescript
@Query(() => User)
@UseGuards(GqlAuthGuard)
whoAmI(@CurrentUser() user: User) {
  return this.usersService.findById(user.id);
}
```

Đối với passport-local strategy, bạn cũng sẽ cần thêm các đối số của ngữ cảnh GraphQL vào request body để Passport có thể truy cập chúng để xác thực. Nếu không, bạn sẽ nhận được một lỗi Unauthorized.

```typescript
@Injectable()
export class GqlLocalAuthGuard extends AuthGuard('local') {
  getRequest(context: ExecutionContext) {
    const gqlExecutionContext = GqlExecutionContext.create(context);
    const gqlContext = gqlExecutionContext.getContext();
    const gqlArgs = gqlExecutionContext.getArgs();

    gqlContext.req.body = { ...gqlContext.req.body, ...gqlArgs };
    return gqlContext.req;
  }
}
```