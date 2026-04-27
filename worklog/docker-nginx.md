# Docker + nginx 踩坑

**最近更新**：2026-04-27

---

## 踩坑记录

### 1. 健康检查失败导致 unhealthy 但容器不重启
**问题**：`docker ps` 显示 `(unhealthy)`，但容器没重启
**原因**：健康检查配置了但 `restart: unless-stopped` 不会因为 unhealthy 就重启
**解决**：
- 用 `docker compose up -d --force-recreate` 强制重建
- 或配置 `restart: always`（但仍不会自动重建 unhealthy 容器）
- 最佳方案：手动 `docker compose up -d --force-recreate <service>`

---

### 2. docker-compose 启动顺序（depends_on）
**问题**：服务启动顺序不对导致连不上
**解决**：`depends_on` 确保顺序，但要注意 **不保证容器内服务已就绪**
```yaml
services:
  web:
    depends_on:
      - redis
  redis:
    image: redis:alpine
```
深度依赖用 `wait-for-it.sh` 或 `healthcheck` 轮询。

---

### 3. volume 持久化后删除容器数据是否保留
**问题**：删了容器，volume 里的数据还在吗
**解决**：
- `docker compose down` → volume 保留
- `docker compose down -v` → volume 删除
- 重建容器前先确认 `down` 还是 `down -v`

---

### 4. 环境变量文件 .env 加载
**问题**：docker-compose 读不到 .env 里的变量
**解决**：`.env` 文件放在 `docker-compose.yml` 同目录，自动加载。
变量名格式：`VARIABLE_NAME=value`，使用时 `${VARIABLE_NAME}`

---

## docker-compose 模板（nginx + cloudflared）

```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: nginx-nginx-1
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - /var/lib/letsencrypt:/var/lib/letsencrypt
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared-tunnel
    restart: unless-stopped
    command: tunnel run --token ${CLOUDFLARED_TOKEN}
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARED_TOKEN}
```

---

## 常用命令

```bash
# 重建并重启服务
docker compose up -d --force-recreate <service>

# 查看日志
docker compose logs -f <service>

# 查看健康状态
docker ps --format "table {{.Names}}\t{{.Status}}"

# 停服务（保留 volume）
docker compose down

# 停服务（删除 volume）
docker compose down -v

# 重启服务
docker compose restart <service>
```
