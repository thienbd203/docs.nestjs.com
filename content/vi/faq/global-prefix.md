### Tiền tố toàn cầu

Để đặt một tiền tố cho **mọi route** được đăng ký trong một ứng dụng HTTP, sử dụng phương thức `setGlobalPrefix()` của instance `INestApplication`.

```typescript
const app = await NestFactory.create(AppModule);
app.setGlobalPrefix('v1');
```

Bạn có thể loại bỏ các route khỏi tiền tố toàn cầu sử dụng cấu trúc sau:

```typescript
app.setGlobalPrefix('v1', {
  exclude: [{ path: 'health', method: RequestMethod.GET }],
});
```

Ngoài ra, bạn có thể chỉ định route như một chuỗi (nó sẽ áp dụng cho mọi phương thức yêu cầu):

```typescript
app.setGlobalPrefix('v1', { exclude: ['cats'] });
```

> info **Gợi ý** Thuộc tính `path` hỗ trợ các tham số wildcard sử dụng gói [path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters). Lưu ý: điều này không chấp nhận dấu sao wildcard `*`. Thay vào đó, bạn phải sử dụng tham số (`:param`) hoặc wildcard được đặt tên (`*splat`).