# Plugin: Tree-config

## 說明

該plugin為第三方開發的外掛，讓Drone檢查每次改變的檔案，並尋找該目錄底下或所有父目錄的.drone.yml，並根據第一個找到的.drone.yml，或所有.drone.yml去跑CI/CD。
[參考作者在github的說明](https://github.com/bitsbeats/drone-tree-config)：
> The extension checks each changed file and looks for a .drone.yml in the directory of the file or any parent directory. Drone will either use the first .drone.yml that matches or optionally run all of them in a multi-machine build.

## 限制
目前該plugin只支援 Github 和 Bitbucket，尚未支援 Gitlab，但因該專案並不大，之後有時間我會嘗試改寫github的部份讓它可以支援gitlab。

## 安裝
```docker-compose
version: '2'
services:
  drone-server:
    image: drone/drone
    ports:
      - 8000:80
    volumes:
      - /var/lib/drone:/data
      - /var/run/docker.sock:/var/run/docker.sock
    links:
      - drone-tree-config
    restart: always
    environment:
      - DRONE_OPEN=true
      - DRONE_SERVER_PROTO=http
      - DRONE_SERVER_HOST=$HOST
      - DRONE_GITHUB=true
      - DRONE_GITHUB_SERVER=https://github.com
      - DRONE_GITHUB_CLIENT_ID=$DRONE_GITHUB_CLIENT_ID
      - DRONE_GITHUB_CLIENT_SECRET=$DRONE_GITHUB_CLIENT_SECRET
      - DRONE_GIT_ALWAYS_AUTH=true
      - DRONE_USER_CREATE=username:jameshwc,admin:true
      - DRONE_RPC_SECRET=$SECRET
      - DRONE_YAML_ENDPOINT=http://drone-tree-config:3000
      - DRONE_YAML_SECRET=654321

  runner:
          image: drone/drone-runner-docker
          ports:
                  - 8005:3000
          environment:
                  - DRONE_RPC_PROTO=http
                  - DRONE_RPC_HOST=drone-server
                  - DRONE_RPC_SECRET=123456
                  - DRONE_RUNNER_NAME=Runner
                  - DRONE_RUNNER_CAPACITY=2
          volumes:
                  - /var/run/docker.sock:/var/run/docker.sock
          privileged: true

  drone-tree-config:
    image: bitsbeats/drone-tree-config
    environment:
      - PLUGIN_DEBUG=true
      - PLUGIN_CONCAT=true
      - PLUGIN_FALLBACK=true
      - PLUGIN_SECRET=654321
      - GITHUB_TOKEN=$GITHUB_TOKEN
    ports:
            - 8030:3000
    restart: always
```

[demo project](https://github.com/jameshwc/monorepo-test)

## debug
先用```docker-compose logs```檢查server、runner、tree-config-plugin彼此有沒有正常運作和溝通
之後在github repo的settings頁面，有欄webhook，檢查github有沒有正常送出request、得到的response又是什麼
## 常見問題
- {"messages": "repository not found"} : 通常是忘了在drone裡面activate