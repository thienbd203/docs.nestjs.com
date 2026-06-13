### Custom providers

Trong các chương trước, chúng tôi đã đề cập đến các khía cạnh khác nhau của **Dependency Injection (DI)** và cách nó được sử dụng trong Nest. Một ví dụ về điều này là [dependency injection dựa trên constructor](https://docs.nestjs.com/providers#dependency-injection) được sử dụng để inject các instances (thường là service providers) vào các class. Bạn sẽ không ngạc nhiên khi biết rằng Dependency Injection được tích hợp vào lõi Nest theo một cách cơ bản. Cho đến nay, chúng tôi chỉ đã khám phá một pattern chính. Khi ứng dụng của bạn phát triển phức tạp hơn, bạn có thể cần tận dụng các tính năng đầy đủ của hệ thống DI, vì vậy hãy khám phá chúng chi tiết hơn.

#### DI fundamentals

Dependency injection là một kỹ thuật [inversion of control (IoC)](https://en.wikipedia.org/wiki/Inversion_of_control) trong đó bạn ủy quyền việc khởi tạo các dependencies cho IoC container (trong trường hợp của chúng ta, hệ thống runtime NestJS), thay vì làm điều đó trong code của bạn một cách mệnh lệnh. Hãy xem xét điều gì đang xảy ra trong ví dụ này từ [chương Providers](https://docs.nestjs.com/providers).

Đầu tiên, chúng ta định nghĩa một provider. Decorator `@Injectable()` đánh dấu class `CatsService` như một provider.

```typescript
@@filename(cats.service)
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor() {
    this.cats = [];
  }

  findAll() {
    return this.cats;
  }
}
```

Sau đó chúng ta yêu cầu Nest inject provider vào class controller của chúng ta:

```typescript
@@filename(cats.controller)
import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
@@switch
import { Controller, Get, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

Cuối cùng, chúng ta đăng ký provider với Nest IoC container:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

Chính xác điều gì đang xảy ra dưới lớp vỏ để làm cho điều này hoạt động? Có ba bước chính trong quá trình:

1. Trong `cats.service.ts`, decorator `@Injectable()` khai báo class `CatsService` như một class có thể được quản lý bởi Nest IoC container.
2. Trong `cats.controller.ts`, `CatsController` khai báo một dependency trên token `CatsService` với constructor injection:

```typescript
  constructor(private catsService: CatsService)
```

3. Trong `app.module.ts`, chúng ta liên kết token `CatsService` với class `CatsService` từ file `cats.service.ts`. Chúng tôi sẽ <a href="/fundamentals/custom-providers#standard-providers">xem bên dưới</a> chính xác cách liên kết này (cũng được gọi là _registration_) xảy ra.

Khi Nest IoC container instantiate một `CatsController`, nó trước tiên tìm kiếm bất kỳ dependencies nào. Khi nó tìm thấy dependency `CatsService`, nó thực hiện một lookup trên token `CatsService`, trả về class `CatsService`, theo bước đăng ký (#3 ở trên). Giả sử scope `SINGLETON` (hành vi mặc định), Nest sau đó sẽ tạo một instance của `CatsService`, cache nó, và trả về nó, hoặc nếu một instance đã được cached, trả về instance hiện có.

*\*Giải thích này được đơn giản hóa một chút để làm rõ điểm. Một khu vực quan trọng mà chúng tôi đã bỏ qua là quá trình phân tích code cho các dependencies rất tinh vi, và xảy ra trong quá trình bootstrapping ứng dụng. Một tính năng chính là việc phân tích dependency (hoặc "tạo dependency graph"), là **transitive**. Trong ví dụ trên, nếu `CatsService` chính nó có các dependencies, những dependencies đó cũng sẽ được giải quyết. Dependency graph đảm bảo rằng các dependencies được giải quyết theo đúng thứ tự - về cơ bản là "bottom up". Cơ chế này giải phóng nhà phát triển khỏi việc phải quản lý các dependency graph phức tạp như vậy.

<app-banner-courses></app-banner-courses>

#### Standard providers

Hãy xem xét kỹ hơn decorator `@Module()`. Trong `app.module`, chúng ta khai báo:

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

Thuộc tính `providers` nhận một mảng các `providers`. Cho đến nay, chúng tôi đã cung cấp các providers đó thông qua một danh sách tên class. Thực tế, cú pháp `providers: [CatsService]` là viết tắt của cú pháp hoàn chỉnh hơn:

```typescript
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

Bây giờ chúng ta thấy cấu trúc rõ ràng này, chúng ta có thể hiểu quá trình đăng ký. Ở đây, chúng ta rõ ràng liên kết token `CatsService` với class `CatsService`. Ký hiệu viết tắt chỉ là sự tiện lợi để đơn giản hóa use case phổ biến nhất, trong đó token được sử dụng để yêu cầu một instance của một class cùng tên.

#### Custom providers

Điều gì xảy ra khi các yêu cầu của bạn vượt quá những gì được cung cấp bởi _Standard providers_? Dưới đây là một số ví dụ:

- Bạn muốn tạo một instance tùy chỉnh thay vì để Nest instantiate (hoặc trả về một instance được cached của) một class
- Bạn muốn tái sử dụng một class hiện có trong một dependency thứ hai
- Bạn muốn ghi đè một class với một phiên bản mock để test

Nest cho phép bạn định nghĩa Custom providers để xử lý các trường hợp này. Nó cung cấp một số cách để định nghĩa custom providers. Hãy đi qua từng cách.

> info **Hint** Nếu bạn đang gặp vấn đề với việc giải quyết dependency bạn có thể đặt biến môi trường `NEST_DEBUG` và nhận thêm logs giải quyết dependency trong quá trình khởi động.

#### Value providers: `useValue`

Cú pháp `useValue` hữu ích để inject một giá trị hằng số, đưa một thư viện bên ngoài vào Nest container, hoặc thay thế một implementation thực bằng một mock object. Hãy nói rằng bạn muốn buộc Nest sử dụng một `CatsService` mock cho mục đích test.

```typescript
import { CatsService } from './cats.service';

const mockCatsService = {
  /* mock implementation
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

Trong ví dụ này, token `CatsService` sẽ resolve thành object mock `mockCatsService`. `useValue` yêu cầu một giá trị - trong trường hợp này là một object literal có cùng interface với class `CatsService` mà nó thay thế. Do [structural typing](https://www.typescriptlang.org/docs/handbook/type-compatibility.html) của TypeScript, bạn có thể sử dụng bất kỳ object nào có interface tương thích, bao gồm một object literal hoặc một class instance được instantiate với `new`.

#### Non-class-based provider tokens

Cho đến nay, chúng tôi đã sử dụng tên class làm provider tokens của chúng ta (giá trị của thuộc tính `provide` trong một provider được liệt kê trong mảng `providers`). Điều này được phù hợp với pattern tiêu chuẩn được sử dụng với [dependency injection dựa trên constructor](https://docs.nestjs.com/providers#dependency-injection), trong đó token cũng là một tên class. (Tham chiếu lại <a href="/fundamentals/custom-providers#di-fundamentals">DI Fundamentals</a> để làm mới về tokens nếu khái niệm này không hoàn toàn rõ ràng). Đôi khi, chúng ta có thể muốn sự linh hoạt để sử dụng chuỗi hoặc symbols làm DI token. Ví dụ:

```typescript
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

Trong ví dụ này, chúng ta đang liên kết một token giá trị chuỗi (`'CONNECTION'`) với một object `connection` đã có từ trước mà chúng tôi đã import từ một file bên ngoài.

> warning **Notice** Ngoài việc sử dụng chuỗi làm giá trị token, bạn cũng có thể sử dụng [symbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) của JavaScript hoặc [enums](https://www.typescriptlang.org/docs/handbook/enums.html) của TypeScript.

Chúng tôi trước đây đã thấy cách inject một provider sử dụng pattern [dependency injection dựa trên constructor](https://docs.nestjs.com/providers#dependency-injection) tiêu chuẩn. Pattern này **yêu cầu** rằng dependency được khai báo với một tên class. Custom provider `'CONNECTION'` sử dụng một token giá trị chuỗi. Hãy xem cách inject một provider như vậy. Để làm điều này, chúng ta sử dụng decorator `@Inject()`. Decorator này nhận một đối số duy nhất - token.

```typescript
@@filename()
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
@@switch
@Injectable()
@Dependencies('CONNECTION')
export class CatsRepository {
  constructor(connection) {}
}
```

> info **Hint** Decorator `@Inject()` được import từ package `@nestjs/common`.

Mặc dù chúng tôi trực tiếp sử dụng chuỗi `'CONNECTION'` trong các ví dụ trên cho mục đích minh họa, để tổ chức code sạch, thực hành tốt nhất là định nghĩa các tokens trong một file riêng biệt, chẳng hạn như `constants.ts`. Coi chúng như bạn sẽ coi các symbols hoặc enums được định nghĩa trong file riêng của chúng và được import ở nơi cần thiết.

#### Class providers: `useClass`

Cú pháp `useClass` cho phép bạn xác định động một class mà một token nên resolve thành. Ví dụ, giả sử chúng ta có một class `ConfigService` trừu tượng (hoặc mặc định). Tùy thuộc vào môi trường hiện tại, chúng ta muốn Nest cung cấp một implementation khác của service cấu hình. Code sau đây thực hiện một chiến lược như vậy.

```typescript
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

Hãy xem một vài chi tiết trong mẫu code này. Bạn sẽ nhận thấy rằng chúng ta định nghĩa `configServiceProvider` với một object literal trước, sau đó truyền nó vào thuộc tính `providers` của decorator module. Đây chỉ là một chút tổ chức code, nhưng chức năng tương đương với các ví dụ chúng ta đã sử dụng cho đến nay trong chương này.

Ngoài ra, chúng tôi đã sử dụng tên class `ConfigService` làm token của chúng ta. Đối với bất kỳ class nào phụ thuộc vào `ConfigService`, Nest sẽ inject một instance của class được cung cấp (`DevelopmentConfigService` hoặc `ProductionConfigService`) ghi đè bất kỳ implementation mặc định nào có thể đã được khai báo ở nơi khác (ví dụ, một `ConfigService` được khai báo với decorator `@Injectable()`).

#### Factory providers: `useFactory`

Cú pháp `useFactory` cho phép tạo providers **động**. Provider thực tế sẽ được cung cấp bởi giá trị được trả về từ một hàm factory. Hàm factory có thể đơn giản hoặc phức tạp tùy theo nhu cầu. Một factory đơn giản có thể không phụ thuộc vào bất kỳ providers nào khác. Một factory phức tạp hơn có thể tự inject các providers khác mà nó cần để tính toán kết quả của nó. Đối với trường hợp sau, cú pháp factory provider có một cặp cơ chế liên quan:

1. Hàm factory có thể chấp nhận các đối số (tùy chọn).
2. Thuộc tính (tùy chọn) `inject` nhận một mảng các providers mà Nest sẽ giải quyết và truyền làm đối số cho hàm factory trong quá trình instantiation. Ngoài ra, các providers này có thể được đánh dấu là tùy chọn. Hai danh sách này nên tương quan: Nest sẽ truyền các instances từ danh sách `inject` làm đối số cho hàm factory theo cùng thứ tự. Ví dụ dưới đây minh họa điều này.

```typescript
@@filename()
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: MyOptionsProvider, optionalProvider?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [MyOptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \______________/             \__________________/
  //        This provider                The provider with this token
  //        is mandatory.                can resolve to `undefined`.
};

@Module({
  providers: [
    connectionProvider,
    MyOptionsProvider, // class-based provider
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
@@switch
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider, optionalProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [MyOptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \______________/            \__________________/
  //        This provider               The provider with this token
  //        is mandatory.               can resolve to `undefined`.
};

@Module({
  providers: [
    connectionProvider,
    MyOptionsProvider, // class-base provider
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
```

#### Alias providers: `useExisting`

Cú pháp `useExisting` cho phép bạn tạo các bí danh cho các providers hiện có. Điều này tạo ra hai cách để truy cập cùng một provider. Trong ví dụ dưới đây, token (dựa trên chuỗi) `'AliasedLoggerService'` là một bí danh cho token (dựa trên class) `LoggerService`. Giả sử chúng ta có hai dependencies khác nhau, một cho `'AliasedLoggerService'` và một cho `LoggerService`. Nếu cả hai dependencies được chỉ định với scope `SINGLETON`, cả hai sẽ resolve thành cùng một instance.

```typescript
@Injectable()
class LoggerService {
  /* implementation details */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

#### Non-service based providers

Mặc dù providers thường cung cấp các services, chúng không bị giới hạn ở việc sử dụng đó. Một provider có thể cung cấp **bất kỳ** giá trị nào. Ví dụ, một provider có thể cung cấp một mảng các object cấu hình dựa trên môi trường hiện tại, như được hiển thị bên dưới:

```typescript
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```

#### Export custom provider

Giống như bất kỳ provider nào, một custom provider được scoped đến module khai báo của nó. Để làm cho nó hiển thị với các module khác, nó phải được export. Để export một custom provider, chúng ta có thể sử dụng token của nó hoặc object provider đầy đủ.

Ví dụ sau đây hiển thị việc export sử dụng token:

```typescript
@@filename()
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
@@switch
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

Ngoài ra, export với object provider đầy đủ:

```typescript
@@filename()
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
@@switch
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```