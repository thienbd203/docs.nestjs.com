### Encryption and Hashing

**Encryption** là quá trình mã hóa thông tin. Quá trình này chuyển đổi biểu diễn gốc của thông tin, được gọi là plaintext, thành một dạng thay thế được gọi là ciphertext. Lý tưởng nhất, chỉ các bên được ủy quyền mới có thể giải mã ciphertext trở lại plaintext và truy cập thông tin gốc. Encryption không tự ngăn chặn sự can thiệp nhưng từ chối nội dung có thể hiểu được đối với một kẻ chặn tiềm năng. Encryption là một hàm hai chiều; những gì được mã hóa có thể được giải mã với key phù hợp.

**Hashing** là quá trình chuyển đổi một key nhất định thành một giá trị khác. Một hàm hash được sử dụng để tạo giá trị mới theo một thuật toán toán học. Sau khi hashing đã được thực hiện, nó nên không thể đi từ đầu ra đến đầu vào.

#### Encryption

Node.js cung cấp một [crypto module](https://nodejs.org/api/crypto.html) tích hợp sẵn mà bạn có thể sử dụng để mã hóa và giải mã các chuỗi, số, buffers, streams, v.v. Nest không cung cấp bất kỳ package bổ sung nào trên top của module này để tránh giới thiệu các abstraction không cần thiết.

Ví dụ, hãy sử dụng thuật toán mã hóa CTR AES (Advanced Encryption System) `'aes-256-ctr'`.

```typescript
import { createCipheriv, randomBytes, scrypt } from 'node:crypto';
import { promisify } from 'node:util';

const iv = randomBytes(16);
const password = 'Password used to generate key';

// The key length is dependent on the algorithm.
// In this case for aes256, it is 32 bytes.
const key = (await promisify(scrypt)(password, 'salt', 32)) as Buffer;
const cipher = createCipheriv('aes-256-ctr', key, iv);

const textToEncrypt = 'Nest';
const encryptedText = Buffer.concat([
  cipher.update(textToEncrypt),
  cipher.final(),
]);
```

Bây giờ để giải mã giá trị `encryptedText`:

```typescript
import { createDecipheriv } from 'node:crypto';

const decipher = createDecipheriv('aes-256-ctr', key, iv);
const decryptedText = Buffer.concat([
  decipher.update(encryptedText),
  decipher.final(),
]);
```

#### Hashing

Đối với hashing, chúng tôi khuyến nghị sử dụng các package [bcrypt](https://www.npmjs.com/package/bcrypt) hoặc [argon2](https://www.npmjs.com/package/argon2). Nest không cung cấp bất kỳ wrappers bổ sung nào trên top của các modules này để tránh giới thiệu các abstraction không cần thiết (làm cho đường cong học tập ngắn).

Ví dụ, hãy sử dụng `bcrypt` để hash một password ngẫu nhiên.

Trước tiên cài đặt các packages cần thiết:

```shell
$ npm i bcrypt
$ npm i -D @types/bcrypt
```

Sau khi cài đặt hoàn tất, bạn có thể sử dụng hàm `hash`, như sau:

```typescript
import * as bcrypt from 'bcrypt';

const saltOrRounds = 10;
const password = 'random_password';
const hash = await bcrypt.hash(password, saltOrRounds);
```

Để tạo một salt, sử dụng hàm `genSalt`:

```typescript
const salt = await bcrypt.genSalt();
```

Để so sánh/kiểm tra một password, sử dụng hàm `compare`:

```typescript
const isMatch = await bcrypt.compare(password, hash);
```

Bạn có thể đọc thêm về các hàm có sẵn [ở đây](https://www.npmjs.com/package/bcrypt).