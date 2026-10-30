=== metrics-dora | Vận hành | DORA và đo lường
>> Không đo thì không cải thiện được, nhưng đo sai còn tệ hơn không đo. Các chỉ số DORA đo hiệu quả của cả quy trình phát hành, và chỉ có ý nghĩa nếu dùng để học chứ không để chấm điểm người.

## Bốn chỉ số

| Chỉ số | Câu hỏi | Cách đo |
|---|---|---|
| **Deployment frequency** (tần suất triển khai) | Bao lâu triển khai lên production một lần? | Đếm số lần triển khai thành công lên production mỗi ngày/tuần |
| **Lead time for changes** (thời gian đưa thay đổi lên) | Từ commit tới chạy trên production mất bao lâu? | Thời điểm triển khai trừ thời điểm commit (hoặc merge); lấy trung vị |
| **Change failure rate** (tỷ lệ thay đổi gây lỗi) | Bao nhiêu phần trăm triển khai gây sự cố cần khắc phục (quay lui, hotfix)? | Số triển khai gây lỗi chia tổng số triển khai |
| **Time to restore** (thời gian khôi phục) | Khi có sự cố, mất bao lâu để khôi phục dịch vụ? | Thời điểm khôi phục trừ thời điểm bắt đầu sự cố; lấy trung vị |

Hai chỉ số đầu đo **tốc độ**, hai chỉ số sau đo **độ ổn định**. Phát hiện chính của nghiên cứu là các nhóm làm tốt thường tốt ở **cả hai**, vì thay đổi nhỏ và thường xuyên vừa nhanh vừa ít rủi ro hơn bản phát hành to và hiếm. Một số báo cáo DORA gần đây còn nhấn mạnh thêm độ tin cậy vận hành và đổi tên hoặc điều chỉnh cách tính chỉ số thứ tư: hãy đọc báo cáo hiện hành thay vì dựa vào bảng cố định.

## Lấy dữ liệu ở đâu

| Cần | Nguồn |
|---|---|
| Thời điểm commit/merge | Git, nền tảng quản lý mã |
| Thời điểm triển khai production | Sự kiện triển khai của CI/CD (GitHub `deployments`, GitLab `environments`), hoặc sự kiện của Argo CD/Flux |
| Triển khai gây lỗi | Gắn nhãn từ hệ thống sự cố (PagerDuty, Jira, issue gắn nhãn `incident`) liên kết tới bản triển khai |
| Thời điểm sự cố bắt đầu/kết thúc | Hệ thống cảnh báo và quản lý sự cố |

Cách đơn giản để bắt đầu: bảng tính cập nhật tay hằng tuần với vài chục triển khai gần nhất, rồi tự động hoá khi thấy giá trị. Có công cụ chuyên dụng (Four Keys là dự án mã nguồn mở mẫu, các nền tảng thương mại, và bảng điều khiển DORA tích hợp trong một số dịch vụ), nhưng đừng mua công cụ trước khi bạn biết mình muốn trả lời câu hỏi gì.

## Dấu mốc triển khai

Ghi mỗi lần triển khai thành một **sự kiện** (phiên bản, người/pipeline, thời điểm, môi trường) và hiển thị lên biểu đồ giám sát. Khi một chỉ số xấu đi, bạn thấy ngay nó bắt đầu sau lần triển khai nào. Đây là công cụ chẩn đoán sự cố hữu ích nhất và gần như miễn phí.

## SLO làm cổng phát hành

Kết nối phát hành với độ tin cậy:

- **SLI**: chỉ số đo trải nghiệm (tỷ lệ yêu cầu thành công, độ trễ p99).
- **SLO**: mục tiêu (ví dụ 99,9% yêu cầu thành công trong 30 ngày).
- **Ngân sách lỗi (error budget)**: phần "được phép hỏng" còn lại (0,1% trong ví dụ). Còn nhiều ngân sách thì phát hành mạnh tay, cạn ngân sách thì ưu tiên ổn định hơn tính năng mới.

