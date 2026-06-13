### Mongo

Nest hỗ trợ hai phương pháp để tích hợp với cơ sở dữ liệu [MongoDB](https://www.mongodb.com/). Bạn có thể sử dụng module [TypeORM](https://github.com/typeorm/typeorm) tích hợp sẵn được mô tả [ở đây](/techniques/database), có một connector cho MongoDB, hoặc sử dụng [Mongoose](https://mongoosejs.com), công cụ mô hình hóa đối tượng MongoDB phổ biến nhất. Trong chương này chúng tôi sẽ mô tả cái sau, sử dụng package chuyên dụng `@nestjs/mongoose`.

Bắt đầu bằng cách cài đặt [các dependency cần thiết](https://github.com/Automattic/mongoose):

```bash
$ npm i @nestjs/mongoose mongoose
```

Sau khi quá trình cài đặt hoàn tất, chúng ta có thể nhập `MongooseModule` vào `AppModule` gốc.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [MongooseModule.forRoot('mongodb://localhost/nest')],
})
export class AppModule {}
```

Phương thức `forRoot()` chấp nhận cùng đối tượng cấu hình như `mongoose.connect()` từ package Mongoose, như được mô tả [ở đây](https://mongoosejs.com/docs/connections.html).

#### Injection model

Với Mongoose, mọi thứ được dẫn xuất từ một [Schema](http://mongoosejs.com/docs/guide.html). Mỗi schema ánh xạ tới một collection MongoDB và định nghĩa hình dạng của các tài liệu trong collection đó. Schemas được sử dụng để định nghĩa [Models](https://mongoosejs.com/docs/models.html). Models chịu trách nhiệm tạo và đọc tài liệu từ cơ sở dữ liệu MongoDB cơ bản.

Schemas có thể được tạo với các decorator NestJS, hoặc với Mongoose thủ công. Sử dụng decorator để tạo schemas giúp giảm đáng kể mã boilerplate và cải thiện khả năng đọc mã tổng thể.

Hãy định nghĩa `CatSchema`:

```typescript
@@filename(schemas/cat.schema)
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';

export type CatDocument = HydratedDocument<Cat>;

@Schema()
export class Cat {
  @Prop()
  name: string;

  @Prop()
  age: number;

  @Prop()
  breed: string;
}

export const CatSchema = SchemaFactory.createForClass(Cat);
```

> info **Gợi ý** Lưu ý bạn cũng có thể tạo một định nghĩa schema thô bằng cách sử dụng class `DefinitionsFactory` (từ `nestjs/mongoose`). Điều này cho phép bạn sửa đổi thủ công định nghĩa schema được tạo dựa trên metadata bạn cung cấp. Điều này hữu ích cho một số trường hợp đặc biệt trong đó có thể khó đại diện cho mọi thứ bằng decorator.

Decorator `@Schema()` đánh dấu một class là một định nghĩa schema. Nó ánh xạ class `Cat` của chúng ta tới một collection MongoDB có cùng tên, nhưng với một "s" bổ sung ở cuối - vì vậy tên collection mongo cuối cùng sẽ là `cats`. Decorator này chấp nhận một đối số tùy chọn duy nhất là một đối tượng tùy chọn schema. Hãy coi nó như đối tượng bạn thường sẽ truyền làm đối số thứ hai của constructor class `mongoose.Schema` (ví dụ, `new mongoose.Schema(_, options)`). Để tìm hiểu thêm về các tùy chọn schema có sẵn, xem [chương này](https://mongoosejs.com/docs/guide.html#options).

Decorator `@Prop()` định nghĩa một thuộc tính trong tài liệu. Ví dụ, trong định nghĩa schema ở trên, chúng ta đã định nghĩa ba thuộc tính: `name`, `age`, và `breed`. Các [schema types](https://mongoosejs.com/docs/schematypes.html) cho các thuộc tính này được tự động suy ra nhờ vào khả năng metadata TypeScript (và reflection). Tuy nhiên, trong các kịch bản phức tạp hơn trong đó các kiểu không thể được suy ra ngầm định (ví dụ, mảng hoặc cấu trúc đối tượng lồng nhau), các kiểu phải được chỉ định rõ ràng, như sau:

```typescript
@Prop([String])
tags: string[];
```

Ngoài ra, decorator `@Prop()` chấp nhận một đối số tùy chọn ([đọc thêm](https://mongoosejs.com/docs/schematypes.html#schematype-options) về các tùy chọn có sẵn). Với điều này, bạn có thể chỉ định xem một thuộc tính có bắt buộc hay không, chỉ định một giá trị mặc định, hoặc đánh dấu nó là bất biến. Ví dụ:

```typescript
@Prop({ required: true })
name: string;
```

Trong trường hợp bạn muốn chỉ định quan hệ với một model khác, sau đó để populate, bạn cũng có thể sử dụng decorator `@Prop()`. Ví dụ, nếu `Cat` có `Owner` được lưu trữ trong một collection khác gọi là `owners`, thuộc tính nên có kiểu và ref. Ví dụ:

```typescript
import * as mongoose from 'mongoose';
import { Owner } from '../owners/schemas/owner.schema';

