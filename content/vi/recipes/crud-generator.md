### CRUD generator (chỉ TypeScript)

Trong suốt vòng đời của một dự án, khi chúng ta xây dựng các tính năng mới, chúng ta thường cần thêm các tài nguyên mới vào ứng dụng của mình. Các tài nguyên này thường yêu cầu nhiều hoạt động lặp lại mà chúng ta phải lặp lại mỗi khi chúng ta định nghĩa một tài nguyên mới.

#### Giới thiệu

Hãy tưởng tượng một kịch bản thực tế, nơi chúng ta cần expose các endpoint CRUD cho 2 thực thể, hãy nói là các thực thể **User** và **Product**.
Theo các thực tiễn tốt nhất, đối với mỗi thực thể chúng ta sẽ phải thực hiện một số hoạt động, như sau:

- Tạo một module (`nest g mo`) để giữ mã được tổ chức và thiết lập các ranh giới rõ ràng (nhóm các thành phần liên quan)
- Tạo một controller (`nest g co`) để định nghĩa các route CRUD (hoặc queries/mutations cho các ứng dụng GraphQL)
- Tạo một service (`nest g s`) để triển khai & cô lập logic kinh doanh
- Tạo một class/interface thực thể để đại diện cho hình dạng dữ liệu tài nguyên
- Tạo các Data Transfer Objects (hoặc inputs cho các ứng dụng GraphQL) để định nghĩa cách dữ liệu sẽ được gửi qua mạng

Đó là nhiều bước!

Để giúp tăng tốc quá trình lặp lại này, [Nest CLI](/cli/overview) cung cấp một generator (schematic) tự động tạo tất cả mã boilerplate để giúp chúng ta tránh làm tất cả điều này, và làm cho trải nghiệm nhà phát triển đơn giản hơn nhiều.

> info **Lưu ý** Schematic hỗ trợ tạo các controller **HTTP**, controller **Microservice**, resolvers **GraphQL** (cả code first và schema first), và Gateways **WebSocket**.

#### Tạo một tài nguyên mới

Để tạo một tài nguyên mới, chỉ cần chạy lệnh sau trong thư mục gốc của dự án của bạn:

```shell
$ nest g resource
```

Lệnh `nest g resource` không chỉ tạo tất cả các khối xây dựng NestJS (module, service, class controller) mà còn cả class thực thể, class DTO cũng như các file kiểm tra (`.spec`).

Dưới đây bạn có thể thấy file controller được tạo (cho REST API):

```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(+id);
  }

  @Patch(':id')
  update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {
    return this.usersService.update(+id, updateUserDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(+id);
  }
}
```

Ngoài ra, nó tự động tạo các placeholder cho tất cả các endpoint CRUD (routes cho REST APIs, queries và mutations cho GraphQL, message subscribes cho cả Microservices và WebSocket Gateways) - tất cả mà không cần phải nhấc ngón tay.

> warning **Lưu ý** Các class service được tạo **không** bị ràng buộc với bất kỳ **ORM (hoặc nguồn dữ liệu)** cụ thể nào. Điều này làm cho generator đủ chung để đáp ứng nhu cầu của bất kỳ dự án nào. Theo mặc định, tất cả các phương thức sẽ chứa các placeholder, cho phép bạn điền chúng với các nguồn dữ liệu cụ thể cho dự án của bạn.

Tương tự, nếu bạn muốn tạo resolvers cho một ứng dụng GraphQL, chỉ cần chọn `GraphQL (code first)` (hoặc `GraphQL (schema first)`) như lớp transport của bạn.

Trong trường hợp này, NestJS sẽ tạo một class resolver thay vì một controller REST API:

```shell
$ nest g resource users

> ? What transport layer do you use? GraphQL (code first)
> ? Would you like to generate CRUD entry points? Yes
> CREATE src/users/users.module.ts (224 bytes)
> CREATE src/users/users.resolver.spec.ts (525 bytes)
> CREATE src/users/users.resolver.ts (1109 bytes)
> CREATE src/users/users.service.spec.ts (453 bytes)
> CREATE src/users/users.service.ts (625 bytes)
> CREATE src/users/dto/create-user.input.ts (195 bytes)
> CREATE src/users/dto/update-user.input.ts (281 bytes)
> CREATE src/users/entities/user.entity.ts (187 bytes)
> UPDATE src/app.module.ts (312 bytes)
```

> info **Gợi ý** Để tránh tạo các file kiểm tra, bạn có thể truyền cờ `--no-spec`, như sau: `nest g resource users --no-spec`

Chúng ta có thể thấy dưới đây, rằng không chỉ tất cả các mutations và queries boilerplate được tạo, mà mọi thứ đều được buộc lại với nhau. Chúng ta đang sử dụng `UsersService`, Thực thể `User`, và DTO của chúng ta.

```typescript
import { Resolver, Query, Mutation, Args, Int } from '@nestjs/graphql';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { CreateUserInput } from './dto/create-user.input';
import { UpdateUserInput } from './dto/update-user.input';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly usersService: UsersService) {}

  @Mutation(() => User)
  createUser(@Args('createUserInput') createUserInput: CreateUserInput) {
    return this.usersService.create(createUserInput);
  }

  @Query(() => [User], { name: 'users' })
  findAll() {
    return this.usersService.findAll();
  }

  @Query(() => User, { name: 'user' })
  findOne(@Args('id', { type: () => Int }) id: number) {
    return this.usersService.findOne(id);
  }

  @Mutation(() => User)
  updateUser(@Args('updateUserInput') updateUserInput: UpdateUserInput) {
    return this.usersService.update(updateUserInput.id, updateUserInput);
  }

  @Mutation(() => User)
  removeUser(@Args('id', { type: () => Int }) id: number) {
    return this.usersService.remove(id);
  }
}
```