### Asynchronous providers

Đôi khi, việc khởi động ứng dụng nên được trì hoãn cho đến khi một hoặc nhiều **asynchronous tasks** được hoàn thành. Ví dụ, bạn có thể không muốn bắt đầu chấp nhận các yêu cầu cho đến khi kết nối với database đã được thiết lập. Bạn có thể đạt được điều này sử dụng asynchronous providers.

Cú pháp cho điều này là sử dụng `async/await` với cú pháp `useFactory`. Factory trả về một `Promise`, và factory function có thể `await` các asynchronous tasks. Nest sẽ await resolution của promise trước khi instantiate bất kỳ class nào phụ thuộc vào (injects) provider đó.

```typescript
{
  provide: 'ASYNC_CONNECTION',
  useFactory: async () => {
    const connection = await createConnection(options);
    return connection;
  },
}
```

> info **Gợi ý** Tìm hiểu thêm về cú pháp custom provider ở đây.

#### Injection

Asynchronous providers được inject đến các components khác bởi các tokens của chúng, giống như bất kỳ provider nào khác. Trong ví dụ trên, bạn sẽ sử dụng cấu trúc `@Inject('ASYNC_CONNECTION')`.

#### Example

Công thức TypeORM có một ví dụ đầy đủ hơn về một asynchronous provider.