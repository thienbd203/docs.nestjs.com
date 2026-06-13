### SQL (TypeORM)

##### Chương này chỉ áp dụng cho TypeScript

> **Cảnh báo** Trong bài viết này, bạn sẽ học cách tạo một `DatabaseModule` dựa trên gói **TypeORM** từ đầu sử dụng cơ chế provider tùy chỉnh. Hậu quả là, giải pháp này chứa nhiều chi phí mà bạn có thể bỏ qua bằng cách sử dụng gói `@nestjs/typeorm` chuyên dụng, sẵn sàng để sử dụng out-of-the-box. Để tìm hiểu thêm, xem [ở đây](/techniques/sql).

[TypeORM](https://github.com/typeorm/typeorm) chắc chắn là Object Relational Mapper (ORM) trưởng thành nhất có sẵn trong thế giới node.js. Vì nó được viết trong TypeScript, nó hoạt động khá tốt với framework Nest.

#### Bắt đầu

Để bắt đầu cuộc phiêu lưu với thư viện này chúng ta phải cài đặt tất cả các phụ thuộc cần thiết:

```bash
$ npm install --save typeorm mysql2
```

Bước đầu tiên chúng ta cần làm là thiết lập kết nối với cơ sở dữ liệu của chúng ta sử dụng class `new DataSource().initialize()` được nhập từ gói `typeorm`. Hàm `initialize()` trả về một `Promise`, và do đó chúng ta phải tạo một [provider async](/fundamentals/async-components).

```typescript
@@filename(database.providers)
import { DataSource } from 'typeorm';

export const databaseProviders = [
  {
    provide: 'DATA_SOURCE',
    useFactory: async () => {
      const dataSource = new DataSource({
        type: 'mysql',
        host: 'localhost',
        port: 3306,
        username: 'root',
        password: 'root',
        database: 'test',
        entities: [
            __dirname + '/../**/*.entity{.ts,.js}',
        ],
        synchronize: true,
      });

      return dataSource.initialize();
    },
  },
];
```

> warning **Cảnh báo** Đặt `synchronize: true` không nên được sử dụng trong production - nếu không bạn có thể mất dữ liệu production.

> info **Gợi ý** Theo các thực tiễn tốt nhất, chúng ta đã khai báo provider tùy chỉnh trong file riêng biệt có hậu tố `*.providers.ts`.

Sau đó, chúng ta cần xuất các provider này để làm cho chúng **có thể truy cập** cho phần còn lại của ứng dụng.

```typescript
@@filename(database.module)
import { Module } from '@nestjs/common';
import { databaseProviders } from './database.providers';

@Module({
  providers: [...databaseProviders],
  exports: [...databaseProviders],
})
export class DatabaseModule {}
```

Bây giờ chúng ta có thể inject đối tượng `DATA_SOURCE` sử dụng decorator `@Inject()`. Mỗi class phụ thuộc vào provider async `DATA_SOURCE` sẽ đợi cho đến khi một `Promise` được giải quyết.

#### Pattern Repository

[TypeORM](https://github.com/typeorm/typeorm) hỗ trợ pattern thiết kế repository, do đó mỗi thực thể có Repository của riêng. Các repository này có thể được lấy từ kết nối cơ sở dữ liệu.

Nhưng đầu tiên, chúng ta cần ít nhất một thực thể. Chúng ta sẽ tái sử dụng thực thể `Photo` từ tài liệu chính thức.

```typescript
@@filename(photo.entity)
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class Photo {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ length: 500 })
  name: string;

  @Column('text')
  description: string;

  @Column()
  filename: string;

  @Column('int')
  views: number;

  @Column()
  isPublished: boolean;
}
```

Thực thể `Photo` thuộc về thư mục `photo`. Thư mục này đại diện cho `PhotoModule`. Bây giờ, hãy tạo một provider **Repository**:

```typescript