// bên trong định nghĩa class
@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })
owner: Owner;
```

Trong trường hợp có nhiều chủ sở hữu, cấu hình thuộc tính của bạn nên trông như sau:

```typescript
@Prop({ type: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' }] })
owners: Owner[];
```

Nếu bạn không có ý định luôn populate một tham chiếu đến một collection khác, hãy cân nhắc sử dụng `mongoose.Types.ObjectId` làm kiểu thay thế:

```typescript
@Prop({ type: { type: mongoose.Schema.Types.ObjectId, ref: 'Owner' } })
// Điều này đảm bảo trường không bị nhầm lẫn với một tham chiếu đã được populate
owner: mongoose.Types.ObjectId;
```

Sau đó, khi bạn cần chọn lọc populate nó sau này, bạn có thể sử dụng một hàm repository chỉ định kiểu đúng:

```typescript
import { Owner } from './schemas/owner.schema';

// ví dụ bên trong một service hoặc repository
async findAllPopulated() {
  return this.catModel.find().populate<{ owner: Owner }>("owner");
}
```

> info **Gợi ý** Nếu không có tài liệu ngoại để populate, kiểu có thể là `Owner | null`, tùy thuộc vào [cấu hình Mongoose](https://mongoosejs.com/docs/populate.html#doc-not-found) của bạn. Ngoài ra, nó có thể ném một lỗi, trong trường hợp đó kiểu sẽ là `Owner`.

Cuối cùng, định nghĩa schema **thô** cũng có thể được truyền đến decorator. Điều này hữu ích khi, ví dụ, một thuộc tính đại diện cho một đối tượng lồng nhau không được định nghĩa như một class. Để làm điều này, sử dụng hàm `raw()` từ package `@nestjs/mongoose`, như sau:

```typescript
@Prop(raw({
  firstName: { type: String },
  lastName: { type: String }
}))
details: Record<string, any>;
```

Ngoài ra, nếu bạn thích **không sử dụng decorator**, bạn có thể định nghĩa một schema thủ công. Ví dụ:

```typescript
export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,
  breed: String,
});
```

File `cat.schema` nằm trong một thư mục trong thư mục `cats`, nơi chúng ta cũng định nghĩa `CatsModule`. Trong khi bạn có thể lưu trữ các file schema ở bất cứ đâu bạn thích, chúng tôi khuyến nghị lưu trữ chúng gần các đối tượng **domain** liên quan, trong thư mục module tương ứng.

Hãy nhìn vào `CatsModule`:

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { Cat, CatSchema } from './schemas/cat.schema';

@Module({
  imports: [MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }])],
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

`MongooseModule` cung cấp phương thức `forFeature()` để cấu hình module, bao gồm định nghĩa các models nào nên được đăng ký trong phạm vi hiện tại. Nếu bạn cũng muốn sử dụng các models trong một module khác, hãy thêm MongooseModule vào phần `exports` của `CatsModule` và nhập `CatsModule` trong module khác.

Sau khi bạn đã đăng ký schema, bạn có thể inject một model `Cat` vào `CatsService` bằng cách sử dụng decorator `@InjectModel()`:

```typescript
@@filename(cats.service)
import { Model } from 'mongoose';
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Cat } from './schemas/cat.schema';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name) private catModel: Model<Cat>) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }
}
@@switch
import { Model } from 'mongoose';
import { Injectable, Dependencies } from '@nestjs/common';
import { getModelToken } from '@nestjs/mongoose';
import { Cat } from './schemas/cat.schema';

@Injectable()
@Dependencies(getModelToken(Cat.name))
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

#### Connection

