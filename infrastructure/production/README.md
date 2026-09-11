# Secret tao TAY — chi con DUY NHAT mot cai

## sops-age (namespace flux-system)

Chua private key age de Flux giai ma cac Secret trong Git.
Private key backup trong password manager cua team.

    kubectl create secret generic sops-age \
      --namespace=flux-system \
      --from-file=age.agekey=$HOME/.config/sops/age/keys.txt

Moi Secret khac deu nam trong Git duoi dang da ma hoa (*.enc.yaml).
