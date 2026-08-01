# maoz-k8s

自宅 k3s を Argo CD(GitOps)で宣言的に管理するリポジトリ。

## sops
```
sops -e secrets/secret.yaml > secrets/secret.enc.yaml
```
## 参考

https://github.com/schnatterer/argocd-autopilot-example/tree/main
