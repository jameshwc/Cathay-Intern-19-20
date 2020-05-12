---
tags: Cathay
---
# Vault Sidecar: 對 Pod 注入 Vault 變數 (EKS 環境)

## 前置作業

### 環境

- 安裝 helm (需至少 3.0+)
- Kubernetes 1.9+
- demo namespace

### 安裝 vault-helm

```bash
$ git clone https://github.com/hashicorp/vault-helm
$ helm install vault --set='server.dev.enabled=true' ./vault-helm
```

### 進 vault 設定

```bash
$ kubectl exec -ti vault-0 /bin/sh 

# prepare the policy.hcl
$ vault policy write app app-policy.hcl
# allow vault to communicate with kubernetes
$ vault auth enable kubernetes
$ vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
$ vault write auth/kubernetes/role/myapp \
   bound_service_account_names=app \
   bound_service_account_namespaces=demo \
   policies=app \
   ttl=1h
```

#### example policy hcl

```hcl
path "secret*" {
  capabilities = ["read"]
}
```

到這裡就完成和 Kubernetes 的連線了，這時可以退出 shell，或繼續待著都可以

## 放 key

若要從外部連到 vault，可以使用 port-forward 或其他進階方式
以下是 port-forward script
```bash
$ kubectl port-forward vault-0 8200:8200
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ export VAULT_TOKEN="root" # default token
$ vault kv put secret/helloworld username=foobaruser password=foobarbazpass
```
## Deploy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: vault-agent-demo
spec:
  selector:
    matchLabels:
      app: vault-agent-demo
  replicas: 1
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-status: "update"
        vault.hashicorp.com/agent-inject-secret-helloworld: "secret/helloworld"
        vault.hashicorp.com/agent-inject-template-helloworld: |
          {{- with secret "secret/helloworld" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end }}
        vault.hashicorp.com/role: "myapp"
      labels:
        app: vault-agent-demo
    spec:
      serviceAccountName: app
      containers:
      - name: app
        image: jweissig/app:0.0.1
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  labels:
    app: vault-agent-demo
```

關鍵是 annotations 那段，也可以用打 patch 的方式

```yaml
# patch-template-annotations.yaml
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-status: "update"
        vault.hashicorp.com/agent-inject-secret-helloworld: "secret/helloworld"
        vault.hashicorp.com/agent-inject-template-helloworld: |
          {{- with secret "secret/helloworld" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end }}
        vault.hashicorp.com/role: "myapp"
```

```bash
$ kubectl apply -f app.yaml
$ kubectl patch deployment app --patch "$(cat patch-template-annotations.yaml)"
$ kubectl exec -ti app-XXXXXXXXX -c app -- cat /vault/secrets/helloworld 
# 輸出:postgresql://foobaruser:foobarbazpass@postgres:5432/wizard
```

## 參考資料
[官方 Blog](https://www.hashicorp.com/blog/injecting-vault-secrets-into-kubernetes-pods-via-a-sidecar/)