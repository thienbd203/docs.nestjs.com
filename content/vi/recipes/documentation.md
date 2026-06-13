### Tài liệu

**Compodoc** là một công cụ tài liệu cho các ứng dụng Angular. Vì Nest và Angular chia sẻ cấu trúc dự án và mã tương tự, **Compodoc** hoạt động với các ứng dụng Nest cũng tốt.

#### Thiết lập

Thiết lập Compodoc trong một dự án Nest hiện có rất đơn giản. Bắt đầu bằng cách thêm dev-dependency với lệnh sau trong terminal OS của bạn:

```bash
$ npm i -D @compodoc/compodoc
```

#### Tạo

Tạo tài liệu dự án sử dụng lệnh sau (npm 6 được yêu cầu để hỗ trợ `npx`). Xem [tài liệu chính thức](https://compodoc.app/guides/usage.html) để biết thêm tùy chọn.

```bash
$ npx @compodoc/compodoc -p tsconfig.json -s
```

Mở trình duyệt của bạn và điều hướng đến [http://localhost:8080](http://localhost:8080). Bạn nên thấy một dự án Nest CLI ban đầu:

<figure><img src="/assets/documentation-compodoc-1.jpg" /></figure>
<figure><img src="/assets/documentation-compodoc-2.jpg" /></figure>

#### Đóng góp

Bạn có thể tham gia và đóng góp cho dự án Compodoc [ở đây](https://github.com/compodoc/compodoc).