Cổng canary tự động ([Progressive delivery](#progressive-delivery)) chính là kiểm tra SLI của bản mới.

## Dùng chỉ số đúng cách

::: warn Chỉ số bị biến thành mục tiêu sẽ bị làm sai lệch (định luật Goodhart)
Nếu đem chỉ số ra chấm điểm cá nhân hay đội, người ta sẽ tối ưu con số thay vì kết quả: tách một thay đổi thành nhiều lần triển khai vô nghĩa để tăng tần suất, hoặc không ghi nhận sự cố để giảm tỷ lệ lỗi. Dùng chỉ số để **thấy nút thắt và so sánh với chính mình theo thời gian**, không để xếp hạng.
:::

- Xem **xu hướng**, không chỉ một con số; so một đội với chính nó qua thời gian, không so các đội khác bản chất.
- Dùng **trung vị và phân vị**, không chỉ trung bình (một vài ca cực đoan làm lệch trung bình).
- Đo cả bốn cùng nhau: tăng tốc độ mà ổn định tụt là tín hiệu xấu.
- Dùng số liệu để **đặt câu hỏi** ("vì sao lead time ở dịch vụ này dài gấp ba?") chứ không để kết luận.

## Tìm nút thắt từ lead time

Tách lead time thành các đoạn: thời gian chờ review, thời gian pipeline chạy, thời gian chờ duyệt triển khai, thời gian từ merge tới production. Đoạn dài nhất là nơi cần cải thiện. Thường là **chờ đợi** (review, duyệt, hàng đợi runner), không phải thời gian máy chạy.

::: lab Tính lead time từ Git
```
# trong một repo có tag phát hành dạng vX.Y.Z
for t in $(git tag --sort=-creatordate | head -5); do
  prev=$(git describe --tags --abbrev=0 "$t^" 2>/dev/null) || continue
  echo "== $prev -> $t"
  git log --format='%ct' "$prev..$t" | sort -n | head -1 | { read first; tagt=$(git log -1 --format=%ct "$t"); echo "  commit cũ nhất tới tag: $(( (tagt - first) / 3600 )) giờ"; }
done
```
Hoặc dùng bảng tính: ghi 10 lần triển khai gần nhất kèm thời điểm commit đầu tiên của mỗi lần. Tính trung vị của lead time và tần suất triển khai mỗi tuần. Đoạn nào trong quy trình của bạn chiếm nhiều thời gian chờ nhất?
:::

=== pipeline-ops | Vận hành | Hiệu năng và sự cố của pipeline
>> Pipeline chậm hoặc hay lỗi ngẫu nhiên làm cả nhóm mất niềm tin và bỏ qua nó. Trang này gồm cách tăng tốc và cách chẩn đoán các lỗi phổ biến.

## Tăng tốc

| Kỹ thuật | Cách làm |
|---|---|
| **Cache phụ thuộc** | Khoá cache theo hash của tệp khoá (`package-lock.json`, `requirements.txt`, `go.sum`); đừng cache thư mục build đầy đủ không kiểm soát |
| **Cache layer Docker** | `cache-from/cache-to` của buildx (loại `gha` hoặc `registry`); sắp xếp Dockerfile để phụ thuộc ở trên ([Dockerfile tốt](https://tracviethustcs.github.io/docker-container-notes/#build-best)) |
| **Song song hoá** | Tách job độc lập chạy cùng lúc; chia bộ test thành nhiều phần (sharding) |
| **Chỉ chạy việc liên quan** | Bộ lọc đường dẫn (`paths:` của Actions, `rules: changes:` của GitLab) để thay đổi tài liệu không build image |
| **Dừng sớm** | Lint và test nhanh chạy trước; `fail-fast`; huỷ lần chạy cũ khi có commit mới (`concurrency`) |
| **Runner đủ mạnh** | Với job nặng, runner lớn hơn thường rẻ hơn tổng thời gian chờ của lập trình viên |
| **Build tăng dần (incremental)** | Công cụ build có cache từ xa (Bazel, Gradle, Nx, Turborepo) cho monorepo |

```
on:
  pull_request:
    paths:
      - "src/**"
      - "Dockerfile"
      - ".github/workflows/ci.yml"        # đổi chính pipeline cũng phải chạy
```

Đo trước khi tối ưu: nền tảng cho biết thời gian từng bước; tìm bước chiếm nhiều nhất. Tối ưu bước 2 giây là vô ích khi bước chậm là 8 phút.

## Pipeline với monorepo

Chỉ build và kiểm thử **thành phần bị ảnh hưởng**: lọc theo đường dẫn, hoặc công cụ hiểu đồ thị phụ thuộc (Nx, Bazel). Giữ một bước kiểm tra toàn bộ ở nhánh chính hoặc theo lịch để bắt các phụ thuộc bị bỏ sót.

## Các lỗi thường gặp

| Triệu chứng | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|
| "Chạy được trên máy tôi" nhưng đỏ trên CI | Khác phiên bản công cụ, biến môi trường, hệ điều hành, thứ tự test, dữ liệu | Ghim phiên bản; chạy cùng lệnh qua `make`/script; chạy bằng cùng image (container) cục bộ |
| Test lúc xanh lúc đỏ | Test không ổn định: phụ thuộc thời gian, thứ tự, dữ liệu dùng chung | Cách ly, sửa gốc, không chỉ `retry` ([Kiểm thử](#testing-strategy)) |
| `permission denied` với Docker | Runner không có quyền vào `docker.sock`, hoặc đang chạy non-root | Dùng runner phù hợp, hoặc build không cần daemon (buildx, Kaniko) |
| `no space left on device` | Runner đầy do image, cache, artifact tích tụ | Dọn (`docker system prune`), runner tạm thời, giới hạn artifact |
| Hết giới hạn tốc độ khi kéo image (`toomanyrequests`) | Docker Hub giới hạn pull | Đăng nhập registry, dùng mirror/cache nội bộ |
| Bí mật rỗng trong job | PR từ fork không có bí mật; bí mật theo môi trường chưa được duyệt; tên sai | Kiểm tra phạm vi bí mật và điều kiện chạy |
| Cache không bao giờ trúng | Khoá cache đổi mỗi lần (có thời gian, SHA), hoặc đường dẫn sai | Khoá theo hash tệp khoá; kiểm tra log "cache hit/miss" |
| Cache cho kết quả sai | Cache chứa thứ lỗi thời hoặc đã bị nhiễm | Đổi khoá/phiên bản cache; không để cache quyết định tính đúng |
| Job bị treo hoặc bị huỷ do hết thời gian | Chờ tài nguyên, vòng lặp chờ vô hạn, test treo | Đặt `timeout-minutes` cho job và bước; xem log tới điểm cuối |
| Hàng đợi runner dài | Quá ít runner so với số job | Thêm runner, runner co giãn tự động, giảm job không cần thiết |
| Triển khai thành công nhưng ứng dụng hỏng | Thiếu kiểm tra sau triển khai | Thêm smoke test và kiểm tra sức khoẻ trong pipeline; quay lui tự động |
| Hai lần triển khai chạy chồng nhau | Không có khoá | `concurrency` (Actions), `resource_group` (GitLab) |

## Gỡ lỗi pipeline

- **Đọc log từ trên xuống**, tìm dòng lỗi đầu tiên: lỗi sau thường chỉ là hệ quả.
- Bật log chi tiết có kiểm soát (`ACTIONS_STEP_DEBUG` của Actions, `CI_DEBUG_TRACE` của GitLab: cẩn thận vì có thể in cả bí mật).
- Tái hiện cục bộ: chạy đúng lệnh trong cùng image (`docker run -it --rm -v "$PWD":/w -w /w IMAGE bash`), hoặc dùng `act` cho GitHub Actions.
- Với lỗi chỉ xảy ra trên runner: cho phép truy cập tạm thời có kiểm soát (một số nền tảng có phiên SSH debug), tránh để lại cho runner dùng chung.
- Thu thập artifact khi lỗi (`if: always()`/`when: always`): báo cáo test, log, ảnh chụp.

## Thói quen bền vững

- Đặt **`timeout-minutes`** cho mọi job.
- Đặt **ngân sách thời gian** cho pipeline (ví dụ phản hồi đầu tiên dưới 10 phút) và theo dõi theo thời gian: pipeline chậm dần là nợ.
- Giữ pipeline **như mã**: review thay đổi, kiểm tra cú pháp (`actionlint`), cập nhật phụ thuộc có kiểm soát.
- Theo dõi **tỷ lệ pipeline xanh** của nhánh chính, và mức flaky.

::: lab Tìm bước chậm nhất và thử một tối ưu
Lấy một workflow bạn có (hoặc workflow ở trang [GitHub Actions](#gha)).

1. Mở một lần chạy gần đây, ghi thời gian từng bước. Bước nào chiếm nhiều nhất?
2. Thêm cache phụ thuộc và so thời gian lần chạy thứ hai với lần đầu.
3. Thêm bộ lọc `paths:` rồi đẩy một commit chỉ đổi tệp tài liệu: pipeline có được bỏ qua không?
4. Thêm `timeout-minutes: 10` vào job và `concurrency` huỷ lần chạy cũ.

Ghi lại thời gian trước và sau mỗi thay đổi.
:::

=== reference-pipelines | Tham chiếu | Pipeline mẫu hoàn chỉnh
>> Một pipeline từ commit tới triển khai, ghép các mảnh ở những trang trước: kiểm thử, build một lần, quét, ký, cập nhật repo GitOps. Chỉ là điểm khởi đầu: điều chỉnh theo dự án.

::: warn Ví dụ chỉ để minh hoạ
Phiên bản action, tên tổ chức, ARN và đường dẫn là giả định. Hãy ghim action theo mã băm commit, thay bằng giá trị thật, kiểm tra bằng `actionlint`, và thử trên repo thử nghiệm trước khi dùng.
:::

## Kiến trúc của pipeline

```ascii
  PR:   lint -> test -> build (không đẩy) -> quét      (không triển khai)

  main: lint -> test -> build+đẩy image (tag SHA) -> quét -> ký
                                                              |
                                              cập nhật tag trong repo cấu hình (GitOps)
                                                              |
                                           Flux/Argo CD đồng bộ dev  -> (kiểm tra) -> PR lên staging -> PR lên prod (duyệt)
```

## Workflow GitHub Actions

```
name: ci-cd
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

env:
  REGISTRY: ghcr.io
  IMAGE: ghcr.io/myorg/web

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.12", cache: pip}
      - run: pip install -r requirements.txt
      - run: pytest -q

  image:
    needs: test
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      contents: read
      packages: write          # đẩy image lên GHCR
      id-token: write          # ký keyless bằng OIDC
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        if: github.event_name == 'push'
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: ${{ env.IMAGE }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: Quét lỗ hổng
        uses: aquasecurity/trivy-action@0.28.0   # ví dụ: nên ghim theo mã băm commit
        with:
          image-ref: ${{ env.IMAGE }}:${{ github.sha }}
          severity: HIGH,CRITICAL
          exit-code: "1"
          ignore-unfixed: true
        if: github.event_name == 'push'
      - uses: sigstore/cosign-installer@v3
        if: github.event_name == 'push'
      - name: Ký image
        if: github.event_name == 'push'
        run: cosign sign --yes "${IMAGE}@${DIGEST}"
        env:
          IMAGE: ${{ env.IMAGE }}
          DIGEST: ${{ steps.push.outputs.digest }}

  update-gitops:
    if: github.event_name == 'push'
    needs: image
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: myorg/deploy-config
          token: ${{ secrets.DEPLOY_CONFIG_TOKEN }}    # token chỉ có quyền ghi vào repo cấu hình
      - name: Cập nhật tag dev
        run: |
          cd envs/dev
          kustomize edit set image web="${IMAGE}@${DIGEST}"
          git config user.name "ci-bot" && git config user.email "ci-bot@example.com"
          git commit -am "dev: web -> ${GITHUB_SHA::7}" && git push
        env:
          IMAGE: ghcr.io/myorg/web
          DIGEST: ${{ needs.image.outputs.digest }}
```

Điểm đáng chú ý:

- Mọi đầu vào do người dùng kiểm soát (không có ở đây) đi qua biến môi trường; `GITHUB_SHA` và digest là giá trị an toàn do hệ thống tạo.
- `packages: write` và `id-token: write` chỉ mở cho job `image`, không cho cả workflow.
- Artifact được định danh bằng **digest** khi cập nhật repo cấu hình: triển khai đúng cái đã quét và đã ký.
- Pull request chỉ build, không đẩy, không ký, không đụng repo cấu hình.
- Token cập nhật repo cấu hình (`DEPLOY_CONFIG_TOKEN`) chỉ có quyền hẹp (một repo); tốt hơn nữa là dùng GitHub App hoặc danh tính ngắn hạn.

## Bản tương đương trên GitLab CI

```
stages: [test, build, scan, deploy]

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

test:
  stage: test
  image: python:3.12-slim
  script:
    - pip install -r requirements.txt
    - pytest -q

build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - /kaniko/executor --context "$CI_PROJECT_DIR" --dockerfile Dockerfile --destination "$IMAGE"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed "$IMAGE"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-dev:
  stage: deploy
  script:
    - ./scripts/update-gitops.sh dev "$CI_COMMIT_SHA"
  environment: dev
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Dùng Kaniko (build không cần daemon, không cần chế độ privileged) thay cho Docker-in-Docker là lựa chọn an toàn hơn trên runner dùng chung. Với image công cụ (`trivy:latest`, `kaniko:debug`) ở môi trường thật hãy ghim phiên bản.

## Điều chỉnh cho dự án của bạn

| Nếu | Thay đổi |
|---|---|
| Không dùng Kubernetes/GitOps | Thay bước `update-gitops` bằng bước triển khai (ssh, `helm upgrade`, `aws ecs update-service`) chạy với danh tính OIDC |
| Cần duyệt trước production | Thêm job `environment: production` có người duyệt bắt buộc, hoặc PR cần review ở repo cấu hình |
| Có nhiều dịch vụ | Dùng workflow tái sử dụng ([GitHub Actions](#gha)) và bộ lọc đường dẫn |
| Cần phát hành theo phiên bản | Thêm job theo tag `v*` dùng `release-please`/`semantic-release` ([Artifact](#artifacts)) |
| Cần canary | Thay Deployment bằng Rollout ([Progressive delivery](#progressive-delivery)) |

::: lab Kiểm tra workflow trước khi đẩy
```
mkdir -p .github/workflows && cat > .github/workflows/ci-cd.yml   # dán workflow ở trên vào
actionlint                          # kiểm tra cú pháp và biểu thức
yamllint -d relaxed .github/workflows/ci-cd.yml
grep -n "uses:" .github/workflows/ci-cd.yml        # còn action nào chưa ghim?
act pull_request -j test             # chạy thử job test cục bộ (cần Docker và act)
```
Liệt kê những chỗ bạn sẽ đổi để dùng cho dự án thật, và những action cần ghim theo mã băm.
:::

=== cicd-cheatsheet | Tham chiếu | Cheat sheet CI/CD
>> Tra nhanh theo nhu cầu: cú pháp hay dùng, quy ước phiên bản, chiến lược phát hành, và những điều nên kiểm tra.

## GitHub Actions: cú pháp hay dùng

| Cần | Viết |
|---|---|
| Kích hoạt | `on: [push, pull_request]`, `workflow_dispatch`, `schedule: [{cron: "0 2 * * *"}]` |
| Giới hạn nhánh, đường dẫn | `on: push: {branches: [main], paths: ["src/**"]}` |
| Quyền tối thiểu | `permissions: {contents: read}` |
| Thứ tự job | `needs: [build]` |
| Điều kiện | `if: github.ref == 'refs/heads/main'` |
| Ma trận | `strategy: {matrix: {python: ["3.11", "3.12"]}, fail-fast: false}` |
| Đầu ra của bước, của job | `echo "k=v" >> "$GITHUB_OUTPUT"`, `outputs: {k: ${{ steps.id.outputs.k }}}` |
| Môi trường (có duyệt) | `environment: production` |
| Chặn chạy chồng | `concurrency: {group: deploy-prod, cancel-in-progress: false}` |
| Giới hạn thời gian | `timeout-minutes: 15` |
| Luôn chạy (dọn dẹp, thu log) | `if: always()` |
| OIDC | `permissions: {id-token: write}` rồi action đăng nhập cloud |
| Dùng lại workflow | `uses: org/repo/.github/workflows/x.yml@v1` kèm `workflow_call` |

## GitLab CI: từ khoá hay dùng

| Cần | Viết |
|---|---|
| Thứ tự | `stages: [...]`, `needs: [job]` |
| Điều kiện | `rules: [{if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH}]`, `rules: [{changes: ["src/**"]}]` |
| Duyệt tay | `when: manual` |
| Môi trường | `environment: {name: production, url: ...}` |
| Chặn chồng nhau | `resource_group: production` |
| Cache, artifact | `cache: {key: {files: [lock]}, paths: [...]}`, `artifacts: {paths: [...], when: always}` |
| Tái sử dụng | `extends:`, `include:`, `!reference [...]` |
| Giới hạn thời gian | `timeout: 15m` |

## Quy ước phiên bản và commit

| Cần | Ghi nhớ |
|---|---|
| Semver | `MAJOR.MINOR.PATCH`: phá vỡ, tính năng, sửa lỗi |
| Conventional Commits | `feat:` (MINOR), `fix:` (PATCH), `feat!:` hoặc `BREAKING CHANGE:` (MAJOR) |
| Tag Git phát hành | `git tag -a v1.4.2 -m "..." && git push origin v1.4.2` |
| Tag artifact | SHA cho mọi build, semver cho phát hành, **digest** khi triển khai |
| Không dùng để triển khai | `latest`, tag di động (`main`, `1.4`) |

## Chiến lược phát hành

| Chiến lược | Dùng khi | Quay lui |
|---|---|---|
| Recreate | Chấp nhận khoảng chết | Triển khai lại |
| Rolling | Mặc định cho dịch vụ web | Rollout ngược |
| Blue-green | Cần quay lui tức thì, có ngân sách | Đổi lại đích traffic |
| Canary | Thay đổi rủi ro, có chỉ số tốt | Rút traffic |
| Shadow | Thử với traffic thật không ảnh hưởng người dùng | Không cần |
| Cờ tính năng | Tách triển khai khỏi phát hành | Tắt cờ |

## Thay đổi database

Mở rộng (thêm, tương thích) rồi triển khai, backfill, chuyển đọc/ghi, và **cuối cùng mới xoá** (contract). Không đổi tên hoặc xoá cột trong cùng lần phát hành với mã dùng nó. Ưu tiên tiến lên hơn quay lui schema.

## Danh sách kiểm tra nhanh cho một pipeline

| Việc | Kiểm tra |
|---|---|
| Phản hồi nhanh | Lint và test nhanh chạy đầu, có cache, có song song |
| Build một lần | Cùng artifact đi qua mọi môi trường, định danh bằng digest |
| Bảo mật | Quét bí mật, phụ thuộc, image, IaC; action ghim theo mã băm; quyền tối thiểu |
| Xác thực | OIDC, không khoá dài hạn trong biến CI |
| Production | Chỉ từ nhánh chính hoặc tag; có duyệt; có khoá chống chồng nhau |
| Quay lui | Có đường quay lui đã thử; artifact cũ còn giữ |
| Quan sát | Dấu mốc triển khai; smoke test; cảnh báo; đo DORA |

=== quiz-ci | Luyện tập | Luyện tập 1: CI, pipeline và bảo mật
>> 6 câu tình huống về nguyên tắc CI/CD, thiết kế pipeline, artifact và bảo mật. Chọn đáp án rồi bấm "Kiểm tra" để xem giải thích. Kết quả được lưu trong trình duyệt.

## Câu hỏi

@quiz
Q: Nhóm bạn merge vào nhánh chính mỗi vài ngày từ các nhánh tính năng sống lâu, và thường xuyên gặp xung đột lớn, test đỏ sau khi merge. Điều chỉnh nào phù hợp nhất với tinh thần CI?
- Thêm nhiều môi trường staging hơn
+ Tích hợp thường xuyên bằng thay đổi nhỏ và nhánh ngắn (trunk-based), ẩn việc chưa xong sau cờ tính năng, và giữ nhánh chính luôn xanh
- Đóng băng nhánh chính trong tuần trước phát hành
- Tăng thời gian review lên gấp đôi
E: CI là tích hợp thường xuyên để lỗi tích hợp lộ ra sớm khi còn nhỏ. Nhánh sống lâu chỉ trì hoãn tích hợp. Cờ tính năng cho phép merge mã chưa hoàn thiện mà không bật cho người dùng.
Q: Pipeline của bạn build lại image riêng cho từng môi trường (dev, staging, production) từ cùng một commit. Vấn đề chính của cách này là gì?
- Tốn dung lượng registry và không có vấn đề nào khác
+ Image được kiểm thử ở staging không còn là image chạy ở production (build lại có thể khác do phụ thuộc đổi, thời gian); nên build một lần rồi đưa chính artifact đó qua các môi trường, khác biệt nằm ở cấu hình
- Không thể gắn tag cho nhiều image
- Image luôn chạy chậm hơn khi build nhiều lần
E: Điều bạn kiểm thử phải là điều bạn triển khai. Build một lần, định danh bằng SHA hoặc digest, rồi promotion bằng cách chỉ định artifact cho môi trường tiếp theo. Cấu hình riêng từng môi trường nằm ngoài artifact.
Q: Chọn TẤT CẢ biện pháp làm giảm rủi ro cho workflow GitHub Actions.
+ Đặt `permissions` tối thiểu cho token và chỉ mở thêm ở job cần
+ Ghim action bên thứ ba theo mã băm commit đầy đủ thay vì tag di động
- Cho workflow `pull_request_target` checkout và chạy mã của PR từ fork để kiểm thử nhanh
+ Truyền dữ liệu như tiêu đề PR qua biến môi trường thay vì chèn trực tiếp `${{ ... }}` vào lệnh `run:`
E: `pull_request_target` chạy với bí mật và quyền ghi của repo gốc, nên chạy mã từ fork trong đó là nguy hiểm. Ba biện pháp còn lại giảm bề mặt tấn công: quyền tối thiểu, chuỗi cung ứng action, và chống chèn script.
Q: Pipeline production dùng khoá truy cập AWS dài hạn lưu trong secret của CI. Cải thiện hợp lý nhất?
- Đổi tên secret thành tên khó đoán hơn
+ Dùng OIDC để job nhận thông tin xác thực ngắn hạn cho đúng repo và nhánh/môi trường, với vai trò quyền tối thiểu và điều kiện `sub` chặt trong trust policy
- Cho cả tổ chức dùng chung khoá để dễ quản lý
- Mã hoá khoá bằng base64 trong file cấu hình
E: Khoá dài hạn là mục tiêu bị đánh cắp và hiếm khi được xoay vòng. OIDC cấp quyền tạm thời theo danh tính của job, và điều kiện `sub` giới hạn đúng repo/nhánh được phép. Base64 không phải mã hoá.
Q: Bộ test của bạn có vài test hay đỏ ngẫu nhiên, và mọi người quen bấm "chạy lại" cho tới khi xanh. Hệ quả nguy hiểm nhất và cách đúng?
- Không sao, miễn cuối cùng pipeline xanh
+ Mọi người mất niềm tin vào pipeline và bỏ qua cả lỗi thật; cần đo, cách ly (quarantine) test flaky khỏi đường chặn, có người chịu trách nhiệm sửa nguyên nhân gốc thay vì chỉ thêm retry
- Xoá hết các test chậm
- Chuyển toàn bộ sang kiểm thử thủ công
E: Test không ổn định làm tín hiệu của pipeline mất giá trị. Cách ly giúp giữ đường chặn đáng tin cậy trong lúc sửa; retry chỉ nên là biện pháp tạm có ghi nhận, không phải lời giải.
Q: Bạn muốn đặt phiên bản tự động và changelog dựa trên lịch sử commit. Điều kiện và cách thực hiện phù hợp?
- Mỗi người tự đặt số phiên bản trong tên tệp
+ Dùng Conventional Commits (`feat:`, `fix:`, `feat!:`) và công cụ như semantic-release hoặc release-please để tự tính semver, tạo tag và changelog; kiểm tra định dạng commit trong CI
- Đặt phiên bản bằng ngày hiện tại và bỏ qua semver
- Dùng tag `latest` cho mọi bản phát hành
E: Quy ước commit có cấu trúc cho phép công cụ suy ra mức tăng phiên bản (MAJOR cho thay đổi phá vỡ, MINOR cho tính năng, PATCH cho sửa lỗi). Cần cả nhóm tuân theo, và nên có kiểm tra tự động.

=== quiz-cd | Luyện tập | Luyện tập 2: Phát hành, GitOps và khôi phục
>> 6 câu tình huống về chiến lược phát hành, GitOps, migration database và rollback.

## Câu hỏi

@quiz
Q: Bạn cập nhật dịch vụ bằng rolling update, nhưng migration đã đổi tên cột `name` thành `full_name` và xoá cột cũ ngay trong cùng lần phát hành. Chuyện gì xảy ra và cách đúng là gì?
- Không có vấn đề vì rolling update thay từng Pod
+ Trong lúc cuốn chiếu, Pod bản cũ vẫn đọc cột `name` đã bị xoá và lỗi; dùng expand/contract: thêm cột mới, ghi cả hai, backfill, chuyển đọc, và chỉ xoá cột cũ ở lần phát hành sau
- Chỉ cần tăng số replica
- Chuyển sang chiến lược recreate để tránh vấn đề
E: Khi rolling hoặc canary, bản cũ và mới chạy cùng lúc trên cùng database, nên schema phải tương thích cả hai. Chia thay đổi phá vỡ thành các bước nhỏ tương thích ngược, mỗi bước triển khai riêng; recreate chỉ che vấn đề bằng khoảng chết.
Q: Bạn cần phát hành thay đổi rủi ro cho một API có lưu lượng lớn và đã có chỉ số tỷ lệ lỗi, độ trễ tốt. Chiến lược nào cân bằng rủi ro và chi phí nhất?
- Recreate vào ban đêm
+ Canary: cho một phần nhỏ traffic vào bản mới, đo chỉ số rồi tăng dần hoặc tự động rút lại khi xấu đi
- Blue-green nhưng không kiểm tra chỉ số
- Triển khai thẳng cho 100% rồi chờ người dùng báo lỗi
E: Canary giới hạn số người chịu rủi ro và tận dụng chỉ số có sẵn để ra quyết định (có thể tự động hoá bằng Argo Rollouts hoặc Flagger). Blue-green chuyển toàn bộ một lần nên mọi người cùng chịu nếu có lỗi; recreate gây gián đoạn.
Q: Chọn TẤT CẢ ưu điểm của GitOps kiểu kéo (Flux, Argo CD) so với việc pipeline CI chạy `kubectl apply` vào cụm.
+ CI không cần giữ thông tin xác thực vào cụm production, giảm bề mặt tấn công
+ Bộ điều khiển trong cụm liên tục đối chiếu và phát hiện hoặc sửa drift
- Không còn cần repo Git hay review thay đổi
+ Quay lui bằng `git revert`, và lịch sử Git ghi lại ai đổi gì, khi nào
E: GitOps lấy Git làm nguồn sự thật nên vẫn cần review và lịch sử Git (đó là ưu điểm, không phải thứ bị bỏ). Việc kéo từ trong cụm tránh phải đưa khoá vào CI, và việc đối chiếu liên tục giúp phát hiện thay đổi tay.
Q: Sau một lần triển khai, tỷ lệ lỗi tăng vọt. Cụm dùng Argo CD đồng bộ từ Git. Cách quay lui đúng nhất là gì?
- Chạy `kubectl rollout undo` trực tiếp rồi để yên
+ `git revert` commit triển khai trong repo cấu hình để Argo CD đưa cụm về trạng thái trước (hoặc rút canary/tắt cờ tính năng nếu có sẵn), vì nếu sửa tay ngoài Git, bộ điều khiển sẽ ghi đè lại
- Xoá Deployment rồi chờ Argo CD tạo lại
- Tắt giám sát để giảm cảnh báo
E: Trong GitOps, Git là trạng thái mong muốn: chỉnh tay trong cụm sẽ bị bộ điều khiển đưa trở lại. Quay lui đúng là đổi trong Git. Cờ tính năng hoặc rút canary còn nhanh hơn nếu đã được thiết lập, vì giảm thiệt hại trong vài giây.
Q: Dịch vụ có 200 request mỗi phút. Bạn đặt canary 5% trong 5 phút với cổng "tỷ lệ lỗi dưới 1%". Vì sao cổng này có thể vô nghĩa, và nên chỉnh thế nào?
- Vì 5% luôn quá nhỏ cho mọi dịch vụ
+ Canary chỉ nhận khoảng 50 request trong 5 phút, một lỗi đơn lẻ đã là 2% và nhiễu thống kê rất lớn; cần thời gian dài hơn, tỷ lệ cao hơn, hoặc ngưỡng theo số lượng lỗi hoặc chỉ số khác phù hợp với lưu lượng
- Vì tỷ lệ lỗi không bao giờ đo được trong canary
- Vì canary không dùng được với dịch vụ web
E: Ý nghĩa thống kê phụ thuộc số mẫu. Với ít request, cổng theo phần trăm dao động mạnh nên gây rollback giả hoặc bỏ lọt lỗi. Thiết kế cổng theo lưu lượng thật: kéo dài thời gian, tăng tỷ lệ, hoặc dùng ngưỡng theo số lượng.
Q: Một đồng nghiệp đề xuất lấy "số lần triển khai mỗi tuần" làm mục tiêu để đánh giá từng lập trình viên. Rủi ro chính?
- Không có rủi ro vì tần suất triển khai là chỉ số khách quan
+ Chỉ số bị biến thành mục tiêu sẽ bị làm sai lệch: người ta tách thay đổi vô nghĩa để tăng con số hoặc né ghi nhận sự cố; nên dùng DORA để thấy nút thắt và so với chính mình theo thời gian, đo cả tốc độ lẫn độ ổn định
- Tần suất triển khai không liên quan tới hiệu quả phát hành
- Chỉ cần đo thêm số dòng mã là đủ
E: Đây là định luật Goodhart. DORA hữu ích để chẩn đoán quy trình của cả nhóm, nhưng dùng để chấm điểm cá nhân sẽ khuyến khích tối ưu con số thay vì kết quả. Bốn chỉ số cần xem cùng nhau.
