### Healthchecks (Terminus)

Terminus integration cung cấp cho bạn các health check **readiness/liveness**. Healthchecks rất quan trọng khi nói đến các thiết lập backend phức tạp. Tóm lại, một health check trong lĩnh vực phát triển web thường bao gồm một địa chỉ đặc biệt, ví dụ, `https://my-website.com/health/readiness`. Một service hoặc thành phần của cơ sở hạ tầng của bạn (ví dụ, [Kubernetes](https://kubernetes.io/) sẽ liên tục kiểm tra địa chỉ này. Tùy thuộc vào mã trạng thái HTTP được trả về từ một yêu cầu `GET` đến địa chỉ này, service sẽ thực hiện hành động khi nó nhận được phản hồi "unhealthy" (không khỏe). Vì định nghĩa của "healthy" (khỏe mạnh) hoặc "unhealthy" thay đổi tùy theo loại service bạn cung cấp, integration **Terminus** hỗ trợ bạn với một tập hợp các **health indicators**.

Ví dụ, nếu web server của bạn sử dụng MongoDB để lưu trữ dữ liệu của mình, thông tin về việc MongoDB vẫn đang chạy hay không sẽ rất quan trọng. Trong trường hợp đó, bạn có thể sử dụng `MongooseHealthIndicator`. Nếu được cấu hình đúng - thêm chi tiết sau - địa chỉ health check của bạn sẽ trả về mã trạng thái HTTP healthy hoặc unhealthy, tùy thuộc vào việc MongoDB có đang chạy hay không.

#### Bắt đầu

Để bắt đầu với `@nestjs/terminus`, chúng ta cần cài đặt dependency cần thiết.

```bash
$ npm install --save @nestjs/terminus
```

#### Thiết lập Healthcheck

Một health check đại diện cho tóm tắt của các **health indicators**. Một health indicator thực hiện kiểm tra một service, xem nó có ở trạng thái healthy hay unhealthy. Một health check là tích cực nếu tất cả các health indicator được gán đều đang chạy. Vì nhiều ứng dụng sẽ cần các health indicator tương tự, [`@nestjs/terminus`](https://github.com/nestjs/terminus) cung cấp một tập hợp các indicators được định nghĩa trước, chẳng hạn như:

- `HttpHealthIndicator`
- `TypeOrmHealthIndicator`
- `MongooseHealthIndicator`
- `SequelizeHealthIndicator`
- `MikroOrmHealthIndicator`
- `PrismaHealthIndicator`
- `MicroserviceHealthIndicator`
- `GRPCHealthIndicator`
- `MemoryHealthIndicator`
- `DiskHealthIndicator`

Để bắt đầu với health check đầu tiên của chúng ta, hãy tạo `HealthModule` và import `TerminusModule` vào trong mảng imports của nó.

> info **Gợi ý** Để tạo module sử dụng [Nest CLI](cli/overview), chỉ cần thực thi lệnh `$ nest g module health`.

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';

@Module({
  imports: [TerminusModule]
})
export class HealthModule {}
```

Healthcheck(s) của chúng ta có thể được thực thi sử dụng một [controller](/controllers), có thể được thiết lập dễ dàng sử dụng [Nest CLI](cli/overview).

```bash
$ nest g controller health
```

> info **Thông tin** Khuyến nghị mạnh mẽ là kích hoạt shutdown hooks trong ứng dụng của bạn. Terminus integration sử dụng sự kiện lifecycle này nếu được kích hoạt. Đọc thêm về shutdown hooks [tại đây](fundamentals/lifecycle-events#application-shutdown).

#### HTTP Healthcheck

Khi chúng ta đã cài đặt `@nestjs/terminus`, import `TerminusModule` của chúng ta và tạo một controller mới, chúng ta đã sẵn sàng để tạo một health check.

`HttpHealthIndicator` yêu cầu package `@nestjs/axios` vì vậy hãy đảm bảo cài đặt nó:

```bash
$ npm i --save @nestjs/axios axios
```

Bây giờ chúng ta có thể thiết lập `HealthController` của mình:

```typescript
@@filename(health.controller)
import { Controller, Get } from '@nestjs/common';
import { HealthCheckService, HttpHealthIndicator, HealthCheck } from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'),
    ]);
  }
}
@@switch
import { Controller, Dependencies, Get } from '@nestjs/common';
import { HealthCheckService, HttpHealthIndicator, HealthCheck } from '@nestjs/terminus';

