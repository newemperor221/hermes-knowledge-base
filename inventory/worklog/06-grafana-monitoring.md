# Grafana 监控仪表盘搭建 (2026-04-27)

## 完成状态

- Prometheus: node + nginx targets up
- node_exporter: 9100
- nginx stub_status: /nginx_status
- nginx-prometheus-exporter: 9113
- Grafana 数据源: Prometheus (uid: bfkbn9vdqh4owf)
- 仪表盘: 3个 (Node Exporter Full, NGINX exporter, Atlanta Node Exporter)

## 关键操作

### 1. nginx stub_status 配置

在 nginx.conf (conf.d/default.conf) 添加独立 server 监听 80 端口处理 stub_status：

```nginx
server {
    listen 80;
    server_name _;
    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        allow 172.0.0.0/8;
        allow 192.168.0.0/16;
        allow 10.0.0.0/8;
        deny all;
    }
    location / {
        return 301 https://$host$request_uri;
    }
}
```

踩坑：nginx 多个 server block 时，server_name 匹配优先于 listen 端口。stub_status 和 HTTPS 重定向混在同一个 server 里会导致 301 跳转。解决：stub_status 单独 server 块用 `server_name _;`（默认 server）。

### 2. nginx-prometheus-exporter

```bash
docker run -d --name nginx-exporter \
  --network host \
  --restart unless-stopped \
  nginx/nginx-prometheus-exporter:1.3.0 \
  --nginx.scrape-uri=http://127.0.0.1/nginx_status
```

踩坑：
- --network host 才能访问宿主机的 127.0.0.1 stub_status
- Docker 内部网络（如 nginx_default 172.18.0.0/16）访问不到宿主机 127.0.0.1
- Prometheus 和 nginx-exporter 都用 host 网络才能互相访问

### 3. Prometheus 配置

```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['23.95.218.144:9100']
  - job_name: 'nginx'
    static_configs:
      - targets: ['23.95.218.144:9113']
```

### 4. Grafana 数据源

```bash
curl -X POST http://admin:admin@127.0.0.1:3000/api/datasources \
  -H 'Content-Type: application/json' \
  -d '{"name":"Prometheus","type":"prometheus","url":"http://127.0.0.1:9090","access":"proxy","isDefault":true}'
```

### 5. 仪表盘导入

三个仪表盘：
- Node Exporter Full (grafana.com ID 1860) - 全面系统指标
- NGINX exporter (grafana.com ID 12708) - nginx 请求/连接指标
- Atlanta Node Exporter (自制) - 快速总览

导入流程：
1. 从 grafana.com 下载 JSON
2. 批量替换 datasource 占位符 `${ds_prometheus}` 为实际 uid
3. POST /api/dashboards/db 上传

### 6. UFW 放行

```bash
sudo ufw allow 9113/tcp   # nginx-prometheus-exporter
```

## 访问地址

- Grafana: http://23.95.218.144:3000 （admin/admin）
- Prometheus: http://23.95.218.144:9090
- node_exporter metrics: http://23.95.218.144:9100/metrics
- nginx metrics: http://23.95.218.144:9113/metrics
