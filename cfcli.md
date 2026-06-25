# WARP 客户端命令（warp-cli）

| 分类         | 命令                                              | 说明                            |
| ---------- | ----------------------------------------------- | ----------------------------- |
| **初始连接**   | `warp-cli registration new`                     | 首次使用，注册设备到 Cloudflare         |
|            | `warp-cli connect`                              | 建立 WARP 连接                    |
|            | `warp-cli disconnect`                           | 断开 WARP 连接                    |
| **状态查看**   | `warp-cli status`                               | 查看当前连接状态和网络健康度                |
|            | `curl https://www.cloudflare.com/cdn-cgi/trace` | 验证 WARP 是否生效（查找 `warp=on`）    |
| **模式切换**   | `warp-cli mode doh`                             | DNS only 模式（仅 DNS over HTTPS） |
|            | `warp-cli mode warp+doh`                        | WARP + DoH 模式（默认）             |
| **隧道协议**   | `warp-cli tunnel protocol set MASQUE`           | 使用 MASQUE 协议（默认）              |
|            | `warp-cli tunnel protocol set WireGuard`        | 切换到 WireGuard 协议              |
| **DNS 过滤** | `warp-cli dns families off`                     | 关闭 1.1.1.1 Families 保护        |
|            | `warp-cli dns families malware`                 | 仅拦截恶意软件                       |
|            | `warp-cli dns families full`                    | 拦截恶意软件 + 成人内容                 |
| **高级功能**   | `warp-cli enable-always-on`                     | 开机自动重连                        |
|            | `warp-cli account`                              | 查看账户信息                        |
|            | `sudo warp-diag`                                | 生成调试信息压缩包                     |
|            | `sudo warp-diag feedback`                       | 提交反馈/报错给官方团队                  |
