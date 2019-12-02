# Drone setup
## 用docker安裝Drone server
```bash
docker run \
  --volume=/var/lib/drone:/data \
  --volume=/var/run/docker.sock:/var/run/docker.sock/ \
  --env=DRONE_AGENTS_ENABLED=true \
  --env=DRONE_GITLAB_SERVER=https://gitlab.com \
  --env=DRONE_GITLAB_CLIENT_ID=${GITLAB_CLIENT_ID}\
  --env=DRONE_GITLAB_CLIENT_SECRET=${GITLAB_CLIENT_SECRET} \
  --env=DRONE_RPC_SECRET=${YOUR_SECRET} \
  --env=DRONE_SERVER_HOST=${HOST_NAME} \
  --env=DRONE_SERVER_PROTO=${HTTP_OR_HTTPS} \
  --env=DRONE_USER_CREATE=username:${USERNAME},admin:true \
  --publish=80:80 \
  --publish=443:443 \
  --restart=always \
  --detach=true \
  --privileged=true \
  --name=drone \
  drone/drone:1
```
- DRONE_USER_CREATE 那行可加可不加，不加的話之後還是可以用drone cli新增admin
- privileged=true是為了使用docker in docker流程

## 用Docker安裝 Docker-runner
```bash
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e DRONE_RPC_PROTO=${HTTP_OR_HTTPS} \
  -e DRONE_RPC_HOST=${DRONE_HOST} \
  -e DRONE_RPC_SECRET=${YOUR_SECRET} \
  -e DRONE_RUNNER_CAPACITY=2 \
  -e DRONE_RUNNER_NAME=${HOST:PORT} \
  --privileged=true \
  -p 8000:3000 \
  --restart always \
  --name runner \
  drone/drone-runner-docker:1
  ```