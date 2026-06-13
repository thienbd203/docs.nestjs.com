### HTTPS

Để tạo một ứng dụng sử dụng giao thức HTTPS, đặt thuộc tính `httpsOptions` trong đối tượng tùy chọn được truyền đến phương thức `create()` của class `NestFactory`:

```typescript
const httpsOptions = {
  key: fs.readFileSync('./secrets/private-key.pem'),
  cert: fs.readFileSync('./secrets/public-certificate.pem'),
};
const app = await NestFactory.create(AppModule, {
  httpsOptions,
});
await app.listen(process.env.PORT ?? 3000);
```

Nếu bạn sử dụng `FastifyAdapter`, tạo ứng dụng như sau:

```typescript
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter({ https: httpsOptions }),
);
```

#### Nhiều máy chủ đồng thời

Công thức sau đây cho thấy cách khởi tạo một ứng dụng Nest lắng nghe trên nhiều cổng (ví dụ, trên một cổng không HTTPS và một cổng HTTPS) đồng thời.

```typescript
const httpsOptions = {
  key: fs.readFileSync('./secrets/private-key.pem'),
  cert: fs.readFileSync('./secrets/public-certificate.pem'),
};

const server = express();
const app = await NestFactory.create(AppModule, new ExpressAdapter(server));
await app.init();

const httpServer = http.createServer(server).listen(3000);
const httpsServer = https.createServer(httpsOptions, server).listen(443);
```

Vì chúng ta đã gọi `http.createServer` / `https.createServer` chính mình, NestJS không đóng chúng khi gọi `app.close` / trên tín hiệu chấm dứt. Chúng ta cần làm điều này chính mình:

```typescript
@Injectable()
export class ShutdownObserver implements OnApplicationShutdown {
  private httpServers: http.Server[] = [];

  public addHttpServer(server: http.Server): void {
    this.httpServers.push(server);
  }

  public async onApplicationShutdown(): Promise<void> {
    await Promise.all(
      this.httpServers.map(
        (server) =>
          new Promise((resolve, reject) => {
            server.close((error) => {
              if (error) {
                reject(error);
              } else {
                resolve(null);
              }
            });
          }),
      ),
    );
  }
}

const shutdownObserver = app.get(ShutdownObserver);
shutdownObserver.addHttpServer(httpServer);
shutdownObserver.addHttpServer(httpsServer);
```

> info **Gợi ý** `ExpressAdapter` được nhập từ gói `@nestjs/platform-express`. Các gói `http` và `https` là các gói Node.js gốc.

> **Cảnh báo** Công thức này không hoạt động với [GraphQL Subscriptions](/graphql/subscriptions).