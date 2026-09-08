# maoz-k8s

自宅 k3s を Argo CD(GitOps)で宣言的に管理するリポジトリ。

## 構成
ApplicationSetsを用いたApplication生成を行う。
また、ApplicationSetsで用いるパラメーターはconfig.jsonで定義するようにする。
各アプリディレクトリ(apps/<app>/)にconfig.jsonを置き、namespaceを指定する。
Directory Generatorはpath.basenameしか渡せず、アプリごとに異なるnamespaceを指定できなかった。Files Generatorに変更し、config.jsonからnamespaceを取得するようにした。

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
https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Generators-Git/