@Controller('health')
@Dependencies(HealthCheckService, HttpHealthIndicator)
export class HealthController {
  constructor(
    private health,
    private http,
  ) { }

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'),
    ])
  }
}
```

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}
@@switch
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}
```

Health check của chúng ta bây giờ sẽ gửi một yêu cầu _GET_ đến địa chỉ `https://docs.nestjs.com`. Nếu chúng ta nhận được phản hồi healthy từ địa chỉ đó, route của chúng ta tại `http://localhost:3000/health` sẽ trả về đối tượng sau với mã trạng thái 200.

```json
{
  "status": "ok",
  "info": {
    "nestjs-docs": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "nestjs-docs": {
      "status": "up"
    }
  }
}
```

Interface của đối tượng phản hồi này có thể được truy cập từ package `@nestjs/terminus` với interface `HealthCheckResult`.

||           |                                                                                                                                                                                             |                                      |
||-----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
|| `status`  | Nếu bất kỳ health indicator nào thất bại, trạng thái sẽ là `'error'`. Nếu ứng dụng NestJS đang tắt nhưng vẫn chấp nhận các yêu cầu HTTP, health check sẽ có trạng thái `'shutting_down'`. | `'error' \| 'ok' \| 'shutting_down'` |
|| `info`    | Đối tượng chứa thông tin của mỗi health indicator có trạng thái `'up'`, hoặc nói cách khác là "healthy".                                                                                 | `object`                             |
|| `error`   | Đối tượng chứa thông tin của mỗi health indicator có trạng thái `'down'`, hoặc nói cách khác là "unhealthy".                                                                              | `object`                             |
|| `details` | Đối tượng chứa tất cả thông tin của mỗi health indicator                                                                                                                                  | `object`                             |

##### Kiểm tra mã phản hồi HTTP cụ thể

Trong một số trường hợp, bạn có thể muốn kiểm tra các tiêu chí cụ thể và xác thực phản hồi. Ví dụ, giả sử `https://my-external-service.com` trả về mã phản hồi `204`. Với `HttpHealthIndicator.responseCheck`, bạn có thể kiểm tra mã phản hồi đó cụ thể và xác định tất cả các mã khác là unhealthy.

Trong trường hợp bất kỳ mã phản hồi nào khác ngoài `204` được trả về, ví dụ sau sẽ là unhealthy. Tham số thứ ba yêu cầu bạn cung cấp một hàm (đồng bộ hoặc bất đồng bộ) trả về boolean xem phản hồi được coi là healthy (`true`) hay unhealthy (`false`).


```typescript
@@filename(health.controller)
// Trong class `HealthController`

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () =>
      this.http.responseCheck(
        'my-external-service',
        'https://my-external-service.com',
        (res) => res.status === 204,
      ),
  ]);
}
```


#### TypeOrm health indicator

Terminus cung cấp khả năng thêm các kiểm tra database vào health check của bạn. Để bắt đầu với health indicator này, bạn nên kiểm tra [chương Database](/techniques/sql) và đảm bảo kết nối database trong ứng dụng của bạn đã được thiết lập.

> info **Gợi ý** Phía sau `TypeOrmHealthIndicator` đơn giản là thực thi lệnh SQL `SELECT 1` thường được sử dụng để xác minh xem database vẫn còn sống hay không. Trong trường hợp bạn sử dụng database Oracle, nó sẽ sử dụng `SELECT 1 FROM DUAL`.

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, TypeOrmHealthIndicator)
export class HealthController {
  constructor(
    private health,
    private db,
  ) { }

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ])
  }
}
```

Nếu database của bạn có thể truy cập được, bạn bây giờ sẽ thấy kết quả JSON sau khi yêu cầu `http://localhost:3000/health` với một yêu cầu `GET`:

