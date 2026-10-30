=== environments | Phát hành | Môi trường và quản lý cấu hình
>> Môi trường là nơi artifact chạy: dev, staging, production, và đôi khi môi trường tạm cho từng pull request. Điều quan trọng là các môi trường giống nhau ở mọi thứ trừ cấu hình.

## Các môi trường điển hình

| Môi trường | Mục đích | Dữ liệu | Triển khai |
|---|---|---|---|
| **Dev** | Phát triển, thử nhanh | Giả, tổng hợp | Tự động, mỗi merge |
| **Staging** (pre-prod) | Kiểm thử giống production trước khi phát hành | Giả hoặc ẩn danh | Tự động sau dev |
| **Production** | Người dùng thật | Thật | Có kiểm soát: duyệt, cửa sổ thời gian, hoặc tự động nếu đủ tin cậy |
| **Preview / ephemeral** | Xem thử một PR cụ thể | Giả | Tạo khi mở PR, xoá khi đóng |

Staging có ích khi nó **gần giống production** (cùng loại hạ tầng, cùng cấu hình, dữ liệu có hình dạng tương tự). Staging khác production nhiều thì "qua staging" không nói được gì. Với một số hệ thống, kiểm thử trực tiếp ở production bằng phát hành dần (canary) kèm quan sát còn đáng tin hơn staging.

## Cấu hình tách khỏi mã

Theo nguyên tắc của "twelve-factor": **cấu hình đổi theo môi trường nằm ngoài artifact**, nhận qua biến môi trường hoặc tệp cấu hình gắn vào lúc chạy.

