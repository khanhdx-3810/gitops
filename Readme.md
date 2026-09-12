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
flux reconcile helmrelease demo-api -n production
flux reconcile helmrelease demo-api -n staging

# Image automation pipeline (ImageRepository -> ImagePolicy -> ImageUpdateAutomation)
flux reconcile image repository web-api -n flux-system
flux reconcile image policy web-api-production -n flux-system
flux reconcile image policy web-api-staging -n flux-system
flux reconcile image update web-api-production -n flux-system
flux reconcile image update web-api-staging -n flux-system
```

## 5. Xem log

```bash
# Cách nhanh nhất: flux CLI tự lọc log theo object (mọi controller)
flux logs --kind=HelmRelease --name=demo-api --namespace=production
flux logs --kind=Kustomization --name=infrastructure
flux logs --all-namespaces           # tail toàn bộ log Flux, mọi controller

# Hoặc lấy trực tiếp log của từng controller
kubectl logs -n flux-system -l app=helm-controller --tail=100 -f
kubectl logs -n flux-system -l app=source-controller --tail=100 -f          # chart/git pull
kubectl logs -n flux-system -l app=kustomize-controller --tail=100 -f
kubectl logs -n flux-system -l app=notification-controller --tail=100 -f   # debug vì sao Slack không nhận alert
kubectl logs -n flux-system -l app=image-reflector-controller --tail=100 -f # ImageRepository/ImagePolicy
kubectl logs -n flux-system -l app=image-automation-controller --tail=100 -f # ImageUpdateAutomation

# Xem điều kiện (Ready/Released/Drifted...) + Event gần nhất của 1 HelmRelease
kubectl describe helmrelease demo-api -n production
kubectl get events -n production --sort-by=.lastTimestamp
```

## 6. Ghi nhớ hành vi hay gây nhầm lẫn

- `flux reconcile helmrelease` ép chạy lại reconcile ngay, nhưng nếu chart/values **không đổi** so với release đang chạy thì helm-controller coi là no-op — không có `helm upgrade` nào chạy, không có Event mới, không có Slack alert.
- Version chart không tồn tại trên registry → Flux **không** áp version lỗi lên cluster. Nó dừng ở bước pull chart (`HelmChart` object fail, `Ready=False reason=SourceNotReady`), release cũ trong cluster vẫn chạy nguyên, an toàn (fail-closed).
- Lỗi chart pull xảy ra ở object con `HelmChart` (namespace `flux-system`), không phải ở `HelmRelease` — nếu muốn Alert bắt được lỗi này, `eventSources` phải khai thêm `kind: HelmChart, namespace: flux-system` (xem `infrastructure/shared/notification-alert.yaml`).
