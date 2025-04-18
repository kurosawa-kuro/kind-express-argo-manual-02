# Hands‑on│ Kind + Helm + Argo CD（Express API 8000 / Manual Sync）

> **ゴール** — EC2 `t3.small` の Ubuntu 上に Kind クラスタを構築し、**ECR プライベートイメージ `container-nodejs-api-8000:latest`** を Helm でデプロイ。
> その後 Argo CD UI で **手動 Sync** まで確認します。Nginx ではなく **Express API (port 8000)** 版のミニハンズオンです。

---

## ✔️ 必要ツール & リソース

| ツール | バージョン目安 | 備考 |
|--------|--------------|------|
| Docker | 20.10+ | rootless でないこと |
| Kind   | ≥ 0.23 | containerdConfigPatches 対応 |
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

## STEP 0 — Kind クラスタ（hostPort 8000 公開 & ECR 認証）

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

## STEP 1 — Helm Chart で Express API デプロイ

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
  nodePort: 30800   # Kind -> host 8000 にマッピング済

containerPort: 8000

ingress:
  enabled: false
resources: {}
YAML
```
> **テンプレート削除禁止** — デフォルトのテンプレは残し、`values.yaml` だけ調整します。

### 1‑3. インストール & 動作確認
```bash
helm install express-demo ./ -f values.yaml --namespace default
kubectl get pods,svc -l app.kubernetes.io/name=express-chart

# 動作確認（ローカル PC）
curl http://<EC2-Public-IP>:8000/health
```
`{"status":"ok"}` 等が返れば成功。

---

## STEP 2 — Argo CD（Manual Sync）

### 2‑1. Argo CD インストール
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

### 2‑2. GitHub リポジトリ（public）に Push
```bash
cd ~/dev/kind-express-argo-manual-02
git init
git add .
git commit -m "express manual sync step"
git branch -M main
git remote add origin https://github.com/<YOUR-USER>/kind-express-argo-manual-02.git
git push -u origin main
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
    repoURL: https://github.com/<YOUR-USER>/kind-express-argo-manual-02.git
    targetRevision: main
    path: express-chart
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy: {}   # Manual Sync
EOF

kubectl apply -f ../argocd-application.yaml
```

### 2‑4. UI で手動 Sync
1. ブラウザ `https://localhost:8080` → Login (`admin` / <初期PW>)
2. `Applications › express-demo` を開き **Sync → Synchronize**
3. 緑色 **Healthy / Synced** で完了

---

## 🔄 ロールバックスクリプト
```bash
helm uninstall express-demo || true
kubectl delete -f argocd-application.yaml || true
kind delete cluster --name express-demo || true
```

---

### 次ステップ候補
- **ECR → automated Sync**：イメージタグを更新して GitOps 自動展開
- **HPA・Ingress**：`values.yaml` で `enabled: true` に切り替えて負荷試験
- **Operator**：公式 Cache Operator サンプルを同様に GitOps 管理

> **メモリ上限** : t3.small では Pod 合計メモリを 1 GiB 未満に抑えると安定します。Express API の `node --max-old-space-size=128` などで制御してみてください。

