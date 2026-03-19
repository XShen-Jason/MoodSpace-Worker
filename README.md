# ⚡ MoodSpace 极速边缘网关 (Cloudflare Worker)

这就是抗住千万级高并发的绝对核心。
作为一个纯粹的“只读”渲染引擎，它直接被部署在全球数以千计的 Cloudflare 边缘服务器上（Serverless 架构）。
它的唯一职责是：**无情拦截所有平台二级子域名的请求，光速去查 KV 里该子域名的所有者配置，并通过抓取 R2 的 HTML 源码，在这颗地球上离用户物理位置最近的节点直接把成品的网页拼装好，甩在用户脸上！**

---

## 🏗️ 核心技术栈
- **运行环境**: Cloudflare Workers
- **存储驱动**: Cloudflare KV (存储路由、白名单配置) + Cloudflare R2 (提取模板的 HTML 骨架)
- **协议**: HTTPS / HTTP/3 原生支持

---

## 💻 开发者：本地调试与部署指南

如果你要在此地修改这段精良的组装逻辑代码：

1. **克隆代码到本地**:
   ```bash
   git clone https://github.com/XShen-Jason/MoodSpace-Worker.git
   cd MoodSpace-Worker
   ```
2. **安装本地依赖**:
   ```bash
   npm install
   ```
2. **连接你的 CF 控制台 (必须)**:
   ```bash
   npx wrangler login
   ```
   *这一步会在你的默认浏览器弹窗登入 Cloudflare，这是你能调试或上传 Worker 的前提。*
   如果你是在自动化的脚本或无界面环境下（比如 CI/CD），你需要通过添加系统环境变量设置：
   `set CLOUDFLARE_API_TOKEN="your-api-token"`（具有 Edit Workers 的权限）。
3. **本地沙盒模拟测试**:
   ```bash
   npm run dev
   ```

---

## ⚙️ 配置文件保姆级详解 (`wrangler.toml`)

当你要把这个超级引擎发往全世界时，唯一的配置文件就是你的 `wrangler.toml`。小白必读：

```toml
name = "moodspace-worker"
main = "src/index.js"
compatibility_date = "2024-03-05"

# 这里的路由用来告诉 CF，当访问哪些网址时，才召唤这个 Worker 处理
# pattern="*." 意味着你把这套网站上无数用户的二级子域名请求全部一把抓进来了
routes = [
  { pattern = "*.moodspace.xyz", zone_name = "moodspace.xyz" }
]

# 把你 Cloudflare 账户的大脑 (KV) 接入给这个引擎
[[kv_namespaces]]
binding = "MOODSPACE_KV"
# 💡 去 CF 面板左边栏 -> Workers & Pages -> KV 里找到那个名字是 MOODSPACE_KV 的那一条
# 把旁边的 "ID" (长长的 32 位乱码字符串) 填进下面这里：
id      = "你必须把这行中文字删了然后粘上那个32位ID"

# 让你所有的模板实体文件都能被这个引擎直接读取
[[r2_buckets]]
binding = "MOODSPACE_R2"
# 必须完全等同于你在 CF 里新建的那个 R2 存储桶名字
bucket_name = "moodspace-templates"
```

修改完上面的内容后，执行核爆命令将它洒向全球：
```bash
npm run deploy
```

---

## 🚨 致命关键设定 1：主站 “免检通道” (None 模式)

这是一件绝对不能忘的事！由于我们在 `wrangler.toml` 中拦截了 `*.moodspace.xyz`，Cloudflare 会忠诚地把类似于 `www.moodspace.xyz` 或者 `api.moodspace.xyz` 的访问也错误地踢进了这个渲染引擎里，从而引发惨烈的“找不对 KV”导致主站大面积报 `500 Internal Server Error` 崩溃的事故！

你必须要登入 Cloudflare 面板，在左侧的 **Workers Routes** 设置里添加几条优先级最高的绕过规则：
1. Route: `www.moodspace.xyz/*` -> Worker 分配给: **None** (意思是不处理，放行给主 VPS)
2. Route: `api.moodspace.xyz/*` -> Worker 分配给: **None**
3. Route: `moodspace.xyz/*` -> Worker 分配给: **None**

---

## 🚨 致命关键设定 2：添加密码 (Secret) 至控制台

这个系统内置了极其强大的 API 隔离防御墙。
但是安全密钥类的东西不能写死在代码或者 `.toml` 配置文件里，**每次你新建或部署了全新名字的 Worker 时，都必须手动加密码！**

去你的 Cloudflare Dashboard -> Workers & Pages -> 点击 `moodspace-worker` -> Settings -> **Variables**。
在这里，点击 `Add variable`，类型选择 **Secret (加密)**：
- Key: 填入 `ADMIN_KEY`
- Value: 填入你在 Backend `.env` 里设计的那个超强密码！

没有这个暗号匹配，后台的 API 指令就会被它全部当做黑客流量拒之门外！
