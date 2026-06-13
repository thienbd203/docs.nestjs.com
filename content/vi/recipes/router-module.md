### Module Router

> info **Gợi ý** Chương này chỉ liên quan đến các ứng dụng dựa trên HTTP.

Trong một ứng dụng HTTP (ví dụ, REST API), đường dẫn route cho một handler được xác định bằng cách nối (tùy chọn) tiền tố được khai báo cho controller (bên trong decorator `@Controller`),
và bất kỳ đường dẫn nào được chỉ định trong decorator của phương thức (ví dụ, `@Get('users')`). Bạn có thể tìm hiểu thêm về điều đó trong [phần này](/controllers#routing). Ngoài ra,
bạn có thể định nghĩa một [tiền tố toàn cầu](/faq/global-prefix) cho tất cả các route được đăng ký trong ứng dụng của bạn, hoặc bật [phiên bản](/techniques/versioning).

Ngoài ra, có các trường hợp edge khi định nghĩa một tiền tố ở cấp module (vì vậy cho tất cả các controller được đăng ký trong module đó) có thể hữu ích.
Ví dụ, hãy tưởng tượng một ứng dụng REST expose một số endpoint khác nhau được sử dụng bởi một phần cụ thể của ứng dụng của bạn gọi là "Dashboard".
Trong trường hợp như vậy, thay vì lặp lại tiền tố `/dashboard` trong mỗi controller, bạn có thể sử dụng một module tiện ích `RouterModule`, như sau:

```typescript
@Module({
  imports: [
    DashboardModule,
    RouterModule.register([
      {
        path: 'dashboard',
        module: DashboardModule,
      },
    ]),
  ],
})
export class AppModule {}
```

> info **Gợi ý** Class `RouterModule` được xuất từ gói `@nestjs/core`.

Ngoài ra, bạn có thể định nghĩa các cấu trúc phân cấp. Điều này có nghĩa là mỗi module có thể có các module `children`.
Các module con sẽ kế thừa tiền tố của cha mẹ chúng. Trong ví dụ sau, chúng ta sẽ đăng ký `AdminModule` như một module cha của `DashboardModule` và `MetricsModule`.

```typescript
@Module({
  imports: [
    AdminModule,
    DashboardModule,
    MetricsModule,
    RouterModule.register([
      {
        path: 'admin',
        module: AdminModule,
        children: [
          {
            path: 'dashboard',
            module: DashboardModule,
          },
          {
            path: 'metrics',
            module: MetricsModule,
          },
        ],
      },
    ])
  ],
});
```

> info **Gợi ý** Tính năng này nên được sử dụng rất cẩn thận, vì việc lạm dụng nó có thể làm cho mã khó bảo trì theo thời gian.

Trong ví dụ trên, bất kỳ controller nào được đăng ký bên trong `DashboardModule` sẽ có một tiền tố bổ sung `/admin/dashboard` (vì module nối các đường dẫn từ trên xuống dưới - đệ quy - cha đến con).
Tương tự, mỗi controller được định nghĩa bên trong `MetricsModule` sẽ có một tiền tố cấp module bổ sung `/admin/metrics`.