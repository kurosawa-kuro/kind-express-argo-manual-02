# Hands‑on│ Kind + Helm + Argo CD（Express API 8000 / Manual Sync）

> **ゴール** — EC2 `t3.small` の Ubuntu 上に Kind クラスタを構築し、**ECR プライベートイメージ `container-nodejs-api-8000:latest`** を Helm でデプロイ。
> その後 Argo CD UI で **手動 Sync** まで確認します。Nginx ではなく **Express API (port 8000)** 版のミニハンズオンです。

---

## ✔️ 必要ツール & リソース

| ツール | バージョン目安 | 備考 |
|--------|--------------|------|
| Docker | 20.10+ | rootless でないこと |
| Kind   | ≥ 0.23 | containerdConfigPatches 対応 |
| kubectl| v1.30± |  |
| Helm   | v3.14± |  |
| Git    | 任意 |  |
| AWS CLI| v2 | ECR ログイン用 |

> **ECR リポジトリ** : `986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000:latest`

---

## 📂 プロジェクト構成

```
~/dev/kind-express-argo-manual-02/
  ├── kind-cluster.yaml
  ├── express-chart/          # helm create で生成
  └── argocd-application.yaml
```

---

## STEP 0 — Kind クラスタ（hostPort 8000 公開 & ECR 認証）

### 0‑1. クラスタ定義
```bash
mkdir -p ~/dev/kind-express-argo-manual-02 && cd $_

# ECR 認証トークン取得
export REGION=ap-northeast-1
export ACCOUNT_ID=986154984217
export ECR_TOKEN=$(aws ecr get-login-password --region $REGION)

cat <<EOF > kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 8000   # Express API 用
    hostPort: 8000
    protocol: TCP
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.auths."${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"]
      username = "AWS"
      password = "${ECR_TOKEN}"
EOF

kind create cluster --config <(envsubst < kind-cluster.yaml) --name express-demo
```

> **t3.small のメモリ節約** : Pod を 1 つだけ起動する想定なので余計な nodePort/Ingress を開けていません。

---

## STEP 1 — Helm Chart で Express API デプロイ

### 1‑1. Chart 生成
```bash
helm create express-chart
cd express-chart
```

### 1‑2. values.yaml だけ編集
```bash
cat > values.yaml <<'YAML'
replicaCount: 1

image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000
  tag: latest
  pullPolicy: Always

service:
  type: NodePort
  port: 8000
  nodePort: 30800   # Kind -> host 8000 にマッピング済

containerPort: 8000

ingress:
  enabled: false
resources: {}

# ServiceAccount設定を追加
serviceAccount:
  create: false
  name: ""
  annotations: {}

# オートスケーリング設定を追加
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

YAML
```
> **テンプレート削除禁止** — デフォルトのテンプレは残し、`values.yaml` だけ調整します。

### 1‑3. インストール & 動作確認
```bash
helm install express-demo ./ -f values.yaml --namespace default
kubectl get pods,svc -l app.kubernetes.io/name=express-chart
```

### 1‑4. アプリケーションへのアクセス方法

#### 方法1: ポートフォワーディング（推奨）
```bash
# Podに直接ポートフォワーディング
kubectl port-forward pod/express-demo-express-chart-<POD-ID> 8000:8000
```

別のターミナルで確認:
```bash
curl http://localhost:8000
```

> **注意**: ポートフォワーディングは、ターミナルセッションが終了すると停止します。バックグラウンドで実行する場合は、`&`を付けて実行し、プロセスIDを記録しておきます。
> ```bash
> kubectl port-forward pod/express-demo-express-chart-<POD-ID> 8000:8000 &
> echo $! > port-forward.pid
> ```
> 後で停止する場合:
> ```bash
> kill $(cat port-forward.pid)
> ```

#### 方法2: NodePort経由
```bash
# 実際に割り当てられたNodePortを確認
kubectl get svc express-demo-express-chart -o jsonpath='{.spec.ports[0].nodePort}'

# 確認したポート番号でアクセス
curl http://localhost:<NODE_PORT>
```

#### 方法3: EC2のパブリックIP経由
```bash
# EC2のパブリックIPを確認
curl http://<EC2-PUBLIC-IP>:8000
```

> **注意**: `values.yaml`で指定したNodePort（30800）と実際に割り当てられるNodePortは異なる場合があります。実際に割り当てられたポート番号を確認して使用してください。

`{"status":"ok"}` 等が返れば成功。

---

## STEP 2 — Argo CD（Manual Sync）

### 2‑1. Argo CD インストール
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ポートフォワード（VSCode Remote SSH ならこれだけ）
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```
初期パスワード：
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

uTkEFf8bdN9PQFbt

### 2‑2. GitHub リポジトリ（public）に Push
```bash
cd ~/dev/kind-express-argo-manual-02
git init
git add .
git commit -m "express manual sync step"
git branch -M development
git remote add origin https://github.com/kurosawa-kuro/kind-express-argo-manual-02.git
git push -u origin development
```

### 2‑3. Application 定義
```bash
cat <<EOF > argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: express-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/kurosawa-kuro/kind-express-argo-manual-02.git
    targetRevision: development
    path: express-chart
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy: {}   # Manual Sync
EOF

