# LineBOT 包版：EKS 結合 S3 和 DynamoDB

- deploy.yml
- configmap.yml

```kubectl create secret docker-registry gitlab --docker-server=registry.gitlab.com --docker-username=$USERNAME --docker-password=$PASSWORD --docker-email=$EMAIL```

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
        active: dev

    line:
      bot:
        channel-token: 'AFFU='
        channel-secret: '0bc28d'
        handler:
          path: /callback
      notify:
        group-token: '<group-token>'
        owner-token: '<owner-token>'
  application-dev.yml: |
    amazon:
      dynamodb:
        endpoint: https://dynamodb.ap-northeast-1.amazonaws.com
        region: ap-northeast-1
      s3:
        accessKey: 123
        secretKey: 123 # useless but still need to fill those vars in source code
        bucketName: cathayct-linebot
        region: ap-northeast-1
  aws_access_key_id: AJV
  aws_secret_access_key: qSFw # need access to s3 and dynamodb
```

```yaml
# deploy.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iam-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: iam-test
  template:
    metadata:
      labels:
        app: iam-test
    spec:
      # serviceAccount: iam-linebot-role
      containers:
      - name: linebot
        image: registry.gitlab.com/cloudtechlab/line-bot
        ports:
        - containerPort: 8000
        volumeMounts:
        - name: application-config
          mountPath: /config
          readOnly: true
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            configMapKeyRef:
              name: linebot
              key: aws_access_key_id
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            configMapKeyRef:
              name: linebot
              key: aws_secret_access_key
      imagePullSecrets:
        - name: gitlab
      volumes:
        - name: application-config
          configMap:
            name: linebot
            items:
            - key: application.yml
              path: application.yml
            - key: application-dev.yml
              path: application-dev.yml
```

