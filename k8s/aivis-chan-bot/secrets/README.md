## aivis-chan-bot Secrets Management

このディレクトリは 2 つの方式をサポートするための雛形です:

1. SOPS + age による Git 上暗号化管理 (ArgoCD + KSOPS で復号)
2. ExternalSecrets Operator (ESO) + 外部 Secret Store (Vault / AWS Secrets Manager など)

現状はプレーンな *plaintext* Secret ファイルを `plain/` に置き、`sops` で暗号化した結果のみ `*.enc.yaml` をコミットする運用を想定しています。

### ディレクトリ構成

```
secrets/
  plain/                           # .gitignore 済 (平文を一時配置)
  aivis-chan-bot-1st-secret.enc.yaml (まだ無い → sops で生成)
  README.md
```

### 例: Discord Bot 1st の Secret (SOPS)

1. `plain/aivis-chan-bot-1st-secret.yaml` (平文テンプレート) を作成:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aivis-chan-bot-1st-secret
  namespace: aivis-chan-bot
type: Opaque
stringData:
  DISCORD_TOKEN: "<actual-token>"
  CLIENT_ID: "<discord-client-id>"
  PATREON_CLIENT_ID: "<patreon-client-id>"
  PATREON_CLIENT_SECRET: "<patreon-client-secret>"
  SENTRY_DSN: "<sentry-dsn>"
  GEMINI_SERVICE_ACCOUNT_JSON: |
    {"type":"service_account", ...}
```

2. 暗号化:
```bash
sops -e plain/aivis-chan-bot-1st-secret.yaml > aivis-chan-bot-1st-secret.enc.yaml
```
3. 平文は commit しない。`aivis-chan-bot-1st-secret.enc.yaml` を commit。
4. ArgoCD 側で KSOPS プラグイン設定が済めば自動復号→ Secret 作成。

### 複数 Bot の管理

同じ手順で `aivis-chan-bot-2nd-secret.yaml` などを作成→暗号化。フィールド名 (DISCORD_TOKEN 等) は統一。

### ExternalSecrets 方式

`../../external-secrets/` に ESO 用 CR を配置。必要に応じて下記 ExternalSecret を作り、外部ストア (例: AWS Secrets Manager) のキーを参照:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: aivis-chan-bot-1st
  namespace: aivis-chan-bot
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
  target:
    name: aivis-chan-bot-1st-secret
    creationPolicy: Owner
  data:
    - secretKey: DISCORD_TOKEN
      remoteRef:
        key: prod/aivis/bot1
        property: DISCORD_TOKEN
    - secretKey: CLIENT_ID
      remoteRef:
        key: prod/aivis/bot1
        property: CLIENT_ID
```

### どちらを使うか

| 要件 | 推奨方式 |
|------|-----------|
| 外部ストアなし / Git 内で暗号化完結 | SOPS + age |
| 既に Vault / AWS SM / GCP Secret Manager 利用 | ExternalSecrets |
| 将来マルチクラウド / ローテ自動化 | ExternalSecrets |

両方式を同時併用する場合、同一 Secret 名の衝突に注意してください (一方だけを kustomization に含める)。

### 次ステップ
1. ルート `.sops.yaml` の `AGE_PUBLIC_KEY_PLACEHOLDER` を自分の age 公開鍵に差し替え。
2. age 鍵を安全な場所に保管 (復旧用バックアップ必須)。
3. KSOPS プラグインを ArgoCD に設定 (未設定なら)。
4. SOPS 方式か ExternalSecrets 方式かを選び `kustomization.yaml` にリソース追加。