Đôi khi bạn có thể cần truy cập đối tượng [Mongoose Connection](https://mongoosejs.com/docs/api.html#Connection) gốc. Ví dụ, bạn có thể muốn thực hiện các cuộc gọi API gốc trên đối tượng kết nối. Bạn có thể inject Mongoose Connection bằng cách sử dụng decorator `@InjectConnection()` như sau:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private connection: Connection) {}
}
```

#### Sessions

Để bắt đầu một session với Mongoose, được khuyến nghị để inject kết nối cơ sở dữ liệu sử dụng `@InjectConnection` thay vì gọi `mongoose.startSession()` trực tiếp. Cách tiếp cận này cho phép tích hợp tốt hơn với hệ thống dependency injection của NestJS, đảm bảo quản lý kết nối đúng cách.

Đây là một ví dụ về cách bắt đầu một session:

```typescript
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private readonly connection: Connection) {}

  async startTransaction() {
    const session = await this.connection.startSession();
    session.startTransaction();
    // Logic giao dịch của bạn ở đây
  }
}
```

Trong ví dụ này, `@InjectConnection()` được sử dụng để inject kết nối Mongoose vào service. Sau khi kết nối được inject, bạn có thể sử dụng `connection.startSession()` để bắt đầu một session mới. Session này có thể được sử dụng để quản lý các giao dịch cơ sở dữ liệu, đảm bảo các hoạt động nguyên tử qua nhiều truy vấn. Sau khi bắt đầu session, hãy nhớ commit hoặc hủy giao dịch dựa trên logic của bạn.

#### Nhiều cơ sở dữ liệu

Một số dự án yêu cầu nhiều kết nối cơ sở dữ liệu. Điều này cũng có thể đạt được với module này. Để làm việc với nhiều kết nối, trước tiên hãy tạo các kết nối. Trong trường hợp này, đặt tên kết nối trở thành **bắt buộc**.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionName: 'cats',
    }),
    MongooseModule.forRoot('mongodb://localhost/users', {
      connectionName: 'users',
    }),
  ],
})
export class AppModule {}
```

> warning **Lưu ý** Vui lòng lưu ý rằng bạn không nên có nhiều kết nối không có tên, hoặc có cùng tên, nếu không chúng sẽ bị ghi đè.

Với cấu hình này, bạn phải nói cho hàm `MongooseModule.forFeature()` biết kết nối nào nên được sử dụng.

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }], 'cats'),
  ],
})
export class CatsModule {}
```

Bạn cũng có thể inject `Connection` cho một kết nối cụ thể:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection('cats') private connection: Connection) {}
}
```

Để inject một `Connection` cụ thể vào một provider tùy chỉnh (ví dụ, factory provider), sử dụng hàm `getConnectionToken()` truyền tên của kết nối làm đối số.

```typescript
{
  provide: CatsService,
  useFactory: (catsConnection: Connection) => {
    return new CatsService(catsConnection);
  },
  inject: [getConnectionToken('cats')],
}
```

Nếu bạn chỉ đang tìm cách inject model từ một cơ sở dữ liệu được đặt tên, bạn có thể sử dụng tên kết nối làm đối số thứ hai cho decorator `@InjectModel()`.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name, 'cats') private catModel: Model<Cat>) {}
}
@@switch
@Injectable()
@Dependencies(getModelToken(Cat.name, 'cats'))
export class CatsService {
  constructor(catModel) {
    this.catModel = catModel;
  }
}
```

#### Hooks (middleware)

Middleware (cũng được gọi là pre và post hooks) là các hàm được truyền điều khiển trong quá trình thực thi các hàm bất đồng bộ. Middleware được chỉ định ở mức schema và hữu ích để viết các plugin ([nguồn](https://mongoosejs.com/docs/middleware.html)). Gọi `pre()` hoặc `post()` sau khi biên dịch một model không hoạt động trong Mongoose. Để đăng ký một hook **trước** đăng ký model, sử dụng phương thức `forFeatureAsync()` của `MongooseModule` cùng với một factory provider (tức là `useFactory`). Với kỹ thuật này, bạn có thể truy cập một đối tượng schema, sau đó sử dụng phương thức `pre()` hoặc `post()` để đăng ký một hook trên schema đó. Xem ví dụ dưới đây:

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatSchema;
          schema.pre('save', function () {
            console.log('Hello from pre save');
          });
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

Giống như các [factory providers](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) khác, hàm factory của chúng ta có thể là `async` và có thể inject dependencies thông qua `inject`.

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => {
          const schema = CatSchema;
          schema.pre('save', function() {
            console.log(
              `${configService.get('APP_NAME')}: Hello from pre save`,
            ),
          });
          return schema;
        },
        inject: [ConfigService],
      },
    ]),
  ],
})
export class AppModule {}
```

#### Plugins

Để đăng ký một [plugin](https://mongoosejs.com/docs/plugins.html) cho một schema cụ thể, sử dụng phương thức `forFeatureAsync()`.

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatSchema;
          schema.plugin(require('mongoose-autopopulate'));
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

Để đăng ký một plugin cho tất cả schemas cùng một lúc, gọi phương thức `.plugin()` của đối tượng `Connection`. Bạn nên truy cập kết nối trước khi các models được tạo; để làm điều này, sử dụng `connectionFactory`:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionFactory: (connection) => {
        connection.plugin(require('mongoose-autopopulate'));
        return connection;
      }
    }),
  ],
})
export class AppModule {}
```

#### Discriminators

[Discriminators](https://mongoosejs.com/docs/discriminators.html) là một cơ chế kế thừa schema. Chúng cho phép bạn có nhiều models với các schema chồng chéo trên cùng một collection MongoDB cơ bản.

Giả sử bạn muốn theo dõi các loại sự kiện khác nhau trong một collection duy nhất. Mỗi sự kiện sẽ có một timestamp.

```typescript
@@filename(event.schema)
@Schema({ discriminatorKey: 'kind' })
export class Event {
  @Prop({
    type: String,
    required: true,
    enum: [ClickedLinkEvent.name, SignUpEvent.name],
  })
  kind: string;

