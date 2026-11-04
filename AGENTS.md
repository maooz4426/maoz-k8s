# AGENTS.md

k3s クラスタ向けの GitOps リポジトリです。`main` に push すると ArgoCD がクラスタへ反映します。

## アーキテクチャ(責務分け)

| パス | 責務 |
| --- | --- |
| `bootstrap/root-app.yaml` | 手元から `kubectl apply` する唯一のファイル。`applicationsets/*.yaml` と `argocd/*/application.yaml` だけを管理する |
| `applicationsets/apps.yaml` | Files generator で `apps/*/config.json` を読み、アプリごとに Application を生成する |
| `apps/<app>/` | アプリ本体。必須ファイルは `config.json`(`{"namespace": "..."}`)と `kustomization.yaml` |
| `argocd/<tool>/` | ArgoCD 周辺ツール(現状は `argocd-image-updater`)。root-app が `application.yaml` を直接管理する |

- 新しいアプリを追加するときは `apps/<app>/` を作る。ApplicationSet は編集しない。
- Application 名はディレクトリ名と同じ。namespace は `config.json` の値。
- ArgoCD 本体はこのリポジトリで管理しない(自己参照を避けるため手元で apply する)。ArgoCD 側の kustomize 設定(`--enable-helm`、ksops)も管理外。
- 同期は `automated`(`prune` + `selfHeal`)。`CreateNamespace=true`。

### アプリと namespace

| アプリ | namespace |
| --- | --- |
| authentik | authentik |
| cloudflared | cloudflared |
| grafana | grafana |
| mao-bot | mao-bot |
| otel-collector / prometheus | monitoring |
| tailscale | tailscale |
| traefik | kube-system |

## Traefik と Tailscale

- Traefik は k3s 同梱を使う。chart は入れない。`apps/traefik/services.yaml` の Service だけを管理する。
  - Service 名 `treafik-tailscale-service` の綴りは既存のまま。変更しない。
  - `loadBalancerClass: tailscale`、`tailscale.com/hostname: k3s-master-01-traefik` で tailnet に公開する。
- Tailscale operator は `apps/tailscale/`(Helm chart)。OAuth 認証情報は ksops で渡す。ProxyClass は使わない。
- ルーティングは標準の `networking.k8s.io/v1` Ingress(`ingressClassName: traefik`)。Traefik の IngressRoute は使わない。
- tailnet 側はホスト `k3s.maooz4426.com` をパスで振り分ける。

  | パス | 転送先 |
  | --- | --- |
  | `/grafana` | grafana:80 |
  | `/prometheus` | prometheus-server:80 |
  | `/otel` | otel-collector:4318 |

  サブパス運用のため、アプリ側にも設定がある(Grafana の `root_url` と `serve_from_sub_path`、Prometheus の `prefixURL`、OTel の `metrics_url_path`)。パスを変えるときはアプリ側も変更する。
- インターネット公開は cloudflared の tunnel(`apps/cloudflared/config.yaml`)。`auth` / `grafana` / `argocd` の各ホストを転送する。Authentik は Ingress を持たず、cloudflared 経由だけで公開する。

## デプロイと ArgoCD Image Updater

- ApplicationSet が `apps/<app>` を kustomize でビルドして適用する。
- Image Updater は `argocd/argocd-image-updater/`。CRD(`ImageUpdater`)で設定する。annotation は使わない。
  - 対象は Application `mao-bot` だけ。イメージは `ghcr.io/maooz4426/mao-bot`。
  - 更新戦略は `newest-build`。`latest` タグは無視する。
  - write-back は `git:secret:argocd/image-updater-git-creds`、`writeBackTarget: kustomization`。`main` に直接 push せず、PR を作成する。
  - 更新先は `apps/mao-bot/kustomization.yaml` の `images[].newTag`。
- `apps/mao-bot/deployment.yaml` の `image` はタグを持たない。タグを `deployment.yaml` に書かない。`newTag` を手で編集しない(Image Updater の PR に任せる)。
- 依存更新は Renovate(`.github/renovate.json`)。PR を作る。

## シークレット

- SOPS(age)と KSOPS で管理する。`.sops.yaml` が `secrets/.*\.yaml$` を暗号化する。
- 構成は `apps/<app>/secrets/*.enc.yaml` と `apps/<app>/secret-generator.yaml`(`kind: ksops`)。`kustomization.yaml` の `generators` から参照する。
- 平文の `secret.yaml` と `age.key` をコミットしない(`.gitignore` で除外済み)。
- 数値は文字列として引用符で囲む。
- 暗号化: `sops -e secrets/secret.yaml > secrets/secret.enc.yaml`

## マニフェストの規約

- すべて Kustomize ベース。第三者アプリは `helmCharts`(`version` 固定、`valuesFile: values.yaml`)を使う。
- `kustomization.yaml` のキー順は `resources`、`helmCharts`、`generators`。`namespace` を明示する。
- すべての Deployment に requests と limits を設定する。
- YAML マニフェストに説明コメントを追加しない。既存のコメントは削除しない。
- 既存の誤記(`prometheus/values.yaml` の `enebled` など)は、確認なしに修正しない。

## Git 規約

- Conventional Commits。本文は日本語。例: `feat: ユーザー認証にOAuth2を追加`
- ブランチ名は `feat/...`、`fix/...`、`chore/...` の形式。PR 経由で `main` にマージする。
- 確認なしにコミットも push もしない。