```json
{
  "status": "ok",
  "info": {
    "database": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "database": {
      "status": "up"
    }
  }
}
```

Trong trường hợp ứng dụng của bạn sử dụng [nhiều database](techniques/database#multiple-databases), bạn cần inject mỗi kết nối vào `HealthController` của mình. Sau đó, bạn có thể đơn giản truyền tham chiếu kết nối đến `TypeOrmHealthIndicator`.

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    @InjectConnection('albumsConnection')
    private albumsConnection: Connection,
    @InjectConnection()
    private defaultConnection: Connection,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('albums-database', { connection: this.albumsConnection }),
      () => this.db.pingCheck('database', { connection: this.defaultConnection }),
    ]);
  }
}
```


#### Disk health indicator

Với `DiskHealthIndicator`, chúng ta có thể kiểm tra bao nhiêu dung lượng lưu trữ đang được sử dụng. Để bắt đầu, hãy đảm bảo inject `DiskHealthIndicator` vào `HealthController` của bạn. Ví dụ sau kiểm tra dung lượng lưu trữ được sử dụng của đường dẫn `/` (hoặc trên Windows bạn có thể sử dụng `C:\\`). Nếu điều đó vượt quá hơn 50% tổng dung lượng lưu trữ, nó sẽ phản hồi với một Health Check unhealthy.

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly disk: DiskHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.disk.checkStorage('storage', { path: '/', thresholdPercent: 0.5 }),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, DiskHealthIndicator)
export class HealthController {
  constructor(health, disk) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.disk.checkStorage('storage', { path: '/', thresholdPercent: 0.5 }),
    ])
  }
}
```

Với hàm `DiskHealthIndicator.checkStorage`, bạn cũng có khả năng kiểm tra một lượng dung lượng cố định. Ví dụ sau sẽ là unhealthy trong trường hợp đường dẫn `/my-app/` vượt quá 250GB.

```typescript
@@filename(health.controller)
// Trong class `HealthController`

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.disk.checkStorage('storage', {  path: '/', threshold: 250 * 1024 * 1024 * 1024, })
  ]);
}
```

#### Memory health indicator

Để đảm bảo process của bạn không vượt quá giới hạn bộ nhớ nhất định, `MemoryHealthIndicator` có thể được sử dụng.
Ví dụ sau có thể được sử dụng để kiểm tra heap của process của bạn.

> info **Gợi ý** Heap là phần bộ nhớ nơi bộ nhớ được cấp phát động nằm (tức là bộ nhớ được cấp phát qua malloc). Bộ nhớ được cấp phát từ heap sẽ vẫn được cấp phát cho đến khi một trong các điều sau xảy ra:
> - Bộ nhớ được _free_
> - Chương trình kết thúc

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private memory: MemoryHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.memory.checkHeap('memory_heap', 150 * 1024 * 1024),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, MemoryHealthIndicator)
export class HealthController {
  constructor(health, memory) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.memory.checkHeap('memory_heap', 150 * 1024 * 1024),
    ])
  }
}
```

Cũng có thể xác minh bộ nhớ RSS của process của bạn với `MemoryHealthIndicator.checkRSS`. Ví dụ này sẽ trả về mã phản hồi unhealthy trong trường hợp process của bạn có nhiều hơn 150MB được cấp phát.

> info **Gợi ý** RSS là Resident Set Size và được sử dụng để hiển thị bao nhiêu bộ nhớ được cấp phát cho process đó và đang trong RAM.
> Nó không bao gồm bộ nhớ đã được swap out. Nó bao gồm bộ nhớ từ các thư viện chia sẻ miễn là các trang từ các thư viện đó thực sự đang trong bộ nhớ. Nó bao gồm tất cả bộ nhớ stack và heap.


```typescript
@@filename(health.controller)
// Trong class `HealthController`

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.memory.checkRSS('memory_rss', 150 * 1024 * 1024),
  ]);
}
```


#### Custom health indicator

Trong một số trường hợp, các health indicator được định nghĩa trước được cung cấp bởi `@nestjs/terminus` không bao phủ tất cả các yêu cầu health check của bạn. Trong trường hợp đó, bạn có thể thiết lập một health indicator tùy chỉnh theo nhu cầu của bạn.

Hãy bắt đầu bằng cách tạo một service sẽ đại diện cho indicator tùy chỉnh của chúng ta. Để có sự hiểu biết cơ bản về cách một indicator được cấu trúc, chúng ta sẽ tạo một ví dụ `DogHealthIndicator`. Service này nên có trạng thái `'up'` nếu mọi đối tượng `Dog` có loại `'goodboy'`. Nếu điều kiện đó không được thỏa mãn thì nó nên throw một lỗi.

```typescript
@@filename(dog.health)
import { Injectable } from '@nestjs/common';
import { HealthIndicatorService } from '@nestjs/terminus';