| Loại | Ví dụ | Nơi để |
|---|---|---|
| Cấu hình không nhạy cảm | URL dịch vụ, mức log, số worker | Repo cấu hình (ConfigMap, Helm values theo môi trường) |
| Bí mật | Mật khẩu, khoá API | Kho bí mật (Vault, Secrets Manager), không trong Git ([Bí mật và OIDC](#secrets-oidc)) |
| Cờ tính năng | Bật/tắt chức năng | Hệ thống cờ tính năng (xem bên dưới) |

Mỗi môi trường có tệp giá trị riêng nhưng dùng **cùng artifact và cùng chart/manifest nền**:

```
deploy/
├── base/                    # manifest chung (Kustomize base hoặc chart)
└── envs/
    ├── dev/values.yaml      # replicas: 1, log: debug
    ├── staging/values.yaml  # replicas: 2
    └── prod/values.yaml     # replicas: 5, tài nguyên lớn hơn
```

Xem [Helm và Kustomize](https://tracviethustcs.github.io/k8s-study-notes/#helm).

## Quy tắc bảo vệ môi trường

- Production: chỉ triển khai từ **nhánh chính hoặc tag**, **bắt buộc người duyệt** (nếu là Continuous Delivery), nhật ký ai triển khai gì và khi nào.
- **Quyền riêng** cho từng môi trường: danh tính của pipeline dev không dùng được ở production.
- **Đồng bộ thứ tự**: không đưa lên production thứ chưa qua staging.
- **Khoá triển khai** để hai lần triển khai không chạy chồng nhau (`concurrency` ở Actions, `resource_group` ở GitLab).
- **Cửa sổ triển khai**: tránh giờ cao điểm hoặc cuối tuần với hệ thống chưa tự tin quay lui.

## Môi trường tạm cho mỗi pull request

Dựng một môi trường nhỏ cho mỗi PR (ví dụ một namespace Kubernetes) để review và kiểm thử trực quan, rồi xoá khi PR đóng:

```
on:
  pull_request:
    types: [opened, synchronize, reopened, closed]
jobs:
  preview:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy-preview.sh "pr-${{ github.event.number }}" "${{ github.sha }}"
  cleanup:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - run: ./destroy-preview.sh "pr-${{ github.event.number }}"
```

Nhớ giới hạn tài nguyên (ResourceQuota) và thời gian sống, nếu không chi phí tăng dần.

## Cờ tính năng

Cờ tính năng (feature flag) tách **triển khai** (mã đã nằm trên production) khỏi **phát hành** (người dùng thấy tính năng):

- Merge và triển khai mã chưa hoàn thiện sau một cờ đang tắt.
- Bật dần theo phần trăm người dùng, nhóm nội bộ, hoặc theo vùng.
- **Công tắc khẩn cấp (kill switch)**: tắt một tính năng lỗi ngay mà không cần triển khai lại.

Chi phí: cờ là mã điều kiện cần được kiểm thử ở cả hai trạng thái, và **phải dọn** khi tính năng đã ổn định (cờ cũ tích tụ thành nợ kỹ thuật). Công cụ: Unleash, OpenFeature (chuẩn mở), các dịch vụ thương mại.

::: lab Cùng một artifact, cấu hình khác nhau
```
mkdir -p ~/env-lab && cd ~/env-lab
mkdir -p base envs/dev envs/prod
printf 'resources: [deploy.yaml]\n' > base/kustomization.yaml
cat > base/deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: {name: web}
spec:
  replicas: 1
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
        - {name: web, image: registry.example.com/web:3f1a9c2, env: [{name: LOG_LEVEL, value: info}]}
EOF
printf 'resources: [../../base]\npatches:\n  - patch: |-\n      - op: replace\n        path: /spec/replicas\n        value: 5\n    target: {kind: Deployment, name: web}\n' > envs/prod/kustomization.yaml
printf 'resources: [../../base]\n' > envs/dev/kustomization.yaml
kubectl kustomize envs/dev | grep -E "replicas|image"
kubectl kustomize envs/prod | grep -E "replicas|image"
```
Quan sát: image giữ nguyên ở hai môi trường, chỉ `replicas` đổi. Đây là "một artifact, nhiều cấu hình".
:::

=== deploy-strategies | Phát hành | Chiến lược phát hành
>> Cách đưa một phiên bản mới thay phiên bản cũ quyết định rủi ro, gián đoạn và chi phí. Không có chiến lược "tốt nhất": chọn theo mức chịu rủi ro và khả năng của hạ tầng.

## Các chiến lược

| Chiến lược | Cách làm | Gián đoạn | Chi phí hạ tầng | Quay lui | Rủi ro |
|---|---|---|---|---|---|
| **Recreate** | Tắt bản cũ, bật bản mới | **Có** (khoảng chết) | Thấp | Chậm (triển khai lại) | Cao với dịch vụ trực tuyến |
| **Rolling** | Thay dần từng Pod/máy | Không (nếu probe đúng) | Thấp (dư một ít) | Trung bình (rolling ngược) | Lỗi lan dần trong lúc thay |
| **Blue-green** | Dựng song song bản mới (green), chuyển toàn bộ traffic từ cũ (blue) sang | Không | **Cao** (gấp đôi tạm thời) | **Rất nhanh** (chuyển lại) | Chuyển một lần toàn bộ: nếu lỗi, mọi người đều bị |
| **Canary** | Cho một phần nhỏ traffic (1%, 5%...) vào bản mới, tăng dần nếu ổn | Không | Thấp đến trung bình | Nhanh (rút traffic) | Thấp: chỉ một phần nhỏ người dùng chịu rủi ro |
| **Shadow / mirror** | Sao chép traffic thật sang bản mới nhưng không trả kết quả cho người dùng | Không | Cao | Không cần | Rất thấp, nhưng cần xử lý tác dụng phụ (ghi dữ liệu, gửi email) |
| **A/B test** | Chia người dùng theo nhóm để so sánh một chỉ số kinh doanh | Không | Trung bình | Dễ | Là kiểm thử sản phẩm, không phải kỹ thuật phát hành |

## Sơ đồ

```ascii
 BLUE-GREEN                                CANARY
 ----------                                ------
   Load balancer / Ingress                   Load balancer / Ingress
      |  (chuyển 100% một lần)                  |  95%            |  5%
      v                                          v                 v
  +--------+     +--------+                 +--------+        +--------+
  |  BLUE  |     | GREEN  |                 | v1 (cũ)|        | v2 (mới)|
  |  v1    |     |  v2    |                 +--------+        +--------+
  +--------+     +--------+                 quan sát chỉ số --> tăng 5% -> 25% -> 50% -> 100%
  bật green, kiểm thử, rồi đổi              (nếu chỉ số xấu: rút về 0%)
  đích của load balancer
```

## Triển khai trên Kubernetes

| Chiến lược | Cách thực hiện |
|---|---|
| Recreate | `strategy: {type: Recreate}` trong Deployment |
| Rolling | Mặc định của Deployment: `maxSurge`, `maxUnavailable`, cùng readiness probe ([Probe](https://tracviethustcs.github.io/k8s-study-notes/#ops-probes)) |
| Blue-green | Hai Deployment, một Service; đổi `selector` của Service (hoặc đổi backend của Ingress) sang bản mới. Công cụ: Argo Rollouts |
| Canary | Chia trọng số bằng Ingress controller (annotation canary), service mesh, Gateway API, hoặc Argo Rollouts/Flagger tự động hoá ([Progressive delivery](#progressive-delivery)) |

## Chọn chiến lược

| Nếu | Nên |
|---|---|
| Chấp nhận khoảng chết ngắn, ứng dụng nội bộ, tiết kiệm | Recreate |
| Dịch vụ web thông thường, thay đổi nhỏ, có probe tốt | Rolling |
| Cần quay lui tức thì và có ngân sách hạ tầng | Blue-green |
| Thay đổi rủi ro hoặc nhiều người dùng, có chỉ số tốt để quan sát | Canary |
| Muốn thử bản mới với traffic thật mà không ảnh hưởng người dùng | Shadow |

Hai điều kiện chung làm cho bất kỳ chiến lược nào an toàn: **tương thích ngược** giữa bản cũ và mới (hai bản chạy cùng lúc trong lúc chuyển, và dùng chung database: xem [Migration database](#db-migrations)), và **giám sát đủ tốt** để biết bản mới có đang hỏng không.

## Kết hợp với cờ tính năng

Triển khai (đưa mã lên) và phát hành (bật cho người dùng) là hai việc. Cờ tính năng cho phép phát hành dần, hoặc tắt ngay một tính năng lỗi mà không triển khai lại: thường nhanh hơn và nhỏ rủi ro hơn rollback cả phiên bản. Canary bảo vệ ở mức **phiên bản**, cờ bảo vệ ở mức **tính năng**; hai thứ bổ sung nhau.

## Những cái bẫy

- **Hai phiên bản cùng chạy** (rolling, canary) nghĩa là mọi thay đổi giao diện, hợp đồng message và schema phải chạy được với cả bản cũ lẫn mới.
- **Phiên (session) và trạng thái trong bộ nhớ** mất khi Pod bị thay: lưu bên ngoài.
- **Cache và client cũ**: client tải về trước đó có thể vẫn gọi giao diện cũ.
- **Blue-green** tốn gấp đôi tài nguyên trong lúc chuyển và dữ liệu (database) vẫn chỉ có một: nó không quay lui được những thay đổi dữ liệu đã làm.
- **Canary không có chỉ số tốt** chỉ là rolling chậm: nếu không đo tỷ lệ lỗi và độ trễ, không có gì để quyết định tăng hay rút.

::: lab Rolling và blue-green trên cụm cục bộ
```
kind create cluster --name cd-lab                      # hoặc dùng cụm bất kỳ
kubectl create deployment web --image=nginx:1.26 --replicas=3
kubectl set image deployment/web nginx=nginx:1.27 && kubectl rollout status deployment/web   # rolling
kubectl rollout undo deployment/web                    # quay lui rolling

# blue-green bằng đổi selector của Service (mỗi Deployment tự có nhãn app=<tên>)
kubectl create deployment blue  --image=nginx:1.26
kubectl create deployment green --image=nginx:1.27
kubectl rollout status deployment/blue && kubectl rollout status deployment/green
kubectl expose deployment blue --name=svc --port=80 --selector=app=blue
kubectl get endpoints svc
kubectl patch service svc -p '{"spec":{"selector":{"app":"green"}}}'   # chuyển 100% traffic sang green
kubectl get endpoints svc
kind delete cluster --name cd-lab
```
Quan sát: danh sách endpoint của Service đổi thế nào khi bạn đổi selector? So với rolling, việc chuyển và quay lui nhanh hơn hay chậm hơn?
:::

=== progressive-delivery | Phát hành | Progressive delivery: canary tự động
>> Progressive delivery là phát hành dần theo các bước, kiểm tra chỉ số ở mỗi bước và tự rút lại khi chỉ số xấu. Trên Kubernetes, Argo Rollouts và Flagger làm việc đó.

## Ý tưởng

Thay vì "bật bản mới cho mọi người rồi hy vọng", ta làm:

1. Cho **một phần nhỏ** traffic vào bản mới.
2. **Đo** các chỉ số (tỷ lệ lỗi, độ trễ, chỉ số nghiệp vụ) trong một khoảng thời gian.
3. Nếu **đạt ngưỡng**, tăng tỷ lệ; nếu **không đạt**, **tự động quay lui**.
4. Lặp lại tới 100%.

Quyết định dựa trên **số liệu thật**, không phải cảm giác. Điều này đòi hỏi bạn đã có chỉ số tốt (Prometheus, [giám sát](https://tracviethustcs.github.io/k8s-study-notes/#tool-observe)) và SLO rõ ràng.

## Argo Rollouts

Argo Rollouts thêm resource `Rollout` thay cho `Deployment`, hỗ trợ canary và blue-green có phân tích chỉ số.

```
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: {name: web}
spec:
  replicas: 5
  selector: {matchLabels: {app: web}}
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - analysis:
            templates:
              - templateName: success-rate
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
        - name: web
          image: registry.example.com/web:3f1a9c2
```

Phân tích chỉ số khai báo bằng `AnalysisTemplate`, ví dụ truy vấn Prometheus:

```
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: {name: success-rate}
spec:
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] >= 0.99
      failureLimit: 1
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{app="web",code!~"5.."}[2m]))
            /
            sum(rate(http_requests_total{app="web"}[2m]))
```

Nếu tỷ lệ thành công thấp hơn 99% vượt giới hạn lỗi, Rollout **tự động huỷ** và đưa traffic về bản ổn định. Việc chia traffic thực sự theo trọng số cần ingress controller hoặc service mesh được hỗ trợ (nếu không, Rollouts chia theo số Pod). Tên trường và hành vi theo phiên bản: xem tài liệu của dự án.

## Flagger

Flagger làm việc tương tự nhưng giữ nguyên `Deployment` và tạo các thành phần phụ trợ; tích hợp nhiều với service mesh và ingress. Chọn giữa hai công cụ theo hệ sinh thái bạn dùng (Argo Rollouts đi cùng Argo CD; Flagger đi cùng Flux).

## Thiết kế chỉ số cho cổng phát hành

| Loại chỉ số | Ví dụ | Lưu ý |
|---|---|---|
| Tỷ lệ lỗi | % phản hồi 5xx | So sánh canary với bản ổn định, không chỉ so với ngưỡng cố định |
| Độ trễ | p95, p99 | Cần đủ lượng request để thống kê có nghĩa |
| Bão hoà | CPU, bộ nhớ, hàng đợi | Phát hiện rò rỉ chậm |
| Nghiệp vụ | Tỷ lệ thanh toán thành công, đăng nhập | Bắt lỗi mà kỹ thuật không thấy |

Những lưu ý hay bị bỏ sót:

- **Lượng traffic đủ lớn**: 5% của một dịch vụ ít request không cho thống kê có ý nghĩa trong 5 phút. Chọn thời gian và tỷ lệ theo lưu lượng.
- **Đo canary tách riêng**: chỉ số phải tách được traffic của bản mới (nhãn `version` hoặc `rollouts-pod-template-hash`), nếu không bản lỗi bị trộn với bản tốt.
- **Cổng nên bảo thủ**: ngưỡng quá chặt gây rollback giả (nhiễu), quá lỏng thì bỏ lọt lỗi.
- Luôn có **đường quay lui thủ công** phòng khi tự động hoá lỗi.

## Khi nào không cần

Nếu dịch vụ chưa có chỉ số đáng tin cậy, hoặc lượng traffic quá nhỏ, progressive delivery tự động chưa mang lại giá trị: hãy bắt đầu bằng rolling update tốt, probe đúng và giám sát, rồi nâng cấp dần.

::: lab Thiết kế cổng canary trên giấy
Cho một API có 200 request/phút. Bạn muốn canary 10% trong 5 phút, cổng là "tỷ lệ 5xx của canary nhỏ hơn 1%".

1. Canary nhận bao nhiêu request trong 5 phút? (10% của 200/phút × 5 phút)
2. Với số request đó, một lỗi đơn lẻ đã đủ gây ra tỷ lệ bao nhiêu? Cổng 1% còn ý nghĩa không?
3. Bạn sẽ đổi thiết kế thế nào (thời gian dài hơn, tỷ lệ cao hơn, ngưỡng theo số lượng lỗi, chỉ số khác)?
:::

=== gitops-cd | Phát hành | GitOps: CD kiểu khai báo
>> GitOps lấy Git làm nguồn sự thật cho trạng thái mong muốn của hệ thống. Một bộ điều khiển trong cụm liên tục kéo trạng thái đó về và kéo cụm khớp với nó, thay vì pipeline CI đẩy thay đổi vào cụm.

## Đẩy và kéo

| | CD kiểu đẩy (push) | GitOps kiểu kéo (pull) |
|---|---|---|
| Ai triển khai | Pipeline CI chạy `kubectl apply` hoặc `helm upgrade` vào cụm | Bộ điều khiển **trong cụm** (Flux, Argo CD) kéo từ Git và áp dụng |
| Thông tin xác thực | Pipeline cần quyền vào cụm (kubeconfig, token) | Cụm chỉ cần **quyền đọc** Git; **không** có thông tin đăng nhập cụm trong CI |
| Phát hiện drift | Không (chỉ biết lúc chạy pipeline) | **Liên tục**: tự sửa hoặc báo khi ai đó đổi tay |
| Lịch sử, kiểm toán | Nhật ký pipeline | Lịch sử Git: ai, khi nào, vì sao (review qua PR) |
| Quay lui | Chạy lại pipeline với bản cũ | **`git revert`**: bộ điều khiển tự đưa cụm về trạng thái cũ |

Điểm mấu chốt về bảo mật: bề mặt tấn công nhỏ hơn vì CI không giữ khoá vào production.

## Tách CI và CD, và tách repo

```ascii
  Repo ỨNG DỤNG                         Registry               Repo CẤU HÌNH (GitOps)                    CỤM
  --------------                        --------               ----------------------                    ----
  commit -> CI: test, build ->  push image :3f1a9c2   -->   CI mở PR/commit: đổi tag thành 3f1a9c2  -->  Flux/Argo CD
                                                            (envs/dev/values.yaml)                        phát hiện thay đổi,
                                                                    |                                      kéo và áp dụng
                                                                    v  review/merge                        vào cụm
                                                            Git là trạng thái mong muốn  <-------- đối chiếu liên tục
```

- **CI** (repo ứng dụng): kiểm thử, build, đẩy image, rồi **cập nhật tag** trong repo cấu hình.
- **CD** (bộ điều khiển trong cụm): thấy repo cấu hình đổi và đồng bộ cụm.
- **Hai repo riêng** (mã ứng dụng, cấu hình triển khai) tránh vòng lặp "commit cấu hình kích hoạt CI" và cho phép quyền khác nhau (lập trình viên đẩy mã, chỉ pipeline/nhóm nền tảng đổi cấu hình production).

## Cập nhật tag image trong repo cấu hình

| Cách | Mô tả |
|---|---|
| Bước trong CI | CI sau khi đẩy image chạy `kustomize edit set image ...` hoặc `yq`, rồi commit/mở PR vào repo cấu hình |
| **Image automation** của Flux | Flux theo dõi registry và tự cập nhật manifest trong Git khi có tag mới khớp chính sách (semver, regex) |
| Argo CD Image Updater | Tương tự cho Argo CD |
| Renovate | Mở PR cập nhật phiên bản image/chart, có thể gộp tự động theo quy tắc |

```
# trong CI của repo ứng dụng, sau khi đẩy image
git clone https://github.com/myorg/deploy-config.git && cd deploy-config
cd envs/dev && kustomize edit set image web=registry.example.com/web:${GITHUB_SHA}
git commit -am "dev: web -> ${GITHUB_SHA::7}" && git push
```

## Promotion giữa các môi trường

Đưa một phiên bản lên môi trường kế tiếp là **thay đổi trong repo cấu hình**:

- Dev: tự động cập nhật tag mỗi merge.
- Staging: PR tự động hoặc bán tự động sao chép tag từ dev sau khi kiểm thử xanh.
- Production: PR **cần người duyệt** (đó là cổng Continuous Delivery), merge là phát hành.

Mỗi môi trường là một thư mục (hoặc nhánh) trong repo cấu hình. Thư mục thường dễ theo dõi hơn nhánh vì thấy được cả ba môi trường cùng lúc.

## Công cụ và thiết lập

| Công cụ | Ghi chú |
|---|---|
| **Flux** | Bộ công cụ GitOps theo từng thành phần (source, kustomize, helm, image automation) |
| **Argo CD** | Giao diện web mạnh, mô hình `Application`, theo dõi trạng thái đồng bộ trực quan |

Cả hai đều theo dõi trạng thái và đồng bộ lại khi lệch. Chi tiết cài đặt, `Kustomization`, `HelmRelease` ở [GitOps: Argo CD và Flux](https://tracviethustcs.github.io/k8s-study-notes/#gitops) và [GitOps bootstrap với Flux](https://tracviethustcs.github.io/k8s-study-notes/#setup-gitops).

## Bí mật trong GitOps

Git không chứa bí mật dạng rõ. Hướng thường dùng: lưu **tham chiếu** trong Git và để Vault Secrets Operator hoặc External Secrets nạp bí mật vào cụm; hoặc mã hoá trong Git bằng Sealed Secrets hay SOPS ([Secrets trong cụm](https://tracviethustcs.github.io/k8s-study-notes/#tool-secrets)).

## Lưu ý vận hành

- **Đồng bộ tự động và `prune`**: bật tự động đồng bộ cho dev; với production cân nhắc đồng bộ thủ công hoặc yêu cầu duyệt, và hiểu hành vi xoá tài nguyên không còn trong Git.
- **Thứ tự phụ thuộc**: CRD và operator trước, resource dùng chúng sau (`dependsOn` của Flux, sync wave của Argo CD).
- **Giám sát chính bộ điều khiển**: cảnh báo khi đồng bộ thất bại hay lệch kéo dài.
- **Hạn chế thay đổi tay** ngoài Git (`kubectl edit` ở production): bộ điều khiển sẽ ghi đè, và đó chính là mục đích.

::: lab Mô phỏng vòng lặp GitOps bằng Git cục bộ
```
mkdir -p ~/gitops-lab && cd ~/gitops-lab && rm -rf cfg && git init -q cfg && cd cfg
mkdir -p envs/dev envs/prod
echo 'image: registry.example.com/web:aaaaaaa' > envs/dev/values.yaml
echo 'image: registry.example.com/web:aaaaaaa' > envs/prod/values.yaml
git add . && git -c user.name=t -c user.email=t@t commit -qm "ban dau"
# CI "đẩy" bản mới vào dev bằng một commit
sed -i 's/aaaaaaa/bbbbbbb/' envs/dev/values.yaml && git -c user.name=ci -c user.email=ci@t commit -qam "dev: web -> bbbbbbb"
# promotion lên prod = một thay đổi có review
sed -i 's/aaaaaaa/bbbbbbb/' envs/prod/values.yaml && git -c user.name=ops -c user.email=ops@t commit -qam "prod: web -> bbbbbbb (đã duyệt)"
git log --oneline
git revert --no-edit HEAD -q && cat envs/prod/values.yaml     # quay lui = git revert
```
Quan sát: lịch sử Git cho thấy ai đổi gì, khi nào. Sau `git revert`, trạng thái mong muốn của prod về bản cũ; trong GitOps, bộ điều khiển sẽ tự kéo cụm về trạng thái đó.
:::

=== db-migrations | Phát hành | Thay đổi database trong phát hành liên tục
>> Mã có thể quay lui trong vài giây; dữ liệu thì không. Thay đổi schema là phần rủi ro nhất của việc phát hành, và cần cách làm riêng để vẫn triển khai thường xuyên và an toàn.

## Vấn đề

Khi triển khai cuốn chiếu hoặc canary, **bản cũ và bản mới chạy cùng lúc trên cùng database**. Nếu migration đổi tên cột hay xoá bảng một lần, bản cũ đang chạy sẽ hỏng ngay. Và nếu bản mới có lỗi, quay lui mã không hoàn tác được thay đổi schema hay dữ liệu.

Hệ quả: schema luôn phải **tương thích với cả phiên bản cũ lẫn mới** trong thời gian chuyển đổi.

## Mẫu expand / contract (parallel change)

Chia một thay đổi phá vỡ thành nhiều bước nhỏ, mỗi bước tương thích ngược và triển khai riêng:

```ascii
  Mục tiêu: đổi tên cột  users.name  ->  users.full_name

  1. EXPAND     thêm cột mới full_name (cho phép NULL)            | bản cũ vẫn chạy
  2. DEPLOY     bản mới GHI vào cả name và full_name, ĐỌC từ name  | tương thích cả hai
  3. BACKFILL   sao chép dữ liệu cũ sang full_name (theo lô)       | chạy nền, kiểm tra
  4. DEPLOY     bản mới ĐỌC từ full_name (vẫn ghi cả hai)         | chuyển dần
  5. DEPLOY     bản mới chỉ dùng full_name                         | không còn tham chiếu name
  6. CONTRACT   xoá cột name                                       | bước phá vỡ cuối, rất muộn
```

Nhiều bước, nhưng mỗi bước **an toàn và quay lui được**; việc xoá thật sự (bước cuối) chỉ làm khi chắc chắn không còn gì dùng nó.

## Quy tắc cho migration an toàn

| Nên | Tránh (trong cùng một lần phát hành) |
|---|---|
| Thêm cột cho phép NULL hoặc có giá trị mặc định | Đổi tên hoặc xoá cột đang được bản cũ dùng |
| Thêm bảng, thêm chỉ mục (tạo **đồng thời** nếu hỗ trợ, ví dụ `CREATE INDEX CONCURRENTLY` trong PostgreSQL) | Thêm cột `NOT NULL` không mặc định trên bảng lớn (khoá bảng) |
| Backfill theo **lô nhỏ** trong tiến trình nền | `UPDATE` cả bảng lớn trong một giao dịch |
| Tách thay đổi schema và thay đổi mã thành các bản phát hành khác nhau | Gộp migration phá vỡ và mã mới trong cùng một lần triển khai |
| Kiểm thử migration trên dữ liệu có kích thước gần với thật | Chạy lần đầu trên production |
| Đặt `lock_timeout` để migration không treo chặn cả hệ thống | Giữ khoá bảng lâu |

Trên bảng lớn, nhiều lệnh `ALTER` có thể khoá bảng và chặn mọi truy vấn: biết lệnh nào khoá trên database của bạn (và công cụ thay đổi schema online như `gh-ost`, `pt-online-schema-change` cho MySQL).

## Chạy migration trong pipeline

| Cách | Ghi chú |
|---|---|
| **Job riêng trước khi triển khai** (Kubernetes Job, Helm hook `pre-upgrade`, Argo CD `PreSync`) | Phổ biến: migration chạy một lần, xong mới rolling update |
| Khi ứng dụng khởi động (mỗi Pod tự migrate) | Dễ nhưng nguy hiểm: nhiều Pod chạy đồng thời; cần khoá phân tán |
| Bước thủ công cho thay đổi lớn | Hợp lý cho migration nặng, có người theo dõi |

Công cụ quản lý migration có phiên bản: **Flyway**, **Liquibase**, **Alembic** (Python), Prisma/Knex/Rails/Django migrations. Chúng ghi lại migration nào đã chạy, để chạy lặp lại an toàn (idempotent theo phiên bản).

```
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-3f1a9c2
  annotations:
    argocd.argoproj.io/hook: PreSync          # chạy trước khi đồng bộ ứng dụng (Argo CD)
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: registry.example.com/web:3f1a9c2
          command: ["./migrate", "--up"]
```

## Quay lui

- **Mã**: quay lui nhanh (phiên bản cũ), miễn là schema vẫn tương thích: đó là lý do dùng expand/contract.
- **Schema**: ưu tiên **tiến lên (roll forward)** bằng migration sửa lỗi, vì migration "xuống" (down) hiếm khi an toàn với dữ liệu đã phát sinh.
- **Dữ liệu**: trước thay đổi rủi ro, có **bản sao lưu hoặc snapshot** vừa lấy và đã biết cách khôi phục ([Backup](https://tracviethustcs.github.io/linux-network-notes/#lx-backup)); với database lớn, xem xét chạy trên bản sao trước.
- **Cờ tính năng** giúp tắt đường mã mới đụng tới cột mới mà không cần đổi schema.

::: lab Thử expand/contract với SQLite
```
cd ~ && rm -f mig.db
sqlite3 mig.db "create table users(id integer primary key, name text); insert into users(name) values ('an'),('binh');"
# 1. EXPAND: thêm cột mới, cho phép NULL
sqlite3 mig.db "alter table users add column full_name text;"
# 2-3. Bản mới ghi cả hai, backfill dữ liệu cũ
sqlite3 mig.db "update users set full_name = name where full_name is null;"
# bản CŨ vẫn đọc được name bình thường:
sqlite3 mig.db "select id, name from users;"
# bản MỚI đọc full_name:
sqlite3 mig.db "select id, full_name from users;"
# 6. CONTRACT (chỉ sau khi không còn gì dùng name)
sqlite3 mig.db "alter table users drop column name;"
sqlite3 mig.db "select * from users;"
```
Tại bước nào nếu xoá cột `name` quá sớm thì bản cũ đang chạy sẽ hỏng? Vì sao chia thành nhiều bước lại cho phép quay lui ở mỗi bước?
:::

=== rollback | Phát hành | Rollback và khôi phục
>> Mọi phát hành đều có thể hỏng. Khả năng khôi phục nhanh quan trọng hơn việc cố không bao giờ hỏng: nó quyết định bạn mất vài phút hay vài giờ.

## Quay lui hay tiến lên

| | Rollback (quay lui) | Roll forward (tiến lên) |
|---|---|---|
| Cách | Triển khai lại **phiên bản trước** | Sửa lỗi và phát hành bản mới |
| Tốc độ | Thường nhanh nhất | Tuỳ thời gian sửa và pipeline |
| Khi hợp | Lỗi rõ ràng trong bản mới, schema vẫn tương thích | Đã có thay đổi dữ liệu/schema không đảo ngược, hoặc bản sửa nhỏ và nhanh |
| Rủi ro | Phiên bản cũ có thể không còn tương thích với dữ liệu mới | Sửa vội có thể thêm lỗi mới |

Quy tắc thực tế: **giảm thiệt hại trước** (quay lui, tắt cờ, rút canary), tìm nguyên nhân gốc sau.

## Các cơ chế quay lui

| Cơ chế | Cách làm | Tốc độ |
|---|---|---|
| **Cờ tính năng / công tắc khẩn cấp** | Tắt tính năng lỗi | Giây |
| **Rút canary / đổi lại Service** | Đưa traffic về bản ổn định (Argo Rollouts, blue-green) | Giây đến phút |
| **Rollout ngược của Kubernetes** | `kubectl rollout undo deployment/web` | Phút |
| **GitOps: revert** | `git revert` commit triển khai; Flux/Argo CD đưa cụm về trạng thái cũ | Phút |
| **Triển khai lại artifact cũ** | Chạy lại pipeline với tag/digest của bản trước (artifact còn nguyên nhờ chính sách giữ lại) | Phút |
| **Khôi phục dữ liệu** | Khôi phục từ snapshot/backup | Phút đến giờ |

```
kubectl rollout history deployment/web
kubectl rollout undo deployment/web                       # về bản trước
kubectl rollout undo deployment/web --to-revision=3
kubectl rollout status deployment/web
```

Với GitOps, **đừng** dùng `kubectl rollout undo` làm cách chính: bộ điều khiển sẽ đưa cụm về lại trạng thái trong Git. Quay lui đúng cách là `git revert` trong repo cấu hình.

## Điều kiện để quay lui thật sự hoạt động

- **Artifact cũ còn tồn tại** và truy cập được (chính sách giữ lại không xoá bản đã phát hành).
- **Schema tương thích** hai chiều ([Migration database](#db-migrations)).
- **Cấu hình và bí mật** của bản cũ vẫn có.
- Quy trình quay lui đã **được thử**, không phải lần đầu thực hiện giữa sự cố. Diễn tập định kỳ.
- Người trực biết **ngưỡng quyết định**: lỗi bao nhiêu hoặc kéo dài bao lâu thì quay lui (không đợi "xem thêm một chút").

## Tự động quay lui

Cổng phát hành dựa trên chỉ số có thể **tự quay lui** khi tỷ lệ lỗi hoặc độ trễ vượt ngưỡng ([Progressive delivery](#progressive-delivery)). Với triển khai thường, probe `readiness` thất bại khiến rolling update dừng lại thay vì thay hết Pod, và `progressDeadlineSeconds` đánh dấu Deployment thất bại để cảnh báo/quay lui.

## Khi sự cố xảy ra

1. **Phát hiện**: cảnh báo từ giám sát, không phải từ người dùng.
2. **Chẩn đoán nhanh**: có phải do thay đổi vừa triển khai không? (Đối chiếu thời điểm lỗi với bản ghi triển khai.)
3. **Giảm thiệt hại**: quay lui hoặc tắt cờ ngay nếu nghi do thay đổi.
4. **Xác nhận khôi phục**: chỉ số trở lại bình thường.
5. **Giao tiếp**: thông báo cho người liên quan.
6. **Sau sự cố**: viết postmortem không đổ lỗi, ghi nguyên nhân gốc, điều gì hoạt động và không, và hành động cải thiện có người chịu trách nhiệm.

Gắn **dấu mốc triển khai** (deploy marker) lên biểu đồ giám sát giúp thấy ngay lỗi bắt đầu sau lần triển khai nào ([DORA và đo lường](#metrics-dora)).

::: lab Diễn tập quay lui trong cụm thử
```
kind create cluster --name rb-lab
kubectl create deployment web --image=nginx:1.26 --replicas=2
kubectl rollout status deployment/web
kubectl set image deployment/web nginx=nginx:khong-ton-tai          # bản "hỏng"
sleep 15; kubectl get pods; kubectl rollout status deployment/web --timeout=20s
kubectl rollout history deployment/web
kubectl rollout undo deployment/web && kubectl rollout status deployment/web
kubectl get pods
kind delete cluster --name rb-lab
```
Quan sát: rolling update với image hỏng có làm cả Pod cũ biến mất không? Vì sao ứng dụng vẫn phục vụ được trong lúc bản mới lỗi?
:::
