# 服务器部署指南

从 `git clone` 到运行验证的完整流程。Docker Compose + Caddy 反向代理，公网只开放 80/443。

新增文件（相对仓库根目录）：

- `compose.server.yaml` — 服务器版 compose（含 Caddy 服务）
- `Caddyfile` — 反向代理配置
- `.env.example` — 环境变量模板

## 架构

```
客户端 → Caddy(80/443, 自动 HTTPS) → opencode2api 容器(8080 API / 8081 WebUI)
```

容器端口不直接映射公网（Docker 会绕过 ufw 写 iptables），只能经 Caddy 访问。

## 前置条件

- Linux 服务器（以 Ubuntu 22.04+ 为例），Docker 已安装
- 域名 A 记录指向服务器（没有域名见[附录 A](#附录-a无域名方案)）
- 防火墙/安全组开放 22、80、443

安装 Docker：

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER && newgrp docker
```

## 部署步骤

### 1. 克隆代码

```bash
cd /opt
sudo git clone https://github.com/5575xyx/opencode2api.git
cd opencode2api
```

### 2. 准备配置

```bash
cp config.example.json config.json
cp .env.example .env
```

编辑 `config.json`，**必改三处**：

| 字段 | 说明 |
| --- | --- |
| `server_keys` | 本地 API Key，用 `openssl rand -hex 24` 生成 |
| `zen_keys` / `go_keys` | 上游 Key；或设 `"anonymous": true` 并清空两者 |
| `webui.password` | WebUI 登录密码，至少 10 位 |

### 3. 配置域名

编辑 `Caddyfile`：两个域名用方案一，一个域名用方案二（路径分流），都需把 `example.com` 换成自己的域名。

### 4. 选择镜像来源

海外服务器：`.env` 默认（GHCR 官方镜像）即可。

国内服务器（ghcr.io 拉取慢）：

```bash
docker build -t opencode2api:local .
# 并把 .env 改为 OPENCODE2API_IMAGE=opencode2api
```

### 5. 防火墙

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

云服务器安全组同步只放行这三个端口。

### 6. 启动

```bash
docker compose -f compose.server.yaml up -d
docker compose -f compose.server.yaml logs -f
```

日志出现 `Initialized ... config.json` 和 `listen 0.0.0.0:8080` 即初始化正常。

### 7. 验证

```bash
docker compose -f compose.server.yaml ps                          # 状态应为 healthy
curl https://你的API域名/healthz                                    # 无需认证
curl https://你的API域名/v1/models -H "Authorization: Bearer 你的server_key"
```

浏览器打开 `https://你的管理域名`，用户名 `admin`，密码为 `config.json` 里的 `webui.password`。

## 运维

### 升级

```bash
cd /opt/opencode2api
git pull
docker compose -f compose.server.yaml pull   # 本地构建方式则：docker build -t opencode2api:local .
docker compose -f compose.server.yaml up -d
docker compose -f compose.server.yaml logs -f --tail 50
```

`.env` 里固定版本号（`OPENCODE2API_VERSION=v1.0.0`）比 `latest` 更可控。

### 修改配置

**宿主机 `config.json` 只在首次启动导入**，之后以状态卷为准：

- 推荐：WebUI → 配置中心修改，保存即热重载
- 或强制重新导入：`docker compose -f compose.server.yaml cp config.json opencode2api:/var/lib/opencode2api/config.json && docker compose -f compose.server.yaml restart`

### 备份与恢复

状态卷含上游 Key 和密码哈希，**属敏感数据，加密后异地存放**：

```bash
mkdir -p /opt/backups
sudo docker run --rm -v opencode2api_opencode2api-state:/data -v /opt/backups:/backup alpine \
  tar czf /backup/opencode2api-state-$(date +%F).tgz -C /data .
```

恢复：

```bash
sudo docker run --rm -v opencode2api_opencode2api-state:/data -v /opt/backups:/backup alpine \
  tar xzf /backup/opencode2api-state-YYYY-MM-DD.tgz -C /data
docker compose -f compose.server.yaml restart
```

### 日志与监控

- 日志：`docker compose -f compose.server.yaml logs -f`（stdout JSON，长期保留需外部收集）
- 监控统计在内存中，重启清空

## 附录 A：无域名方案

- **API 自用**：不改 compose，本地执行 `ssh -L 8080:127.0.0.1:8080 -L 8081:127.0.0.1:8081 user@服务器IP`，访问 `http://localhost:8081`（WebUI）和 `http://localhost:8080`（API）。防火墙只需 22。
- **API 他人使用**：把 `compose.server.yaml` 中 `expose` 改为 `ports: - "8080:8080"`，配强 `server_keys` 并在安全组限制来源 IP。明文 HTTP，仅适合内网或临时场景。

## 附录 B：国内服务器访问上游

上游为 `https://opencode.ai/zen`，国内直连可能超时。在 `config.json` 中配置 `proxies`（如 `["http://user:pass@代理:端口", "direct"]`），或使用 `proxyfile`：

```bash
echo "socks5://127.0.0.1:1080" > proxy.txt
# compose.server.yaml 中取消 proxy.txt 挂载行的注释，config.json 里设 "proxyfile": "proxy.txt"
```

## 常见问题

| 现象 | 排查方向 |
| --- | --- |
| 访问返回 502 | 看 `docker compose -f compose.server.yaml logs caddy`；主服务是否 healthy |
| 证书签发失败 | 80/443 是否开放、域名解析是否正确、Caddy 日志 |
| 镜像拉取超时 | 改用本地构建（步骤 4） |
| 容器 unhealthy | 看主服务日志；`LISTEN_ADDRESS` 若改过需同步调整健康检查端口 |
| API 返回 401 | 请求 Key 与 `config.json` 的 `server_keys` 不一致 |
| 改宿主机 `config.json` 不生效 | 正常，配置在状态卷，用 WebUI 或 `docker compose cp` 重新导入 |
| 请求 502 / 504 | 上游 Key、代理连通性、超时设置，看 WebUI 路由诊断 |
| HTTP 200 但生成失败 | SSE 错误事件，看 WebUI 请求结果 |
