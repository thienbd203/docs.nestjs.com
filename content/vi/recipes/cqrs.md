### CQRS

Dòng chảy của các ứng dụng [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) (Create, Read, Update và Delete) đơn giản có thể được mô tả như sau:

1. Lớp controllers xử lý các yêu cầu HTTP và ủy nhiệm các nhiệm vụ cho lớp services.
2. Lớp services là nơi hầu hết logic kinh doanh sống.
3. Services sử dụng repositories / DAOs để thay đổi / duy trì các thực thể.
4. Các thực thể hoạt động như các vùng chứa cho các giá trị, với setters và getters.

Mặc dù pattern này thường đủ cho các ứng dụng nhỏ và trung bình, nó có thể không phải là lựa chọn tốt nhất cho các ứng dụng lớn hơn, phức tạp hơn. Trong những trường hợp như vậy, mô hình **CQRS** (Command and Query Responsibility Segregation) có thể phù hợp và có thể mở rộng hơn (tùy thuộc vào các yêu cầu của ứng dụng). Lợi ích của mô hình này bao gồm:

- **Tách biệt mối quan tâm**. Mô hình tách các hoạt động đọc và ghi thành các mô hình riêng biệt.
- **Khả năng mở rộng**. Các hoạt động đọc và ghi có thể được mở rộng độc lập.
- **Tính linh hoạt**. Mô hình cho phép sử dụng các cửa hàng dữ liệu khác nhau cho các hoạt động đọc và ghi.
- **Hiệu suất**. Mô hình cho phép sử dụng các cửa hàng dữ liệu khác nhau được tối ưu hóa cho các hoạt động đọc và ghi.