export interface Dog {
  name: string;
  type: string;
}

@Injectable()
export class DogHealthIndicator {
  constructor(
    private readonly healthIndicatorService: HealthIndicatorService
  ) {}

  private dogs: Dog[] = [
    { name: 'Fido', type: 'goodboy' },
    { name: 'Rex', type: 'badboy' },
  ];

  async isHealthy(key: string){
    const indicator = this.healthIndicatorService.check(key);
    const badboys = this.dogs.filter(dog => dog.type === 'badboy');
    const isHealthy = badboys.length === 0;

    if (!isHealthy) {
      return indicator.down({ badboys: badboys.length });
    }

    return indicator.up();
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { HealthIndicatorService } from '@nestjs/terminus';

@Injectable()
@Dependencies(HealthIndicatorService)
export class DogHealthIndicator {
  constructor(healthIndicatorService) {
    this.healthIndicatorService = healthIndicatorService;
  }

  private dogs = [
    { name: 'Fido', type: 'goodboy' },
    { name: 'Rex', type: 'badboy' },
  ];

  async isHealthy(key){
    const indicator = this.healthIndicatorService.check(key);
    const badboys = this.dogs.filter(dog => dog.type === 'badboy');
    const isHealthy = badboys.length === 0;

    if (!isHealthy) {
      return indicator.down({ badboys: badboys.length });
    }

    return indicator.up();
  }
}
```

Điều tiếp theo chúng ta cần làm là đăng ký health indicator như một provider.

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { DogHealthIndicator } from './dog.health';

@Module({
  controllers: [HealthController],
  imports: [TerminusModule],
  providers: [DogHealthIndicator]
})
export class HealthModule { }
```

> info **Gợi ý** Trong một ứng dụng thực tế, `DogHealthIndicator` nên được cung cấp trong một module riêng biệt, ví dụ, `DogModule`, sau đó sẽ được import bởi `HealthModule`.

Bước cuối cùng cần thiết là thêm health indicator hiện có vào endpoint health check cần thiết. Để làm điều đó, chúng ta quay lại `HealthController` của mình và thêm nó vào hàm `check`.

```typescript
@@filename(health.controller)
import { HealthCheckService, HealthCheck } from '@nestjs/terminus';
import { Injectable, Dependencies, Get } from '@nestjs/common';
import { DogHealthIndicator } from './dog.health';

@Injectable()
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private dogHealthIndicator: DogHealthIndicator
  ) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.dogHealthIndicator.isHealthy('dog'),
    ])
  }
}
@@switch
import { HealthCheckService, HealthCheck } from '@nestjs/terminus';
import { Injectable, Get } from '@nestjs/common';
import { DogHealthIndicator } from './dog.health';

@Injectable()
@Dependencies(HealthCheckService, DogHealthIndicator)
export class HealthController {
  constructor(
    health,
    dogHealthIndicator
  ) {
    this.health = health;
    this.dogHealthIndicator = dogHealthIndicator;
  }

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.dogHealthIndicator.isHealthy('dog'),
    ])
  }
}
```

#### Logging

Terminus chỉ log các thông báo lỗi, ví dụ khi một Healthcheck thất bại. Với phương thức `TerminusModule.forRoot()`, bạn có nhiều quyền kiểm soát hơn về cách các lỗi được logging cũng như hoàn toàn tiếp quản việc logging.

Trong phần này, chúng ta sẽ hướng dẫn bạn cách tạo một logger tùy chỉnh `TerminusLogger`. Logger này mở rộng logger tích hợp sẵn. Do đó bạn có thể chọn và chọn phần nào của logger bạn muốn ghi đè

> info **Thông tin** Nếu bạn muốn tìm hiểu thêm về các logger tùy chỉnh trong NestJS, [đọc thêm tại đây](/techniques/logger#injecting-a-custom-logger).


```typescript
@@filename(terminus-logger.service)
import { Injectable, Scope, ConsoleLogger } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class TerminusLogger extends ConsoleLogger {
  error(message: any, stack?: string, context?: string): void;
  error(message: any, ...optionalParams: any[]): void;
  error(
    message: unknown,
    stack?: unknown,
    context?: unknown,
    ...rest: unknown[]
  ): void {
    // Ghi đè ở đây cách các thông báo lỗi nên được logging
  }
}
```

Khi bạn đã tạo logger tùy chỉnh của mình, tất cả những gì bạn cần làm là đơn giản truyền nó vào `TerminusModule.forRoot()` như sau.

```typescript
@@filename(health.module)
@Module({
imports: [
  TerminusModule.forRoot({
    logger: TerminusLogger,
  }),
],
})
export class HealthModule {}
```


Để hoàn toàn ngăn chặn bất kỳ thông báo log nào đến từ Terminus, bao gồm cả thông báo lỗi, cấu hình Terminus như sau.

```typescript
@@filename(health.module)
@Module({
imports: [
  TerminusModule.forRoot({
    logger: false,
  }),
],
})
export class HealthModule {}
```



Terminus cho phép bạn cấu hình cách các lỗi Healthcheck nên được hiển thị trong logs của bạn.

|| Kiểu Log Lỗi          | Mô tả                                                                                                                        | Ví dụ                                                              |
||:------------------|:-----------------------------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------------------|
|| `json`  (mặc định) | In tóm tắt kết quả health check trong trường hợp lỗi như đối tượng JSON                                                     | <figure><img src="/assets/Terminus_Error_Log_Json.png" /></figure>   |
|| `pretty`          | In tóm tắt kết quả health check trong trường hợp lỗi trong các hộp được định dạng và làm nổi bật kết quả thành công/lỗi | <figure><img src="/assets/Terminus_Error_Log_Pretty.png" /></figure> |

Bạn có thể thay đổi kiểu log sử dụng tùy chọn cấu hình `errorLogStyle` như trong đoạn sau.

```typescript
@@filename(health.module)
@Module({
  imports: [
    TerminusModule.forRoot({
      errorLogStyle: 'pretty',
    }),
  ]
})
export class HealthModule {}
```

#### Graceful shutdown timeout

Nếu ứng dụng của bạn yêu cầu hoãn quá trình tắt của nó, Terminus có thể xử lý điều đó cho bạn.
Cài đặt này có thể đặc biệt hữu ích khi làm việc với một orchestrator như Kubernetes.
Bằng cách thiết lập độ trễ hơi dài hơn khoảng thời gian kiểm tra readiness, bạn có thể đạt được zero downtime khi tắt containers.

```typescript
@@filename(health.module)
@Module({
  imports: [
    TerminusModule.forRoot({
      gracefulShutdownTimeoutMs: 1000,
    }),
  ]
})
export class HealthModule {}
```

#### Thêm ví dụ

Thêm ví dụ đang hoạt động có sẵn [tại đây](https://github.com/nestjs/terminus/tree/master/sample).