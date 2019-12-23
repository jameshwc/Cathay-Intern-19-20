# Code Commit

## How to git clone
-  適用於沒有權限創建IAM user的帳戶
> 先備條件：安裝aws cli、git，並完成```aws configure```
```
$ git config --global credential.helper '!aws codecommit credential-helper $@'
$ git config --global credential.UseHttpPath true
$ git clone $REPO_URL
```
