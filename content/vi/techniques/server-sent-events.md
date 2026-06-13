### Server-Sent Events

Server-Sent Events (SSE) là một công nghệ server push cho phép client nhận các cập nhật tự động từ server thông qua kết nối HTTP. Mỗi thông báo được gửi dưới dạng một khối văn bản được kết thúc bởi một cặp dòng mới (tìm hiểu thêm [tại đây](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)).

#### Sử dụng

Để bật Server-Sent events trên một route (route được đăng ký trong một **controller class**), chú thích phương thức handler với decorator `@Sse()`.

```typescript
@Sse('sse')
sse(): Observable<MessageEvent> {
  return interval(1000).pipe(map((_) => ({ data: { hello: 'world' } })));
}
```

> info **Gợi ý** Decorator `@Sse()` và interface `MessageEvent` được nhập từ `@nestjs/common`, trong khi `Observable`, `interval`, và `map` được nhập từ package `rxjs`.

> warning **Cảnh báo** Các route Server-Sent Events phải trả về một luồng `Observable`.

Trong ví dụ trên, chúng ta đã định nghĩa một route có tên `sse` sẽ cho phép chúng ta truyền tải các cập nhật thời gian thực. Các sự kiện này có thể được lắng nghe bằng cách sử dụng [EventSource API](https://developer.mozilla.org/en-US/docs/Web/API/EventSource).

Phương thức `sse` trả về một `Observable` phát ra nhiều `MessageEvent` (trong ví dụ này, nó phát ra một `MessageEvent` mới mỗi giây). Đối tượng `MessageEvent` nên tôn trọng interface sau để khớp với đặc tả:

```typescript
export interface MessageEvent {
  data: string | object;
  id?: string;
  type?: string;
  retry?: number;
}
```

Với điều này, chúng ta bây giờ có thể tạo một instance của class `EventSource` trong ứng dụng client-side của chúng ta, truyền route `/sse` (khớp với endpoint mà chúng ta đã truyền vào decorator `@Sse()` ở trên) làm đối số constructor.

Instance `EventSource` mở một kết nối persist đến một server HTTP, gửi các sự kiện ở định dạng `text/event-stream`. Kết nối vẫn mở cho đến khi được đóng bằng cách gọi `EventSource.close()`.

Sau khi kết nối được mở, các tin nhắn incoming từ server được chuyển đến code của bạn dưới dạng sự kiện. Nếu có trường sự kiện trong tin nhắn incoming, sự kiện được kích hoạt giống như giá trị trường sự kiện. Nếu không có trường sự kiện hiện diện, thì một sự kiện `message` chung được kích hoạt ([nguồn](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)).

```javascript
const eventSource = new EventSource('/sse');
eventSource.onmessage = ({ data }) => {
  console.log('New message', JSON.parse(data));
};
```

#### Ngắt kết nối client

Khi một client đóng kết nối SSE (ví dụ, `eventSource.close()`), NestJS tự động unsubscribe từ Observable được trả về, dừng luồng sự kiện và dọn dẹp bất kỳ tài nguyên liên quan nào — bao gồm bộ đếm thời gian interval trong ví dụ trên.

Để chạy logic teardown tùy chỉnh khi một client ngắt kết nối, sử dụng toán tử `finalize`:

```typescript
@Sse('sse')
sse(): Observable<MessageEvent> {
  return interval(1000).pipe(
    map((_) => ({ data: { hello: 'world' } })),
    finalize(() => console.log('Client disconnected')),
  );
}
```

> info **Gợi ý** Toán tử `finalize` (được nhập từ `rxjs`) thực thi callback của nó bất cứ khi nào Observable chấm dứt — bởi hoàn thành, lỗi, hoặc unsubscribe (bao gồm ngắt kết nối client). Điều này làm cho nó trở thành nơi thích hợp để giải phóng các tài nguyên bên ngoài như con trỏ cơ sở dữ liệu hoặc bộ xử lý file được liên kết với luồng.

#### Ví dụ

Một ví dụ hoạt động có sẵn [tại đây](https://github.com/nestjs/nest/tree/master/sample/28-sse).