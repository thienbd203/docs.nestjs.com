### Circular dependency

Một circular dependency xảy ra khi hai classes phụ thuộc vào nhau. Ví dụ, class A cần class B, và class B cũng cần class A. Circular dependencies có thể phát sinh trong Nest giữa modules và giữa providers.

Mặc dù circular dependencies nên được tránh khi có thể, bạn không thể luôn làm như vậy. Trong những trường hợp như vậy, Nest cho phép resolving circular dependencies giữa providers theo hai cách. Trong chương này, chúng tôi mô tả sử dụng **forward referencing** như một kỹ thuật, và sử dụng class **ModuleRef** để retrieve một provider instance từ DI container như một cách khác.

Chúng tôi cũng mô tả resolving circular dependencies giữa modules.

> warning **Cảnh báo** Một circular dependency cũng có thể được gây ra khi sử dụng "barrel files"/index.ts files để nhóm imports. Barrel files nên được bỏ qua khi nói đến module/provider classes. Ví dụ, barrel files không nên được sử dụng khi importing files trong cùng một directory với barrel file, nghĩa là `cats/cats.controller` không nên import `cats` để import file `cats/cats.service`. Để biết thêm chi tiết vui lòng xem GitHub issue này.

#### Forward reference

Một **forward reference** cho phép Nest tham chiếu đến các classes chưa được định nghĩa sử dụng utility function `forwardRef()`. Ví dụ, nếu `CatsService` và `CommonService` phụ thuộc vào nhau, cả hai phía của mối quan hệ có thể sử dụng `@Inject()` và utility `forwardRef()` để resolve circular dependency. Nếu không Nest sẽ không instantiate chúng vì tất cả metadata thiết yếu sẽ không có sẵn. Đây là một ví dụ:

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    @Inject(forwardRef(() => CommonService))
    private commonService: CommonService,
  ) {}
}
@@switch
@Injectable()
@Dependencies(forwardRef(() => CommonService))
export class CatsService {
  constructor(commonService) {
    this.commonService = commonService;
  }
}
```

> info **Gợi ý** Function `forwardRef()` được import từ package `@nestjs/common`.

Điều đó bao phủ một phía của mối quan hệ. Bây giờ hãy làm điều tương tự với `CommonService`:

```typescript
@@filename(common.service)
@Injectable()
export class CommonService {
  constructor(
    @Inject(forwardRef(() => CatsService))
    private catsService: CatsService,
  ) {}
}
@@switch
@Injectable()
@Dependencies(forwardRef(() => CatsService))
export class CommonService {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

> warning **Cảnh báo** Thứ tự của instantiation là không thể xác định. Hãy đảm bảo code của bạn không phụ thuộc vào constructor nào được gọi trước. Có circular dependencies phụ thuộc vào providers với `Scope.REQUEST` có thể dẫn đến undefined dependencies. Thông tin thêm có sẵn ở đây.

#### ModuleRef class alternative

Một thay thế cho việc sử dụng `forwardRef()` là refactor code của bạn và sử dụng class `ModuleRef` để retrieve một provider trên một phía của mối quan hệ circular (nếu không). Tìm hiểu thêm về utility class `ModuleRef` ở đây.

#### Module forward reference

Để resolve circular dependencies giữa modules, sử dụng cùng utility function `forwardRef()` trên cả hai phía của module association. Ví dụ:

```typescript
@@filename(common.module)
@Module({
  imports: [forwardRef(() => CatsModule)],
})
export class CommonModule {}
```

Điều đó bao phủ một phía của mối quan hệ. Bây giờ hãy làm điều tương tự với `CatsModule`:

```typescript
@@filename(cats.module)
@Module({
  imports: [forwardRef(() => CommonModule)],
})
export class CatsModule {}
```