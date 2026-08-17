# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

自宅 Kubernetes (RKE2) クラスタを ArgoCD + Kustomize で管理する GitOps リポジトリ。
アプリケーションコードは無く、全て YAML マニフェスト。ビルド/テスト/リントの仕組みは無い。

## 検証コマンド

`kustomize` はこの環境に標準では入っていないため、必要なら取得する:

```sh
curl -sSL https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.7.1/kustomize_v5.7.1_linux_amd64.tar.gz | tar xz
```

**マニフェストを変更したら必ず全 overlay のビルドを通すこと**（唯一の自動検証手段）:

```sh
for d in apps/*/overlays; do kustomize build "$d" > /dev/null || echo "FAIL $d"; done
kustomize build cluster/applications > /dev/null
```

意図しない差分が出ていないかは、変更前後のレンダリング結果を比較するのが確実:

```sh
for d in apps/*/overlays; do n=$(echo $d|cut -d/ -f2); kustomize build "$d" > /tmp/after/$n.yaml; done
git stash && for d in apps/*/overlays; do n=$(echo $d|cut -d/ -f2); kustomize build "$d" > /tmp/before/$n.yaml; done && git stash pop
diff -ru /tmp/before /tmp/after
```

`cluster/bootstrap/` 配下はリモートリソース（ArgoCD 本体、MetalLB、ingress-nginx、sealed-secrets）を参照するためビルドにネットワークが必要。

## 適用コマンド

```sh
kubectl apply -k cluster/bootstrap        # 初回のみ。ArgoCD/MetalLB/ingress/CSI/sealed-secrets
kubectl apply -k cluster/applications     # ArgoCD Application の作成・更新（後述の通り手動）
./get-argocd-password.sh                  # ArgoCD 初期パスワード取得
```

## アーキテクチャ

### 二層構造

- **`cluster/bootstrap/`** — クラスタ基盤。ArgoCD 自身を含むため ArgoCD では管理されず、`kubectl apply -k` で手動適用する。
- **`cluster/applications/`** — アプリごとの ArgoCD `Application` (9個)。
- **`apps/<name>/{base,overlays}/`** — 各アプリのマニフェスト。ArgoCD が見るのは `overlays` のみ。

### ⚠️ `cluster/applications/` は ArgoCD 管理外

**このリポジトリには app-of-apps の親 Application が存在しない。** `cluster/applications/` を指す Application が無いため、`Application` リソースの追加・変更は commit しただけでは反映されず、**必ず手動で `kubectl apply -k cluster/applications` が必要**。

この構造のおかげで repoURL 変更時の自己参照デッドロックが無い（後述の移行計画で重要）。

### base / overlays の役割分担

- **base** — Service / Deployment / PVC などの本体。**イメージにタグを書かない**（`image: goegoe0212/home-api` のようにタグ無し）。
- **overlays** — `namespace:` の指定、`namespace.yaml`、ConfigMap、SealedSecret、そして **`images:` によるタグ固定**。

**イメージのバージョン更新は overlays の `images:` を編集する。** base 側にタグを書かないのが規約。overlay は 1 アプリにつき 1 つだけで、環境別の overlay は無い。

可変タグ（`latest` / `alpine` / `release` 等）は使わない。GitOps の再現性が壊れ、次回 sync で意図しないバージョンに上がるため。

### 外部公開の 2 経路

1. **Cloudflare Tunnel**（主経路）— `apps/cloudflare/base/cloudflared-deploy.yaml` の ConfigMap にホスト名とバックエンドの対応が書かれている。metabase / argocd / immich はここ経由。**新しくドメインを生やす場合はこの ConfigMap を編集する。**
2. **nginx Ingress** — home-api の `/dev` パスのみ。host 指定なし。

### MetalLB の固定 IP 割り当て

プール `192.168.20.100-192.168.20.150`（`cluster/bootstrap/metallb/metallb-config.yaml`）。
`loadBalancerIP` は各マニフェストに散らばっているので、**新規に LoadBalancer を作る際は衝突を避けること**:

| IP | 用途 |
|---|---|
| `.100` | minecraft-server |
| `.101` | redis-service (`apps/redis`) |
| `.102` | postgres-service (`apps/postgresql`) |
| `.103` | immich-server-lb |
| `.110` | ingress-nginx-controller |

### ストレージ

- **`qnap-iscsi-standard`** — QNAP Trident CSI。`reclaimPolicy: Retain`。immich / minecraft / postgresql が使用。
- **NFS** — home-api のみ。`192.168.10.20:/youtube/temporary` を PV 直書きで参照。

### Redis が 2 系統ある（混同注意）

- `apps/redis` — 汎用。`redis` イメージ、LoadBalancer `.101`。home-api と discord-bot が `redis-service.redis` として参照。
- `apps/immich/base/redis-deploy.yaml` — immich 専用。**valkey** イメージ、immich namespace 内に閉じている。

同様に PostgreSQL も別物:

- `apps/postgresql` — timescaledb。metabase と temp-collector が `postgres-service.postgresql.svc` として参照。
- `apps/immich/base/database-deploy.yaml` — immich 専用。ベクトル拡張入りの `ghcr.io/immich-app/postgres`（`DB_VECTOR_EXTENSION: pgvecto.rs`）で、immich namespace 内に閉じている。

### Secret 管理

Sealed Secrets を使用（cloudflare / discord-bot / temp-collector の 3 つ）。
**`kubeseal` とクラスタの公開鍵が無い環境では SealedSecret を新規生成できない。** 暗号化が必要な作業は手順の提示までしかできない。

