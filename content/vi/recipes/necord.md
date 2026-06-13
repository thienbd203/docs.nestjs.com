### Necord

Necord là một module mạnh mẽ đơn giản hóa việc tạo các bot [Discord](https://discord.com), cho phép tích hợp liền mạch với ứng dụng NestJS của bạn.

> info **Lưu ý** Necord là một gói bên thứ ba và không được duy trì chính thức bởi nhóm cốt lõi NestJS. Nếu bạn gặp bất kỳ vấn đề nào, vui lòng báo cáo chúng trong [repository chính thức](https://github.com/necordjs/necord).

#### Cài đặt

Để bắt đầu, bạn cần cài đặt Necord cùng với phụ thuộc của nó, [`Discord.js`](https://discord.js.org).

```bash
$ npm install necord discord.js
```

#### Sử dụng

Để sử dụng Necord trong dự án của bạn, nhập `NecordModule` và cấu hình nó với các tùy chọn cần thiết.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { NecordModule } from 'necord';
import { IntentsBitField } from 'discord.js';
import { AppService } from './app.service';

@Module({
  imports: [
    NecordModule.forRoot({
      token: process.env.DISCORD_TOKEN,
      intents: [IntentsBitField.Flags.Guilds],
      development: [process.env.DISCORD_DEVELOPMENT_GUILD_ID],
    }),
  ],
  providers: [AppService],
})
export class AppModule {}
```

> info **Gợi ý** Bạn có thể tìm một danh sách toàn diện các intents có sẵn [ở đây](https://discord.com/developers/docs/topics/gateway#gateway-intents).

Với thiết lập này, bạn có thể inject `AppService` vào các provider của bạn để dễ dàng đăng ký các lệnh, sự kiện, và nhiều hơn nữa.

```typescript
@@filename(app.service)
import { Injectable, Logger } from '@nestjs/common';
import { Context, On, Once, ContextOf } from 'necord';
import { Client } from 'discord.js';

@Injectable()
export class AppService {
  private readonly logger = new Logger(AppService.name);

  @Once('ready')
  public onReady(@Context() [client]: ContextOf<'ready'>) {
    this.logger.log(`Bot logged in as ${client.user.username}`);
  }

  @On('warn')
  public onWarn(@Context() [message]: ContextOf<'warn'>) {
    this.logger.warn(message);
  }
}
```

##### Hiểu ngữ cảnh

Bạn có thể đã nhận thấy decorator `@Context` trong các ví dụ trên. Decorator này inject ngữ cảnh sự kiện vào phương thức của bạn, cho phép bạn truy cập nhiều dữ liệu cụ thể cho sự kiện. Vì có nhiều loại sự kiện, kiểu ngữ cảnh được suy ra sử dụng kiểu `ContextOf<type: string>`. Bạn có thể dễ dàng truy cập các biến ngữ cảnh bằng cách sử dụng decorator `@Context()`, điền biến với một mảng các đối số liên quan đến sự kiện.

#### Lệnh văn bản

> warning **Thận trọng** Lệnh văn bản phụ thuộc vào nội dung tin nhắn, được đặt là sẽ không được dùng nữa cho các bot đã xác minh và các ứng dụng với hơn 100 máy chủ. Điều này có nghĩa là nếu bot của bạn không thể truy cập nội dung tin nhắn, các lệnh văn bản sẽ không hoạt động. Đọc thêm về thay đổi này [ở đây](https://support-dev.discord.com/hc/en-us/articles/4404772028055-Message-Content-Access-Deprecation-for-Verified-Bots).

Đây là cách tạo một handler lệnh đơn giản cho tin nhắn sử dụng decorator `@TextCommand`.

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, TextCommand, TextCommandContext, Arguments } from 'necord';

@Injectable()
export class AppCommands {
  @TextCommand({
    name: 'ping',
    description: 'Responds with pong!',
  })
  public onPing(
    @Context() [message]: TextCommandContext,
    @Arguments() args: string[],
  ) {
    return message.reply('pong!');
  }
}
```

#### Lệnh ứng dụng

Lệnh ứng dụng cung cấp một cách gốc cho người dùng tương tác với ứng dụng của bạn trong client Discord. Có ba loại lệnh ứng dụng có thể được truy cập thông qua các giao diện khác nhau: đầu vào chat, menu ngữ cảnh tin nhắn (truy cập bằng cách nhấp chuột phải vào tin nhắn), và menu ngữ cảnh người dùng (truy cập bằng cách nhấp chuột phải vào người dùng).

<figure><img class="illustrative-image" src="https://i.imgur.com/4EmG8G8.png" /></figure>

#### Lệnh slash

Lệnh slash là một cách tuyệt vời để tương tác với người dùng theo cách có cấu trúc. Chúng cho phép bạn tạo các lệnh với các đối số và tùy chọn chính xác, nâng cao đáng kể trải nghiệm người dùng.

Để định nghĩa một lệnh slash sử dụng Necord, bạn có thể sử dụng decorator `SlashCommand`.

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, SlashCommand, SlashCommandContext } from 'necord';

@Injectable()
export class AppCommands {
  @SlashCommand({
    name: 'ping',
    description: 'Responds with pong!',
  })
  public async onPing(@Context() [interaction]: SlashCommandContext) {
    return interaction.reply({ content: 'Pong!' });
  }
}
```

> info **Gợi ý** Khi client bot của bạn đăng nhập, nó sẽ tự động đăng ký tất cả các lệnh được định nghĩa. Lưu ý rằng các lệnh toàn cầu được cache trong tối đa một giờ. Để tránh các vấn đề với cache toàn cầu, sử dụng đối số `development` trong module Necord, hạn chế khả năng hiển thị của lệnh đến một guild duy nhất.

##### Tùy chọn

Bạn có thể định nghĩa các tham số cho các lệnh slash của bạn sử dụng các decorator tùy chọn. Hãy tạo một class `TextDto` cho mục đích này:

```typescript
@@filename(text.dto)
import { StringOption } from 'necord';

export class TextDto {
  @StringOption({
    name: 'text',
    description: 'Input your text here',
    required: true,
  })
  text: string;
}
```

Sau đó bạn có thể sử dụng DTO này trong class `AppCommands`:

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, SlashCommand, Options, SlashCommandContext } from 'necord';
import { TextDto } from './length.dto';

@Injectable()
export class AppCommands {
  @SlashCommand({
    name: 'length',
    description: 'Calculate the length of your text',
  })
  public async onLength(
    @Context() [interaction]: SlashCommandContext,
    @Options() { text }: TextDto,
  ) {
    return interaction.reply({
      content: `The length of your text is: ${text.length}`,
    });
  }
}
```

Để biết danh sách đầy đủ các decorator tùy chọn tích hợp sẵn, hãy xem [tài liệu này](https://necord.org/interactions/slash-commands#options).

##### Autocomplete

Để triển khai chức năng autocomplete cho các lệnh slash của bạn, bạn sẽ cần tạo một interceptor. Interceptor này sẽ xử lý các yêu cầu khi người dùng nhập vào trường autocomplete.

```typescript
@@filename(cats-autocomplete.interceptor)
import { Injectable } from '@nestjs/common';
import { AutocompleteInteraction } from 'discord.js';
import { AutocompleteInterceptor } from 'necord';

@Injectable()
class CatsAutocompleteInterceptor extends AutocompleteInterceptor {
  public transformOptions(interaction: AutocompleteInteraction) {
    const focused = interaction.options.getFocused(true);
    let choices: string[];

    if (focused.name === 'cat') {
      choices = ['Siamese', 'Persian', 'Maine Coon'];
    }

    return interaction.respond(
      choices
        .filter((choice) => choice.startsWith(focused.value.toString()))
        .map((choice) => ({ name: choice, value: choice })),
    );
  }
}
```

Bạn cũng sẽ cần đánh dấu class tùy chọn của bạn với `autocomplete: true`:

```typescript
@@filename(cat.dto)
import { StringOption } from 'necord';

export class CatDto {
  @StringOption({
    name: 'cat',
    description: 'Choose a cat breed',
    autocomplete: true,
    required: true,
  })
  cat: string;
}
```

Cuối cùng, áp dụng interceptor vào lệnh slash của bạn:

```typescript
@@filename(cats.commands)
import { Injectable, UseInterceptors } from '@nestjs/common';
import { Context, SlashCommand, Options, SlashCommandContext } from 'necord';
import { CatDto } from '/cat.dto';
import { CatsAutocompleteInterceptor } from './cats-autocomplete.interceptor';

@Injectable()
export class CatsCommands {
  @UseInterceptors(CatsAutocompleteInterceptor)
  @SlashCommand({
    name: 'cat',
    description: 'Retrieve information about a specific cat breed',
  })
  public async onSearch(
    @Context() [interaction]: SlashCommandContext,
    @Options() { cat }: CatDto,
  ) {
    return interaction.reply({
      content: `I found information on the breed of ${cat} cat!`,
    });
  }
}
```

#### Menu ngữ cảnh người dùng

Lệnh người dùng xuất hiện trên menu ngữ cảnh xuất hiện khi nhấp chuột phải (hoặc nhấn) vào người dùng. Các lệnh này cung cấp các hành động nhanh nhắm trực tiếp đến người dùng.

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, UserCommand, UserCommandContext, TargetUser } from 'necord';
import { User } from 'discord.js';

@Injectable()
export class AppCommands {
  @UserCommand({ name: 'Get avatar' })
  public async getUserAvatar(
    @Context() [interaction]: UserCommandContext,
    @TargetUser() user: User,
  ) {
    return interaction.reply({
      embeds: [
        new MessageEmbed()
          .setTitle(`Avatar of ${user.username}`)
          .setImage(user.displayAvatarURL({ size: 4096, dynamic: true })),
      ],
    });
  }
}
```

#### Menu ngữ cảnh tin nhắn

Lệnh tin nhắn xuất hiện trong menu ngữ cảnh khi nhấp chuột phải vào tin nhắn, cho phép các hành động nhanh liên quan đến những tin nhắn đó.

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, MessageCommand, MessageCommandContext, TargetMessage } from 'necord';
import { Message } from 'discord.js';

@Injectable()
export class AppCommands {
  @MessageCommand({ name: 'Copy Message' })
  public async copyMessage(
    @Context() [interaction]: MessageCommandContext,
    @TargetMessage() message: Message,
  ) {
    return interaction.reply({ content: message.content });
  }
}
```

#### Nút

[Nút](https://discord.com/developers/docs/interactions/message-components#buttons) là các yếu tố tương tác có thể được bao gồm trong tin nhắn. Khi được nhấp, chúng gửi một [interaction](https://discord.com/developers/docs/interactions/receiving-and-responding#interaction-object) đến ứng dụng của bạn.

```typescript
@@filename(app.components)
import { Injectable } from '@nestjs/common';
import { Context, Button, ButtonContext } from 'necord';

@Injectable()
export class AppComponents {
  @Button('BUTTON')
  public onButtonClick(@Context() [interaction]: ButtonContext) {
    return interaction.reply({ content: 'Button clicked!' });
  }
}
```

#### Menu chọn

[Menu chọn](https://discord.com/developers/docs/interactions/message-components#select-menus) là một loại thành phần tương tác khác xuất hiện trên tin nhắn. Chúng cung cấp một UI giống dropdown cho người dùng chọn các tùy chọn.

```typescript
@@filename(app.components)
import { Injectable } from '@nestjs/common';
import { Context, StringSelect, StringSelectContext, SelectedStrings } from 'necord';

@Injectable()
export class AppComponents {
  @StringSelect('SELECT_MENU')
  public onSelectMenu(
    @Context() [interaction]: StringSelectContext,
    @SelectedStrings() values: string[],
  ) {
    return interaction.reply({ content: `You selected: ${values.join(', ')}` });
  }
}
```

Để biết danh sách đầy đủ các thành phần menu chọn tích hợp sẵn, hãy truy cập [liên kết này](https://necord.org/interactions/message-components#select-menu).

#### Modals

Modals là các dạng bật lên cho phép người dùng gửi đầu vào được định dạng. Đây là cách tạo và xử lý modals sử dụng Necord:

```typescript
@@filename(app.modals)
import { Injectable } from '@nestjs/common';
import { Context, Modal, ModalContext } from 'necord';

@Injectable()
export class AppModals {
  @Modal('pizza')
  public onModal(@Context() [interaction]: ModalContext) {
    return interaction.reply({
      content: `Your fav pizza : ${interaction.fields.getTextInputValue('pizza')}`
    });
  }
}
```

#### Thêm thông tin

Truy cập trang web [Necord](https://necord.org) để biết thêm thông tin.