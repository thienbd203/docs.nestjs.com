### Discovery service

`DiscoveryService` được cung cấp bởi package `@nestjs/core` là một utility mạnh mẽ cho phép các nhà phát triển dynamically inspect và retrieve providers, controllers, và metadata khác trong một ứng dụng NestJS. Điều này đặc biệt hữu ích khi xây dựng plugins, decorators, hoặc các tính năng nâng cao phụ thuộc vào runtime introspection. Bằng cách tận dụng `DiscoveryService`, các nhà phát triển có thể tạo các kiến trúc linh hoạt động và module hóa hơn, cho phép automation và dynamic behavior trong các ứng dụng của họ.

#### Getting started

Trước khi sử dụng `DiscoveryService`, bạn cần import `DiscoveryModule` trong module nơi bạn định sử dụng nó. Điều này đảm bảo rằng service có sẵn cho dependency injection. Dưới đây là một ví dụ về cách cấu hình nó trong một module NestJS:

```typescript
import { Module } from '@nestjs/common';
import { DiscoveryModule } from '@nestjs/core';
import { ExampleService } from './example.service';

@Module({
  imports: [DiscoveryModule],
  providers: [ExampleService],
})
export class ExampleModule {}
```

Khi module được thiết lập, `DiscoveryService` có thể được inject vào bất kỳ provider hoặc service nào nơi dynamic discovery được yêu cầu.

```typescript
@@filename(example.service)
@Injectable()
export class ExampleService {
  constructor(private readonly discoveryService: DiscoveryService) {}
}
@@switch
@Injectable()
@Dependencies(DiscoveryService)
export class ExampleService {
  constructor(discoveryService) {
    this.discoveryService = discoveryService;
  }
}
```

#### Discovering providers và controllers

Một trong những khả năng chính của `DiscoveryService` là retrieving tất cả các providers được đăng ký trong ứng dụng. Điều này hữu ích để dynamically xử lý providers dựa trên các điều kiện cụ thể. Snippet sau đây minh họa cách truy cập tất cả providers:

```typescript
const providers = this.discoveryService.getProviders();
console.log(providers);
```

Mỗi đối tượng provider chứa thông tin như instance, token, và metadata của nó. Tương tự, nếu bạn cần retrieve tất cả các controllers được đăng ký trong ứng dụng, bạn có thể làm như vậy với:

```typescript
const controllers = this.discoveryService.getControllers();
console.log(controllers);
```

Tính năng này đặc biệt có lợi cho các kịch bản nơi controllers cần được xử lý dynamically, chẳng hạn như analytics tracking, hoặc các cơ chế đăng ký tự động.

#### Extracting metadata

Ngoài việc discovering providers và controllers, `DiscoveryService` cũng cho phép retrieval của metadata được attach đến các components này. Điều này đặc biệt có giá trị khi làm việc với các decorators tùy chỉnh lưu trữ metadata tại runtime.

Ví dụ, hãy xem xét một trường hợp nơi một decorator tùy chỉnh được sử dụng để tag providers với metadata cụ thể:

```typescript
import { DiscoveryService } from '@nestjs/core';

export const FeatureFlag = DiscoveryService.createDecorator();
```

Áp dụng decorator này đến một service cho phép nó lưu trữ metadata có thể được query sau:

```typescript
import { Injectable } from '@nestjs/common';
import { FeatureFlag } from './custom-metadata.decorator';

@Injectable()
@FeatureFlag('experimental')
export class CustomService {}
```

Khi metadata được attach đến providers theo cách này, `DiscoveryService` giúp dễ dàng filter providers dựa trên metadata được gán. Code snippet sau đây minh họa cách retrieve providers đã được tag với một giá trị metadata cụ thể:

```typescript
const providers = this.discoveryService.getProviders();

const [provider] = providers.filter(
  (item) =>
    this.discoveryService.getMetadataByDecorator(FeatureFlag, item) ===
    'experimental',
);

console.log(
  'Providers with the "experimental" feature flag metadata:',
  provider,
);
```

#### Conclusion

`DiscoveryService` là một công cụ đa năng và mạnh mẽ cho phép runtime introspection trong các ứng dụng NestJS. Bằng cách cho phép dynamic discovery của providers, controllers, và metadata, nó đóng một vai trò quan trọng trong việc xây dựng các framework có thể mở rộng, plugins, và các tính năng được điều khiển bởi automation. Cho dù bạn cần quét và xử lý providers, extract metadata để xử lý nâng cao, hoặc tạo các kiến trúc module hóa và có khả năng mở rộng, `DiscoveryService` cung cấp một cách tiếp cận hiệu quả và có cấu trúc để đạt được các mục tiêu này.