# GitOps repo — cheatsheet

Cluster: minikube, Flux bootstrap path `clusters/production`, repo `khanhdx-3810/gitops` (branch `main`).

## 1. Bootstrap trên máy/cluster mới (clone repo về lần đầu)

```bash
# Cài flux CLI (nếu chưa có)
curl -s https://fluxcd.io/install.sh | sudo bash

# Export GitHub token (cần quyền tạo deploy key / repo)
export GITHUB_TOKEN=<personal-access-token>

# Bootstrap: cài Flux controllers lên cluster + tạo deploy key + sync với path đang dùng
flux bootstrap github \
  --owner=khanhdx-3810 \
  --repository=gitops \
  --branch=main \
  --path=clusters/production \
  --personal \
  --read-write-key=true
```

`--read-write-key=true` là bắt buộc cho repo này — deploy key phải có quyền **write** vì
`ImageUpdateAutomation` (`web-api-production`, `web-api-staging`) tự commit + push ảnh mới
lên branch `flux-image-updates`/`main`. Thiếu flag này, bootstrap tạo deploy key read-only
(mặc định) và mọi lần push của image-automation-controller sẽ fail permission denied.
Verify lại quyền deploy key: `gh api repos/khanhdx-3810/gitops/keys` → field `"read_only": false`.

Lệnh bootstrap idempotent — chạy lại trên repo/cluster đã bootstrap chỉ để đồng bộ lại
`clusters/production/flux-system/`, không phá dữ liệu, không tạo deploy key thứ 2 nếu key cũ
còn hợp lệ.

## 2. Tạo SOPS age key (giải mã secret .enc.yaml)

```bash
# Cài age
sudo apt install age    # hoặc: brew install age

# Sinh key mới, lưu vào đúng đường dẫn mặc định sops/age tự dò (KHÔNG lưu trong repo này)
# Linux:   ~/.config/sops/age/keys.txt
# macOS:   ~/Library/Application Support/sops/age/keys.txt
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
# In ra "Public key: age1..." — copy giá trị này

# Đưa public key vào .sops.yaml (recipient) để sops biết mã hoá cho ai
# .sops.yaml đã có sẵn:
#   creation_rules:
#     - path_regex: .*\.enc\.yaml$
#       age: age1ml87v52sj3sfqw9um5ywmu6cxlymv6n9nx4laqtlaxmny0j70y7qcvw3mn

# Tạo secret cho Flux (kustomize-controller) dùng để giải mã khi apply trên cluster
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=~/.config/sops/age/keys.txt

# QUAN TRỌNG:
# - keys.txt chứa PRIVATE key — không commit vào git, không để trong thư mục repo.
# - Đặt ở ~/.config/sops/age/keys.txt để lệnh `sops` tự tìm ra khi sửa file .enc.yaml,
#   khỏi phải set SOPS_AGE_KEY_FILE mỗi lần.
# - Mỗi máy/người cần giải mã (sửa secret) đều phải có riêng file keys.txt này —
#   chia sẻ qua kênh bảo mật (password manager, Vault...), không qua Slack/email.
```

Kustomization CR nào apply thư mục có file `*.enc.yaml` thì phải khai thêm block sau (xem `clusters/production/infrastructure.yaml`, `infrastructure-shared.yaml`):

```yaml
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

### Mã hoá / sửa 1 secret

```bash
# Tạo mới hoặc sửa file .enc.yaml (sops tự đọc .sops.yaml theo path_regex)
sops infrastructure/production/harbor-creds.enc.yaml

# Mã hoá 1 file plaintext có sẵn thành .enc.yaml
sops --encrypt --in-place infrastructure/shared/slack-webhook.enc.yaml
```

## 3. Flux — xem trạng thái

```bash
flux get kustomizations           # trạng thái apply từng thư mục (infra/apps)
flux get sources git              # trạng thái GitRepository
flux get helmreleases -A          # trạng thái HelmRelease mọi namespace
flux get images all               # ImageRepository + ImagePolicy + ImageUpdateAutomation
flux get images update            # riêng ImageUpdateAutomation
```

## 4. Flux — ép reconcile ngay (bỏ qua interval)

```bash
# Kustomization (kéo git mới nhất rồi apply lại)
flux reconcile kustomization apps --with-source
flux reconcile kustomization apps-staging --with-source
flux reconcile kustomization infrastructure --with-source
flux reconcile kustomization infrastructure-shared --with-source

