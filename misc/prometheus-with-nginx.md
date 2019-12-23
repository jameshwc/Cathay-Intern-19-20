# Prometheus Setup with reverse proxy in nginx

以前自己的網站是使用subdomain導向不同的server，但現在手上沒有第二個subdomain了，所以希望在nginx實現用不同的location導向不同server，例如：example.com/prometheus/ 會導向 http://localhost:9090/ ; example.com/grafana/ 導向 http://localhost:9099/ 。

## nginx setup
``` sh
# /etc/nginx/nginx.conf
location /prometheus/ {
    proxy_pass http://127.0.0.1:9090/;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
}
```

## Prometheus setup
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
            - --web.external-url=http://cathay-james.csie.org/prometheus
            volumes:
            - ./prometheus.yml:/etc/prometheus/prometheus.yml
```

`$ docker-compose up -d`