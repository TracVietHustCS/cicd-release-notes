=== overview | Bắt đầu | Tổng quan và cách học
>> Bộ ghi chú về CI/CD và phát hành phần mềm: từ nguyên tắc, giải phẫu một pipeline, GitHub Actions và GitLab CI, kiểm thử, artifact, bảo mật pipeline, đến chiến lược phát hành, GitOps, migration database, rollback và đo lường bằng DORA.

::: warn Đọc trước khi dùng
Đây là ghi chú **tự học, tự soạn**, không phải tài liệu chính thức của GitHub, GitLab hay bất kỳ nhà cung cấp nào. Cú pháp workflow, tên và phiên bản action, tính năng của nền tảng thay đổi nhanh. Ghi chú không tra cứu trực tuyến nên có thể lỗi thời: hãy đối chiếu tài liệu hiện hành, thử trên repo thử nghiệm, và ghim phiên bản các thành phần bạn dùng. Pipeline có quyền triển khai: đừng đưa nó vào production nếu bạn chưa kiểm soát được ai và cái gì chạy được trong đó.
:::

## Bạn sẽ làm được gì sau khi học xong

- Phân biệt Continuous Integration, Continuous Delivery và Continuous Deployment, và giải thích vì sao tổ chức chọn mức nào.
- Thiết kế pipeline gọn: build một lần, kiểm thử theo tầng, quét bảo mật, đóng gói, triển khai lặp lại được.
- Viết workflow GitHub Actions và `.gitlab-ci.yml` có cache, matrix, môi trường và duyệt thủ công.
- Chọn chiến lược phát hành (rolling, blue-green, canary), kèm cờ tính năng và cổng kiểm tra bằng số đo.
- Xử lý thay đổi database an toàn, quay lui khi sự cố, và đo hiệu quả bằng các chỉ số DORA.

## Cấu trúc

