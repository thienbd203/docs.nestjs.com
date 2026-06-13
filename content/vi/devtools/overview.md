### Tổng quan

> info **Gợi ý** Chương này bao gồm tích hợp Nest Devtools với framework Nest. Nếu bạn đang tìm kiếm ứng dụng Devtools, hãy truy cập [Devtools](https://devtools.nestjs.com).

Để bắt đầu debug ứng dụng local của bạn, mở file `main.ts` và đảm bảo đặt thuộc tính `snapshot` thành `true` trong object tùy chọn ứng dụng, như sau:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    snapshot: true,
  });
  await app.listen(process.env.PORT ?? 3000);
}
```

Điều này sẽ hướng dẫn framework thu thập metadata cần thiết để Nest Devtools có thể trực quan hóa đồ thị ứng dụng của bạn.

Tiếp theo, hãy cài đặt dependency cần thiết:

```bash
$ npm i @nestjs/devtools-integration
```

> warning **Cảnh báo** Nếu bạn đang sử dụng package `@nestjs/graphql` trong ứng dụng, hãy đảm bảo cài đặt phiên bản mới nhất (`npm i @nestjs/graphql@11`).

Với dependency này, hãy mở file `app.module.ts` và import `DevtoolsModule` mà chúng ta vừa cài đặt:

```typescript
@Module({
  imports: [
    DevtoolsModule.register({
      http: process.env.NODE_ENV !== 'production',
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

> warning **Cảnh báo** Lý do chúng ta kiểm tra biến môi trường `NODE_ENV` ở đây là bạn không bao giờ nên sử dụng module này trong production!

Khi `DevtoolsModule` đã được import và ứng dụng của bạn đang chạy (`npm run start:dev`), bạn có thể điều hướng đến URL [Devtools](https://devtools.nestjs.com) và xem đồ thị đã được introspect.

<figure><img src="/assets/devtools/modules-graph.png" /></figure>

> info **Gợi ý** Như bạn có thể thấy trên ảnh chụp màn hình ở trên, mỗi module kết nối với `InternalCoreModule`. `InternalCoreModule` là một module toàn cục luôn được import vào module gốc. Vì nó được đăng ký sebagai một node toàn cục, Nest tự động tạo các cạnh giữa tất cả các module và node `InternalCoreModule`. Bây giờ, nếu bạn muốn ẩn các module toàn cục khỏi đồ thị, bạn có thể sử dụng checkbox "**Hide global modules**" (trong sidebar).

Như vậy chúng ta có thể thấy, `DevtoolsModule` làm cho ứng dụng của bạn expose một HTTP server bổ sung (trên cổng 8000) mà ứng dụng Devtools sẽ sử dụng để introspect ứng dụng của bạn.

Để kiểm tra lại mọi thứ hoạt động như mong đợi, hãy đổi chế độ xem đồ thị thành "Classes". Bạn sẽ thấy màn hình sau:

<figure><img src="/assets/devtools/classes-graph.png" /></figure>

Để tập trung vào một node cụ thể, nhấp vào hình chữ nhật và đồ thị sẽ hiển thị cửa sổ popup với nút **"Focus"**. Bạn cũng có thể sử dụng thanh tìm kiếm (nằm ở sidebar) để tìm một node cụ thể.

> info **Gợi ý** Nếu bạn nhấp vào nút **Inspect**, ứng dụng sẽ đưa bạn đến trang `/debug` với node cụ thể đó được chọn.

<figure><img src="/assets/devtools/node-popup.png" /></figure>

> info **Gợi ý** Để xuất đồ thị thành hình ảnh, nhấp vào nút **Export as PNG** ở góc phải của đồ thị.

Sử dụng các controls form nằm ở sidebar (bên trái), bạn có thể kiểm soát khoảng cách của các cạnh, ví dụ để trực quan hóa một cây con cụ thể của ứng dụng:

<figure><img src="/assets/devtools/subtree-view.png" /></figure>

Điều này đặc biệt hữu ích khi bạn có **nhà phát triển mới** trong nhóm và bạn muốn cho họ thấy ứng dụng của bạn được cấu trúc như thế nào. Bạn cũng có thể sử dụng tính năng này để trực quan hóa một module cụ thể (ví dụ `TasksModule`) và tất cả các dependency của nó, điều này rất hữu ích khi bạn đang chia nhỏ một ứng dụng lớn thành các module nhỏ hơn (ví dụ, các micro-service riêng lẻ).

Bạn có thể xem video này để thấy tính năng **Graph Explorer** trong hành động:

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/bW8V-ssfnvM"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### Điều tra lỗi "Cannot resolve dependency"

> info **Ghi chú** Tính năng này được hỗ trợ cho `@nestjs/core` >= `v9.3.10`.

Có lẽ thông báo lỗi phổ biến nhất mà bạn có thể đã thấy là về việc Nest không thể giải quyết các dependency của một provider. Sử dụng Nest Devtools, bạn có thể dễ dàng xác định vấn đề và học cách giải quyết nó.

Đầu tiên, mở file `main.ts` và cập nhật lời gọi `bootstrap()`, như sau:

```typescript
bootstrap().catch((err) => {
  fs.writeFileSync('graph.json', PartialGraphHost.toString() ?? '');
  process.exit(1);
});
```

Ngoài ra, hãy đảm bảo đặt `abortOnError` thành `false`:

```typescript
const app = await NestFactory.create(AppModule, {
  snapshot: true,
  abortOnError: false, // <--- NÀY
});
```

Bây giờ mỗi khi ứng dụng của bạn không thể bootstrap do lỗi **"Cannot resolve dependency"**, bạn sẽ tìm thấy file `graph.json` (đại diện cho một đồ thị một phần) trong thư mục gốc. Sau đó bạn có thể kéo & thả file này vào Devtools (đảm bảo chuyển chế độ hiện tại từ "Interactive" sang "Preview"):

<figure><img src="/assets/devtools/drag-and-drop.png" /></figure>

Sau khi tải lên thành công, bạn sẽ thấy đồ thị & cửa sổ dialog sau:

<figure><img src="/assets/devtools/partial-graph-modules-view.png" /></figure>

Như bạn có thể thấy, `TasksModule` được highlight là module chúng ta nên xem xét. Ngoài ra, trong cửa sổ dialog bạn có thể thấy một số hướng dẫn về cách sửa vấn đề này.

Nếu chúng ta chuyển sang chế độ xem "Classes", đây là những gì chúng ta sẽ thấy:

<figure><img src="/assets/devtools/partial-graph-classes-view.png" /></figure>

Đồ thị này minh họa rằng `DiagnosticsService` mà chúng ta muốn inject vào `TasksService` không được tìm thấy trong ngữ cảnh của module `TasksModule`, và chúng ta có lẽ chỉ cần import `DiagnosticsModule` vào module `TasksModule` để sửa lỗi này!

#### Routes explorer

Khi bạn điều hướng đến trang **Routes explorer**, bạn sẽ thấy tất cả các entrypoint đã đăng ký:

<figure><img src="/assets/devtools/routes.png" /></figure>

> info **Gợi ý** Trang này không chỉ hiển thị các route HTTP, mà còn tất cả các entrypoint khác (ví dụ WebSockets, gRPC, GraphQL resolvers, v.v.).

Các entrypoint được nhóm theo controller host của chúng. Bạn cũng có thể sử dụng thanh tìm kiếm để tìm một entrypoint cụ thể.

Nếu bạn nhấp vào một entrypoint cụ thể, **một đồ thị luồng** sẽ được hiển thị. Đồ thị này hiển thị luồng thực thi của entrypoint (ví dụ guards, interceptors, pipes, v.v. được bind đến route này). Điều này đặc biệt hữu ích khi bạn muốn hiểu chu kỳ request/response trông như thế nào cho một route cụ thể, hoặc khi khắc phục sự cố tại sao một guard/interceptor/pipe cụ thể không được thực thi.

#### Sandbox

Để thực thi code JavaScript ngay lập tức & tương tác với ứng dụng của bạn theo thời gian thực, điều hướng đến trang **Sandbox**:

<figure><img src="/assets/devtools/sandbox.png" /></figure>

Playground có thể được sử dụng để test và debug các API endpoint theo **thời gian thực**, cho phép các nhà phát triển nhanh chóng xác định và sửa các vấn đề mà không cần sử dụng, ví dụ, một HTTP client. Chúng ta cũng có thể bỏ qua lớp xác thực, vì vậy chúng ta không còn cần bước thêm đó của việc đăng nhập, hoặc thậm chí một tài khoản người dùng đặc biệt cho mục đích test. Đối với các ứng dụng driven bởi sự kiện, chúng ta cũng có thể kích hoạt sự kiện trực tiếp từ playground, và xem ứng dụng phản ứng như thế nào với chúng.

Bất cứ thứ gì được ghi xuống đều được chuyển đến console của playground, vì vậy chúng ta có thể dễ dàng xem những gì đang diễn ra.

Chỉ cần thực thi code **ngay lập tức** và xem kết quả ngay lập tức, mà không cần rebuild ứng dụng và restart server.

<figure><img src="/assets/devtools/sandbox-table.png" /></figure>

> info **Gợi ý** Để hiển thị đẹp một mảng các đối tượng, sử dụng hàm `console.table()` (hoặc chỉ `table()`).

Bạn có thể xem video này để thấy tính năng **Interactive Playground** trong hành động:

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/liSxEN_VXKM"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### Bootstrap performance analyzer

Để xem danh sách tất cả các node class (controllers, providers, enhancers, v.v.) và thời gian khởi tạo tương ứng của chúng, điều hướng đến trang **Bootstrap performance**:

<figure><img src="/assets/devtools/bootstrap-performance.png" /></figure>

Trang này đặc biệt hữu ích khi bạn muốn xác định các phần chậm nhất của quy trình bootstrap ứng dụng (ví dụ khi bạn muốn tối ưu hóa thời gian khởi động ứng dụng điều này rất quan trọng cho, ví dụ, môi trường serverless).

#### Audit

Để xem audit được tạo tự động - các lỗi/cảnh báo/gợi ý mà ứng dụng đưa ra khi phân tích đồ thị đã serialize của bạn, điều hướng đến trang **Audit**:

<figure><img src="/assets/devtools/audit.png" /></figure>

> info **Gợi ý** Ảnh chụp màn hình ở trên không hiển thị tất cả các quy tắc audit có sẵn.

Trang này rất hữu ích khi bạn muốn xác định các vấn đề tiềm ẩn trong ứng dụng của bạn.

#### Xem trước tệp tĩnh

Để lưu một đồ thị đã serialize vào một tệp, sử dụng code sau:

```typescript
await app.listen(process.env.PORT ?? 3000); // HOẶC await app.init()
fs.writeFileSync('./graph.json', app.get(SerializedGraph).toString());
```

> info **Gợi ý** `SerializedGraph` được xuất từ package `@nestjs/core`.

Sau đó bạn có thể kéo và thả/tải lên tệp này:

<figure><img src="/assets/devtools/drag-and-drop.png" /></figure>

Điều này hữu ích khi bạn muốn chia sẻ đồ thị của mình với người khác (ví dụ, đồng nghiệp), hoặc khi bạn muốn phân tích nó ngoại tuyến.