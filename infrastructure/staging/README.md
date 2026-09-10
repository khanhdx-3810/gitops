# Secret tao TAY, chua nam trong Git

Hai secret duoi day duoc tao bang kubectl, CHUA duoc quan ly boi GitOps.
Se chuyen sang SOPS o Phan 6.

Ca hai dung cung mot tai khoan: robot$rnd+flux (chi co quyen Pull).

## 1. namespace flux-system — cho source-controller keo CHART (buoc 5)

    kubectl create secret docker-registry harbor-creds \
      --namespace flux-system \
      --docker-server=harbor.dangxuankhanh.io.vn \
      --docker-username='robot$rnd+flux' \
      --docker-password="$FLUX_TOKEN"

## 2. namespace production va staging — cho kubelet keo IMAGE (buoc 7)

    kubectl create secret docker-registry harbor-creds \
      --namespace production \
      --docker-server=harbor.dangxuankhanh.io.vn \
      --docker-username='robot$rnd+flux' \
      --docker-password="$FLUX_TOKEN"

## Command

    flux reconcile image repository web-api            # quét Harbor ngay

    flux reconcile image update web-api-production     # kiểm tra và commit ngay

    flux reconcile source git flux-system              # kéo repo về ngay
