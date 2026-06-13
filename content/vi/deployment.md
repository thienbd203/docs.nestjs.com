### Deployment

Khi bạn sẵn sàng triển khai ứng dụng NestJS của mình lên production, có các bước chính bạn có thể thực hiện để đảm bảo nó chạy càng hiệu quả càng tốt. Trong hướng dẫn này, chúng tôi sẽ khám phá các mẹo và thực hành tốt nhất thiết yếu để giúp bạn triển khai ứng dụng NestJS của mình thành công.

#### Điều kiện tiên quyết

Trước khi triển khai ứng dụng NestJS của bạn, hãy đảm bảo bạn có:

- Một ứng dụng NestJS đang hoạt động và sẵn sàng để triển khai.
- Truy cập vào một nền tảng triển khai hoặc máy chủ nơi bạn có thể lưu trữ ứng dụng của mình.
- Tất cả các biến môi trường cần thiết được thiết lập cho ứng dụng của bạn.
- Bất kỳ dịch vụ cần thiết nào, như cơ sở dữ liệu, được thiết lập và sẵn sàng hoạt động.
- Ít nhất một phiên bản LTS của Node.js được cài đặt trên nền tảng triển khai của bạn.

> info **Gợi ý** Nếu bạn đang tìm kiếm một nền tảng dựa trên đám mây để triển khai ứng dụng NestJS của mình, hãy xem [Mau](https://mau.nestjs.com/ 'Deploy Nest'), nền tảng chính thức của chúng tôi để triển khai các ứng dụng NestJS trên AWS. Với Mau, việc triển khai ứng dụng NestJS của bạn đơn giản như nhấp vào một vài nút và chạy một lệnh duy nhất:
>
> ```bash
> $ npm install -g @nestjs/mau
> $ mau deploy
> ```
>
> Sau khi triển khai hoàn tất, bạn sẽ có ứng dụng NestJS của mình chạy trên AWS trong vài giây!

#### Xây dựng ứng dụng của bạn

Để xây dựng ứng dụng NestJS của bạn, bạn cần biên dịch mã TypeScript của bạn thành JavaScript. Quá trình này tạo ra một thư mục `dist` chứa các tệp được biên dịch. Bạn có thể xây dựng ứng dụng của mình bằng cách chạy lệnh sau:

```bash
$ npm run build
```

Lệnh này thường chạy lệnh `nest build` dưới bề mặt, về cơ bản là một wrapper quanh trình biên dịch TypeScript với một số tính năng bổ sung (sao chép tài sản, v.v.). Trong trường hợp bạn có một tập lệnh xây dựng tùy chỉnh, bạn có thể chạy nó trực tiếp. Ngoài ra, đối với các mono-repo NestJS CLI, hãy đảm bảo truyền tên dự án để xây dựng làm đối số (`npm run build my-app`).

Sau khi biên dịch thành công, bạn sẽ thấy một thư mục `dist` trong thư mục gốc dự án của bạn chứa các tệp được biên dịch, với điểm vào là `main.js`. Nếu bạn có bất kỳ tệp `.ts` nào nằm trong thư mục gốc của dự án của bạn (và `tsconfig.json` của bạn được cấu hình để biên dịch chúng), chúng cũng sẽ được sao chép vào thư mục `dist`, thay đổi cấu trúc thư mục một chút (thay vì `dist/main.js`, bạn sẽ có `dist/src/main.js` vì vậy hãy nhớ điều này khi cấu hình máy chủ của bạn).

#### Môi trường production

Môi trường production của bạn là nơi ứng dụng của bạn sẽ có thể truy cập được cho người dùng bên ngoài. Điều này có thể là một nền tảng dựa trên đám mây như [AWS](https://aws.amazon.com/) (với EC2, ECS, v.v.), [Azure](https://azure.microsoft.com/), hoặc [Google Cloud](https://cloud.google.com/), hoặc thậm chí là một máy chủ chuyên dụng mà bạn quản lý, chẳng hạn như [Hetzner](https://www.hetzner.com/).

Để đơn giản hóa quá trình triển khai và tránh thiết lập thủ công, bạn có thể sử dụng một dịch vụ như [Mau](https://mau.nestjs.com/ 'Deploy Nest'), nền tảng chính thức của chúng tôi để triển khai các ứng dụng NestJS trên AWS. Để biết thêm chi tiết, hãy xem [phần này](todo).

Một số ưu điểm của việc sử dụng một **nền tảng dựa trên đám mây** hoặc dịch vụ như [Mau](https://mau.nestjs.com/ 'Deploy Nest') bao gồm:

- Khả năng mở rộng: Dễ dàng mở rộng ứng dụng của bạn khi cơ sở người dùng tăng lên.
- Bảo mật: Tận hưởng các tính năng bảo mật tích hợp sẵn và các chứng chỉ tuân thủ.
- Giám sát: Giám sát hiệu suất và sức khỏe của ứng dụng của bạn theo thời gian thực.
- Độ tin cậy: Đảm bảo ứng dụng của bạn luôn khả dụng với các đảm bảo thời gian hoạt động cao.

Mặt khác, các nền tảng dựa trên đám mây thường đắt hơn tự lưu trữ, và bạn có thể ít kiểm soát hơn đối với cơ sở hạ tầng cơ bản. VPS đơn giản có thể là một lựa chọn tốt nếu bạn đang tìm kiếm giải pháp tiết kiệm chi phí hơn và có chuyên môn kỹ thuật để quản lý máy chủ của riêng mình, nhưng hãy nhớ rằng bạn sẽ cần xử lý các nhiệm vụ như bảo trì máy chủ, bảo mật và sao lưu thủ công.

#### NODE_ENV=production

Trong khi về mặt kỹ thuật không có sự khác biệt giữa development và production trong Node.js và NestJS, đó là một thực hành tốt để đặt biến môi trường `NODE_ENV` thành `production` khi chạy ứng dụng của bạn trong môi trường production, vì một số thư viện trong hệ sinh thái có thể hoạt động khác nhau dựa trên biến này (ví dụ: bật hoặc tắt đầu ra debug, v.v.).

Bạn có thể đặt biến môi trường `NODE_ENV` khi khởi động ứng dụng của bạn như sau:

```bash
$ NODE_ENV=production node dist/main.js
```

Hoặc chỉ cần đặt nó trong bảng điều khiển của nhà cung cấp đám mây/Mau của bạn.

#### Chạy ứng dụng của bạn

Để chạy ứng dụng NestJS của bạn trong production, chỉ cần sử dụng lệnh sau:

```bash
$ node dist/main.js # Điều chỉnh điều này dựa trên vị trí điểm vào của bạn
```

Lệnh này khởi động ứng dụng của bạn, sẽ lắng nghe trên cổng được chỉ định (thường là `3000` theo mặc định). Đảm bảo rằng điều này khớp với cổng bạn đã cấu hình trong ứng dụng của bạn.

Ngoài ra, bạn có thể sử dụng lệnh `nest start`. Lệnh này là một wrapper xung quanh `node dist/main.js`, nhưng nó có một sự khác biệt chính: nó tự động chạy `nest build` trước khi khởi động ứng dụng, vì vậy bạn không cần thực thi thủ công `npm run build`.

#### Health checks

Health checks là thiết yếu để giám sát sức khỏe và trạng thái của ứng dụng NestJS của bạn trong production. Bằng cách thiết lập một endpoint health check, bạn có thể thường xuyên xác minh rằng ứng dụng của bạn đang chạy như mong đợi và phản hồi các vấn đề trước khi chúng trở nên quan trọng.

Trong NestJS, bạn có thể dễ dàng triển khai health checks sử dụng package **@nestjs/terminus**, cung cấp một công cụ mạnh mẽ để thêm health checks, bao gồm các kết nối cơ sở dữ liệu, dịch vụ bên ngoài và các checks tùy chỉnh.

Xem [hướng dẫn này](/recipes/terminus) để tìm hiểu cách triển khai health checks trong ứng dụng NestJS của bạn, và đảm bảo ứng dụng của bạn luôn được giám sát và phản hồi.

#### Logging

Logging là thiết yếu cho bất kỳ ứng dụng sẵn sàng cho production nào. Nó giúp theo dõi lỗi, giám sát hành vi và khắc phục sự cố. Trong NestJS, bạn có thể dễ dàng quản lý logging với logger tích hợp hoặc chọn các thư viện bên ngoài nếu bạn cần các tính năng nâng cao hơn.

Thực hành tốt nhất cho logging:

- Ghi nhật ký lỗi, không phải Exceptions: Tập trung vào việc ghi nhật ký các thông báo lỗi chi tiết để tăng tốc debug và giải quyết vấn đề.
- Tránh dữ liệu nhạy cảm: Không bao giờ ghi nhật ký thông tin nhạy cảm như mật khẩu hoặc tokens để bảo vệ bảo mật.
- Sử dụng ID tương quan: Trong các hệ thống phân tán, bao gồm các định danh duy nhất (như ID tương quan) trong logs của bạn để theo dõi các yêu cầu qua các dịch vụ khác nhau.
- Sử dụng các mức độ log: Phân loại logs theo mức độ nghiêm trọng (ví dụ: `info`, `warn`, `error`) và vô hiệu hóa logs debug hoặc verbose trong production để giảm nhiễu.

> info **Gợi ý** Nếu bạn đang sử dụng [AWS](https://aws.amazon.com/) (với [Mau](https://mau.nestjs.com/ 'Deploy Nest') hoặc trực tiếp), hãy cân nhắc logging JSON để làm cho việc phân tích cú pháp và phân tích logs của bạn dễ dàng hơn.

Đối với các ứng dụng phân tán, sử dụng một dịch vụ logging tập trung như ElasticSearch, Loggly hoặc Datadog có thể cực kỳ hữu ích. Các công cụ này cung cấp các tính năng mạnh mẽ như tổng hợp log, tìm kiếm và trực quan hóa, làm cho việc giám sát và phân tích hiệu suất và hành vi của ứng dụng của bạn dễ dàng hơn.

#### Mở rộng theo chiều dọc hoặc ngang

Mở rộng ứng dụng NestJS của bạn một cách hiệu quả là rất quan trọng để xử lý lưu lượng truy cập tăng lên và đảm bảo hiệu suất tối ưu. Có hai chiến lược chính để mở rộng: **mở rộng theo chiều dọc** và **mở rộng theo chiều ngang**. Hiểu rõ các cách tiếp cận này sẽ giúp bạn thiết kế ứng dụng của mình để quản lý tải hiệu quả.

**Mở rộng theo chiều dọc**, thường được gọi là "mở rộng lên" liên quan đến việc tăng tài nguyên của một máy chủ duy nhất để nâng cao hiệu suất của nó. Điều này có thể có nghĩa là thêm nhiều CPU, RAM hoặc bộ nhớ hơn vào máy hiện có của bạn. Dưới đây là một số điểm chính để xem xét:

- Tính đơn giản: Mở rộng theo chiều dốc thường đơn giản hơn để triển khai vì bạn chỉ cần nâng cấp máy chủ hiện có của mình thay vì quản lý nhiều instances.
- Hạn chế: Có giới hạn vật lý về mức độ bạn có thể mở rộng một máy duy nhất. Khi bạn đạt đến công suất tối đa, bạn có thể cần xem xét các tùy chọn khác.
- Hiệu quả chi phí: Đối với các ứng dụng có lưu lượng truy cập vừa phải, mở rộng theo chiều dọc có thể tiết kiệm chi phí, vì nó giảm nhu cầu về cơ sở hạ tầng bổ sung.

Ví dụ: Nếu ứng dụng NestJS của bạn được lưu trữ trên một máy ảo và bạn nhận thấy nó chạy chậm trong giờ cao điểm, bạn có thể nâng cấp máy ảo của mình lên một instance lớn hơn với nhiều tài nguyên hơn. Để nâng cấp máy ảo của bạn, chỉ cần điều hướng đến bảng điều khiển của nhà cung cấp hiện tại của bạn và chọn một loại instance lớn hơn.

**Mở rộng theo chiều ngang**, hoặc "mở rộng ra" liên quan đến việc thêm nhiều máy chủ hoặc instances để phân phối tải. Chiến lược này được sử dụng rộng rãi trong các môi trường đám mây và là thiết yếu cho các ứng dụng mong đợi lưu lượng truy cập cao. Dưới đây là các lợi ích và cân nhắc:

- Tăng dung lượng: Bằng cách thêm nhiều instances của ứng dụng của bạn, bạn có thể xử lý một số lượng lớn hơn người dùng đồng thời mà không làm giảm hiệu suất.
- Dư thừa: Mở rộng theo chiều ngang cung cấp dư thừa, vì sự thất bại của một máy chủ sẽ không làm sập toàn bộ ứng dụng của bạn. Lưu lượng truy cập có thể được phân phối lại giữa các máy chủ còn lại.
- Cân bằng tải: Để quản lý nhiều instances hiệu quả, sử dụng các bộ cân bằng tải (như Nginx hoặc AWS Elastic Load Balancing) để phân phối lưu lượng truy cập đến đều nhau trên các máy chủ của bạn.

Ví dụ: Đối với một ứng dụng NestJS trải qua lưu lượng truy cập cao, bạn có thể triển khai nhiều instances của ứng dụng của bạn trong một môi trường đám mây và sử dụng một bộ cân bằng tải để định tuyến các yêu cầu, đảm bảo rằng không có instance nào trở thành nút thắt cổ chai.

Quá trình này đơn giản với các công nghệ containerization như [Docker](https://www.docker.com/) và các nền tảng điều phối container như [Kubernetes](https://kubernetes.io/). Ngoài ra, bạn có thể tận dụng các bộ cân bằng tải cụ thể của đám mây như [AWS Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/) hoặc [Azure Load Balancer](https://azure.microsoft.com/en-us/services/load-balancer/) để phân phối lưu lượng truy cập trên các instances ứng dụng của bạn.

> info **Gợi ý** [Mau](https://mau.nestjs.com/ 'Deploy Nest') cung cấp hỗ trợ tích hợp cho mở rộng theo chiều ngang trên AWS, cho phép bạn dễ dàng triển khai nhiều instances của ứng dụng NestJS của bạn và quản lý chúng chỉ với một vài cú nhấp chuột.

#### Một số mẹo khác

Có một vài mẹo khác cần ghi nhớ khi triển khai ứng dụng NestJS của bạn:

- **Bảo mật**: Đảm bảo ứng dụng của bạn an toàn và được bảo vệ khỏi các mối đe dọa phổ biến như SQL injection, XSS, v.v. Xem danh mục "Security" để biết thêm chi tiết.
- **Giám sát**: Sử dụng các công cụ giám sát như [Prometheus](https://prometheus.io/) hoặc [New Relic](https://newrelic.com/) để theo dõi hiệu suất và sức khỏe của ứng dụng của bạn. Nếu bạn đang sử dụng nhà cung cấp đám mây/Mau, họ có thể cung cấp các dịch vụ giám sát tích hợp (như [AWS CloudWatch](https://aws.amazon.com/cloudwatch/) v.v.)
- **Không hardcode các biến môi trường**: Tránh hardcode thông tin nhạy cảm như khóa API, mật khẩu hoặc tokens trong mã của bạn. Sử dụng các biến môi trường hoặc một trình quản lý bí mật để lưu trữ và truy cập các giá trị này một cách an toàn.
- **Sao lưu**: Sao lưu dữ liệu thường xuyên để ngăn mất dữ liệu trong trường hợp sự cố.
- **Tự động hóa triển khai**: Sử dụng các đường ống CI/CD để tự động hóa quy trình triển khai của bạn và đảm bảo tính nhất quán trên các môi trường.
- **Giới hạn tốc độ**: Triển khai giới hạn tốc độ để ngăn chặn lạm dụng và bảo vệ ứng dụng của bạn khỏi các cuộc tấn công DDoS. Xem chương [Giới hạn tốc độ](/security/rate-limiting) để biết thêm chi tiết, hoặc sử dụng một dịch vụ như [AWS WAF](https://aws.amazon.com/waf/) để bảo vệ nâng cao.

#### Docker hóa ứng dụng của bạn

[Docker](https://www.docker.com/) là một nền tảng sử dụng containerization để cho phép các nhà phát triển đóng gói ứng dụng cùng với các dependencies của chúng thành một đơn vị chuẩn hóa được gọi là container. Các container nhẹ, di động và cô lập, làm cho chúng lý tưởng để triển khai các ứng dụng trong nhiều môi trường khác nhau, từ development cục bộ đến production.

Lợi ích của việc Docker hóa ứng dụng NestJS của bạn:

- Tính nhất quán: Docker đảm bảo rằng ứng dụng của bạn chạy theo cùng một cách trên bất kỳ máy nào, loại bỏ vấn đề "nó hoạt động trên máy của tôi".
- Cô lập: Mỗi container chạy trong môi trường cô lập của nó, ngăn chặn xung đột giữa các dependencies.
- Khả năng mở rộng: Docker giúp bạn dễ dàng mở rộng ứng dụng của mình bằng cách chạy nhiều containers trên các máy hoặc instances đám mây khác nhau.
- Tính di động: Các container có thể dễ dàng di chuyển giữa các môi trường, làm cho việc triển khai ứng dụng của bạn trên các nền tảng khác nhau trở nên đơn giản.

Để cài đặt Docker, hãy làm theo hướng dẫn trên [trang web chính thức](https://www.docker.com/get-started). Sau khi Docker được cài đặt, bạn có thể tạo một `Dockerfile` trong dự án NestJS của bạn để định nghĩa các bước để xây dựng hình ảnh container của bạn.

`Dockerfile` là một tệp văn bản chứa các hướng dẫn Docker sử dụng để xây dựng hình ảnh container của bạn.

Dưới đây là một Dockerfile mẫu cho một ứng dụng NestJS:

```bash
# Sử dụng hình ảnh Node.js chính thức làm hình ảnh cơ sở
FROM node:20

# Đặt thư mục làm việc bên trong container
WORKDIR /usr/src/app

# Sao chép package.json và package-lock.json vào thư mục làm việc
COPY package*.json ./

# Cài đặt các dependencies ứng dụng
RUN npm install

# Sao chép phần còn lại của các tệp ứng dụng
COPY . .

# Xây dựng ứng dụng NestJS
RUN npm run build

# Tiếp lộ cổng ứng dụng
EXPOSE 3000

# Lệnh để chạy ứng dụng
CMD ["node", "dist/main"]
```

> info **Gợi ý** Đảm bảo thay thế `node:20` bằng phiên bản Node.js thích hợp mà bạn đang sử dụng trong dự án của mình. Bạn có thể tìm thấy các hình ảnh Docker Node.js có sẵn trên [repository Docker Hub chính thức](https://hub.docker.com/_/node).

Đây là một Dockerfile cơ bản thiết lập môi trường Node.js, cài đặt các dependencies ứng dụng, xây dựng ứng dụng NestJS và chạy nó. Bạn có thể tùy chỉnh tệp này dựa trên yêu cầu dự án của bạn (ví dụ: sử dụng các hình ảnh cơ sở khác nhau, tối ưu hóa quy trình xây dựng, chỉ cài đặt các dependencies production, v.v.).

Hãy cũng tạo một tệp `.dockerignore` để chỉ định các tệp và thư mục mà Docker nên bỏ qua khi xây dựng hình ảnh. Tạo một tệp `.dockerignore` trong thư mục gốc dự án của bạn:

```bash
node_modules
dist
*.log
*.md
.git
```

Tệp này đảm bảo rằng các tệp không cần thiết không được bao gồm trong hình ảnh container, giữ cho nó nhẹ. Bây giờ bạn đã thiết lập Dockerfile, bạn có thể xây dựng hình ảnh Docker của mình. Mở terminal của bạn, điều hướng đến thư mục dự án của bạn và chạy lệnh sau:

```bash
docker build -t my-nestjs-app .
```

Trong lệnh này:

- `-t my-nestjs-app`: Gắn thẻ hình ảnh với tên `my-nestjs-app`.
- `.`: Chỉ định thư mục hiện tại làm bối cảnh xây dựng.

Sau khi xây dựng hình ảnh, bạn có thể chạy nó như một container. Thực thi lệnh sau:

```bash
docker run -p 3000:3000 my-nestjs-app
```

Trong lệnh này:

- `-p 3000:3000`: Ánh xạ cổng 3000 trên máy chủ của bạn đến cổng 3000 trong container.
- `my-nestjs-app`: Chỉ định hình ảnh để chạy.

Ứng dụng NestJS của bạn bây giờ nên đang chạy bên trong một container Docker.

Nếu bạn muốn triển khai hình ảnh Docker của mình đến một nhà cung cấp đám mây hoặc chia sẻ nó với những người khác, bạn sẽ cần đẩy nó đến một registry Docker (như [Docker Hub](https://hub.docker.com/), [AWS ECR](https://aws.amazon.com/ecr/), hoặc [Google Container Registry](https://cloud.google.com/container-registry)).

Khi bạn quyết định một registry, bạn có thể đẩy hình ảnh của mình bằng cách làm theo các bước sau:

```bash
docker login # Đăng nhập vào registry Docker của bạn
docker tag my-nestjs-app your-dockerhub-username/my-nestjs-app # Gắn thẻ hình ảnh của bạn
docker push your-dockerhub-username/my-nestjs-app # Đẩy hình ảnh của bạn
```

Thay thế `your-dockerhub-username` bằng tên người dùng Docker Hub của bạn hoặc URL registry thích hợp. Sau khi đẩy hình ảnh của bạn, bạn có thể kéo nó trên bất kỳ máy nào và chạy nó như một container.

Các nhà cung cấp đám mây như AWS, Azure và Google Cloud cung cấp các dịch vụ container được quản lý đơn giản hóa việc triển khai và quản lý các containers ở quy mô lớn. Các dịch vụ này cung cấp các tính năng như tự động mở rộng, cân bằng tải và giám sát, làm cho việc chạy ứng dụng NestJS của bạn trong production dễ dàng hơn.

#### Triển khai dễ dàng với Mau

[Mau](https://mau.nestjs.com/ 'Deploy Nest') là nền tảng chính thức của chúng tôi để triển khai các ứng dụng NestJS trên [AWS](https://aws.amazon.com/). Nếu bạn chưa sẵn sàng để quản lý cơ sở hạ tầng của mình thủ công (hoặc chỉ muốn tiết kiệm thời gian), Mau là giải pháp hoàn hảo cho bạn.

Với Mau, việc cung cấp và duy trì cơ sở hạ tầng của bạn đơn giản như nhấp chỉ một vài nút. Mau được thiết kế để đơn giản và trực quan, vì vậy bạn có thể tập trung vào việc xây dựng các ứng dụng của mình và không phải lo lắng về cơ sở hạ tầng cơ bản. Dưới bề mặt, chúng tôi sử dụng **Amazon Web Services** để cung cấp cho bạn một nền tảng mạnh mẽ và đáng tin cậy, trong khi trừu tượng hóa tất cả sự phức tạp của AWS. Chúng tôi xử lý tất cả công việc nặng nhọc cho bạn, vì vậy bạn có thể tập trung vào việc xây dựng các ứng dụng của mình và phát triển doanh nghiệp của mình.

[Mau](https://mau.nestjs.com/ 'Deploy Nest') hoàn hảo cho các startup, doanh nghiệp vừa và nhỏ, các doanh nghiệp lớn và các nhà phát triển muốn bắt đầu và chạy nhanh chóng mà không cần dành nhiều thời gian để học hỏi và quản lý cơ sở hạ tầng. Nó cực kỳ dễ sử dụng, và bạn có thể có cơ sở hạ tầng của mình chạy trong vài phút. Nó cũng tận dụng AWS phía sau, mang lại cho bạn tất cả các lợi ích của AWS mà không gặp rắc rối khi quản lý sự phức tạp của nó.

<figure><img src="/assets/mau-metrics.png" /></figure>

Với [Mau](https://mau.nestjs.com/ 'Deploy Nest'), bạn có thể:

- Triển khai các ứng dụng NestJS của bạn chỉ với một vài cú nhấp chuột (APIs, microservices, v.v.).
- Cung cấp **cơ sở dữ liệu** như:
  - PostgreSQL
  - MySQL
  - MongoDB (DocumentDB)
  - Redis
  - và nhiều hơn nữa
- Thiết lập các dịch vụ broker như:
  - RabbitMQ
  - Kafka
  - NATS
- Triển khai các nhiệm vụ được lên lịch (**CRON jobs**) và các workers nền.
- Triển khai các hàm lambda và các ứng dụng serverless.
- Thiết lập các đường ống **CI/CD** cho các triển khai tự động.
- Và nhiều hơn nữa!

Để triển khai ứng dụng NestJS của bạn với Mau, chỉ cần chạy lệnh sau:

```bash
$ npm install -g @nestjs/mau
$ mau deploy
```

Đăng ký ngay hôm nay và [Triển khai với Mau](https://mau.nestjs.com/ 'Deploy Nest') để đưa các ứng dụng NestJS của bạn chạy trên AWS trong vài phút!