# ⚡ Nginx 全栈性能与安全大满贯：动态 TLS + HTTP/2/3 + Brotli + TLS 1.3 终极配置

## 🖥️ Nginx - Server [[Nginx Version]](https://nginx.org)

这是一份兼顾**极致传输速度 (Speed)**、**边缘缓存控制 (Caching)** 与 **顶级现代安全 (Security)** 的 Nginx 站点配置。通过动态 TLS 记录减小首包延迟，利用 Brotli 算法压缩体积，同时加入 HSTS 强制纯血安全握手，并针对静态资源部署强缓存策略，彻底阻断用户的二次无效请求。

---

## 🛠️ 前置依赖说明

> ⚠️ **核心注意：** 本配置中部分高级指令非 Nginx 官方标准内置：
1. **`ssl_dyn_rec_*`**：由 **Cloudflare 动态 TLS 记录大小补丁** 提供。
2. **`brotli*`**：由 Google 开源的 `ngx_brotli` 模块提供，需在编译时引入。

*如果你使用的是 standard nginx，建议更换为预集成了上述特性的发行版（如阿里 **Tengine**、或社区集成了扩展补丁的编译版本）。*

---

## 🚀 终极全能优化配置模板

请将以下配置应用到你的 Nginx `server` 块中：

```nginx
server {
    # --------------------------------------------------
    # 🌐 1. 多协议网络监听与双栈
    # --------------------------------------------------
    listen 443 ssl;                 # 传统 TCP 监听
    listen 443 quic reuseport;      # HTTP/3 (QUIC) 专属 UDP 监听 (需 Nginx 1.25.0+)
    
    server_name example.com;

    # --------------------------------------------------
    # 🔒 2. 顶级安全硬化：TLS 1.3 & HSTS
    # --------------------------------------------------
    ssl_certificate     cert.pem;
    ssl_certificate_key cert.key;

    # 彻底废弃过时的 TLS 1.0/1.1/1.2，仅允许纯血 TLS 1.3
    ssl_protocols TLSv1.3;
    
    # 选用 TLS 1.3 速度与安全性最佳的加密算法
    ssl_ciphers TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256;
    
    # 开启会话复用缓存，10m 约可缓存 40000 个 Session
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets on;       # 结合 H3 提速建连

    # 【关键安全头：HSTS】强制浏览器在未来 1 年内（31536000秒）自动将该域名的所有 HTTP 请求转为 HTTPS 握手
    # includeSubDomains: 包含子域名; preload: 允许加入 HSTS 预加载列表
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # --------------------------------------------------
    # ⚡ 3. 动态 TLS 记录大小优化 (仅对 HTTP/1.1 & HTTP/2 生效)
    # --------------------------------------------------
    ssl_dyn_rec_enable    on;     # 激活动态调整机制
    ssl_dyn_rec_size_lo   1369;   # 初始小记录（完美容纳 1 个明文 TCP 包，秒开首屏）
    ssl_dyn_rec_size_hi   4229;   # 承上启下记录（适应约 3 个 TCP 包，慢启动过渡）
    ssl_dyn_rec_threshold 40;     # 发送 40 个记录后升级到全速大包 (16KB) 模式
    ssl_dyn_rec_timeout   1s;     # 连接空闲 1 秒自动降级，优化下一次交互

    # --------------------------------------------------
    # 🏎️ 4. HTTP/2 / HTTP/3 协议层深度加速
    # --------------------------------------------------
    http2 on;                     # 显式开启 HTTP/2
    http2_max_concurrent_streams 128; 
    http2_recv_buffer_size       256k;
    http2_chunk_size             8k;  # 小块响应，配合 TLS 动态小包极速吐出关键资源

    http3 on;                     # 显式开启 HTTP/3
    add_header Alt-Svc 'h3=":443"; ma=86400'; # 通告 H3 支持
    ssl_early_data on;            # 允许 QUIC 0-RTT 极速重建连

    # --------------------------------------------------
    # 📦 5. 智能双压缩战略：Brotli 优先 + Gzip 兼容
    # --------------------------------------------------
    brotli on;
    brotli_comp_level 6;          # 兼顾 CPU 消耗与高压缩比的黄金级别
    brotli_buffers 16 8k;
    brotli_min_length 20;
    brotli_types text/plain text/css text/xml application/javascript application/json image/svg+xml;

    gzip on;
    gzip_comp_level 5;
    gzip_min_length 256;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/plain text/css text/xml application/javascript application/json image/svg+xml;
    gzip_vary on; 

    # --------------------------------------------------
    # 💾 6. 静态资源极限强缓存策略 (Cache-Control & Expires)
    # --------------------------------------------------
    # 针对：图片、音视频、字体等极其稳定的多媒体资源
    location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc|woff|woff2|ttf)\$ {
        expires 365d;             # 向旧版客户端发送 1 年的 Expires 绝对时间
        access_log off;           # 关闭静态资源日志，减轻磁盘 I/O 负担
        # public: 允许中间代理/CDN及浏览器缓存; immutable: 告知浏览器该资源永远不会变，无需发起任何验证请求(304)
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    # 针对：可能随版本更新而经常发生代码变动的 CSS 和 JS 资源 (推荐使用文件 Hash 后缀命名)
    location ~* \.(?:css|js)\$ {
        expires 30d;              # 强缓存 30 天
        access_log off;
        add_header Cache-Control "public, max-age=2592000, must-revalidate"; # 允许缓存，但过期后必须回源校验
    }
}
```

---

## 📊 新增优化策略深度解析

| 策略维度 | 核心指令 | 优化原理与效果 |
| :--- | :--- | :--- |
| **🛡️ 纯血安全 (HSTS)** | `Strict-Transport-Security` | 拒绝中间人篡改。让浏览器记住“此生只走 HTTPS”，哪怕用户手动输入 `http://`，浏览器也会在本地拦截并秒转加密请求，**规避首次 HTTP 握手的劫持风险**。 |
| **💾 极限强缓存 (Immutable)** | `max-age=31536000, immutable` | **0 次网络请求**。带有 `immutable` 的资源，即使用户在浏览器中狂按 F5 刷新，浏览器也绝不会向服务器发送一条查询请求（连 `304 Not Modified` 都不发），直接从本地内存/磁盘秒秒级加载！ |
| **⚡ 动态校验 (Revalidate)** | `must-revalidate` | 安全过渡。对前端打包构建出的 `app.hash.js` / `style.hash.css` 文件进行 30 天缓存。若后端版本发生变化，文件 Hash 一变立刻回源拉取最新版，兼顾速度与业务更新。 |

---

## 🏆 终极收益
至此，你的 Nginx 已经进化为完全体：
1. **首次访问**：通过 **HSTS** 极速切入安全轨道，使用 **TLS 1.3** + **动态 TLS 记录** 达成最快的首字节建连（TTFB）。
2. **内容下载**：应用层体积经过 **Brotli** 高比例压缩，数据块大小与 TCP 底层标准 MTU 极限契合，顺着 **HTTP/2/3** 管道并发表白。
3. **二次访问**：所有音视频、字体、图片全部命中本地 **Immutable 强缓存**，服务器直接收回 0 条请求，下秒直接呈现。
