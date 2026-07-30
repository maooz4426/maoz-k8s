# maoz-k8s

自宅 k3s を Argo CD(GitOps)で宣言的に管理するリポジトリ。
現在のゴールは **Argo CD 経由で Grafana をデプロイする** こと。

## 構成(app-of-apps)

```
bootstrap/root-app.yaml           親 Application。これ1枚だけ手動 apply する
apps/grafana/application.yaml     Grafana(Helm)
apps/authentik/application.yaml   Authentik(Helm)
apps/authentik/secret-app.yaml    Secret用Application(config/ を kustomize+KSOPS で処理)
apps/authentik/config/            KSOPS一式(kustomization / secret-generator / secrets/secret.enc.yaml)
docs/やること.md                  ロードマップ / 進捗チェックリスト
```

`root` が `apps/` を監視し、配下の Application を自動登録する。以降アプリの追加・変更は
`apps/` にファイルを置いて git push するだけ。

## 初回セットアップ

前提: k3s に接続でき、Argo CD がインストール済みで、本リポジトリは public。

```bash
kubectl apply -f bootstrap/root-app.yaml   # 親を一度だけ登録
kubectl -n argocd get applications         # root / grafana が Synced/Healthy を確認
```

Grafana へのアクセス(管理者パスワード取得と port-forward)や今後の作業は
[docs/やること.md](docs/やること.md) を参照。


## sops
```
sops -e secrets/secret.yaml > secrets/secret.enc.yaml
```
## 参考

https://github.com/schnatterer/argocd-autopilot-example/tree/main
