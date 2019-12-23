## Reverse proxy

以前自己的網站是使用subdomain導向不同的server，但現在手上沒有第二個subdomain了，但又只想在一台機器上開多個服務，所以希望在nginx實現用不同的location導向不同server，例如：example.com/prometheus/ 會導向 http://localhost:9090/ ; example.com/grafana/ 導向 http://localhost:3000/ 。

## nginx setup
``` sh
# /etc/nginx/nginx.conf
location /prometheus/ {
    proxy_pass http://127.0.0.1:9090/;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
}
location /grafana/ {
    proxy_pass http://127.0.0.1:3000/;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X_Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
}
```

## Prometheus and Grafana setup
前置作業：將預設的檔案從容器裡提出來，方便之後mount使用
```bash
$ docker cp prometheus:/etc/prometheus/prometheus.yml .
$ docker cp grafana:/var/lib/grafana .
```
```yaml
# docker-compose.yml
version: '3.2'
services:
          prometheus:
            image: prom/prometheus
            ports:
            - 9090:9090
            command:
            - --config.file=/etc/prometheus/prometheus.yml
            - --web.route-prefix=/
            - --web.external-url=http://cathay-james.csie.org/prometheus # set for reverse proxy
            volumes:
            - ./prometheus.yml:/etc/prometheus/prometheus.yml
           grafana:
            image: grafana/grafana
            volumes:
                - grafana_data:/var/lib/grafana
            environment:
              - GF_SECURITY_ADMIN_PASSWORD=pass
              - GF_SERVER_ROOT_URL=http://cathay-james.csie.org/grafana/ # set for reverse proxy
            depends_on:
              - prometheus
            ports:
              - '3000:3000'
```

`$ docker-compose up -d`