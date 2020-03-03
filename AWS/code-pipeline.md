# Code Pipeline Guide

## CodeCommit

1. Create repository
 ![](https://i.imgur.com/CPs9Voj.png)

2. Clone the repo

> 先備條件：安裝aws cli、git，並完成```aws configure```

```bash
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true
git clone $REPO_URL
```

## CodeBuild

1. Create a CodeBuild project
![](https://i.imgur.com/j3yiX0y.png)
記得要開Cloudwatch才能在web console看到build logs

2. 在repository裡新增 buildspec.yml
    > 如果指令不長，也可以在project的設定裡選擇不使用yaml，這裡因為IaC原則還是用yaml做示範

    buildspec.yml的細則可以參考[官網文件](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)
    ```yaml
    version: 0.2

    phases:
      build:
        commands:
          - go test ./...
          - go build
    ```
    * versions: 目前最新版本為0.2
    * phases: 分成install, pre_build, build, post_build等階段
        * install: 安裝build階段需要用到的package，例如rspec
        * 其他階段說明請參考文件
    * 前一個phase失敗就不執行後面的phases
    * 支援*artifact, env, report*等

3. 將buildspec.yml 使用codecommit push之後，在第一步創建的project裡**手動**build看看成果（要自動化請參考CodePipeline）
4. demo:
![](https://i.imgur.com/Og2VrZD.png)

## CodePipeline

1. Create a CodePipeline project
![](https://i.imgur.com/zEsS4CC.png)

> source可以有很多個選擇，這裡當然使用的是CodeCommit；另外build也可以選擇Jenkins，同理我選了剛剛創建的codebuild project

2. 創好之後應該會先自動的跑一次整個流程。另外如果剛剛選項選的是Amazon CloudWatch Events（1. 的圖），則這時候git push也會觸發pipeline。

3. demo:

![](https://i.imgur.com/SY1ZKcn.png)
