### SQL (Sequelize)

##### Chương này chỉ áp dụng cho TypeScript

> **Cảnh báo** Trong bài viết này, bạn sẽ học cách tạo một `DatabaseModule` dựa trên gói **Sequelize** từ đầu sử dụng các thành phần tùy chỉnh. Hậu quả là, kỹ thuật này chứa nhiều chi phí mà bạn có thể tránh bằng cách sử dụng gói `@nestjs/sequelize` chuyên dụng, sẵn sàng để sử dụng out-of-the-box. Để tìm hiểu thêm, xem [ở đây](/techniques/database#sequelize-integration).

[Sequelize](https://github.com/sequelize/sequelize) là một Object Relational Mapper (ORM) phổ biến được viết trong JavaScript thuần túy, nhưng có một wrapper TypeScript [sequelize-typescript](https://github.com/RobinBuschmann/sequelize-typescript) cung cấp một bộ decorator và các tiện ích khác cho sequelize cơ bản.

#### Bắt đầu

Để bắt đầu cuộc phiêu lưu với thư viện này chúng ta phải cài đặt các phụ thuộc sau:

```bash
$ npm install --save sequelize sequelize-typescript mysql2
$ npm install --save-dev @types/sequelize
```

Bước đầu tiên chúng ta cần làm là tạo một instance **Sequelize** với một đối tượng tùy chọn được truyền vào constructor. Ngoài ra, chúng ta cần thêm tất cả các model (thay thế là sử dụng thuộc tính `modelPaths`) và `sync()` các bảng cơ sở dữ liệu của chúng ta.

```typescript
@@filename(database.providers)
import { Sequelize } from 'sequelize-typescript';
import { Cat } from '../cats/cat.entity';

export const databaseProviders = [
  {
    provide: 'SEQUELIZE',
    useFactory: async () => {
      const sequelize = new Sequelize({
        dialect: 'mysql',
        host: 'localhost',
        port: 3306,
        username: 'root',
        password: 'password',
        database: 'nest',
      });
      sequelize.addModels([Cat]);
      await sequelize.sync();
      return sequelize;
    },
  },
];
```

> info **Gợi ý** Theo các thực tiễn tốt nhất, chúng ta đã khai báo provider tùy chỉnh trong file riêng biệt có hậu tố `*.providers.ts`.

Sau đó, chúng ta cần xuất các provider này để làm cho chúng **có thể truy cập** cho phần còn lại của ứng dụng.

```typescript
import { Module } from '@nestjs/common';
import { databaseProviders } from './database.providers';

@Module({
  providers: [...databaseProviders],
  exports: [...databaseProviders],
})
export class DatabaseModule {}
```

Bây giờ chúng ta có thể inject đối tượng `Sequelize` sử dụng decorator `@Inject()`. Mỗi class phụ thuộc vào provider async `Sequelize` sẽ đợi cho đến khi một `Promise` được giải quyết.

#### Injection Model

Trong [Sequelize](https://github.com/sequelize/sequelize) **Model** định nghĩa một bảng trong cơ sở dữ liệu. Các instance của class này đại diện cho một hàng cơ sở dữ liệu. Đầu tiên, chúng ta cần ít nhất một thực thể:

```typescript
@@filename(cat.entity)
import { Table, Column, Model } from 'sequelize-typescript';

@Table
export class Cat extends Model {
  @Column
  name: string;

  @Column
  age: number;

  @Column
  breed: string;
}
```

Thực thể `Cat` thuộc về thư mục `cats`. Thư mục này đại diện cho `CatsModule`. Bây giờ là lúc để tạo một provider **Repository**:

```typescript
@@filename(cats.providers)
import { Cat } from './cat.entity';

export const catsProviders = [
  {
    provide: 'CATS_REPOSITORY',
    useValue: Cat,
  },
];
```

> warning **Cảnh báo** Trong các ứng dụng thực tế bạn nên tránh **chuỗi ma thuật**. Cả `CATS_REPOSITORY` và `SEQUELIZE` nên được giữ trong file `constants.ts` riêng biệt.

Trong Sequelize, chúng ta sử dụng các phương thức tĩnh để thao tác dữ liệu, và do đó chúng ta đã tạo một **alias** ở đây.