### Events

Package [Event Emitter](https://www.npmjs.com/package/@nestjs/event-emitter) (`@nestjs/event-emitter`) cung cấp một implementation observer đơn giản, cho phép bạn subscribe và lắng nghe các sự kiện khác nhau xảy ra trong ứng dụng của bạn. Các sự kiện đóng vai trò là một cách tuyệt vời để decouple các khía cạnh khác nhau của ứng dụng của bạn, vì một sự kiện đơn lẻ có thể có nhiều listener không phụ thuộc vào nhau.

`EventEmitterModule` sử dụng nội bộ package [eventemitter2](https://github.com/EventEmitter2/EventEmitter2).

#### Bắt đầu

Đầu tiên cài đặt package cần thiết:

```shell
$ npm i --save @nestjs/event-emitter
```

Sau khi cài đặt hoàn tất, nhập `EventEmitterModule` vào `AppModule` gốc và chạy phương thức tĩnh `forRoot()` như hiển thị bên dưới:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    EventEmitterModule.forRoot()
  ],
})
export class AppModule {}
```

Lời gọi `.forRoot()` khởi tạo event emitter và đăng ký bất kỳ event listener declarative nào tồn tại trong ứng dụng của bạn. Đăng ký xảy ra khi lifecycle hook `onApplicationBootstrap` diễn ra, đảm bảo rằng tất cả các module đã tải và khai báo bất kỳ công việc đã lên lịch nào.

Để cấu hình instance `EventEmitter` cơ bản, truyền đối tượng cấu hình cho phương thức `.forRoot()`, như sau:

```typescript
EventEmitterModule.forRoot({
  // set this to `true` to use wildcards
  wildcard: false,
  // the delimiter used to segment namespaces
  delimiter: '.',
  // set this to `true` if you want to emit the newListener event
  newListener: false,
  // set this to `true` if you want to emit the removeListener event
  removeListener: false,
  // the maximum amount of listeners that can be assigned to an event
  maxListeners: 10,
  // show event name in memory leak message when more than maximum amount of listeners is assigned
  verboseMemoryLeak: false,
  // disable throwing uncaughtException if an error event is emitted and it has no listeners
  ignoreErrors: false,
});
```

#### Dispatching sự kiện

Để dispatch (tức là fire) một sự kiện, trước tiên inject `EventEmitter2` bằng cách sử dụng constructor injection tiêu chuẩn:

```typescript
constructor(private eventEmitter: EventEmitter2) {}
```

> info **Gợi ý** Import `EventEmitter2` từ package `@nestjs/event-emitter`.

Sau đó sử dụng nó trong một class như sau:

```typescript
this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({
    orderId: 1,
    payload: {},
  }),
);
```

#### Lắng nghe sự kiện

Để khai báo một event listener, trang trí một phương thức với decorator `@OnEvent()` đứng trước định nghĩa phương thức chứa code để được thực thi, như sau:

```typescript
@OnEvent('order.created')
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // handle and process "OrderCreatedEvent" event
}
```

> warning **Cảnh báo** Event subscribers không thể là request-scoped.

Đối số đầu tiên có thể là một `string` hoặc `symbol` cho một event emitter đơn giản và một `string | symbol | Array<string | symbol>` trong trường hợp của một wildcard emitter.

Đối số thứ hai (tùy chọn) là một đối tượng tùy chọn listener như sau:

```typescript
export type OnEventOptions = OnOptions & {
  /**
   * If "true", prepends (instead of append) the given listener to the array of listeners.
   *
   * @see https://github.com/EventEmitter2/EventEmitter2#emitterprependlistenerevent-listener-options
   *
   * @default false
   */
  prependListener?: boolean;

  /**
   * If "true", the onEvent callback will not throw an error while handling the event. Otherwise, if "false" it will throw an error.
   *
   * @default true
   */
  suppressErrors?: boolean;
};
```

> info **Gợi ý** Đọc thêm về đối tượng tùy chọn `OnOptions` từ [`eventemitter2`](https://github.com/EventEmitter2/EventEmitter2#emitteronevent-listener-options-objectboolean).

```typescript
@OnEvent('order.created', { async: true })
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // handle and process "OrderCreatedEvent" event
}
```

Để sử dụng namespaces/wildcards, truyền tùy chọn `wildcard` vào phương thức `EventEmitterModule#forRoot()`. Khi namespaces/wildcards được bật, các sự kiện có thể là chuỗi (`foo.bar`) được phân tách bởi một delimiter hoặc mảng (`['foo', 'bar']`). Delimiter cũng có thể cấu hình làm thuộc tính cấu hình (`delimiter`). Với tính năng namespace được bật, bạn có thể subscribe các sự kiện bằng cách sử dụng wildcard:

```typescript
@OnEvent('order.*')
handleOrderEvents(payload: OrderCreatedEvent | OrderRemovedEvent | OrderUpdatedEvent) {
  // handle and process an event
}
```

Lưu ý rằng wildcard như vậy chỉ áp dụng cho một block. Đối số `order.*` sẽ khớp, ví dụ, các sự kiện `order.created` và `order.shipped` nhưng không phải `order.delayed.out_of_stock`. Để lắng nghe các sự kiện như vậy, sử dụng pattern `multilevel wildcard` (tức là, `**`), được mô tả trong [tài liệu](https://github.com/EventEmitter2/EventEmitter2#multi-level-wildcards) của `EventEmitter2`.

Với pattern này, bạn có thể, ví dụ, tạo một event listener bắt tất cả các sự kiện.

```typescript
@OnEvent('**')
handleEverything(payload: any) {
  // handle and process an event
}
```

> info **Gợi ý** Class `EventEmitter2` cung cấp một số phương thức hữu ích để tương tác với các sự kiện, như `waitFor` và `onAny`. Bạn có thể đọc thêm về chúng [tại đây](https://github.com/EventEmitter2/EventEmitter2).

#### Ngăn chặn mất sự kiện

Các sự kiện được kích hoạt trước hoặc trong lifecycle hook `onApplicationBootstrap` - chẳng hạn như từ các constructor module hoặc phương thức `onModuleInit` - có thể bị bỏ lỡ vì `EventSubscribersLoader` có thể chưa hoàn tất việc thiết lập các listener.

Để tránh vấn đề này, bạn có thể sử dụng phương thức `waitUntilReady` của `EventEmitterReadinessWatcher`, trả về một promise giải quyết sau khi tất cả các listener đã được đăng ký. Phương thức này có thể được gọi trong lifecycle hook `onApplicationBootstrap` của một module để đảm bảo rằng tất cả các sự kiện được capture đúng cách.

```typescript
await this.eventEmitterReadinessWatcher.waitUntilReady();
this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({ orderId: 1, payload: {} }),
);
```

> info **Lưu ý** Điều này chỉ cần thiết cho các sự kiện được phát ra trước khi lifecycle hook `onApplicationBootstrap` hoàn tất.

#### Ví dụ

Một ví dụ hoạt động có sẵn [tại đây](https://github.com/nestjs/nest/tree/master/sample/30-event-emitter).