kubectl apply -f argocd-application.yaml
```

### 2‑4. UI で手動 Sync
1. ブラウザ `https://localhost:8080` → Login (`admin` / <初期PW>)
2. `Applications › express-demo` を開き **Sync → Synchronize**
3. 緑色 **Healthy / Synced** で完了

---

## 🔄 ロールバックスクリプト
```bash
helm uninstall express-demo || true
kubectl delete -f argocd-application.yaml || true
kind delete cluster --name express-demo || true
```

---

## 🔍 トラブルシューティング

### Helmチャートのインストールエラー
- **エラー**: `template: express-chart/templates/serviceaccount.yaml:1:14: executing "express-chart/templates/serviceaccount.yaml" at <.Values.serviceAccount.create>: nil pointer evaluating interface {}.create`
- **解決策**: `values.yaml`に`serviceAccount`の設定を追加する
  ```yaml
  serviceAccount:
    create: false
    name: ""
    annotations: {}
  ```

- **エラー**: `template: express-chart/templates/hpa.yaml:1:14: executing "express-chart/templates/hpa.yaml" at <.Values.autoscaling.enabled>: nil pointer evaluating interface {}.enabled`
- **解決策**: `values.yaml`に`autoscaling`の設定を追加する
  ```yaml
  autoscaling:
    enabled: false
    minReplicas: 1
    maxReplicas: 3
    targetCPUUtilizationPercentage: 80
    targetMemoryUtilizationPercentage: 80
  ```

### アプリケーションへのアクセスエラー
- **エラー**: `curl: (56) Recv failure: Connection reset by peer`
- **解決策**: ポートフォワーディングを使用して直接Podにアクセスする
  ```bash
  kubectl port-forward pod/<POD-NAME> 8000:8000
  curl http://localhost:8000
  ```

### ArgoCDへのアクセスエラー
- **エラー**: `curl: (7) Failed to connect to localhost port 8080 after 0 ms: Couldn't connect to server`
- **解決策**: ポートフォワーディングが正しく設定されているか確認する
  ```bash
  # ポートフォワーディングを再設定
  kubectl port-forward svc/argocd-server -n argocd 8080:443
  ```

---

### 次ステップ候補
- **ECR → automated Sync**：イメージタグを更新して GitOps 自動展開
- **HPA・Ingress**：`values.yaml` で `enabled: true` に切り替えて負荷試験
- **Operator**：公式 Cache Operator サンプルを同様に GitOps 管理

> **メモリ上限** : t3.small では Pod 合計メモリを 1 GiB 未満に抑えると安定します。Express API の `node --max-old-space-size=128` などで制御してみてください。

## 🔄 イメージ更新手順

### タグ更新方針

このプロジェクトでは、イメージのバージョン管理のためにタグ更新方針を採用しています。`values.yaml`で特定のタグを指定し、新しいバージョンをリリースする際にタグを更新します。

```yaml
image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000
  tag: v1.0.0  # バージョン番号を指定
  pullPolicy: Always
```

### 新しいバージョンのリリース手順

1. **新しいイメージのビルドとプッシュ**:
   ```bash
   # ECRにログイン
   aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com

   # イメージをビルド
   docker build -t container-nodejs-api-8000:v1.0.1 .

   # ECRリポジトリにタグ付け
   docker tag container-nodejs-api-8000:v1.0.1 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000:v1.0.1

   # ECRにプッシュ
   docker push 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000:v1.0.1
   ```

2. **values.yamlの更新**:
   ```bash
   # values.yamlを編集してタグを更新
   sed -i 's/tag: v1.0.0/tag: v1.0.1/' express-chart/values.yaml
   ```

3. **変更をGitリポジトリにコミット**:
   ```bash
   git add express-chart/values.yaml
   git commit -m "Update image tag to v1.0.1"
   git push origin development
   ```

4. **ArgoCDを使用してアプリケーションを更新**:
   ```bash
   # ArgoCDにログイン
   argocd login localhost:8080

   # アプリケーションを同期
   argocd app sync express-demo
   ```

5. **更新の確認**:
   ```bash
   # ポートフォワーディングを設定
   kubectl port-forward pod/$(kubectl get pods -l app.kubernetes.io/name=express-chart -o jsonpath='{.items[0].metadata.name}') 8000:8000

   # 別のターミナルで確認
   curl http://localhost:8000/posts
   ```

### ロールバック手順

特定のバージョンにロールバックする場合は、以下の手順を実行します：

1. **values.yamlのタグを更新**:
   ```bash
   # values.yamlを編集してタグを更新
   sed -i 's/tag: v1.0.1/tag: v1.0.0/' express-chart/values.yaml
   ```

2. **変更をGitリポジトリにコミット**:
   ```bash
   git add express-chart/values.yaml
   git commit -m "Rollback to v1.0.0"
   git push origin development
   ```

3. **ArgoCDを使用してアプリケーションを更新**:
   ```bash
   argocd app sync express-demo
   ```

### タグ更新方針のメリット

1. **バージョン管理**: 各リリースに明確なバージョン番号が付与されるため、どのバージョンがデプロイされているかが分かりやすくなります。

2. **ロールバックの容易さ**: 問題が発生した場合、以前のバージョンに簡単にロールバックできます。

3. **環境の一貫性**: 開発、テスト、本番環境で同じバージョンのイメージを使用することで、環境間の一貫性が保たれます。

4. **変更の追跡**: どのバージョンでどのような変更が行われたかを追跡しやすくなります。