| Nhóm | Trang | Nội dung |
|---|---|---|
| Khái niệm | [Nguyên tắc CI/CD](#cicd-principles) | Định nghĩa, trunk-based, DORA |
| Pipeline | [Giải phẫu pipeline](#pipeline-anatomy), [GitHub Actions](#gha), [GitLab CI và công cụ khác](#gitlab-others), [Chiến lược kiểm thử](#testing-strategy), [Artifact và phiên bản](#artifacts) | Xây pipeline |
| An toàn | [Bảo mật pipeline](#pipeline-security), [Bí mật và OIDC](#secrets-oidc) | Giảm rủi ro chuỗi cung ứng |
| Phát hành | [Môi trường](#environments), [Chiến lược phát hành](#deploy-strategies), [Progressive delivery](#progressive-delivery), [GitOps](#gitops-cd), [Migration database](#db-migrations), [Rollback](#rollback) | Đưa thay đổi lên production |
| Vận hành | [DORA và đo lường](#metrics-dora), [Hiệu năng và sự cố pipeline](#pipeline-ops) | Cải thiện liên tục |
| Tham chiếu | [Pipeline mẫu hoàn chỉnh](#reference-pipelines), [Cheat sheet](#cicd-cheatsheet), [Luyện tập 1](#quiz-ci), [Luyện tập 2](#quiz-cd) | Tra nhanh và tự kiểm tra |

## Môi trường thực hành

Bạn chỉ cần một tài khoản GitHub (hoặc GitLab) miễn phí và một repo thử nghiệm nhỏ có vài bài kiểm thử. Với phần triển khai, có thể dùng cụm Kubernetes cục bộ (`kind`) hoặc một VM thử.

```
git --version
docker --version
gh --version        # GitHub CLI (tuỳ chọn)
```

Công cụ bổ trợ hay dùng: `act` (chạy thử workflow GitHub Actions trên máy), `actionlint` (kiểm tra cú pháp workflow), `yamllint`.

## Liên quan tới các ghi chú khác

Pipeline thường build và chạy [Docker](https://tracviethustcs.github.io/docker-container-notes/), triển khai hạ tầng bằng [Terraform và Ansible](https://tracviethustcs.github.io/iac-terraform-ansible-notes/) và đưa ứng dụng lên [Kubernetes](https://tracviethustcs.github.io/k8s-study-notes/) bằng GitOps. Phần chuỗi cung ứng image (quét, SBOM, ký) có ở [Bảo mật chuỗi cung ứng image](https://tracviethustcs.github.io/k8s-study-notes/#adv-supply-chain).

=== cicd-principles | Khái niệm | Nguyên tắc CI/CD, trunk-based và DORA
>> CI/CD không phải là công cụ mà là cách làm việc: tích hợp thường xuyên, luôn có thể phát hành, và đưa thay đổi nhỏ lên production liên tục. Công cụ chỉ tự động hoá thực hành đó.

## Ba khái niệm hay bị lẫn

| | Continuous Integration (CI) | Continuous Delivery | Continuous Deployment |
|---|---|---|---|
| Ý nghĩa | Mọi thay đổi được **tích hợp** vào nhánh chính thường xuyên (ít nhất mỗi ngày) và được **build, kiểm thử tự động** | Mọi thay đổi đã qua pipeline đều **sẵn sàng phát hành** bất kỳ lúc nào; việc phát hành lên production là **quyết định của con người** (một cú nhấp) | Mọi thay đổi qua pipeline được **tự động đưa lên production**, không cần duyệt thủ công |
| Mục tiêu | Phát hiện lỗi tích hợp sớm | Giảm rủi ro và công sức của việc phát hành | Tốc độ tối đa, vòng phản hồi ngắn nhất |
| Điều kiện | Kiểm thử tự động đáng tin cậy | CI tốt, triển khai tự động hoá hoàn toàn | Kiểm thử, giám sát và rollback đủ tốt để tin tưởng |

"CD" có thể là Delivery hoặc Deployment tuỳ ngữ cảnh: hãy hỏi lại. Phần lớn tổ chức bắt đầu ở Continuous Delivery và chỉ tiến tới Deployment cho những dịch vụ có đủ độ tin cậy.

## Nguyên tắc cốt lõi

1. **Tích hợp thường xuyên, thay đổi nhỏ.** Thay đổi nhỏ dễ review, dễ kiểm thử, dễ tìm nguyên nhân khi hỏng và dễ quay lui.
2. **Một nguồn sự thật**: mã, cấu hình và định nghĩa pipeline đều nằm trong Git (pipeline as code).
3. **Build một lần, triển khai nhiều nơi.** Artifact được tạo một lần rồi đi qua các môi trường; không build lại cho từng môi trường ([Artifact](#artifacts)).
4. **Pipeline nhanh và đáng tin cậy.** Pipeline chậm hoặc hay lỗi ngẫu nhiên khiến người ta bỏ qua nó. Mục tiêu thường là vài phút cho phản hồi đầu tiên.
5. **Hỏng thì sửa ngay.** Nhánh chính luôn xanh; build đỏ là việc ưu tiên số một của cả nhóm.
6. **Tự động hoá mọi thứ lặp lại**, gồm cả triển khai và quay lui.
7. **Dịch chuyển trái (shift-left)**: kiểm thử và bảo mật ở giai đoạn càng sớm càng rẻ.
8. **Có thể quan sát và quay lui**: biết bản nào đang chạy, và quay về bản trước dễ dàng ([Rollback](#rollback)).

## Phát triển trên nhánh chính (trunk-based)

CI chỉ thật sự có nghĩa khi mọi người tích hợp thường xuyên. Nhánh tính năng sống nhiều ngày là "tích hợp trì hoãn": xung đột và lỗi dồn lại tới lúc merge.

| | Trunk-based | Nhánh tính năng dài hạn (GitFlow) |
|---|---|---|
| Nhánh | Nhánh chính duy nhất; nhánh ngắn (dưới 1 đến 2 ngày) hoặc commit thẳng | Nhiều nhánh `develop`, `release`, `feature` sống lâu |
| Tích hợp | Liên tục | Dồn lại tới lúc merge |
| Phát hành | Từ nhánh chính (kèm cờ tính năng để ẩn việc chưa xong) | Theo nhánh release |
| Hợp với | CI/CD thường xuyên, dịch vụ web | Phần mềm đóng gói phát hành theo phiên bản dài hạn |

Việc chưa hoàn thành được **ẩn sau cờ tính năng** (feature flag) để vẫn merge được mã vào nhánh chính mà không bật cho người dùng ([Chiến lược phát hành](#deploy-strategies)).

## Đo hiệu quả: bốn chỉ số DORA

Nghiên cứu DORA tìm ra bốn chỉ số liên quan chặt tới hiệu quả của nhóm phát triển:

| Chỉ số | Đo gì | Nhóm |
|---|---|---|
| **Deployment frequency** | Bao lâu triển khai lên production một lần | Tốc độ |
| **Lead time for changes** | Từ commit đến chạy trên production mất bao lâu | Tốc độ |
| **Change failure rate** | Tỷ lệ triển khai gây sự cố cần khắc phục | Ổn định |
| **Time to restore service** | Mất bao lâu để khôi phục khi có sự cố (các báo cáo mới gọi là thời gian phục hồi sau triển khai lỗi) | Ổn định |

Điểm đáng chú ý của nghiên cứu: **tốc độ và ổn định đi cùng nhau**, không đánh đổi nhau; nhóm triển khai thường xuyên với thay đổi nhỏ lại ít sự cố hơn. Chi tiết cách đo ở [DORA và đo lường](#metrics-dora). Tên chỉ số và cách phân hạng thay đổi theo từng báo cáo hằng năm: hãy xem báo cáo hiện hành.

## Những hiểu lầm hay gặp

- "CI/CD là cài Jenkins hoặc GitHub Actions": công cụ chỉ là phần nhỏ; không có kiểm thử tự động tốt và thói quen tích hợp thường xuyên thì pipeline chỉ chạy nhanh những thứ vô nghĩa.
- "Triển khai thường xuyên thì rủi ro hơn": ngược lại, thay đổi nhỏ và thường xuyên giảm rủi ro mỗi lần. Rủi ro lớn nằm ở bản phát hành to, hiếm.
- "Cần 100% tự động mới là CI/CD": Continuous Delivery với một bước duyệt thủ công vẫn là CD hợp lệ.

::: lab Đánh giá nhanh quy trình hiện tại của bạn
Chọn một repo bạn làm việc và trả lời bằng số liệu thực:

1. Nhánh chính có luôn xanh không? Lần gần nhất nó đỏ là khi nào, và mất bao lâu để sửa?
2. Từ lúc bạn commit đến lúc thay đổi đó chạy trên production mất bao lâu (lead time)?
3. Bao lâu triển khai một lần? Mỗi lần triển khai là thay đổi nhỏ hay dồn lại?
4. Triển khai gần nhất có cần ai làm tay không? Bao nhiêu bước?
5. Nếu bản vừa triển khai hỏng, quay lui mất bao lâu và bằng cách nào?

Ghi các câu trả lời lại: đó là đường cơ sở để so sánh sau khi cải thiện pipeline.
:::

=== pipeline-anatomy | Pipeline | Giải phẫu một pipeline
>> Pipeline là chuỗi bước tự động đưa thay đổi từ commit tới production. Các nền tảng khác nhau về cú pháp nhưng giống nhau về cấu trúc.

## Các giai đoạn điển hình

```ascii
 commit/PR  -->  CHECKOUT  -->  BUILD  -->  TEST  -->  SCAN  -->  PACKAGE  -->  PUBLISH  -->  DEPLOY  -->  VERIFY
                                  |           |          |           |             |            |           |
                              biên dịch   unit, tích   SAST, SCA,  image,        đẩy lên      dev -> stg   smoke test,
                              phụ thuộc   hợp, lint    secret scan  gói          registry/    -> prod      giám sát,
                                                                                  repo artifact             rollback
```

| Giai đoạn | Làm gì | Phản hồi nhanh? |
|---|---|---|
| Lint, kiểm tra tĩnh | Định dạng, phong cách, lỗi cú pháp | Rất nhanh, chạy đầu tiên |
| Build | Biên dịch, cài phụ thuộc, tạo sản phẩm | Nhanh nhờ cache |
| Kiểm thử đơn vị | Kiểm thử logic từng phần | Nhanh |
| Kiểm thử tích hợp | Kiểm thử với database, dịch vụ phụ thuộc | Chậm hơn |
| Quét bảo mật | Mã (SAST), thư viện (SCA), bí mật, image, IaC | Song song với test |
| Đóng gói | Tạo artifact bất biến (image, gói) | |
| Xuất bản | Đẩy artifact lên registry/kho, kèm metadata | |
| Triển khai | Đưa artifact lên môi trường | Có thể cần duyệt |
| Xác minh | Smoke test, kiểm tra sức khoẻ, đo chỉ số | Quyết định giữ hay quay lui |

Nguyên tắc thiết kế: **chạy thứ rẻ và nhanh trước** (lint, unit test) để lỗi lộ ra sớm, thứ đắt và chậm sau; **song song hoá** những việc độc lập; **dừng sớm** khi một giai đoạn quan trọng thất bại.

## Khái niệm chung của mọi nền tảng

| Khái niệm | GitHub Actions | GitLab CI | Ý nghĩa |
|---|---|---|---|
| Định nghĩa | `.github/workflows/*.yml` | `.gitlab-ci.yml` | Pipeline là mã nằm trong repo |
| Kích hoạt | `on:` (push, pull_request, schedule...) | `rules:`/`workflow:` | Khi nào chạy |
| Đơn vị công việc | `job` | `job` | Chạy trên một máy chạy (runner) |
| Bước | `steps` | `script` | Lệnh trong một job |
| Thứ tự | `needs:` | `stages:` và `needs:` | Job nào chờ job nào |
| Máy chạy | `runs-on:` (runner) | runner với executor | Máy ảo/container thực thi |
| Cache | `actions/cache` | `cache:` | Giữ phụ thuộc giữa các lần chạy |
| Artifact | `upload-artifact` | `artifacts:` | Truyền kết quả giữa job và lưu lại |
| Bí mật | `secrets.*` | CI/CD variables (masked, protected) | Giá trị nhạy cảm |
| Môi trường | `environment:` | `environment:` | Nơi triển khai, có quy tắc bảo vệ |

## Runner

Runner là máy thực thi job. Có hai loại:

- **Runner do nhà cung cấp lưu trữ** (hosted): máy sạch tạo mới cho mỗi job, không cần quản lý, nhưng có giới hạn tài nguyên và thời gian, và tính phí ngoài hạn mức.
- **Runner tự lưu trữ** (self-hosted): bạn kiểm soát cấu hình, mạng nội bộ, phần cứng. Đổi lại phải vận hành và **bảo mật**: runner tự lưu trữ chạy mã của pipeline nên không nên dùng cho repo công khai nhận pull request từ bên ngoài, và nên là **tạm thời (ephemeral)**, mỗi job một máy mới, để job không để lại dấu vết cho job sau.

## Bất biến và lặp lại được

Một pipeline tốt cho **cùng kết quả khi chạy lại** cùng một commit:

- Ghim phiên bản công cụ, image nền, action; tránh `latest`.
- Không phụ thuộc trạng thái còn lại trên runner (cache chỉ là tối ưu, không phải điều kiện đúng).
- Không dùng thời gian hay giá trị ngẫu nhiên không kiểm soát trong build.
- Artifact đặt tên theo commit và có thể truy ngược về pipeline sinh ra nó.

## Điều kiện chạy

Không phải mọi thay đổi cần chạy mọi thứ:

| Sự kiện | Gợi ý |
|---|---|
| Pull request | Lint, build, test, quét; **không** triển khai |
| Merge vào nhánh chính | Đầy đủ: build, test, đóng gói, xuất bản, triển khai dev/staging |
| Tag phát hành | Đóng gói bản chính thức, triển khai production (hoặc chờ duyệt) |
| Theo lịch | Quét lỗ hổng định kỳ, kiểm thử chậm, phát hiện drift |
| Thủ công | Triển khai lại, chạy tác vụ bảo trì |

Dùng bộ lọc đường dẫn để bỏ qua việc không liên quan (chỉ đổi tài liệu thì không cần build image): xem [Hiệu năng pipeline](#pipeline-ops).

::: lab Vẽ pipeline của dự án bạn
Trên giấy hoặc bằng ASCII, vẽ pipeline lý tưởng cho một dịch vụ web của bạn: liệt kê từng giai đoạn, bước nào song song, bước nào chặn (blocking), thời gian ước lượng của mỗi bước, và điều kiện kích hoạt (PR, merge, tag). Sau đó so với pipeline thật đang có: thiếu giai đoạn nào, bước nào chậm nhất?
:::

=== gha | Pipeline | GitHub Actions
>> GitHub Actions chạy workflow định nghĩa bằng YAML trong repo, kích hoạt bởi sự kiện của GitHub. Trang này gồm các khái niệm, một workflow CI hoàn chỉnh, cache, matrix, môi trường, workflow tái sử dụng và các điểm bảo mật cần nhớ.

## Khái niệm

| Thành phần | Ý nghĩa |
|---|---|
| **Workflow** | Một tệp YAML trong `.github/workflows/` |
| **Event** | Sự kiện kích hoạt: `push`, `pull_request`, `workflow_dispatch` (thủ công), `schedule` (cron), `release`, `workflow_call` |
| **Job** | Nhóm bước chạy trên một runner; các job chạy **song song** trừ khi có `needs` |
| **Step** | Một lệnh (`run`) hoặc một action (`uses`) |
| **Action** | Đơn vị tái sử dụng (`actions/checkout`, `docker/build-push-action`...) |
| **Runner** | Máy chạy job (`ubuntu-latest`, hoặc tự lưu trữ) |
| **Context** | Biến truy cập bằng `${{ ... }}`: `github`, `env`, `secrets`, `vars`, `needs`, `matrix` |

## Workflow CI hoàn chỉnh

```
name: ci
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read              # quyền tối thiểu cho GITHUB_TOKEN

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true    # huỷ lần chạy cũ khi có commit mới trên cùng nhánh

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install ruff
      - run: ruff check .

  test:
    needs: lint
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python: ["3.11", "3.12"]
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 5s --health-retries 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python }}
          cache: pip
      - run: pip install -r requirements.txt
      - run: pytest --maxfail=1 -q
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/postgres

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          tags: app:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Các điểm cần nhớ:

- **`permissions`** đặt ở đầu workflow để token tự động của job chỉ có quyền cần thiết (mặc định có thể rộng hơn); mở thêm quyền ở job nào thật sự cần.
- **`concurrency`** tránh nhiều lần chạy chồng nhau (và tiết kiệm phút chạy). Với job triển khai production nên **không** huỷ giữa chừng (`cancel-in-progress: false`), chỉ xếp hàng.
- **`needs`** tạo thứ tự; job không `needs` nhau chạy song song.
- **`matrix`** chạy cùng job với nhiều tổ hợp tham số (phiên bản ngôn ngữ, hệ điều hành); `fail-fast: false` để một tổ hợp lỗi không huỷ các tổ hợp còn lại.
- **`services`** dựng container phụ thuộc (database) cho job kiểm thử, với healthcheck.
- **Cache**: `setup-python` có `cache: pip`, hoặc dùng `actions/cache` với khoá theo hash của tệp khoá phụ thuộc. Cache Docker layer qua `cache-from/cache-to` của buildx.

## Truyền dữ liệu giữa các bước và job

```
steps:
  - id: ver
    run: echo "version=$(cat VERSION)" >> "$GITHUB_OUTPUT"       # đầu ra của bước
  - run: echo "Phiên bản ${{ steps.ver.outputs.version }}"

jobs:
  build:
    outputs:
      version: ${{ steps.ver.outputs.version }}                 # đầu ra của job
    # ...
  deploy:
    needs: build
    steps:
      - run: echo "Triển khai ${{ needs.build.outputs.version }}"
```

Truyền tệp giữa job bằng `actions/upload-artifact` và `actions/download-artifact`.

## Môi trường và duyệt thủ công

```
deploy-prod:
  needs: deploy-stg
  runs-on: ubuntu-latest
  environment: production        # bảo vệ: người duyệt bắt buộc, giới hạn nhánh, bí mật riêng
  steps:
    - run: ./deploy.sh production
```

Trong phần cài đặt repo, môi trường `production` có thể yêu cầu **người duyệt**, **chỉ nhánh `main` mới được triển khai**, và có **bí mật/biến riêng** cho môi trường đó. Job dừng chờ duyệt trước khi chạy.

## Workflow tái sử dụng

Tránh lặp pipeline giữa nhiều repo bằng workflow có `workflow_call`:

```
# .github/workflows/build-image.yml  (trong repo "platform-workflows")
on:
  workflow_call:
    inputs:
      image-name: {type: string, required: true}
    secrets:
      registry-token: {required: true}

# nơi gọi
jobs:
  image:
    uses: myorg/platform-workflows/.github/workflows/build-image.yml@v1
    with: {image-name: web}
    secrets:
      registry-token: ${{ secrets.REGISTRY_TOKEN }}
```

Ghim phiên bản (`@v1`, hoặc mã băm commit) cho workflow tái sử dụng và cho action của bên thứ ba.

## Bảo mật workflow: những điều bắt buộc biết

- **Ghim action theo mã băm commit đầy đủ** thay vì tag di động (`uses: actions/checkout@<sha>`), nhất là action của bên thứ ba: tag có thể bị đổi sang mã độc ([Bảo mật pipeline](#pipeline-security)).
- **`pull_request_target`** chạy với quyền ghi và bí mật của repo gốc ngay cả với PR từ fork: **không bao giờ** checkout và chạy mã của PR trong workflow này.
- **Chèn script (script injection)**: không đặt dữ liệu người dùng kiểm soát (tiêu đề PR, tên nhánh) trực tiếp vào `run:` bằng `${{ ... }}`; truyền qua biến môi trường.
- Dùng **OIDC** để lấy quyền cloud ngắn hạn thay vì lưu khoá dài hạn ([Bí mật và OIDC](#secrets-oidc)).
- Với repo công khai, đừng dùng runner tự lưu trữ không tạm thời.

::: lab Chạy thử một workflow trên máy
```
# trong một repo thử có Python
mkdir -p .github/workflows
cat > .github/workflows/ci.yml <<'EOF'
name: ci
on: [push, pull_request]
permissions:
  contents: read
jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Xin chào từ workflow, commit ${{ github.sha }}"
      - run: python3 -c "print(2+2)"
EOF
actionlint                       # kiểm tra cú pháp (nếu đã cài)
act -j hello                     # chạy thử cục bộ bằng Docker (nếu đã cài act)
git add . && git commit -m "ci: thêm workflow" && git push
```
Mở tab Actions trên GitHub, xem log từng bước. Thêm một bước cố ý lỗi và quan sát việc các bước sau bị bỏ qua thế nào.
:::
