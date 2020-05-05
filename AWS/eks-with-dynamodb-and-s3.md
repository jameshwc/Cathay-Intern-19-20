# LineBOT 包版：EKS 結合 S3 和 DynamoDB

- deploy.yml
- configmap.yml
- loadBalancer.yml
- extern-dns.yml

```kubectl create secret docker-registry gitlab --docker-server=registry.gitlab.com --docker-username=$USERNAME --docker-password=$PASSWORD --docker-email=$EMAIL```

```bash
kubectl apply -f extern-dns.yml
kubectl apply -f configmap.yml
kubectl apply -f deploy.yml
kubectl apply -f loadBalancer.yml
```

```yaml
# configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: linebot
  labels:
    app: linebot
data:
  application.yml: |
    spring:
      profiles:
        active: pro

    line:
      bot:
        channel-token: '123'
        channel-secret: '123'
        handler:
          path: /callback
      notify:
        group-token: '<group-token>'
        owner-token: '<owner-token>'
  application-pro.yml: |
    amazon:
      dynamodb:
        accesskey: 123
        secretkey: 123
        region: ap-northeast-1
      s3:
        accessKey: 123
        secretKey: 123
        bucketName: cathayct-linebot
        region: ap-northeast-1
```

```yaml
# deploy.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linebot
spec:
  selector:
    matchLabels:
      app: linebot
  template:
    metadata:
      labels:
        app: linebot
    spec:
      containers:
      - image: registry.gitlab.com/cloudtechlab/line-bot:latest
        name: linebot
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: application-config
          mountPath: /config
          readOnly: true
      imagePullSecrets:
        - name: gitlab
      volumes:
        - name: application-config
          configMap:
            name: linebot
            items:
            - key: application.yml
              path: application.yml
            - key: application-pro.yml
              path: application-pro.yml
```
```yaml
# loadBalancer.yml
apiVersion: v1
kind: Service
metadata:
  name: linebot
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert:  arn:aws:acm:$REGION:$USER_ID:certificate/$CERTIFICATE_ID
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "https"
    external-dns.alpha.kubernetes.io/hostname: linebot.cathaycloud.online.
spec:
  type: LoadBalancer
  selector:
    app: linebot
  ports:
    - protocol: TCP
      port: 443
      name: https
      targetPort: 8080
```
```yaml
# extern-dns.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  # If you're using Amazon EKS with IAM Roles for Service Accounts, specify the following annotation.
  # Otherwise, you may safely omit it.
  annotations:
    # Substitute your account ID and IAM service role name below.
    eks.amazonaws.com/role-arn: arn:aws:iam::${USER_ID}:role/${ROLE_ID}
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
      # If you're using kiam or kube2iam, specify the following annotation.
      # Otherwise, you may safely omit it.
      annotations:
        iam.amazonaws.com/role: arn:aws:iam::${USER_ID}:role/${ROLE_ID}
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.opensource.zalan.do/teapot/external-dns:latest
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=cathaycloud.online # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
        - --provider=aws
        - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
        - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
        - --registry=txt
        - --txt-owner-id=my-hostedzone-identifier
      securityContext:
        fsGroup: 65534 # For ExternalDNS to be able to read Kubernetes and AWS token files
```
## 參考資料


[external dns](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md)
[eks with https](https://aws.amazon.com/tw/premiumsupport/knowledge-center/terminate-https-traffic-eks-acm/)
