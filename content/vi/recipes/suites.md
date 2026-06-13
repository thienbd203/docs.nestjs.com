### Suites

[Suites](https://suites.dev) là một framework kiểm tra đơn vị [mã nguồn mở](https://github.com/suites-dev/suites) cho các framework dependency injection TypeScript. Nó được sử dụng như một **sự thay thế** để tạo mocks thủ công, thiết lập kiểm tra dài dòng với nhiều cấu hình mock, hoặc làm việc với các test doubles không có kiểu (như mocks và stubs).

Suites đọc metadata từ các dịch vụ nestjs tại runtime và tự động tạo mocks đầy đủ kiểu cho tất cả các phụ thuộc.
Điều này loại bỏ thiết lập mock boilerplate và đảm bảo các bài kiểm tra type-safe. Trong khi Suites có thể được sử dụng cùng với `Test.createTestingModule()`, nó xuất sắc trong kiểm tra đơn vị tập trung.
Sử dụng `Test.createTestingModule()` khi xác nhận cấu hình module, decorators, guards, và interceptors.
Sử dụng Suites cho các bài kiểm tra đơn vị nhanh với tạo mock tự động.

Để biết thêm thông tin về kiểm tra dựa trên module, xem chương [cơ bản kiểm tra](/fundamentals/testing).

> info **Lưu ý** `Suites` là một gói bên thứ ba và không được duy trì bởi nhóm cốt lõi NestJS. Vui lòng báo cáo bất kỳ vấn đề nào đến [repository thích hợp](https://github.com/suites-dev/suites).

#### Bắt đầu

Hướng dẫn này minh họa việc sử dụng Suites để kiểm tra các dịch vụ NestJS. Nó bao gồm cả kiểm tra cô lập (tất cả các phụ thuộc được mock) và kiểm tra xã hội (các triển khai thực tế được chọn).

#### Cài đặt Suites

Xác minh các phụ thuộc runtime NestJS đã được cài đặt:

```bash
$ npm install @nestjs/common @nestjs/core reflect-metadata
```

Cài đặt Suites core, adapter NestJS, và adapter doubles:

```bash
$ npm install --save-dev @suites/unit @suites/di.nestjs @suites/doubles.jest
```

Adapter doubles (`@suites/doubles.jest`) cung cấp các wrapper xung quanh khả năng mocking của Jest. Nó expose các hàm `mock()` và `stub()` tạo ra test doubles type-safe.

Đảm bảo Jest và TypeScript có sẵn:

```bash
$ npm install --save-dev ts-jest @types/jest jest typescript
```

<details><summary>Mở rộng nếu bạn đang sử dụng Vitest</summary>

```bash
$ npm install --save-dev @suites/unit @suites/di.nestjs @suites/doubles.vitest
```

</details>

<details><summary>Mở rộng nếu bạn đang sử dụng Sinon</summary>

```bash
$ npm install --save-dev @suites/unit @suites/di.nestjs @suites/doubles.sinon
```

</details>

> info **Gợi ý** Đảm bảo có `"emitDecoratorMetadata": true` trong tsconfig `compilerOptions` của bạn (tiêu chuẩn NestJS).

#### Thiết lập định nghĩa kiểu

Tạo `global.d.ts` tại gốc dự án của bạn:

```typescript
/// <reference types="@suites/doubles.jest/unit" />
/// <reference types="@suites/di.nestjs/types" />
```

#### Tạo một dịch vụ mẫu

Hướng dẫn này sử dụng một `UserService` đơn giản với hai phụ thuộc:

```typescript
@@filename(user.repository)
import { Injectable } from '@nestjs/common';

@Injectable()
export class UserRepository {
  async findById(id: string): Promise<User | null> {
    // Truy vấn cơ sở dữ liệu
  }

  async save(user: User): Promise<User> {
    // Lưu cơ sở dữ liệu
  }
}
```
```typescript
@@filename(user.service)
import { Injectable, NotFoundException } from '@nestjs/common';
import { Logger } from '@nestjs/common';

@Injectable()
export class UserService {
  constructor(
    private repository: UserRepository,
    private logger: Logger,
  ) {}

  async findById(id: string): Promise<User> {
    const user = await this.repository.findById(id);
    if (!user) {
      throw new NotFoundException(`User ${id} not found`);
    }
    this.logger.log(`Found user ${id}`);
    return user;
  }

  async create(email: string, name: string): Promise<User> {
    const user = { id: generateId(), email, name };
    await this.repository.save(user);
    this.logger.log(`Created user ${user.id}`);
    return user;
  }
}
```

#### Viết một bài kiểm tra đơn vị

Sử dụng `TestBed.solitary()` để tạo các bài kiểm tra cô lập với tất cả các phụ thuộc được mock:

```typescript
@@filename(user.service.spec)
import { TestBed, type Mocked } from '@suites/unit';
import { UserService } from './user.service';
import { UserRepository } from './user.repository';
import { Logger } from '@nestjs/common';

describe('User Service Unit Spec', () => {
  let userService: UserService;
  let repository: Mocked<UserRepository>;
  let logger: Mocked<Logger>;

  beforeAll(async () => {
    const { unit, unitRef } = await TestBed.solitary(UserService).compile();

    userService = unit;
    repository = unitRef.get(UserRepository);
    logger = unitRef.get(Logger);
  });

  it('should find user by id', async () => {
    const user = { id: '1', email: 'test@example.com', name: 'Test' };
    repository.findById.mockResolvedValue(user);

    const result = await userService.findById('1');

    expect(result).toEqual(user);
    expect(logger.log).toHaveBeenCalled();
  });
});
```

`TestBed.solitary()` phân tích constructor và tạo mocks kiểu cho tất cả các phụ thuộc.
Kiểu `Mocked<T>` cung cấp hỗ trợ IntelliSense cho cấu hình mock.

#### Cấu hình mock biên dịch trước

Cấu hình hành vi mock trước biên dịch sử dụng `.mock().impl()`:

```typescript
@@filename(user.service.spec)
import { TestBed } from '@suites/unit';
import { UserService } from './user.service';
import { UserRepository } from './user.repository';

describe('User Service Unit Spec - pre-configured', () => {
  let unit: UserService;
  let repository: Mocked<UserRepository>;
  
  beforeAll(async () => {
    const { unit: underTest, unitRef } = await TestBed.solitary(UserService)
      .mock(UserRepository)
      .impl(stubFn => ({
        findById: stubFn().mockResolvedValue({ id: '1', email: 'test@example.com', name: 'Test' })
      }))
      .compile();
    
    repository = unitRef.get(UserRepository);
    unit = underTest;
  })
  
  it('should find user with pre-configured mock', async () => {
    const result = await unit.findById('1');
    
    expect(repository.findById).toHaveBeenCalled();
    expect(result.email).toBe('test@example.com');
  });
});
```

Tham số `stubFn` tương ứng với adapter doubles đã cài đặt (`jest.fn()` cho Jest, `vi.fn()` cho Vitest, `sinon.stub()` cho Sinon).

#### Kiểm tra với các phụ thuộc thực

Sử dụng `TestBed.sociable()` với `.expose()` để sử dụng các triển khai thực tế cho các phụ thuộc cụ thể:

```typescript
@@filename(user.service.spec)
import { TestBed, Mocked } from '@suites/unit';
import { UserService } from './user.service';
import { UserRepository } from './user.repository';
import { Logger } from '@nestjs/common';

describe('UserService - with real logger', () => {
  let userService: UserService;
  let repository: Mocked<UserRepository>;

  beforeAll(async () => {
    const { unit, unitRef } = await TestBed.sociable(UserService)
      .expose(Logger)
      .compile();

    userService = unit;
    repository = unitRef.get(UserRepository);
  });

  it('should log when finding user', async () => {
    const user = { id: '1', email: 'test@example.com' };
    repository.findById.mockResolvedValue(user);

    await userService.findById('1');

    // Logger thực sự thực thi, không cần mock
  });
});
```

`.expose(Logger)` khởi tạo `Logger` với triển khai thực của nó trong khi giữ các phụ thuộc khác được mock.

#### Các phụ thuộc dựa trên token

Suites xử lý các token injection tùy chỉnh (chuỗi hoặc symbols):

```typescript
@@filename(config.service)
import { Injectable, Inject } from '@nestjs/common';

export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';

@Injectable()
export class ConfigService {
  constructor(
    @Inject(CONFIG_OPTIONS) private options: { apiKey: string },
  ) {}

  getApiKey(): string {
    return this.options.apiKey;
  }
}
```

Truy cập các phụ thuộc dựa trên token với `unitRef.get()`:

```typescript
@@filename(config.service.spec)
import { TestBed } from '@suites/unit';
import { ConfigService, CONFIG_OPTIONS, ConfigOptions } from './config.service';

describe('Config Service Unit Spec', () => {
  let configService: ConfigService;
  let options: ConfigOptions;

  beforeAll(async () => {
    const { unit, unitRef } = await TestBed.solitary(ConfigService).compile();
    configService = unit;

    options = unitRef.get<ConfigOptions>(CONFIG_OPTIONS);
  });

  it('should return api key', () => { ... });
});
```

#### Sử dụng mock() và stub() trực tiếp

Đối với những người thích kiểm soát trực tiếp mà không cần `TestBed`, gói adapter doubles cung cấp các hàm `mock()` và `stub()`:

```typescript
@@filename(user.service.spec)
import { mock } from '@suites/unit';
import { UserRepository } from './user.repository';

describe('User Service Unit Spec', () => {
  it('should work with direct mocks', async () => {
    const repository = mock<UserRepository>();
    const logger = mock<Logger>();

    const service = new UserService(repository, logger);

    // ...
  });
});
```

`mock()` tạo một đối tượng mock kiểu, và `stub()` bọc thư viện mocking cơ bản (Jest trong ví dụ này) để cung cấp các phương thức như `mockResolvedValue()`
Các hàm này đến từ adapter doubles đã cài đặt (`@suites/doubles.jest`), điều này thích ứng các khả năng mocking gốc của framework kiểm tra.

> info **Gợi ý** Hàm `mock()` là một sự thay thế cho `createMock` từ `@golevelup/ts-jest`. Cả hai đều tạo các đối tượng mock kiểu. Xem chương [cơ bản kiểm tra](/fundamentals/testing#auto-mocking) để biết thêm về `createMock`.

#### Tóm tắt

**Sử dụng `Test.createTestingModule()` cho:**
- Xác nhận cấu hình module và dây chuyền provider
- Kiểm tra decorators, guards, interceptors, và pipes
- Xác minh dependency injection giữa các module
- Kiểm tra ngữ cảnh ứng dụng đầy đủ với middleware

**Sử dụng Suites cho:**
- Các bài kiểm tra đơn vị nhanh tập trung vào logic kinh doanh
- Tạo mock tự động cho nhiều phụ thuộc
- Test doubles type-safe với IntelliSense

Tổ chức các bài kiểm tra theo mục đích: sử dụng Suites cho các bài kiểm tra đơn vị xác nhận hành vi dịch vụ riêng lẻ, và sử dụng `Test.createTestingModule()` cho các bài kiểm tra tích hợp xác nhận cấu hình module.

Để biết thêm thông tin:
- [Tài liệu Suites](https://suites.dev/docs)
- [Repository GitHub Suites](https://github.com/suites-dev/suites)
- [Tài liệu Kiểm tra NestJS](/fundamentals/testing)