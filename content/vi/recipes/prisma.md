### Prisma

[Prisma](https://www.prisma.io) là một ORM [mã nguồn mở](https://github.com/prisma/prisma) cho Node.js và TypeScript. Nó được sử dụng như một **sự thay thế** cho việc viết SQL thuần, hoặc sử dụng một công cụ truy cập database khác như các trình xây dựng truy vấn SQL (như [knex.js](https://knexjs.org/)) hoặc ORMs (như [TypeORM](https://typeorm.io/) và [Sequelize](https://sequelize.org/)). Prisma hiện hỗ trợ PostgreSQL, MySQL, SQL Server, SQLite, MongoDB và CockroachDB ([Preview](https://www.prisma.io/docs/orm/reference/supported-databases)).

Mặc dù Prisma có thể được sử dụng với JavaScript thuần, nó chấp nhận TypeScript và cung cấp mức độ type-safety vượt xa các đảm bảo của các ORMs khác trong hệ sinh thái TypeScript. Bạn có thể tìm thấy so sánh chuyên sâu về các đảm bảo type-safety của Prisma và TypeORM [tại đây](https://www.prisma.io/docs/orm/more/comparisons/prisma-and-typeorm#type-safety).

> info **Lưu ý** Nếu bạn muốn có tổng quan nhanh về cách Prisma hoạt động, bạn có thể làm theo [Quickstart](https://www.prisma.io/docs/getting-started/prisma-orm/quickstart/prisma-postgres) hoặc đọc [Introduction](https://www.prisma.io/docs/orm/overview/introduction/what-is-prisma) trong [tài liệu](https://www.prisma.io/docs). Ngoài ra còn có các ví dụ sẵn sàng để chạy cho [REST](https://github.com/prisma/prisma-examples/tree/b53fad046a6d55f0090ddce9fd17ec3f9b95cab3/orm/nest) và [GraphQL](https://github.com/prisma/prisma-examples/tree/b53fad046a6d55f0090ddce9fd17ec3f9b95cab3/orm/nest-graphql) trong repo [`prisma-examples`](https://github.com/prisma/prisma-examples/).

#### Bắt đầu

Trong hướng dẫn này, bạn sẽ học cách bắt đầu với NestJS và Prisma từ đầu. Bạn sẽ xây dựng một ứng dụng NestJS mẫu với REST API có thể đọc và ghi dữ liệu trong một database.

Để mục đích của hướng dẫn này, bạn sẽ sử dụng database [SQLite](https://sqlite.org/) để lưu chi phí thiết lập một database server. Lưu ý rằng bạn vẫn có thể làm theo hướng dẫn này, ngay cả khi bạn đang sử dụng PostgreSQL hoặc MySQL – bạn sẽ nhận được hướng dẫn bổ sung để sử dụng các database này ở những nơi thích hợp.

> info **Lưu ý** Nếu bạn đã có một dự án hiện có và xem xét việc chuyển sang Prisma, bạn có thể làm theo hướng dẫn cho [thêm Prisma vào một dự án hiện có](https://www.prisma.io/docs/getting-started/setup-prisma/add-to-existing-project-typescript-postgres). Nếu bạn đang chuyển từ TypeORM, bạn có thể đọc hướng dẫn [Chuyển từ TypeORM sang Prisma](https://www.prisma.io/docs/guides/migrate-from-typeorm).

#### Tạo dự án NestJS của bạn

Để bắt đầu, cài đặt NestJS CLI và tạo khung ứng dụng của bạn với các lệnh sau:

```bash
$ npm install -g @nestjs/cli
$ nest new hello-prisma
```

Xem trang [First steps](https://docs.nestjs.com/first-steps) để tìm hiểu thêm về các file dự án được tạo bởi lệnh này. Lưu ý thêm rằng bây giờ bạn có thể chạy `npm start` để khởi động ứng dụng của bạn. REST API chạy tại `http://localhost:3000/` hiện phục vụ một route đơn lẻ được triển khai trong `src/app.controller.ts`. Trong suốt hướng dẫn này, bạn sẽ triển khai các routes bổ sung để lưu trữ và truy xuất dữ liệu về _users_ và _posts_.

#### Thiết lập Prisma

Bắt đầu bằng cách cài đặt Prisma CLI như một dependency phát triển trong dự án của bạn:

```bash
$ cd hello-prisma
$ npm install prisma --save-dev
```

Trong các bước sau, chúng ta sẽ sử dụng [Prisma CLI](https://www.prisma.io/docs/orm/tools/prisma-cli). Là một phương pháp hay nhất, được khuyến nghị gọi CLI theo cục bộ bằng cách thêm tiền tố `npx`:

```bash
$ npx prisma
```

<details><summary>Mở rộng nếu bạn đang sử dụng Yarn</summary>

Nếu bạn đang sử dụng Yarn, thì bạn có thể cài đặt Prisma CLI như sau:

```bash
$ yarn add prisma --dev
```

Khi đã cài đặt, bạn có thể gọi nó bằng cách thêm tiền tố `yarn`:

```bash
$ yarn prisma
```

</details>

Bây giờ tạo thiết lập Prisma ban đầu của bạn sử dụng lệnh `init` của Prisma CLI:

```bash
$ npx prisma init
```

Lệnh này tạo một thư mục `prisma` mới với nội dung sau:

- `schema.prisma`: Chỉ định kết nối database của bạn và chứa schema database
- `prisma.config.ts`: Một file cấu hình cho các dự án của bạn
- `.env`: Một file [dotenv](https://github.com/motdotla/dotenv), thường được sử dụng để lưu trữ credentials database của bạn trong một nhóm các biến môi trường

#### Thiết lập đường dẫn đầu ra của generator

Chỉ định đường dẫn `output` của bạn cho Prisma client được tạo ra bằng cách truyền `--output ../src/generated/prisma` trong prisma init, hoặc trực tiếp trong Prisma schema của bạn:

```groovy
generator client {
  provider        = "prisma-client"
  output          = "../src/generated/prisma"
}
```

#### Cấu hình định dạng module

Thiết lập `moduleFormat` trong generator thành `cjs`:

```groovy
generator client {
  provider        = "prisma-client"
  output          = "../src/generated/prisma"
  moduleFormat    = "cjs"
}
```

> info **Lưu ý** Cấu hình `moduleFormat` là bắt buộc vì Prisma v7 được vận chuyển như một ES module theo mặc định, không hoạt động với thiết lập CommonJS của NestJS. Thiết lập `moduleFormat` thành `cjs` buộc Prisma tạo một module CommonJS thay vì ESM.

#### Thiết lập kết nối database

Kết nối database của bạn được cấu hình trong block `datasource` trong file `schema.prisma` của bạn. Theo mặc định nó được thiết lập thành `postgresql`, nhưng vì bạn đang sử dụng database SQLite trong hướng dẫn này bạn cần điều chỉnh trường `provider` của block `datasource` thành `sqlite`:

```groovy
datasource db {
  provider = "sqlite"
}

generator client {
  provider      = "prisma-client"
  output        = "../src/generated/prisma"
  moduleFormat  = "cjs"
}
```

Bây giờ, mở `.env` và điều chỉnh biến môi trường `DATABASE_URL` để trông như sau:

```bash
DATABASE_URL="file:./dev.db"
```

Hãy đảm bảo bạn có [ConfigModule](https://docs.nestjs.com/techniques/configuration) được cấu hình, nếu không biến `DATABASE_URL` sẽ không được lấy từ `.env`.

Database SQLite là các file đơn giản; không cần server để sử dụng database SQLite. Vì vậy thay vì cấu hình một URL kết nối với một _host_ và _port_, bạn có thể chỉ trỏ nó đến một file cục bộ mà trong trường hợp này được gọi là `dev.db`. File này sẽ được tạo trong bước tiếp theo.

<details><summary>Mở rộng nếu bạn đang sử dụng PostgreSQL, MySQL, MsSQL hoặc Azure SQL</summary>

Với PostgreSQL và MySQL, bạn cần cấu hình URL kết nối để trỏ đến _database server_. Bạn có thể tìm hiểu thêm về định dạng URL kết nối cần thiết [tại đây](https://www.prisma.io/docs/orm/reference/connection-urls).

**PostgreSQL**

Nếu bạn đang sử dụng PostgreSQL, bạn phải điều chỉnh các file `schema.prisma` và `.env` như sau:

**`schema.prisma`**

```groovy
datasource db {
  provider = "postgresql"
}

generator client {
  provider      = "prisma-client"
  output          = "../src/generated/prisma"
  moduleFormat  = "cjs"
}
```

**`.env`**

```bash
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=SCHEMA"
```

Thay thế các placeholders được viết bằng chữ in hoa bằng credentials database của bạn. Lưu ý rằng nếu bạn không chắc chắn phải cung cấp gì cho placeholder `SCHEMA`, rất có thể là giá trị mặc định `public`:

```bash
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=public"
```

Nếu bạn muốn tìm hiểu cách thiết lập database PostgreSQL, bạn có thể làm theo hướng dẫn này về [thiết lập database PostgreSQL miễn phí trên Heroku](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1).

**MySQL**

Nếu bạn đang sử dụng MySQL, bạn phải điều chỉnh các file `schema.prisma` và `.env` như sau:

**`schema.prisma`**

```groovy
datasource db {
  provider = "mysql"
}

generator client {
  provider      = "prisma-client"
  output          = "../src/generated/prisma"
  moduleFormat  = "cjs"
}
```

**`.env`**

```bash
DATABASE_URL="mysql://USER:PASSWORD@HOST:PORT/DATABASE"
```

Thay thế các placeholders được viết bằng chữ in hoa bằng credentials database của bạn.

**Microsoft SQL Server / Azure SQL Server**

Nếu bạn đang sử dụng Microsoft SQL Server hoặc Azure SQL Server, bạn phải điều chỉnh các file `schema.prisma` và `.env` như sau:

**`schema.prisma`**

```groovy
datasource db {
  provider = "sqlserver"
}

generator client {
  provider      = "prisma-client"
  output          = "../src/generated/prisma"
  moduleFormat  = "cjs"
}
```

**`.env`**

Thay thế các placeholders được viết bằng chữ in hoa bằng credentials database của bạn. Lưu ý rằng nếu bạn không chắc chắn phải cung cấp gì cho placeholder `encrypt`, rất có thể là giá trị mặc định `true`:

```bash
DATABASE_URL="sqlserver://HOST:PORT;database=DATABASE;user=USER;password=PASSWORD;encrypt=true"
```

</details>

#### Tạo hai bảng database với Prisma Migrate

Trong phần này, bạn sẽ tạo hai bảng mới trong database của bạn sử dụng [Prisma Migrate](https://www.prisma.io/docs/orm/prisma-migrate/getting-started). Prisma Migrate tạo ra các file migration SQL cho định nghĩa mô hình dữ liệu khai báo của bạn trong Prisma schema. Các file migration này hoàn toàn có thể tùy chỉnh để bạn có thể cấu hình bất kỳ tính năng bổ sung nào của database cơ bản hoặc bao gồm các lệnh bổ sung, ví dụ cho seeding.

Thêm hai mô hình sau vào file `schema.prisma` của bạn:

```groovy
model User {
  id    Int     @default(autoincrement()) @id
  email String  @unique
  name  String?
  posts Post[]
}

model Post {
  id        Int      @default(autoincrement()) @id
  title     String
  content   String?
  published Boolean? @default(false)
  author    User?    @relation(fields: [authorId], references: [id])
  authorId  Int?
}
```

Với các mô hình Prisma của bạn có chỗ, bạn có thể tạo các file migration SQL của mình và chạy chúng trên database. Chạy các lệnh sau trong terminal của bạn:

```bash
$ npx prisma migrate dev --name init
```

Lệnh `prisma migrate dev` này tạo ra các file SQL và chạy chúng trực tiếp trên database. Trong trường hợp này, các file migration sau đã được tạo trong thư mục `prisma` hiện có:

```bash
$ tree prisma
prisma
├── dev.db
├── migrations
│   └── 20201207100915_init
│       └── migration.sql
└── schema.prisma
```

<details><summary>Mở rộng để xem các câu lệnh SQL được tạo</summary>

Các bảng sau đã được tạo trong database SQLite của bạn:

```sql
-- CreateTable
CREATE TABLE "User" (
    "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    "email" TEXT NOT NULL,
    "name" TEXT
);

-- CreateTable
CREATE TABLE "Post" (
    "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    "title" TEXT NOT NULL,
    "content" TEXT,
    "published" BOOLEAN DEFAULT false,
    "authorId" INTEGER,

    FOREIGN KEY ("authorId") REFERENCES "User"("id") ON DELETE SET NULL ON UPDATE CASCADE
);

-- CreateIndex
CREATE UNIQUE INDEX "User.email_unique" ON "User"("email");
```

</details>

#### Cài đặt và tạo Prisma Client

Prisma Client là một database client type-safe được _tạo_ từ định nghĩa mô hình Prisma của bạn. Vì cách tiếp cận này, Prisma Client có thể expose các hoạt động [CRUD](https://www.prisma.io/docs/orm/prisma-client/queries/crud) được _tùy chỉnh_ cụ thể cho các mô hình của bạn.

Để cài đặt Prisma Client trong dự án của bạn, chạy lệnh sau trong terminal của bạn:

```bash
$ npm install @prisma/client
```

Khi đã cài đặt, bạn có thể chạy lệnh generate để tạo các types và Client cần thiết cho dự án của bạn. Nếu bất kỳ thay đổi nào được thực hiện cho schema của bạn, bạn sẽ cần chạy lại lệnh `generate` để giữ các types đó đồng bộ.

```bash
$ npx prisma generate
```

Ngoài Prisma Client, bạn cũng cần một adapter driver cho loại database bạn đang làm việc. Đối với SQLite, bạn có thể cài đặt driver `@prisma/adapter-better-sqlite3`.

```bash
npm install @prisma/adapter-better-sqlite3
```

<details> <summary>Mở rộng nếu bạn đang sử dụng PostgreSQL, MySQL, MsSQL, hoặc AzureSQL</summary>

- Đối với PostgreSQL

```bash
npm install @prisma/adapter-pg
```

- Đối với MySQL, MsSQL, AzureSQL:

```bash
npm install @prisma/adapter-mariadb
```

</details>

#### Sử dụng Prisma Client trong các dịch vụ NestJS của bạn

Bây giờ bạn có thể gửi các truy vấn database với Prisma Client. Nếu bạn muốn tìm hiểu thêm về việc xây dựng các truy vấn với Prisma Client, hãy xem [tài liệu API](https://www.prisma.io/docs/orm/reference/prisma-client-reference).

Khi thiết lập ứng dụng NestJS của bạn, bạn sẽ muốn trừu tượng hóa API Prisma Client cho các truy vấn database trong một service. Để bắt đầu, bạn có thể tạo một `PrismaService` mới lo việc khởi tạo `PrismaClient` và kết nối với database của bạn.

Bên trong thư mục `src`, tạo một file mới được gọi là `prisma.service.ts` và thêm code sau vào đó:

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaClient } from './generated/prisma/client';
import { PrismaBetterSqlite3 } from '@prisma/adapter-better-sqlite3';

@Injectable()
export class PrismaService extends PrismaClient {
  constructor() {
    const adapter = new PrismaBetterSqlite3({ url: process.env.DATABASE_URL });
    super({ adapter });
  }
}
```

Tiếp theo, bạn có thể viết các services mà bạn có thể sử dụng để thực hiện các cuộc gọi database cho các mô hình `User` và `Post` từ Prisma schema của bạn.

Vẫn bên trong thư mục `src`, tạo một file mới được gọi là `user.service.ts` và thêm code sau vào đó:

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { User, Prisma } from 'generated/prisma';

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  async user(
    userWhereUniqueInput: Prisma.UserWhereUniqueInput,
  ): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: userWhereUniqueInput,
    });
  }

  async users(params: {
    skip?: number;
    take?: number;
    cursor?: Prisma.UserWhereUniqueInput;
    where?: Prisma.UserWhereInput;
    orderBy?: Prisma.UserOrderByWithRelationInput;
  }): Promise<User[]> {
    const { skip, take, cursor, where, orderBy } = params;
    return this.prisma.user.findMany({
      skip,
      take,
      cursor,
      where,
      orderBy,
    });
  }

  async createUser(data: Prisma.UserCreateInput): Promise<User> {
    return this.prisma.user.create({
      data,
    });
  }

  async updateUser(params: {
    where: Prisma.UserWhereUniqueInput;
    data: Prisma.UserUpdateInput;
  }): Promise<User> {
    const { where, data } = params;
    return this.prisma.user.update({
      data,
      where,
    });
  }

  async deleteUser(where: Prisma.UserWhereUniqueInput): Promise<User> {
    return this.prisma.user.delete({
      where,
    });
  }
}
```

Lưu ý cách bạn đang sử dụng các types được tạo bởi Prisma Client để đảm bảo rằng các methods được expose bởi service của bạn được gõ đúng. Do đó bạn tiết kiệm boilerplate của việc gõ các mô hình của bạn và tạo các file interface hoặc DTO bổ sung.

Bây giờ làm điều tương tự cho mô hình `Post`.

Vẫn bên trong thư mục `src`, tạo một file mới được gọi là `post.service.ts` và thêm code sau vào đó:

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { Post, Prisma } from 'generated/prisma';

@Injectable()
export class PostsService {
  constructor(private prisma: PrismaService) {}

  async post(
    postWhereUniqueInput: Prisma.PostWhereUniqueInput,
  ): Promise<Post | null> {
    return this.prisma.post.findUnique({
      where: postWhereUniqueInput,
    });
  }

  async posts(params: {
    skip?: number;
    take?: number;
    cursor?: Prisma.PostWhereUniqueInput;
    where?: Prisma.PostWhereInput;
    orderBy?: Prisma.PostOrderByWithRelationInput;
  }): Promise<Post[]> {
    const { skip, take, cursor, where, orderBy } = params;
    return this.prisma.post.findMany({
      skip,
      take,
      cursor,
      where,
      orderBy,
    });
  }

  async createPost(data: Prisma.PostCreateInput): Promise<Post> {
    return this.prisma.post.create({
      data,
    });
  }

  async updatePost(params: {
    where: Prisma.PostWhereUniqueInput;
    data: Prisma.PostUpdateInput;
  }): Promise<Post> {
    const { data, where } = params;
    return this.prisma.post.update({
      data,
      where,
    });
  }

  async deletePost(where: Prisma.PostWhereUniqueInput): Promise<Post> {
    return this.prisma.post.delete({
      where,
    });
  }
}
```

`UsersService` và `PostsService` của bạn hiện bao bọc các truy vấn CRUD có sẵn trong Prisma Client. Trong một ứng dụng thực tế, service cũng sẽ là nơi để thêm logic kinh doanh vào ứng dụng của bạn. Ví dụ, bạn có thể có một phương thức được gọi là `updatePassword` bên trong `UsersService` sẽ chịu trách nhiệm cập nhật password của người dùng.

Hãy nhớ đăng ký các services mới trong app module.

##### Triển khai các routes REST API của bạn trong app controller chính

Cuối cùng, bạn sẽ sử dụng các services bạn đã tạo trong các phần trước để triển khai các routes khác nhau của ứng dụng của bạn. Để mục đích của hướng dẫn này, bạn sẽ đặt tất cả các routes của mình vào class `AppController` đã tồn tại.

Thay thế nội dung của file `app.controller.ts` bằng code sau:

```typescript
import {
  Controller,
  Get,
  Param,
  Post,
  Body,
  Put,
  Delete,
} from '@nestjs/common';
import { UsersService } from './user.service';
import { PostsService } from './post.service';
import { User as UserModel, Post as PostModel } from 'generated/prisma';

@Controller()
export class AppController {
  constructor(
    private readonly userService: UsersService,
    private readonly postService: PostsService,
  ) {}

  @Get('post/:id')
  async getPostById(@Param('id') id: string): Promise<PostModel> {
    return this.postService.post({ id: Number(id) });
  }

  @Get('feed')
  async getPublishedPosts(): Promise<PostModel[]> {
    return this.postService.posts({
      where: { published: true },
    });
  }

  @Get('filtered-posts/:searchString')
  async getFilteredPosts(
    @Param('searchString') searchString: string,
  ): Promise<PostModel[]> {
    return this.postService.posts({
      where: {
        OR: [
          {
            title: { contains: searchString },
          },
          {
            content: { contains: searchString },
          },
        ],
      },
    });
  }

  @Post('post')
  async createDraft(
    @Body() postData: { title: string; content?: string; authorEmail: string },
  ): Promise<PostModel> {
    const { title, content, authorEmail } = postData;
    return this.postService.createPost({
      title,
      content,
      author: {
        connect: { email: authorEmail },
      },
    });
  }

  @Post('user')
  async signupUser(
    @Body() userData: { name?: string; email: string },
  ): Promise<UserModel> {
    return this.userService.createUser(userData);
  }

  @Put('publish/:id')
  async publishPost(@Param('id') id: string): Promise<PostModel> {
    return this.postService.updatePost({
      where: { id: Number(id) },
      data: { published: true },
    });
  }

  @Delete('post/:id')
  async deletePost(@Param('id') id: string): Promise<PostModel> {
    return this.postService.deletePost({ id: Number(id) });
  }
}
```

Controller này triển khai các routes sau:

###### `GET`

- `/post/:id`: Lấy một bài đăng duy nhất theo `id` của nó
- `/feed`: Lấy tất cả các bài đăng _đã xuất bản_
- `/filter-posts/:searchString`: Lọc bài đăng theo `title` hoặc `content`

###### `POST`

- `/post`: Tạo một bài đăng mới
  - Body:
    - `title: String` (bắt buộc): Tiêu đề của bài đăng
    - `content: String` (tùy chọn): Nội dung của bài đăng
    - `authorEmail: String` (bắt buộc): Email của người dùng tạo bài đăng
- `/user`: Tạo một người dùng mới
  - Body:
    - `email: String` (bắt buộc): Địa chỉ email của người dùng
    - `name: String` (tùy chọn): Tên của người dùng

###### `PUT`

- `/publish/:id`: Xuất bản một bài đăng theo `id` của nó

###### `DELETE`

- `/post/:id`: Xóa một bài đăng theo `id` của nó

#### Tóm tắt

Trong hướng dẫn này, bạn đã học cách sử dụng Prisma cùng với NestJS để triển khai một REST API. Controller triển khai các routes của API đang gọi một `PrismaService` mà lần lượt sử dụng Prisma Client để gửi các truy vấn đến một database để đáp ứng các nhu cầu dữ liệu của các yêu cầu đến.

Nếu bạn muốn tìm hiểu thêm về việc sử dụng NestJS với Prisma, hãy chắc chắn kiểm tra các tài nguyên sau:

- [NestJS & Prisma](https://www.prisma.io/nestjs)
- [Các dự án ví dụ sẵn sàng để chạy cho REST & GraphQL](https://github.com/prisma/prisma-examples/)
- [Bộ khởi đầu sẵn sàng cho production](https://github.com/notiz-dev/nestjs-prisma-starter#instructions)
- [Video: Truy cập Database sử dụng NestJS với Prisma (5min)](https://www.youtube.com/watch?v=UlVJ340UEuk&ab_channel=Prisma) bởi [Marc Stammerjohann](https://github.com/marcjulian)