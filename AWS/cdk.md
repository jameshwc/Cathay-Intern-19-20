# CDK

## Prerequisites
### 安裝 nodejs (>= 10.3.0)
根據[nodejs的官方網站](https://nodejs.org/download/release/)下載>=10.3.0的版本

```bash
$ wget https://nodejs.org/download/release/latest-v13.x/node-v13.8.0-linux-x64.tar.gz
$ tar --strip-components 1 -xzvf node-v* -C /usr/local
```

### 安裝cdk
```bash
$ npm install cdk
$ npm install -g aws-cdk
```

### npm安裝aws-cdk底下的套件
```npm i @aws-cdk/aws-apigateway```