  @Prop({ type: Date, required: true })
  time: Date;
}

export const EventSchema = SchemaFactory.createForClass(Event);
```

> info **Gợi ý** Cách mongoose phân biệt các models discriminator khác nhau là bằng "discriminator key", mặc định là `__t`. Mongoose thêm một đường dẫn String gọi là `__t` vào schemas của bạn mà nó sử dụng để theo dõi discriminator mà tài liệu này là một instance của.
> Bạn cũng có thể sử dụng tùy chọn `discriminatorKey` để định nghĩa đường dẫn cho phân biệt.

Các instance `SignUpEvent` và `ClickedLinkEvent` sẽ được lưu trữ trong cùng collection như các sự kiện chung chung.

Bây giờ, hãy định nghĩa class `ClickedLinkEvent`, như sau:

```typescript
@@filename(click-link-event.schema)
@Schema()
export class ClickedLinkEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  url: string;
}

export const ClickedLinkEventSchema = SchemaFactory.createForClass(ClickedLinkEvent);
```

Và class `SignUpEvent`:

```typescript
@@filename(sign-up-event.schema)
@Schema()
export class SignUpEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  user: string;
}

export const SignUpEventSchema = SchemaFactory.createForClass(SignUpEvent);
```

Với điều này đã sẵn sàng, sử dụng tùy chọn `discriminators` để đăng ký một discriminator cho một schema cụ thể. Nó hoạt động trên cả `MongooseModule.forFeature` và `MongooseModule.forFeatureAsync`:

```typescript
@@filename(event.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: Event.name,
        schema: EventSchema,
        discriminators: [
          { name: ClickedLinkEvent.name, schema: ClickedLinkEventSchema },
          { name: SignUpEvent.name, schema: SignUpEventSchema },
        ],
      },
    ]),
  ]
})
export class EventsModule {}
```

#### Testing

Khi kiểm thử đơn vị một ứng dụng, chúng ta thường muốn tránh bất kỳ kết nối cơ sở dữ liệu nào, làm cho bộ kiểm thử của chúng ta đơn giản hơn để thiết lập và thực thi nhanh hơn. Nhưng các class của chúng ta có thể phụ thuộc vào các models được kéo từ instance kết nối. Làm thế nào để chúng ta giải quyết các class này? Giải pháp là tạo các models giả (mock).

Để làm điều này dễ dàng hơn, package `@nestjs/mongoose` expose một hàm `getModelToken()` trả về một [injection token](https://docs.nestjs.com/fundamentals/custom-providers#di-fundamentals) được chuẩn bị dựa trên một tên token. Sử dụng token này, bạn có thể dễ dàng cung cấp một triển khai giả sử dụng bất kỳ kỹ thuật [custom provider](/fundamentals/custom-providers) tiêu chuẩn nào, bao gồm `useClass`, `useValue`, và `useFactory`. Ví dụ:

```typescript
@Module({
  providers: [
    CatsService,
    {
      provide: getModelToken(Cat.name),
      useValue: catModel,
    },
  ],
})
export class CatsModule {}
```

Trong ví dụ này, một `catModel` được hardcode (instance đối tượng) sẽ được cung cấp bất cứ khi nào bất kỳ consumer nào inject một `Model<Cat>` sử dụng một decorator `@InjectModel()`.

<app-banner-courses></app-banner-courses>

#### Cấu hình async

Khi bạn cần truyền các tùy chọn module một cách bất đồng bộ thay vì tĩnh, sử dụng phương thức `forRootAsync()`. Giống như hầu hết các module động, Nest cung cấp một số kỹ thuật để xử lý cấu hình async.

Một kỹ thuật là sử dụng một hàm factory:

```typescript
MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: 'mongodb://localhost/nest',
  }),
});
```

Giống như các [factory providers](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) khác, hàm factory của chúng ta có thể là `async` và có thể inject dependencies thông qua `inject`.

```typescript
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    uri: configService.get<string>('MONGODB_URI'),
  }),
  inject: [ConfigService],
});
```

Ngoài ra, bạn có thể cấu hình `MongooseModule` sử dụng một class thay vì một factory, như được hiển thị dưới đây:

```typescript
MongooseModule.forRootAsync({
  useClass: MongooseConfigService,
});
```