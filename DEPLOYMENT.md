# UGREEN NAS Docker 部署指南

本方案只通过 Tailscale 私网访问，由 Nginx Proxy Manager（NPM）负责域名和 HTTPS。Redis 与 Webdis 均不暴露宿主机端口。

```text
Tailnet 客户端 -> NPM (HTTPS) -> cloud-clipboard-web:80 -> Webdis:7379 -> Redis:6379
```

## 前提

- UGREEN NAS 已安装 Docker，并支持 `docker compose`。
- Tailscale 和 NPM 已正常运行，域名通过 Tailnet 指向 NPM，客户端信任该域名的 HTTPS 证书。
- NPM 已加入 Docker 网络 `nginx_proxy_manager_default`。
- 客户端可以访问 jsDelivr；本项目的 Bootstrap、Bootstrap Icons 和 Axios 仍从 CDN 加载。

在 NAS 上确认 NPM 网络名：

```bash
docker network inspect nginx_proxy_manager_default
```

如果实际名称不同，修改 `docker-compose.yml` 中 `networks.npm.name` 的值。

## 准备文件

将整个仓库复制或克隆到 NAS 的一个目录，确保至少包含以下文件：

```text
docker-compose.yml
nginx.conf
webdis.json
index.html
favicon.png
apple-touch-icon.png
```

进入项目目录并验证配置。该命令只解析配置，不会启动容器：

```bash
cd /path/to/cloud-clipboard
docker compose config --quiet
```

## 配置 NPM

在 NPM 中新增 Proxy Host：

- Domain Names：填写你的私有域名。
- Scheme：`http`。
- Forward Hostname / IP：`cloud-clipboard-web`。
- Forward Port：`80`。
- SSL：选择有效证书，并启用 Force SSL。

在 Advanced 中加入：

```nginx
client_max_body_size 128m;
proxy_buffering off;
proxy_request_buffering off;
proxy_read_timeout 86400s;
proxy_send_timeout 86400s;
```

不要在路由器上将该域名对应的 80/443 端口开放到公网。本方案依赖 Tailscale 控制访问身份，因此没有额外启用 Basic Auth。

## 启动

由你在 NAS 上执行：

```bash
docker compose pull
docker compose up -d
docker compose ps
```

查看启动日志：

```bash
docker compose logs --tail=100 web webdis redis
```

## 首次使用

通过 `https://你的域名` 打开网页。默认 Redis 配置为：

```json
{
  "host": "/webdis/",
  "prefix": "cloud-clipboard"
}
```

如果同一域名以前保存过其他配置，网页会优先使用浏览器 `localStorage` 中的旧值。可在页面底部修改 Redis 信息，或在浏览器控制台执行：

```javascript
localStorage.removeItem('redis');
location.reload();
```

浏览器首次读取或写入剪贴板时，需要允许剪贴板权限并按提示进行用户操作。Linux、Windows 或 iOS 客户端使用完整 Webdis 地址：

```text
https://你的域名/webdis/
```

地址末尾的 `/` 不能省略。

## 验收

将下面命令中的域名替换为实际域名。

1. Webdis 可用：

   ```bash
   curl -sS https://你的域名/webdis/PING
   ```

   应返回包含 `PONG` 的 JSON。

2. 危险命令被拒绝：

   ```bash
   curl -i https://你的域名/webdis/FLUSHALL
   ```

   应返回 HTTP 403。

3. 内部端口未暴露：

   ```bash
   curl --connect-timeout 3 http://NAS_IP:7379/PING
   nc -vz -w 3 NAS_IP 6379
   ```

   两个连接都应失败。Web 服务同样没有映射宿主机端口，只能由共享 Docker 网络中的 NPM 访问。

4. 使用两台 Tailnet 设备分别验证文本、图片和文件的推送、拉取与删除。

5. 保持页面打开超过 60 秒，再从另一台设备更新内容，确认页面能够实时刷新。

6. 重启服务后确认数据仍存在：

   ```bash
   docker compose restart
   ```

7. 小于 128 MiB 的上传应成功；超过限制的请求应返回 HTTP 413。

## 数据与日常维护

Redis 使用 AOF，每秒同步一次，数据保存在命名卷 `cloud-clipboard_redis-data` 中。查看卷信息：

```bash
docker volume inspect cloud-clipboard_redis-data
```

更新项目和镜像：

```bash
git pull --ff-only
docker compose pull
docker compose up -d
```

停止或删除容器不会删除数据卷：

```bash
docker compose stop
docker compose down
```

以下命令会永久删除 Redis 数据，仅在确认不再需要数据时使用：

```bash
docker compose down -v
```

## 故障排查

### NPM 返回 502

- 确认 `docker network inspect nginx_proxy_manager_default` 能看到 NPM 和 `cloud-clipboard-web`。
- 确认 NPM 上游使用 `cloud-clipboard-web:80`，Scheme 为 `http`。
- 检查 `docker compose ps` 和 `docker compose logs web webdis redis`。

### 上传返回 413

- 确认 NPM Advanced 中存在 `client_max_body_size 128m;`。
- 项目 Nginx 和 Webdis 的上限均为 128 MiB；更大的文件会被有意拒绝。

### 网页无法读取或写入剪贴板

- 必须使用证书受信任的 HTTPS 域名，不能直接使用普通 HTTP NAS 地址。
- 检查浏览器的剪贴板权限，并通过页面按钮触发操作。

### 实时更新停止

- 确认 NPM 已关闭请求和响应缓冲，并将读写超时设为 86400 秒。
- 在浏览器开发者工具中确认 `/webdis/SUBSCRIBE/cloud-clipboard` 请求保持连接。

### 网页仍连接公共 Webdis

清除该域名下的 `localStorage.redis`，然后刷新页面。

### 重启后数据丢失

- 确认 `cloud-clipboard_redis-data` 卷存在并挂载到 Redis 的 `/data`。
- 检查 Redis 日志中是否存在 AOF 加载或写入错误。
- 确认没有执行过 `docker compose down -v`。

### 页面样式缺失或请求代码未加载

确认客户端能够访问 `cdn.jsdelivr.net`。本部署没有内置 CDN 资源。
