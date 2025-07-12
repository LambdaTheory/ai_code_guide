# DokPloy 部署常见问题 & 故障排查清单

## Dokply 常见问题

### 容器是 Healthy 状态 Dokploy 才能解析域名

**只有健康状态的容器才会注册到内部 DNS**，如果健康检查未通过，容器就无法在 DNS 中注册，从而导致无法通过服务名访问。

注意⚠️ 服务状态正常并不意味着容器是 Healthy 的，具体要看配置的 healthcheck。

## Docker 常见问题

### 如何做 healthcheck

1. **在 Dockerfile 中声明 HEALTHCHECK**

   ```Dockerfile
   HEALTHCHECK --interval=30s --timeout=5s --retries=3 CMD curl -f http://localhost:8080/health || exit 1
   ```

   * `--interval`：两次健康检查之间的间隔时间。
   * `--timeout`：单次检查的超时时间。
   * `--retries`：连续失败次数，超过即标记为 `unhealthy`。

2. **在 ************`docker-compose.yml`************ 中编写 healthcheck 块**

   ```yaml
   services:
     web:
       image: example/web:latest
       healthcheck:
         test: ["CMD", "wget", "-q", "-O", "-", "http://localhost:8080/health"]
         interval: 30s
         timeout: 5s
         retries: 3
   ```

3. **验证健康状态**

   ```bash
   docker ps --format '{{.Names}}\t{{.Status}}'
   docker inspect --format '{{json .State.Health}}' <container_id>
   ```

4. **常见陷阱与最佳实践**

   * **依赖尚未就绪**：应用在依赖数据库或消息队列未启动完成前返回失败，可在检查脚本中增加依赖检测逻辑。
   * **命令返回码**：健康检查脚本必须在成功情况下返回 0；任何非零返回码都会被 Docker 视为失败。
   * **内部端口**：健康检测应直接访问容器内部端口，避免使用宿主机映射端口。

5. 没有 curl 命令时，可以使用 wget，或根据情况使用 python/nodejs

### A 容器无法访问 B 容器的服务

1. **内部 DNS 解析规则**

   * Compose 中的 **服务名** = 容器主机名；其他容器通过该名称解析。
   * 调试命令：`getent hosts <service>`（解析不到时检查网络/拼写）。

2. **自定义网络的重要性**

   * 所需互联服务应加入同一 `networks:`（如 `app-net`）。
   * 否则处于默认隔离网络，DNS 互不可见。

3. **监听地址 vs 连接地址**

   * 服务进程应监听 **0.0.0.0**（即所有网卡），否则其他容器无法访问。
   * 客户端应通过 `http://<service>:<port>` 进行连接，而不是 127.0.0.1 或宿主机 IP。
   * 遇到 `Connection refused` 时，先确认服务进程监听地址、端口及防火墙规则。

4. **端口映射与安全**

   * **仅内部通信**：删除 `ports:`，利用 Docker 内部网络进行通信，降低暴露面。
   * **需要外部访问**：使用显式映射，例如 `8080:8080`，方便运维与监控。
   * **避免随机端口**：随机高位口会破坏自动化脚本及安全策略；生产环境应固定端口。
   * **访问控制**：通过防火墙、反向代理或安全组限制来源，防止未授权外部访问。