Để tạo điều kiện cho mô hình đó, Nest cung cấp một [module CQRS](https://github.com/nestjs/cqrs) nhẹ nhàng. Chương này mô tả cách sử dụng nó.

#### Cài đặt

Trước tiên cài đặt gói cần thiết:

```bash
$ npm install --save @nestjs/cqrs
```

Khi cài đặt hoàn tất, điều hướng đến module gốc của ứng dụng của bạn (thường là `AppModule`), và nhập `CqrsModule.forRoot()`:

```typescript
import { Module } from '@nestjs/common';
import { CqrsModule } from '@nestjs/cqrs';

@Module({
  imports: [CqrsModule.forRoot()],
})
export class AppModule {}
```

Module này chấp nhận một đối tượng cấu hình tùy chọn. Các tùy chọn sau có sẵn:

|| Thuộc tính                     | Mô tả                                                                                                                  | Mặc định                           |
|| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
|| `commandPublisher`            | Publisher chịu trách nhiệm gửi các lệnh đến hệ thống.                                                            | `DefaultCommandPubSub`            |
|| `eventPublisher`              | Publisher được sử dụng để xuất bản các sự kiện, cho phép chúng được phát sóng hoặc xử lý.                                          | `DefaultPubSub`                   |
|| `queryPublisher`              | Publisher được sử dụng để xuất bản các truy vấn, có thể kích hoạt các hoạt động truy xuất dữ liệu.                                      | `DefaultQueryPubSub`              |
|| `unhandledExceptionPublisher` | Publisher chịu trách nhiệm xử lý các ngoại lệ không được xử lý, đảm bảo chúng được theo dõi và báo cáo.                             | `DefaultUnhandledExceptionPubSub` |
|| `eventIdProvider`             | Service cung cấp các ID sự kiện duy nhất bằng cách tạo hoặc truy xuất chúng từ các instance sự kiện.                                | `DefaultEventIdProvider`          |
|| `rethrowUnhandled`            | Xác định xem các ngoại lệ không được xử lý có nên được ném lại sau khi được xử lý hay không, hữu ích cho gỡ lỗi và quản lý lỗi. | `false`                           |

#### Commands

Commands được sử dụng để thay đổi trạng thái ứng dụng. Chúng nên dựa trên nhiệm vụ, thay vì tập trung vào dữ liệu. Khi một command được gửi, nó được xử lý bởi một **Command Handler** tương ứng. Handler chịu trách nhiệm cập nhật trạng thái ứng dụng.

```typescript
@@filename(heroes-game.service)
@Injectable()
export class HeroesGameService {
  constructor(private commandBus: CommandBus) {}

  async killDragon(heroId: string, killDragonDto: KillDragonDto) {
    return this.commandBus.execute(
      new KillDragonCommand(heroId, killDragonDto.dragonId)
    );
  }
}
@@switch
@Injectable()
@Dependencies(CommandBus)
export class HeroesGameService {
  constructor(commandBus) {
    this.commandBus = commandBus;
  }

  async killDragon(heroId, killDragonDto) {
    return this.commandBus.execute(
      new KillDragonCommand(heroId, killDragonDto.dragonId)
    );
  }
}
```

Trong đoạn mã trên, chúng ta khởi tạo class `KillDragonCommand` và truyền nó đến phương thức `execute()` của `CommandBus`. Đây là class command được minh họa:

```typescript
@@filename(kill-dragon.command)
export class KillDragonCommand extends Command<{
  actionId: string // Kiểu này đại diện cho kết quả thực thi command
}> {
  constructor(
    public readonly heroId: string,
    public readonly dragonId: string,
  ) {
    super();
  }
}
@@switch
export class KillDragonCommand extends Command {
  constructor(heroId, dragonId) {
    this.heroId = heroId;
    this.dragonId = dragonId;
  }
}
```

Như bạn có thể thấy, class `KillDragonCommand` mở rộng class `Command`. Class `Command` là một class tiện ích đơn giản được xuất từ gói `@nestjs/cqrs` cho phép bạn định nghĩa kiểu trả về của command. Trong trường hợp này, kiểu trả về là một đối tượng với thuộc tính `actionId`. Bây giờ, bất cứ khi nào command `KillDragonCommand` được gửi, kiểu trả về của phương thức `CommandBus#execute()` sẽ được suy ra là `Promise<{ actionId: string }>`. Điều này hữu ích khi bạn muốn trả về một số dữ liệu từ command handler.

> info **Gợi ý** Kế thừa từ class `Command` là tùy chọn. Nó chỉ cần thiết nếu bạn muốn định nghĩa kiểu trả về của command.

`CommandBus` đại diện cho một **stream** của các command. Nó chịu trách nhiệm gửi các command đến các handler phù hợp. Phương thức `execute()` trả về một promise, giải quyết thành giá trị được trả về bởi handler.

Hãy tạo một handler cho command `KillDragonCommand`.

```typescript
@@filename(kill-dragon.handler)
@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
  constructor(private repository: HeroesRepository) {}

  async execute(command: KillDragonCommand) {
    const { heroId, dragonId } = command;
    const hero = this.repository.findOneById(+heroId);

    hero.killEnemy(dragonId);
    await this.repository.persist(hero);

    // "ICommandHandler<KillDragonCommand>" buộc bạn trả về một giá trị khớp với kiểu trả về của command
    return {
      actionId: crypto.randomUUID(), // Giá trị này sẽ được trả về cho người gọi
    }
  }
}
@@switch
@CommandHandler(KillDragonCommand)
@Dependencies(HeroesRepository)
export class KillDragonHandler {
  constructor(repository) {
    this.repository = repository;
  }

  async execute(command) {
    const { heroId, dragonId } = command;
    const hero = this.repository.findOneById(+heroId);

    hero.killEnemy(dragonId);
    await this.repository.persist(hero);

    // "ICommandHandler<KillDragonCommand>" buộc bạn trả về một giá trị khớp với kiểu trả về của command
    return {
      actionId: crypto.randomUUID(), // Giá trị này sẽ được trả về cho người gọi
    }
  }
}
```

Handler này truy xuất thực thể `Hero` từ repository, gọi phương thức `killEnemy()`, sau đó duy trì các thay đổi. Class `KillDragonHandler` thực hiện interface `ICommandHandler`, yêu cầu thực hiện phương thức `execute()`. Phương thức `execute()` nhận đối tượng command như một đối số.

Lưu ý rằng `ICommandHandler<KillDragonCommand>` buộc bạn trả về một giá trị khớp với kiểu trả về của command. Trong trường hợp này, kiểu trả về là một đối tượng với thuộc tính `actionId`. Điều này chỉ áp dụng cho các command kế thừa từ class `Command`. Nếu không, bạn có thể trả về bất cứ gì bạn muốn.

Cuối cùng, đảm bảo đăng ký `KillDragonHandler` như một provider trong một module:

```typescript
providers: [KillDragonHandler];
```

#### Queries

Queries được sử dụng để truy xuất dữ liệu từ trạng thái ứng dụng. Chúng nên tập trung vào dữ liệu, thay vì dựa trên nhiệm vụ. Khi một query được gửi, nó được xử lý bởi một **Query Handler** tương ứng. Handler chịu trách nhiệm truy xuất dữ liệu.

`QueryBus` tuân theo cùng một mẫu như `CommandBus`. Query handlers nên thực hiện interface `IQueryHandler` và được chú thích với decorator `@QueryHandler()`. Xem ví dụ sau:

```typescript
export class GetHeroQuery extends Query<Hero> {
  constructor(public readonly heroId: string) {}
}
```

Tương tự như class `Command`, class `Query` là một class tiện ích đơn giản được xuất từ gói `@nestjs/cqrs` cho phép bạn định nghĩa kiểu trả về của query. Trong trường hợp này, kiểu trả về là một đối tượng `Hero`. Bây giờ, bất cứ khi nào query `GetHeroQuery` được gửi, kiểu trả về của phương thức `QueryBus#execute()` sẽ được suy ra là `Promise<Hero>`.

Để truy xuất hero, chúng ta cần tạo một query handler:

```typescript
@@filename(get-hero.handler)
@QueryHandler(GetHeroQuery)
export class GetHeroHandler implements IQueryHandler<GetHeroQuery> {
  constructor(private repository: HeroesRepository) {}

  async execute(query: GetHeroQuery) {
    return this.repository.findOneById(query.heroId);
  }
}
@@switch
@QueryHandler(GetHeroQuery)
@Dependencies(HeroesRepository)
export class GetHeroHandler {
  constructor(repository) {
    this.repository = repository;
  }

  async execute(query) {
    return this.repository.findOneById(query.hero);
  }
}
```

Class `GetHeroHandler` thực hiện interface `IQueryHandler`, yêu cầu thực hiện phương thức `execute()`. Phương thức `execute()` nhận đối tượng query như một đối số, và phải trả về dữ liệu khớp với kiểu trả về của query (trong trường hợp này, một đối tượng `Hero`).

Cuối cùng, đảm bảo đăng ký `GetHeroHandler` như một provider trong một module:

```typescript
providers: [GetHeroHandler];
```

Bây giờ, để gửi query, sử dụng `QueryBus`:

```typescript
const hero = await this.queryBus.execute(new GetHeroQuery(heroId)); // "hero" sẽ được tự động suy ra là kiểu "Hero"
```

#### Events

Events được sử dụng để thông báo cho các phần khác của ứng dụng về các thay đổi trong trạng thái ứng dụng. Chúng được gửi bởi **models** hoặc trực tiếp sử dụng `EventBus`. Khi một sự kiện được gửi, nó được xử lý bởi các **Event Handler** tương ứng. Sau đó Handlers có thể, ví dụ, cập nhật mô hình đọc.

Để minh họa, hãy tạo một class sự kiện:

```typescript
@@filename(hero-killed-dragon.event)
export class HeroKilledDragonEvent {
  constructor(
    public readonly heroId: string,
    public readonly dragonId: string,
  ) {}
}
@@switch
export class HeroKilledDragonEvent {
  constructor(heroId, dragonId) {
    this.heroId = heroId;
    this.dragonId = dragonId;
  }
}
```

Bây giờ trong khi các sự kiện có thể được gửi trực tiếp sử dụng phương thức `EventBus.publish()`, chúng ta cũng có thể gửi chúng từ model. Hãy cập nhật model `Hero` để gửi sự kiện `HeroKilledDragonEvent` khi phương thức `killEnemy()` được gọi.

```typescript
@@filename(hero.model)
export class Hero extends AggregateRoot {
  constructor(private id: string) {
    super();
  }

  killEnemy(enemyId: string) {
    // Logic kinh doanh
    this.apply(new HeroKilledDragonEvent(this.id, enemyId));
  }
}
@@switch
export class Hero extends AggregateRoot {
  constructor(id) {
    super();
    this.id = id;
  }

  killEnemy(enemyId) {
    // Logic kinh doanh
    this.apply(new HeroKilledDragonEvent(this.id, enemyId));
  }
}
```

Phương thức `apply()` được sử dụng để gửi các sự kiện. Nó chấp nhận một đối tượng sự kiện như một đối số. Tuy nhiên, vì model của chúng ta không biết về `EventBus`, chúng ta cần liên kết nó với model. Chúng ta có thể làm điều đó bằng cách sử dụng class `EventPublisher`.

```typescript
@@filename(kill-dragon.handler)
@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
  constructor(
    private repository: HeroesRepository,
    private publisher: EventPublisher,
  ) {}

  async execute(command: KillDragonCommand) {
    const { heroId, dragonId } = command;
    const hero = this.publisher.mergeObjectContext(
      await this.repository.findOneById(+heroId),
    );
    hero.killEnemy(dragonId);
    hero.commit();
  }
}
@@switch
@CommandHandler(KillDragonCommand)
@Dependencies(HeroesRepository, EventPublisher)
export class KillDragonHandler {
  constructor(repository, publisher) {
    this.repository = repository;
    this.publisher = publisher;
  }

  async execute(command) {
    const { heroId, dragonId } = command;
    const hero = this.publisher.mergeObjectContext(
      await this.repository.findOneById(+heroId),
    );
    hero.killEnemy(dragonId);
    hero.commit();
  }
}
```

Phương thức `EventPublisher#mergeObjectContext` hợp nhất event publisher vào đối tượng được cung cấp. Đối tượng này phải thực hiện interface `IAggregateRoot` (hoặc mở rộng class `AggregateRoot`). Khi được hợp nhất, đối tượng sẽ có thể xuất bản các sự kiện đến stream sự kiện.

Lưu ý rằng trong ví dụ này chúng ta cũng gọi phương thức `commit()` trên model. Phương thức này được sử dụng để gửi bất kỳ sự kiện chưa gửi nào. Để tự động gửi các sự kiện, chúng ta có thể đặt thuộc tính `autoCommit` thành `true`:

```typescript
export class Hero extends AggregateRoot {
  constructor(private id: string) {
    super();
    this.autoCommit = true;
  }
}
```

Trong trường hợp chúng ta muốn hợp nhất event publisher vào một đối tượng không tồn tại, mà thay vào đó vào một class, chúng ta có thể sử dụng phương thức `EventPublisher#mergeClassContext`:

```typescript
const HeroModel = this.publisher.mergeClassContext(Hero);
const hero = new HeroModel('id'); // <-- HeroModel là một class
```

Bây giờ mọi instance của class `HeroModel` sẽ có thể xuất bản các sự kiện mà không cần sử dụng phương thức `mergeObjectContext()`.

#### Aggregate Roots linh hoạt

Class `AggregateRoot` là một triển khai cụ thể mà bạn có thể mở rộng để thêm khả năng dẫn hướng sự kiện vào các mô hình domain của bạn. Tuy nhiên, cách tiếp cận này yêu cầu các thực thể domain trực tiếp mở rộng `AggregateRoot`, có thể là một hạn chế nếu các ứng dụng của bạn đã có một hệ thống phân cấp kế thừa thực thể được thiết lập (ví dụ, một class cơ sở `Entity` hoặc các class cơ bản cụ thể cho domain như `Monster`, `Vehicle`, v.v.).

Để cung cấp sự linh hoạt hơn, gói `@nestjs/cqrs` cung cấp ba cách tiếp cận khác nhau để triển khai aggregate roots:

**Cách tiếp cận 1: Truyền thống (Kế thừa Class)**

Đây là cách tiếp cận tiêu chuẩn được hiển thị trong các ví dụ trước. Nó hoạt động hoàn hảo cho các kịch bản đơn giản hoặc dự án greenfield.

```typescript
export class Hero extends AggregateRoot {
  constructor(private id: string) {
    super();
  }

  killEnemy(enemyId: string) {
    this.apply(new HeroKilledDragonEvent(this.id, enemyId));
  }
}
```

**Cách tiếp cận 2: Mixin (Đối với các hệ thống phân cấp hiện có)**

Nếu bạn đã có một class cơ sở và không thể mở rộng `AggregateRoot` trực tiếp, bạn có thể sử dụng hàm mixin `WithAggregateRoot<EventBase, TBase>()`. Điều này cho phép bạn áp dụng hành vi aggregate root vào bất kỳ class cơ sở hiện có nào.

```typescript
@@filename(dragon.model)
abstract class Monster {
  constructor(protected readonly id: string) {}
  abstract roar(): void;
}

export class Dragon extends WithAggregateRoot(Monster) {
  roar(): void {
    console.log('Roarrrr!');
  }

  die(): void {
    this.roar();
    this.apply(new DragonDiedEvent(this.id)); // Bây giờ có sẵn qua mixin!
  }
}
@@switch
abstract class Monster {
  constructor(id) {
    this.id = id;
  }
}

export class Dragon extends WithAggregateRoot(Monster) {
  roar() {
    console.log('Roarrrr!');
  }

  die() {
    this.roar();
    this.apply(new DragonDiedEvent(this.id));
  }
}
```

**Cách tiếp cận 3: Triển khai tùy chỉnh**

Để kiểm soát tối đa, hoặc nếu bạn muốn giữ layer domain của mình hoàn toàn không phụ thuộc vào framework, bạn có thể thực hiện trực tiếp interface `IAggregateRoot`. `EventPublisher` chấp nhận bất kỳ đối tượng nào thực hiện interface này.

```typescript
export class CustomEntity implements IAggregateRoot {
  private events: IEvent[] = [];

  getUncommittedEvents() {
    return this.events;
  }

  publish(event: IEvent) {
    // logic tùy chỉnh
  }

  commit() {
    // logic tùy chỉnh
  }

  uncommit() {
    // logic tùy chỉnh
  }

  apply(event: IEvent) {
    this.events.push(event);
  }

  loadFromHistory(history: IEvent[]) {
    // logic tùy chỉnh
  }
}
```

Tất cả ba cách tiếp cận hoạt động liền mạch với `EventPublisher`, chấp nhận bất kỳ đối tượng nào thực hiện interface `IAggregateRoot`.
#### Xuất bản sự kiện thủ công

```typescript
this.eventBus.publish(new HeroKilledDragonEvent());
```

> info **Gợi ý** `EventBus` là một class có thể inject.

Mỗi sự kiện có thể có nhiều **Event Handler**.

```typescript
@@filename(hero-killed-dragon.handler)
@EventsHandler(HeroKilledDragonEvent)
export class HeroKilledDragonHandler implements IEventHandler<HeroKilledDragonEvent> {
  constructor(private repository: HeroesRepository) {}

  handle(event: HeroKilledDragonEvent) {
    // Logic kinh doanh
  }
}
```

> info **Gợi ý** Hãy lưu ý rằng khi bạn bắt đầu sử dụng event handlers bạn sẽ ra khỏi ngữ cảnh web HTTP truyền thống.
>
> - Lỗi trong `CommandHandlers` vẫn có thể được bắt bởi [Exception filters](/exception-filters) tích hợp sẵn.
> - Lỗi trong `EventHandlers` không thể được bắt bởi Exception filters: bạn sẽ phải xử lý chúng thủ công. Hoặc bằng một `try/catch` đơn giản, sử dụng [Sagas](/recipes/cqrs#sagas) bằng cách kích hoạt một sự kiện bù trừ, hoặc bất kỳ giải pháp nào khác bạn chọn.
> - Các phản hồi HTTP trong `CommandHandlers` vẫn có thể được gửi lại cho client.
> - Các phản hồi HTTP trong `EventHandlers` không thể. Nếu bạn muốn gửi thông tin cho client bạn có thể sử dụng [WebSocket](/websockets/gateways), [SSE](/techniques/server-sent-events), hoặc bất kỳ giải pháp nào khác bạn chọn.

Giống như commands và queries, đảm bảo đăng ký `HeroKilledDragonHandler` như một provider trong một module:

```typescript
providers: [HeroKilledDragonHandler];
```

#### Sagas

Saga là một quá trình chạy dài lắng nghe các sự kiện và có thể kích hoạt các command mới. Nó thường được sử dụng để quản lý các workflow phức tạp trong ứng dụng. Ví dụ, khi một người dùng đăng ký, một saga có thể lắng nghe `UserRegisteredEvent` và gửi email chào mừng cho người dùng.

Sagas là một tính năng cực kỳ mạnh mẽ. Một saga đơn lẻ có thể lắng nghe 1..* sự kiện. Sử dụng thư viện [RxJS](https://github.com/ReactiveX/rxjs), chúng ta có thể lọc, ánh xạ, fork, và hợp nhất các stream sự kiện để tạo các workflow tinh vi. Mỗi saga trả về một Observable tạo ra một instance command. Command này sau đó được gửi **bất đồng bộ** bởi `CommandBus`.

Hãy tạo một saga lắng nghe `HeroKilledDragonEvent` và gửi command `DropAncientItemCommand`.

```typescript
@@filename(heroes-game.saga)
@Injectable()
export class HeroesGameSagas {
  @Saga()
  dragonKilled = (events$: Observable<any>): Observable<ICommand> => {
    return events$.pipe(
      ofType(HeroKilledDragonEvent),
      map((event) => new DropAncientItemCommand(event.heroId, fakeItemID)),
    );
  }
}
@@switch
@Injectable()
export class HeroesGameSagas {
  @Saga()
  dragonKilled = (events$) => {
    return events$.pipe(
      ofType(HeroKilledDragonEvent),
      map((event) => new DropAncientItemCommand(event.heroId, fakeItemID)),
    );
  }
}
```

> info **Gợi ý** Toán tử `ofType` và decorator `@Saga()` được xuất từ gói `@nestjs/cqrs`.

Decorator `@Saga()` đánh dấu phương thức như một saga. Đối số `events$` là một Observable stream của tất cả các sự kiện. Toán tử `ofType` lọc stream theo loại sự kiện được chỉ định. Toán tử `map` ánh xạ sự kiện đến một instance command mới.

Trong ví dụ này, chúng ta ánh xạ `HeroKilledDragonEvent` đến command `DropAncientItemCommand`. Command `DropAncientItemCommand` sau đó được tự động gửi bởi `CommandBus`.

Giống như query, command, và event handlers, đảm bảo đăng ký `HeroesGameSagas` như một provider trong một module:

```typescript
providers: [HeroesGameSagas];
```

#### Các ngoại lệ không được xử lý

Event handlers được thực thi bất đồng bộ, vì vậy chúng phải luôn xử lý các ngoại lệ đúng cách để ngăn ứng dụng nhập vào một trạng thái không nhất quán. Nếu một ngoại lệ không được xử lý, `EventBus` sẽ tạo một đối tượng `UnhandledExceptionInfo` và đẩy nó đến stream `UnhandledExceptionBus`. Stream này là một `Observable` có thể được sử dụng để xử lý các ngoại lệ không được xử lý.

```typescript
private destroy$ = new Subject<void>();

constructor(private unhandledExceptionsBus: UnhandledExceptionBus) {
  this.unhandledExceptionsBus
    .pipe(takeUntil(this.destroy$))
    .subscribe((exceptionInfo) => {
      // Xử lý ngoại lệ ở đây
      // ví dụ gửi đến dịch vụ bên ngoài, chấm dứt quá trình, hoặc xuất bản một sự kiện mới
    });
}

onModuleDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

Để lọc ra các ngoại lệ, chúng ta có thể sử dụng toán tử `ofType`, như sau:

```typescript
this.unhandledExceptionsBus
  .pipe(
    takeUntil(this.destroy$),
    UnhandledExceptionBus.ofType(TransactionNotAllowedException),
  )
  .subscribe((exceptionInfo) => {
    // Xử lý ngoại lệ ở đây
  });
```

Ở đó `TransactionNotAllowedException` là ngoại lệ chúng ta muốn lọc ra.

Đối tượng `UnhandledExceptionInfo` chứa các thuộc tính sau:

```typescript
export interface UnhandledExceptionInfo<
  Cause = IEvent | ICommand,
  Exception = any,
> {
  /**
   * Ngoại lệ đã được ném.
   */
  exception: Exception;
  /**
   * Nguyên nhân của ngoại lệ (tham chiếu sự kiện hoặc command).
   */
  cause: Cause;
}
```

#### Đăng ký tất cả các sự kiện

`CommandBus`, `QueryBus` và `EventBus` đều là **Observables**. Điều này có nghĩa là chúng ta có thể đăng ký vào toàn bộ stream và, ví dụ, xử lý tất cả các sự kiện. Ví dụ, chúng ta có thể ghi nhật ký tất cả các sự kiện vào console, hoặc lưu chúng vào cửa hàng sự kiện.

```typescript
private destroy$ = new Subject<void>();

constructor(private eventBus: EventBus) {
  this.eventBus
    .pipe(takeUntil(this.destroy$))
    .subscribe((event) => {
      // Lưu sự kiện vào cơ sở dữ liệu
    });
}

onModuleDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

#### Phạm vi dựa trên yêu cầu

Đối với những người đến từ các nền tảng ngôn ngữ lập trình khác, có thể đáng ngạc nhiên khi biết rằng trong Nest, hầu hết mọi thứ được chia sẻ trên các yêu cầu đến. Điều này bao gồm một pool kết nối đến cơ sở dữ liệu, các service singleton với trạng thái toàn cầu, và nhiều hơn. Hãy nhớ rằng Node.js không tuân theo mô hình stateless đa luồng yêu cầu/phản hồi, trong đó mỗi yêu cầu được xử lý bởi một luồng riêng biệt. Kết quả là, sử dụng các instance singleton là **an toàn** cho các ứng dụng của chúng ta.

Tuy nhiên, có các trường hợp edge nơi một vòng đời dựa trên yêu cầu cho handler có thể được mong muốn. Điều này có thể bao gồm các kịch bản như caching theo yêu cầu trong các ứng dụng GraphQL, theo dõi yêu cầu, hoặc đa tenancy. Bạn có thể tìm hiểu thêm về cách kiểm soát các phạm vi [ở đây](/fundamentals/injection-scopes).

Sử dụng các provider phạm vi yêu cầu cùng với CQRS có thể phức tạp vì `CommandBus`, `QueryBus`, và `EventBus` là các singleton. May mắn thay, gói `@nestjs/cqrs` đơn giản hóa điều này bằng cách tự động tạo một instance mới của các handler phạm vi yêu cầu cho mỗi command, query, hoặc sự kiện được xử lý.

Để làm cho một handler phạm vi yêu cầu, bạn có thể:

1. Phụ thuộc vào một provider phạm vi yêu cầu.
2. Đặt rõ ràng phạm vi của nó thành `REQUEST` sử dụng decorator `@CommandHandler`, `@QueryHandler`, hoặc `@EventsHandler`, như được hiển thị:

```typescript
@CommandHandler(KillDragonCommand, {
  scope: Scope.REQUEST,
})
export class KillDragonHandler {
  // Triển khai ở đây
}
```

Để inject payload yêu cầu vào bất kỳ provider phạm vi yêu cầu nào, bạn sử dụng decorator `@Inject(REQUEST)`. Tuy nhiên, bản chất của payload yêu cầu trong CQRS phụ thuộc vào ngữ cảnh—nó có thể là một yêu cầu HTTP, một công việc được lên lịch, hoặc bất kỳ hoạt động nào khác kích hoạt một command.

Payload phải là một instance của một class mở rộng `AsyncContext` (được cung cấp bởi `@nestjs/cqrs`), hoạt động như ngữ cảnh yêu cầu và giữ dữ liệu có thể truy cập trong suốt vòng đời yêu cầu.

```typescript
import { AsyncContext } from '@nestjs/cqrs';

export class MyRequest extends AsyncContext {
  constructor(public readonly user: User) {
    super();
  }
}
```

Khi thực thi một command, chuyển ngữ cảnh yêu cầu tùy chỉnh như đối số thứ hai đến phương thức `CommandBus#execute`:

```typescript
const myRequest = new MyRequest(user);
await this.commandBus.execute(
  new KillDragonCommand(heroId, killDragonDto.dragonId),
  myRequest,
);
```

Điều này làm cho instance `MyRequest` có sẵn như provider `REQUEST` cho handler tương ứng:

```typescript
@CommandHandler(KillDragonCommand, {
  scope: Scope.REQUEST,
})
export class KillDragonHandler {
  constructor(
    @Inject(REQUEST) private request: MyRequest, // Inject ngữ cảnh yêu cầu
  ) {}

  // Triển khai handler ở đây
}
```

Bạn có thể theo cùng cách tiếp cận cho các truy vấn:

```typescript
const myRequest = new MyRequest(user);
const hero = await this.queryBus.execute(new GetHeroQuery(heroId), myRequest);
```

Và trong query handler:

```typescript
@QueryHandler(GetHeroQuery, {
  scope: Scope.REQUEST,
})
export class GetHeroHandler {
  constructor(
    @Inject(REQUEST) private request: MyRequest, // Inject ngữ cảnh yêu cầu
  ) {}

  // Triển khai handler ở đây
}
```

Đối với các sự kiện, trong khi bạn có thể chuyển provider yêu cầu đến `EventBus#publish`, điều này ít phổ biến hơn. Thay vào đó, sử dụng `EventPublisher` để hợp nhất provider yêu cầu vào một model:

```typescript
const hero = this.publisher.mergeObjectContext(
  await this.repository.findOneById(+heroId),
  this.request, // Inject ngữ cảnh yêu cầu ở đây
);
```

Event handlers phạm vi yêu cầu đăng ký vào các sự kiện này sẽ có quyền truy cập vào provider yêu cầu.

Sagas luôn là các instance singleton vì chúng quản lý các quá trình chạy dài. Tuy nhiên, bạn có thể truy xuất provider yêu cầu từ các đối tượng sự kiện:

```typescript
@Saga()
dragonKilled = (events$: Observable<any>): Observable<ICommand> => {
  return events$.pipe(
    ofType(HeroKilledDragonEvent),
    map((event) => {
      const request = AsyncContext.of(event); // Truy xuất ngữ cảnh yêu cầu
      const command = new DropAncientItemCommand(event.heroId, fakeItemID);

      AsyncContext.merge(request, command); // Hợp nhất ngữ cảnh yêu cầu vào command
      return command;
    }),
  );
}
```

Ngoài ra, sử dụng phương thức `request.attachTo(command)` để gắn ngữ cảnh yêu cầu đến command.

#### Ví dụ

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/kamilmysliwiec/nest-cqrs-example).