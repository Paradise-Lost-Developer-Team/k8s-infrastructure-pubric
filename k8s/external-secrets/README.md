## ExternalSecrets (ESO) Scaffold

このフォルダは External Secrets Operator を利用して外部シークレットストア (AWS Secrets Manager / HashiCorp Vault / GCP Secret Manager 等) から Kubernetes Secret を同期するための雛形です。

### 1. Operator 導入 (例: Helm)
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm upgrade --install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

### 2. ClusterSecretStore 例 (AWS) - `secretstore-aws-example.yaml`
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key
            namespace: external-secrets
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-key
            namespace: external-secrets
```

### 3. ExternalSecret 例 (aivis bot 1st)
`aivis-chan-bot-1st-externalsecret.yaml` を参照。必要な remoteRef.key / property を実ストアに合わせて編集。

### 4. 運用ポイント
| 項目 | 推奨 |
|------|------|
| ローテ | 外部ストア側でローテ → refreshInterval 内で自動反映 |
| 権限 | 最小 IAM/Vault policy (対象キーに限定) |
| 名前付け | `prod/aivis/<botN>` のように階層化 |
| 監査 | Operator の events + 外部ストアのアクセスログを併用 |

### 5. aivis-chan-bot Deployment での利用
Secret 名 (例: `aivis-chan-bot-1st-secret`) を envFrom か個別 env 参照に追加する Patch を Kustomize の `patchesStrategicMerge` で適用してください。
