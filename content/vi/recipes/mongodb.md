### MongoDB (Mongoose)

> **Cảnh báo** Trong bài viết này, bạn sẽ học cách tạo một `DatabaseModule` dựa trên gói **Mongoose** từ đầu sử dụng các thành phần tùy chỉnh. Hậu quả là, giải pháp này chứa nhiều chi phí mà bạn có thể bỏ qua sử dụng gói `@nestjs/mongoose` chuyên dụng sẵn sàng để sử dụng có sẵn out-of-the-box. Để tìm hiểu thêm, xem [ở đây](/techniques/mongodb).

[Mongoose](https://mongoosejs.com) là công cụ mô hình hóa đối tượng [MongoDB](https://www.mongodb.org/) phổ biến nhất.

#### Bắt đầu

Để bắt đầu cuộc phiêu lưu với thư viện này chúng ta phải cài đặt tất cả các phụ thuộc cần thiết:

```typescript
$ npm install --save mongoose
```

Bước đầu tiên chúng ta cần làm là thiết lập kết nối với cơ sở dữ liệu của chúng ta sử dụng hàm `connect()`. Hàm `connect()` trả về một `Promise`, và do đó chúng ta phải tạo một [provider async](/fundamentals/async-components).

```typescript
@@filename(database.providers)
import * as mongoose from 'mongoose';

export const databaseProviders = [
  {
    provide: 'DATABASE_CONNECTION',
    useFactory: (): Promise<typeof mongoose> =>
      mongoose.connect('mongodb://localhost/nest'),
  },
];
@@switch
import * as mongoose from 'mongoose';

export const databaseProviders = [
  {
    provide: 'DATABASE_CONNECTION',
    useFactory: () => mongoose.connect('mongodb://localhost/nest'),
  },
];
```

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

Bây giờ chúng ta có thể inject đối tượng `Connection` sử dụng decorator `@Inject()`. Mỗi class phụ thuộc vào provider async `Connection` sẽ đợi cho đến khi một `Promise` được giải quyết.

#### Injection Model

Với Mongoose, mọi thứ đều có nguồn gốc từ một [Schema](https://mongoosejs.com/docs/guide.html). Hãy định nghĩa `CatSchema`:

```typescript
@@filename(schemas/cat.schema)
import * as mongoose from 'mongoose';

export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,
  breed: String,
});
```

`CatSchema` thuộc về thư mục `cats`. Thư mục này đại diện cho `CatsModule`.

Bây giờ là lúc để tạo một provider **Model**:

```typescript
@@filename(cats.providers)
import { Connection } from 'mongoose';
import { CatSchema } from './schemas/cat.schema';

export const catsProviders = [
  {
    provide: 'CAT_MODEL',
    useFactory: (connection: Connection) => connection.model('Cat', CatSchema),
    inject: ['DATABASE_CONNECTION'],
  },
];
@@switch
import { CatSchema } from './schemas/cat.schema';

export const catsProviders = [
  {
    provide: 'CAT_MODEL',
    useFactory: (connection) => connection.model('Cat', CatSchema),
    inject: ['DATABASE_CONNECTION'],
  },
];
```

> warning **Cảnh báo** Trong các ứng dụng thực tế bạn nên tránh **chuỗi ma thuật**. Cả `CAT_MODEL` và `DATABASE_CONNECTION` nên được giữ trong file `constants.ts` riêng biệt.

Bây giờ chúng ta có thể inject `CAT_MODEL` vào `CatsService` sử dụng decorator `@Inject()`:

```typescript
@@filename(cats.service)
import { Model } from 'mongoose';
import { Injectable, Inject } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(
    @Inject('CAT_MODEL')
    private catModel: Model<Cat>,
  ) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';

@Injectable()
@Dependencies('CAT_MODEL')
export class CatsService {
  constructor(catModel) {
    this.catModel = catModel;
  }

  async create(createCatDto) {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll() {
    return this.catModel.find().exec();
  }
}
```

Trong ví dụ trên chúng ta đã sử dụng interface `Cat`. Interface này mở rộng `Document` từ gói mongoose:

```typescript
import { Document } from 'mongoose';

export interface Cat extends Document {
  readonly name: string;
  readonly age: number;
  readonly breed: string;
}
```

Kết nối cơ sở dữ liệu là **bất đồng bộ**, nhưng Nest làm cho quá trình này hoàn toàn vô hình cho người dùng cuối. Class `CatModel` đang đợi kết nối db, và `CatsService` bị chậm trễ cho đến khi model sẵn sàng để sử dụng. Toàn bộ ứng dụng có thể bắt đầu khi mỗi class được khởi tạo.

Dưới đây là `CatsModule` cuối cùng:

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { catsProviders } from './cats.providers';
import { DatabaseModule } from '../database/database.module';

@Module({
  imports: [DatabaseModule],
  controllers: [CatsController],
  providers: [
    CatsService,
    ...catsProviders,
  ],
})
export class CatsModule {}
```

> info **Gợi ý** Đừng quên nhập `CatsModule` vào `AppModule` gốc.

#### Ví dụ

Một ví dụ hoạt động có sẵn [ở đây](https://github.com/nestjs/nest/tree/master/sample/14-mongoose-base).