### Platform agnosticism

Nest là một framework platform-agnostic. Điều này có nghĩa là bạn có thể phát triển **các phần logic có thể tái sử dụng** có thể được sử dụng trên các loại ứng dụng khác nhau. Ví dụ, hầu hết các components có thể được tái sử dụng mà không cần thay đổi trên các framework HTTP server cơ bản khác nhau (ví dụ, Express và Fastify), và thậm chí trên các _types_ ứng dụng khác nhau (ví dụ, frameworks HTTP server, Microservices với các lớp transport khác nhau, và Web Sockets).

#### Build once, use everywhere

Phần **Overview** của tài liệu chủ yếu hiển thị các kỹ thuật coding sử dụng frameworks HTTP server (ví dụ, các ứng dụng cung cấp REST API hoặc cung cấp ứng dụng server-side rendered kiểu MVC). Tuy nhiên, tất cả các building blocks đó có thể được sử dụng trên các lớp transport khác nhau (microservices hoặc websockets).

Hơn nữa, Nest đi kèm với một module GraphQL chuyên dụng. Bạn có thể sử dụng GraphQL như một layer API có thể thay thế với việc cung cấp REST API.

Ngoài ra, tính năng application context giúp tạo bất kỳ loại ứng dụng Node.js nào - bao gồm cả những thứ như CRON jobs và CLI apps - trên nền Nest.

Nest khao khát trở thành một platform đầy đủ cho các ứng dụng Node.js mang lại mức độ modularity và khả năng tái sử dụng cao hơn cho ứng dụng của bạn. Build once, use everywhere!