## 規約

- コミットメッセージは**日本語**、`feat:` / `fix:` プレフィックス付き（例: `feat: timescaledbの新しいタグを2.24.0-pg17に更新`）。
- ファイル末尾の改行有無が統一されていない。既存ファイルを編集する際は元の形式を維持する。

---

# 進行中の課題

## Org 移行計画（未実施）

個人リポジトリ `goegoe0212/home-kubernetes` を Org へ移行する計画。ArgoCD が本リポジトリを参照しているため慎重に進める必要がある。

### 前提（調査済みの事実）

| 項目 | 状態 | 意味 |
|---|---|---|
| 可視性 | **public** | ArgoCD は匿名 clone。認証情報の移行が不要 |
| ArgoCD repo Secret | `url` と `project` のみ、**認証情報なし** | public 前提。再発行するものが無い |
| `cluster/applications` の親 Application | **無し** | 手動適用のため自己参照デッドロックが無い |
| repoURL の記述箇所 | **10 箇所**（Application × 9 + repo Secret × 1） | ApplicationSet 化すれば 1 箇所になる |

### 方針: transfer 方式（新規リポジトリ作成は禁止）

GitHub の repository transfer は**旧 URL への git 操作を新 URL へ 301 リダイレクトする**。そのため repoURL を変更しないまま ArgoCD は sync し続け、「移行」と「repoURL 更新」を分離できる。

**❌ やってはいけない: Org に空リポジトリを作って push する方式**

> 空リポジトリを作る → repoURL をそちらへ向ける → ArgoCD が fetch に**成功**しマニフェスト 0 件と認識 → `prune: true` の cloudflare / home-api / immich が**全リソース削除**

fetch 失敗なら Unknown 止まりで何も消えないが、「空リポジトリの fetch 成功」が最悪のケース。transfer ならこの状態が発生しない。

### 手順

0. 保険として auto-sync を一時停止（`prune: true` の 3 アプリ）
   ```sh
   for a in cloudflare home-api immich; do
     kubectl -n argocd patch app $a --type=merge -p '{"spec":{"syncPolicy":null}}'
   done
   ```
1. GitHub UI で Org へ transfer（Claude の権限では不可、手作業）
2. **可視性を確認** ⚠️ 唯一の落とし穴。private に落ちると ArgoCD が匿名 clone できず全アプリ sync 不能。private にするなら**先に** deploy key か GitHub App の認証情報を `kube-home-repo` Secret へ入れる
3. Claude Code の GitHub App を Org にインストール（`contents: write`）
4. repoURL を更新（10 箇所）して commit → `kubectl apply -k cluster/applications`
5. auto-sync を戻す

Step 4 まではロールバック可能（transfer は Org → 個人へ戻せる）。

### 順序についての注意

**移行を実施すると決めたなら、リファクタリングより先に移行する。** ApplicationSet 化は `cluster/applications/*.yaml` を丸ごと作り直す変更で、移行の repoURL 更新と同じファイルを触るため、逆順だと同じ作業を 2 回やることになる。

移行が当面先送りなら、構造整理から着手してよい。ApplicationSet 化で repoURL が 1 箇所に集約されるため、将来の移行コストはむしろ下がる。

## リファクタリング backlog（方針決定待ち）

### 未決定

- **ApplicationSet 化** — Application 9 個は repoURL と path 以外ほぼ同一。git directory generator（`apps/*/overlays`、`{{index .path.segments 1}}` でアプリ名を導出）で 1 個に集約できる。
  - ⚠️ **`syncPolicy.automated` が cloudflare / home-api / immich の 3 つにしか無い**（ドリフト）。一律 automated にすると手動投入したリソースが prune される恐れがあるため、方針の合意が必要。
- **home-api の PV/PVC** — `api-deploy.yaml` の PVC に `namespace: home-app` がハードコードされているが overlay の `namespace: home-api` に上書きされる**デッドコード**。また NFS の PV/PVC がバインド条件を持たず非決定的。`volumeName` の後付けは immutable field エラーで sync が落ちる可能性があるため、`kubectl get pvc -n home-api fastapi-nfs-pvc -o yaml` で現状確認してから決める。

### 合意済み・未着手

- `cloudflared-service`（Service, port 80）が**死んでいる** — cloudflared コンテナは 80 を listen していない。削除するか `--metrics` で 2000 に向ける
- `get-argocd-password.sh` → `scripts/` へ移動し shebang と `set -euo pipefail` を追加（現在シェバンなし）
- `imagePullPolicy: Always` — タグ固定後は無意味（cloudflare / redis / discord-bot / home-api）
- README がほぼ空、`.gitignore` が空
- discord-bot と home-api の ConfigMap（`TZ` / `REDIS_*`）が完全重複 → kustomize component に括り出せる
- CI（GitHub Actions）— 全 overlay の `kustomize build`、`kubeconform` 検証、可変タグ再混入のガード

### 将来対応

- **平文パスワードの排除** — `Passw0rd` / `postgres` が postgresql / metabase / temp-collector / immich に残存。「クラウドから遠隔取得」の要望があるため **External Secrets Operator** が候補（Git に秘密が一切入らず、ローテーションがクラスタ側で完結）。既存 SealedSecrets と併存可能で段階移行できる
- **タグ更新の自動化** — 現状は手動。Renovate 等の導入余地あり
