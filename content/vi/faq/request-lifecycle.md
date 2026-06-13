### Vòng đời yêu cầu

Các ứng dụng Nest xử lý các yêu cầu và tạo ra các phản hồi trong một chuỗi mà chúng ta gọi là **vòng đời yêu cầu**. Với việc sử dụng middleware, pipes, guards, và interceptors, có thể khó khăn để theo dõi nơi một đoạn mã cụ thể thực thi trong vòng đời yêu cầu, đặc biệt là khi các thành phần toàn cầu, cấp controller và cấp route được đưa vào hoạt động. Nói chung, một yêu cầu chảy qua middleware đến guards, sau đó đến interceptors, sau đó đến pipes và cuối cùng quay lại interceptors trên đường trả về (khi phản hồi được tạo).

#### Middleware

Middleware được thực thi trong một chuỗi cụ thể. Trước tiên, Nest chạy middleware được ràng buộc toàn cầu (như middleware được ràng buộc với `app.use`) và sau đó nó chạy [middleware được ràng buộc module](/middleware), được xác định trên các đường dẫn. Middleware chạy tuần tự theo thứ tự chúng được ràng buộc, tương tự như cách middleware trong Express hoạt động. Trong trường hợp middleware được ràng buộc trên các module khác nhau, middleware được ràng buộc với module gốc sẽ chạy trước, và sau đó middleware sẽ chạy theo thứ tự mà các module được thêm vào mảng imports.

#### Guards

Thực thi guard bắt đầu với các guard toàn cầu, sau đó tiến đến các guard controller, và cuối cùng đến các guard route. Giống như middleware, guards chạy theo thứ tự mà chúng được ràng buộc. Ví dụ:

```typescript
@UseGuards(Guard1, Guard2)
@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @UseGuards(Guard3)
  @Get()
  getCats(): Cats[] {
    return this.catsService.getCats();
  }
}
```

`Guard1` sẽ thực thi trước `Guard2` và cả hai sẽ thực thi trước `Guard3`.

> info **Gợi ý** Khi nói về ràng buộc toàn cầu so với controller hoặc ràng buộc cục bộ, sự khác biệt là nơi guard (hoặc thành phần khác được ràng buộc). Nếu bạn đang sử dụng `app.useGlobalGuards()` hoặc cung cấp thành phần thông qua một module, nó được ràng buộc toàn cầu. Nếu không, nó được ràng buộc với một controller nếu decorator đứng trước một class controller, hoặc với một route nếu decorator đứng trước một khai báo route.

#### Interceptors

Interceptors, phần lớn, tuân theo cùng một mẫu như guards, với một điểm bắt bắt: vì interceptors trả về [RxJS Observables](https://github.com/ReactiveX/rxjs), các observables sẽ được giải quyết theo cách vào trước ra sau. Vì vậy, các yêu cầu đến sẽ đi qua việc giải quyết cấp toàn cầu, controller, route tiêu chuẩn, nhưng phía phản hồi của yêu cầu (tức là sau khi trả về từ trình xử lý phương thức controller) sẽ được giải quyết từ route đến controller đến toàn cầu. Ngoài ra, bất kỳ lỗi nào được ném bởi pipes, controllers, hoặc services có thể được đọc trong toán tử `catchError` của một interceptor.

#### Pipes

Pipes tuân theo chuỗi ràng buộc toàn cầu đến controller đến route tiêu chuẩn, với cùng vào trước ra trước liên quan đến các tham số `@UsePipes()`. Tuy nhiên, ở cấp tham số route, nếu bạn có nhiều pipe đang chạy, chúng sẽ chạy theo thứ tự của tham số cuối cùng với một pipe đến tham số đầu tiên. Điều này cũng áp dụng cho các pipe cấp route và cấp controller. Ví dụ, nếu chúng ta có controller sau:

```typescript
@UsePipes(GeneralValidationPipe)
@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @UsePipes(RouteSpecificPipe)
  @Patch(':id')
  updateCat(
    @Body() body: UpdateCatDTO,
    @Param() params: UpdateCatParams,
    @Query() query: UpdateCatQuery,
  ) {
    return this.catsService.updateCat(body, params, query);
  }
}
```

sau đó `GeneralValidationPipe` sẽ chạy cho `query`, sau đó `params`, và sau đó các đối tượng `body` trước khi chuyển sang `RouteSpecificPipe`, tuân theo cùng thứ tự. Nếu bất kỳ pipe cụ thể tham số nào được đặt, chúng sẽ chạy (lần nữa, từ tham số cuối cùng đến tham số đầu tiên) sau các pipe cấp controller và route.

#### Filters

Filters là thành phần duy nhất không giải quyết toàn cầu trước. Thay vào đó, filters giải quyết từ mức thấp nhất có thể, có nghĩa là thực thi bắt đầu với bất kỳ filter được ràng buộc route và tiến tiếp đến cấp controller, và cuối cùng đến các filter toàn cầu. Lưu ý rằng các ngoại lệ không thể được truyền từ filter sang filter; nếu một filter cấp route bắt ngoại lệ, một filter cấp controller hoặc toàn cầu không thể bắt cùng một ngoại lệ. Cách duy nhất để đạt được một hiệu ứng như vậy là sử dụng sự kế thừa giữa các filter.

> info **Gợi ý** Filters chỉ được thực thi nếu bất kỳ ngoại lệ chưa được bắt nào xảy ra trong quá trình yêu cầu. Các ngoại lệ đã được bắt, chẳng hạn như những cái được bắt với `try/catch` sẽ không kích hoạt Exception Fires. Ngay khi một ngoại lệ chưa được bắt được gặp phải, phần còn lại của vòng đời bị bỏ qua và yêu cầu bỏ thẳng đến filter.

#### Tóm tắt

Nói chung, vòng đời yêu cầu trông như sau:

1. Yêu cầu đến
2. Middleware
   - 2.1. Middleware được ràng buộc toàn cầu
   - 2.2. Middleware được ràng buộc module
3. Guards
   - 3.1 Global guards
   - 3.2 Controller guards
   - 3.3 Route guards
4. Interceptors (trước controller)
   - 4.1 Global interceptors
   - 4.2 Controller interceptors
   - 4.3 Route interceptors
5. Pipes
   - 5.1 Global pipes
   - 5.2 Controller pipes
   - 5.3 Route pipes
   - 5.4 Route parameter pipes
6. Controller (trình xử lý phương thức)
7. Service (nếu có)
8. Interceptors (sau yêu cầu)
   - 8.1 Route interceptor
   - 8.2 Controller interceptor
   - 8.3 Global interceptor
9. Exception filters
   - 9.1 route
   - 9.2 controller
   - 9.3 global
10. Phản hồi máy chủ