# HelmRelease (chỉ hiệu quả nếu chart/values thực sự đổi, xem mục 6)
flux reconcile helmrelease demo-api -n production              # chart CŨ
flux reconcile helmrelease demo-api -n production --with-source # hỏi lại registry
flux reconcile helmrelease demo-api -n staging

# Image automation pipeline (ImageRepository -> ImagePolicy -> ImageUpdateAutomation)
flux reconcile image repository web-api -n flux-system
flux reconcile image policy web-api-production -n flux-system
flux reconcile image policy web-api-staging -n flux-system
flux reconcile image update web-api-production -n flux-system
flux reconcile image update web-api-staging -n flux-system

# Lấy lại chart mới nhất
flux reconcile source chart production-demo-api -n flux-system
```

## 5. Xem log và debug

### 5.1 6 controller làm gì

| Controller | Quản lý CRD | Log có gì | Xem khi nào |
|---|---|---|---|
| `source-controller` | `GitRepository`, `HelmRepository`, `OCIRepository`, `HelmChart` | Clone Git, tải chart, tạo Artifact, lỗi xác thực | Repo không kéo được, chart không tải được, sai credential |
| `kustomize-controller` | `Kustomization` | Build Kustomize, apply, prune, giải mã SOPS, health check | `kustomize build failed`, object không apply, Secret ra `ENC[...]` |
| `helm-controller` | `HelmRelease` | `helm install/upgrade/rollback`, drift detection, remediation | Upgrade thất bại, rollback tự động, drift |
| `notification-controller` | `Provider`, `Alert`, `Receiver` | Gửi Slack, nhận webhook, xác minh chữ ký | Slack không nhận tin, webhook trả 401/404 |
| `image-reflector-controller` | `ImageRepository`, `ImagePolicy` | Quét registry, lọc tag, chọn tag | `unauthorized to list tags`, policy chọn sai tag |
| `image-automation-controller` | `ImageUpdateAutomation` | Tìm marker, sửa file, commit, push | Không commit gì, `permission denied` khi push |

### 5.2 Lệnh xem log

| Lệnh | Dùng khi |
|---|---|
| `flux logs --kind=HelmRelease --name=demo-api -n production` | Lọc log theo đúng một object — **cách nhanh nhất** |
| `flux logs --level=error --since=15m` | Chỉ xem lỗi gần đây |
| `flux logs --follow --tail=200` | Theo dõi liên tục, mọi controller |
| `flux logs --all-namespaces` | Khi có Flux ở nhiều namespace |
| `kubectl -n flux-system logs deploy/<controller> --tail=100 -f` | Xem thô log một controller cụ thể |

> `flux logs` mặc định chỉ lấy 10 dòng cuối mỗi controller — nhớ thêm `--tail` nếu thấy trống.

### 5.3 Log từng controller

| Controller | Lệnh |
|---|---|
| source | `kubectl -n flux-system logs deploy/source-controller --tail=100 -f` |
| kustomize | `kubectl -n flux-system logs deploy/kustomize-controller --tail=100 -f` |
| helm | `kubectl -n flux-system logs deploy/helm-controller --tail=100 -f` |
| notification | `kubectl -n flux-system logs deploy/notification-controller --tail=100 -f` |
| image-reflector | `kubectl -n flux-system logs deploy/image-reflector-controller --tail=100 -f` |
| image-automation | `kubectl -n flux-system logs deploy/image-automation-controller --tail=100 -f` |

### 5.4 Xem trạng thái và event của một object

| Lệnh | Cho biết |
|---|---|
| `flux get all -A --status-selector ready=false` | **Chạy đầu tiên** — cái gì đang lỗi |
| `kubectl describe helmrelease demo-api -n production` | Điều kiện Ready/Released/Drifted + event gần nhất |
| `flux events --for HelmRelease/demo-api -n production` | Dòng thời gian của một object |
| `kubectl get events -n production --sort-by=.lastTimestamp` | Mọi event trong namespace |
| `flux trace deployment demo-api -n production` | Object này do ai tạo, từ chart/commit nào |
| `flux tree helmrelease demo-api -n production` | `HelmRelease` này quản lý những object nào |

### 5.5 Chọn controller theo triệu chứng

| Triệu chứng | Xem log |
|---|---|
| `flux get sources git` đỏ | `source-controller` |
| `no chart version found` | `source-controller` |
| `kustomize build failed` | `kustomize-controller` |
| Secret ra `ENC[...]` thay vì giá trị thật | `kustomize-controller` |
| `HelmRelease` kẹt `InProgress`, hoặc tự rollback | `helm-controller` |
| Sửa tay không bị ghi đè | `helm-controller` |
| Slack im lặng | `notification-controller` |
| Webhook GitHub trả 401/404 | `notification-controller` |
| Push tag mới mà `flux get images policy` không đổi | `image-reflector-controller` |
| Policy chọn đúng tag nhưng không có commit nào | `image-automation-controller` |

### 5.6 Quy trình debug theo chặng

Đi từ trên xuống, tìm chặng **đầu tiên** đứt.

| # | Chặng | Lệnh kiểm tra |
|---|---|---|
| 1 | Tổng quan | `flux get all -A --status-selector ready=false` |
| 2 | Repo về chưa | `flux get sources git` (so với `git rev-parse --short HEAD`) |
| 3 | Chart về chưa | `flux get sources helm` và `flux get sources chart` |
| 4 | Manifest apply chưa | `flux get kustomizations` |
| 5 | Helm chạy chưa | `flux get helmreleases -A` |
| 6 | Pod chạy chưa | `kubectl -n <ns> get pod` và `describe pod` |
## 6. GitHub Actions — PR tự động cho image update

`.github/workflows/image-update-pr.yml` chạy khi có push lên branch `flux-image-updates`
(nhánh mà `ImageUpdateAutomation` push commit tới, xem mục 1), tự mở PR `flux-image-updates -> main`
bằng `gh pr create` với `GITHUB_TOKEN` mặc định.

**Gotcha:** dù workflow đã khai `permissions: pull-requests: write`, GitHub vẫn chặn với lỗi:
```
pull request create failed: GraphQL: GitHub Actions is not permitted to create or approve pull requests
```
vì có 1 công tắc riêng ở cấp **repo** (không nằm trong workflow yaml): Settings → Actions → General →
Workflow permissions → "Allow GitHub Actions to create and approve pull requests" — mặc định **tắt**.
Bật bằng CLI (repo cá nhân, chỉ 1 workflow dùng quyền này nên an toàn để bật):

```bash
gh api -X PUT repos/khanhdx-3810/gitops/actions/permissions/workflow \
  -f default_workflow_permissions=read \
  -F can_approve_pull_request_reviews=true

# verify
gh api repos/khanhdx-3810/gitops/actions/permissions/workflow
# → {"default_workflow_permissions":"read","can_approve_pull_request_reviews":true}
```

Test lại không cần đợi image mới: `gh run rerun <run-id>` (lấy run-id từ `gh run list`), hoặc push nhẹ
1 commit lên `flux-image-updates`.

## 7. Ghi nhớ hành vi hay gây nhầm lẫn

- `flux reconcile helmrelease` ép chạy lại reconcile ngay, nhưng nếu chart/values **không đổi** so với release đang chạy thì helm-controller coi là no-op — không có `helm upgrade` nào chạy, không có Event mới, không có Slack alert.
- Version chart không tồn tại trên registry → Flux **không** áp version lỗi lên cluster. Nó dừng ở bước pull chart (`HelmChart` object fail, `Ready=False reason=SourceNotReady`), release cũ trong cluster vẫn chạy nguyên, an toàn (fail-closed).
- Lỗi chart pull xảy ra ở object con `HelmChart` (namespace `flux-system`), không phải ở `HelmRelease` — nếu muốn Alert bắt được lỗi này, `eventSources` phải khai thêm `kind: HelmChart, namespace: flux-system` (xem `infrastructure/shared/notification-alert.yaml`).
