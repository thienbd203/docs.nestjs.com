### Tích hợp CI/CD

> info **Gợi ý** Chương này bao gồm tích hợp Nest Devtools với framework Nest. Nếu bạn đang tìm kiếm ứng dụng Devtools, hãy truy cập [Devtools](https://devtools.nestjs.com).

Tích hợp CI/CD có sẵn cho người dùng với gói **Enterprise**.

Bạn có thể xem video này để tìm hiểu tại sao & cách tích hợp CI/CD có thể giúp bạn:

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/r5RXcBrnEQ8"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### Xuất bản đồ thị

Đầu tiên, hãy cấu hình file bootstrap ứng dụng (`main.ts`) để sử dụng class `GraphPublisher` (được xuất từ `@nestjs/devtools-integration` - xem chương trước để biết thêm chi tiết), như sau:

```typescript
async function bootstrap() {
  const shouldPublishGraph = process.env.PUBLISH_GRAPH === "true";

  const app = await NestFactory.create(AppModule, {
    snapshot: true,
    preview: shouldPublishGraph,
  });

  if (shouldPublishGraph) {
    await app.init();

    const publishOptions = { ... } // GHI CHÚ: object tùy chọn này sẽ thay đổi tùy thuộc vào nhà cung cấp CI/CD bạn đang sử dụng
    const graphPublisher = new GraphPublisher(app);
    await graphPublisher.publish(publishOptions);

    await app.close();
  } else {
    await app.listen(process.env.PORT ?? 3000);
  }
}
```

Như chúng ta có thể thấy, chúng ta đang sử dụng `GraphPublisher` ở đây để xuất bản đồ thị đã serialize của mình đến registry tập trung. `PUBLISH_GRAPH` là một biến môi trường tùy chỉnh sẽ cho phép chúng ta kiểm soát xem đồ thị có nên được xuất bản hay không (workflow CI/CD), hay không (bootstrap ứng dụng thông thường). Ngoài ra, chúng ta đặt thuộc tính `preview` thành `true` ở đây. Với flag này được bật, ứng dụng của chúng ta sẽ bootstrap ở chế độ preview - điều này cơ bản có nghĩa là các constructor (và lifecycle hooks) của tất cả controllers, enhancers, và providers trong ứng dụng của chúng ta sẽ không được thực thi. Lưu ý - điều này không **bắt buộc**, nhưng làm cho mọi thứ đơn giản hơn cho chúng ta vì trong trường hợp này chúng ta sẽ không thực sự cần kết nối đến database v.v. khi chạy ứng dụng trong pipeline CI/CD.

Object `publishOptions` sẽ thay đổi tùy thuộc vào nhà cung cấp CI/CD bạn đang sử dụng. Chúng tôi sẽ cung cấp cho bạn hướng dẫn cho các nhà cung cấp CI/CD phổ biến nhất bên dưới, trong các phần sau.

Khi đồ thị được xuất bản thành công, bạn sẽ thấy output sau trong chế độ xem workflow của mình:

<figure><img src="/assets/devtools/graph-published-terminal.png" /></figure>

Mỗi khi đồ thị của chúng ta được xuất bản, chúng ta nên thấy một mục mới trong trang tương ứng của dự án:

<figure><img src="/assets/devtools/project.png" /></figure>

#### Báo cáo

Devtools tạo báo cáo cho mỗi build **NẾU** có một snapshot tương ứng đã được lưu trong registry tập trung. Vì vậy ví dụ, nếu bạn tạo một PR đối với nhánh `master` mà đồ thị đã được xuất bản - thì ứng dụng sẽ có thể phát hiện sự khác biệt và tạo báo cáo. Nếu không, báo cáo sẽ không được tạo.

Để xem báo cáo, điều hướng đến trang tương ứng của dự án (xem organizations).

<figure><img src="/assets/devtools/report.png" /></figure>

Điều này đặc biệt hữu ích trong việc xác định các thay đổi có thể đã bị bỏ qua trong quá trình review code. Ví dụ, giả sử ai đó đã thay đổi phạm vi của một **provider lồng sâu**. Thay đổi này có thể không ngay lập tức rõ ràng với người review, nhưng với Devtools, chúng ta có thể dễ dàng phát hiện các thay đổi như vậy và đảm bảo rằng chúng là có chủ đích. Hoặc nếu chúng ta xóa một guard khỏi một endpoint cụ thể, nó sẽ hiển thị là bị ảnh hưởng trong báo cáo. Bây giờ nếu chúng ta không có integration hoặc e2e test cho route đó, chúng ta có thể không nhận ra rằng nó không còn được bảo vệ, và đến khi chúng ta nhận ra, có thể đã quá muộn.

Tương tự, nếu chúng ta đang làm việc trên một **codebase lớn** và chúng ta sửa đổi một module để trở thành toàn cục, chúng ta sẽ thấy bao nhiêu cạnh được thêm vào đồ thị, và điều này - trong hầu hết các trường hợp - là dấu hiệu rằng chúng ta đang làm sai.

#### Xem trước build

Đối với mỗi đồ thị được xuất bản, chúng ta có thể quay lại thời gian và xem trước nó trông như thế nào trước đó bằng cách nhấp vào nút **Preview**. Hơn nữa, nếu báo cáo được tạo, chúng ta sẽ thấy sự khác biệt được highlight trên đồ thị của mình:

- các node màu xanh lá cây đại diện cho các phần tử được thêm
- các node màu trắng nhạt đại diện cho các phần tử được cập nhật
- các node màu đỏ đại diện cho các phần tử được xóa

Xem ảnh chụp màn hình bên dưới:

<figure><img src="/assets/devtools/nodes-selection.png" /></figure>

Khả năng quay lại thời gian cho phép bạn điều tra và khắc phục sự cố bằng cách so sánh đồ thị hiện tại với đồ thị trước đó. Tùy thuộc vào cách bạn thiết lập, mỗi pull request (hoặc thậm chí mỗi commit) sẽ có một snapshot tương ứng trong registry, vì vậy bạn có thể dễ dàng quay lại thời gian và xem những gì đã thay đổi. Hãy nghĩ về Devtools như Git nhưng với sự hiểu biết về cách Nest xây dựng đồ thị ứng dụng của bạn, và với khả năng **trực quan hóa** nó.

#### Tích hợp: GitHub Actions

Đầu tiên, hãy bắt đầu bằng cách tạo một GitHub workflow mới trong thư mục `.github/workflows` trong dự án của chúng ta và gọi nó, ví dụ, `publish-graph.yml`. Bên trong file này, hãy sử dụng định nghĩa sau:

```yaml
name: Devtools

on:
  push:
    branches:
      - master
  pull_request:
    branches:
      - '*'

jobs:
  publish:
    if: github.actor!= 'dependabot[bot]'
    name: Publish graph
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '16'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Setup Environment (PR)
        if: {{ '${{' }} github.event_name == 'pull_request' {{ '}}' }}
        shell: bash
        run: |
          echo "COMMIT_SHA={{ '${{' }} github.event.pull_request.head.sha {{ '}}' }}" >>\${GITHUB_ENV}
      - name: Setup Environment (Push)
        if: {{ '${{' }} github.event_name == 'push' {{ '}}' }}
        shell: bash
        run: |
          echo "COMMIT_SHA=\${GITHUB_SHA}" >> \${GITHUB_ENV}
      - name: Publish
        run: PUBLISH_GRAPH=true npm run start
        env:
          DEVTOOLS_API_KEY: CHANGE_THIS_TO_YOUR_API_KEY
          REPOSITORY_NAME: {{ '${{' }} github.event.repository.name {{ '}}' }}
          BRANCH_NAME: {{ '${{' }} github.head_ref || github.ref_name {{ '}}' }}
          TARGET_SHA: {{ '${{' }} github.event.pull_request.base.sha {{ '}}' }}
```

Lý tưởng nhất, biến môi trường `DEVTOOLS_API_KEY` nên được lấy từ GitHub Secrets, đọc thêm [ở đây](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository).

Workflow này sẽ chạy cho mỗi pull request nhắm đến nhánh `master` HOẶC trong trường hợp có commit trực tiếp đến nhánh `master`. Tự do điều chỉnh cấu hình này để phù hợp với nhu cầu dự án của bạn. Điều quan trọng ở đây là chúng ta cung cấp các biến môi trường cần thiết cho class `GraphPublisher` của chúng ta (để chạy).

Tuy nhiên, có một biến cần được cập nhật trước khi chúng ta có thể bắt đầu sử dụng workflow này - `DEVTOOLS_API_KEY`. Chúng ta có thể tạo một API key dành riêng cho dự án của chúng ta trên [trang](https://devtools.nestjs.com/settings/manage-api-keys) này.

Cuối cùng, hãy điều hướng đến file `main.ts` một lần nữa và cập nhật object `publishOptions` mà chúng ta đã để trống trước đó.

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.REPOSITORY_NAME,
  owner: process.env.GITHUB_REPOSITORY_OWNER,
  sha: process.env.COMMIT_SHA,
  target: process.env.TARGET_SHA,
  trigger: process.env.GITHUB_BASE_REF ? 'pull' : 'push',
  branch: process.env.BRANCH_NAME,
};
```

Để có trải nghiệm nhà phát triển tốt nhất, hãy đảm bảo tích hợp **ứng dụng GitHub** cho dự án của bạn bằng cách nhấp vào nút "Integrate GitHub app" (xem ảnh chụp màn hình bên dưới). Lưu ý - điều này không bắt buộc.

<figure><img src="/assets/devtools/integrate-github-app.png" /></figure>

Với tích hợp này, bạn sẽ có thể thấy trạng thái của quy trình tạo preview/báo cáo ngay trong pull request của mình:

<figure><img src="/assets/devtools/actions-preview.png" /></figure>

#### Tích hợp: Gitlab Pipelines

Đầu tiên, hãy bắt đầu bằng cách tạo một file cấu hình Gitlab CI mới trong thư mục gốc của dự án của chúng ta và gọi nó, ví dụ, `.gitlab-ci.yml`. Bên trong file này, hãy sử dụng định nghĩa sau:

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.REPOSITORY_NAME,
  owner: process.env.GITHUB_REPOSITORY_OWNER,
  sha: process.env.COMMIT_SHA,
  target: process.env.TARGET_SHA,
  trigger: process.env.GITHUB_BASE_REF ? 'pull' : 'push',
  branch: process.env.BRANCH_NAME,
};
```

> info **Gợi ý** Lý tưởng nhất, biến môi trường `DEVTOOLS_API_KEY` nên được lấy từ secrets.

Workflow này sẽ chạy cho mỗi pull request nhắm đến nhánh `master` HOẶC trong trường hợp có commit trực tiếp đến nhánh `master`. Tự do điều chỉnh cấu hình này để phù hợp với nhu cầu dự án của bạn. Điều quan trọng ở đây là chúng ta cung cấp các biến môi trường cần thiết cho class `GraphPublisher` của chúng ta (để chạy).

Tuy nhiên, có một biến (trong định nghĩa workflow này) cần được cập nhật trước khi chúng ta có thể bắt đầu sử dụng workflow này - `DEVTOOLS_API_KEY`. Chúng ta có thể tạo một API key dành riêng cho dự án của chúng ta trên **trang** này.

Cuối cùng, hãy điều hướng đến file `main.ts` một lần nữa và cập nhật object `publishOptions` mà chúng ta đã để trống trước đó.

```yaml
image: node:16

stages:
  - build

cache:
  key:
    files:
      - package-lock.json
  paths:
    - node_modules/

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: always
    - if: $CI_COMMIT_BRANCH == "master" && $CI_PIPELINE_SOURCE == "push"
      when: always
    - when: never

install_dependencies:
  stage: build
  script:
    - npm ci

publish_graph:
  stage: build
  needs:
    - install_dependencies
  script: npm run start
  variables:
    PUBLISH_GRAPH: 'true'
    DEVTOOLS_API_KEY: 'CHANGE_THIS_TO_YOUR_API_KEY'
```

#### Các công cụ CI/CD khác

Tích hợp CI/CD Nest Devtools có thể được sử dụng với bất kỳ công cụ CI/CD nào bạn chọn (ví dụ, [Bitbucket Pipelines](https://bitbucket.org/product/features/pipelines), [CircleCI](https://circleci.com/), v.v.) vì vậy đừng cảm thấy bị giới hạn bởi các nhà cung cấp chúng tôi mô tả ở đây.

Hãy xem cấu hình object `publishOptions` sau để hiểu thông tin nào được yêu cầu để xuất bản đồ thị cho một commit/build/PR cụ thể.

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.CI_PROJECT_NAME,
  owner: process.env.CI_PROJECT_ROOT_NAMESPACE,
  sha: process.env.CI_COMMIT_SHA,
  target: process.env.CI_MERGE_REQUEST_DIFF_BASE_SHA,
  trigger: process.env.CI_MERGE_REQUEST_DIFF_BASE_SHA ? 'pull' : 'push',
  branch: process.env.CI_COMMIT_BRANCH ?? process.env.CI_MERGE_REQUEST_SOURCE_BRANCH_NAME,
};
```

Hầu hết thông tin này được cung cấp thông qua các biến môi trường tích hợp sẵn của CI/CD (xem [danh sách biến môi trường tích hợp sẵn của CircleCI](https://circleci.com/docs/variables/#built-in-environment-variables) và [biến của Bitbucket](https://support.atlassian.com/bitbucket-cloud/docs/variables-and-secrets/)).

Khi nói đến cấu hình pipeline để xuất bản đồ thị, chúng tôi khuyến nghị sử dụng các triggers sau:

- sự kiện `push` - chỉ nếu nhánh hiện tại đại diện cho môi trường deployment, ví dụ `master`, `main`, `staging`, `production`, v.v.
- sự kiện `pull request` - luôn luôn, hoặc khi **nhánh đích** đại diện cho môi trường deployment (xem ở trên)