# 1. Ép ImageRepository quét lại registry tìm tag mới
flux reconcile image repository web-api -n flux-system

# 2. Ép ImagePolicy đánh giá lại theo policy (semver...)
flux reconcile image policy web-api-production -n flux-system

flux reconcile image policy web-api-staging -n flux-system

# 3. Ép ImageUpdateAutomation chạy ngay (checkout, patch setter, commit, push)
flux reconcile image update web-api-production -n flux-system

flux reconcile image update web-api-staging -n flux-system
