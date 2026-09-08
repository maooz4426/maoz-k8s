# maoz-k8s

自宅 k3s を Argo CD(GitOps)で宣言的に管理するリポジトリ。

## 構成

```
.
├── bootstrap/              # 手元からapplyするファイルを管理
├── applicationsets/
├── argocd/                 # Argo CD周辺ツール(root-appが直接管理)
└── apps/                   # アプリ全般
```

## sops
```
sops -e secrets/secret.yaml > secrets/secret.enc.yaml
```
## 参考

https://github.com/schnatterer/argocd-autopilot-example/tree/main
