# 分布式部署方案 (宝塔面板版)

根据您的需求，我们将采用 **3台服务器** 进行分布式部署，并使用宝塔面板进行管理。

## 服务器规划
| 服务器角色          | 最低配置  | 部署内容                  | 说明                  |
| :------------- | :---- | :-------------------- | :------------------ |
| **Server Web** | 2核 4G | 前端静态资源 (Nginx)        | 负责静态页面分发、反向代理、多域名接入 |
| **Server API** | 4核 8G | 后端服务 (NestJS + PM2)   | 负责业务逻辑、API 处理       |
| **Server DB**  | 4核 8G | MySQL 8.0 + Redis 7.0 | 负责数据存储，需开启远程访问      |

## 详细部署步骤

### 第一步：数据库服务器 (Server DB)
**目标**：安装数据库并开启远程访问，供 Server API 连接。

1.  **安装软件**：
    - 登录宝塔面板 -> **软件商店**。
    - 安装 **MySQL 8.0** 和 **Redis 7.0**。
2.  **配置 MySQL 远程访问**：
    - 宝塔 -> **数据库** -> **root密码** (记下密码)。
    - 点击 **权限** (或在“安全”页面放行 3306 端口)。
    - 建议：仅允许 **Server API 的 IP 地址** 访问 3306 端口（更安全），或者为了测试方便暂时允许所有 IP。
    - **导入数据**：将现有服务器的数据库导出为 SQL，在此处导入。
3.  **配置 Redis 远程访问**：
    - 宝塔 -> **软件商店** -> Redis -> **设置**。
    - **配置文件**：找到 `bind 127.0.0.1`，改为 `# bind 127.0.0.1` (注释掉) 或 `bind 0.0.0.0`。
    - **requirepass**：设置一个复杂的密码。
    - **安全**：在宝塔“安全”页面放行 6379 端口。
    - 重启 Redis 服务。

### 第二步：后端服务器 (Server API)
**目标**：部署 NestJS 后端，连接远程数据库。

1.  **环境准备**：
    - 宝塔 -> **网站** -> **Node项目** -> **Node版本管理器** -> 安装 **Node v20+** (建议 v20 或 v22)。
    - 安装 **PM2** (通常 Node 管理器会自带，或手动 `npm install -g pm2`)。
2.  **部署代码**：
    - 将 `Admin` 文件夹上传到服务器（如 `/www/wwwroot/pi-admin-backend`）。
    - 进入目录，删除 `node_modules` (如果以前有)，重新运行 `npm install`。
3.  **配置环境变量 (.env)**：
    - 修改 `.env` 文件，指向 **Server DB** 的 IP。
    ```env
    DATABASE_URL="mysql://root:DB密码@Server_DB_IP:3306/pi_admin?schema=public"
    REDIS_HOST="Server_DB_IP"
    REDIS_PORT=6379
    REDIS_PASSWORD="Redis密码"
    # 前端域名配置 (用于CORS允许跨域)
    FRONTEND_URL="https://www.example.com" 
    # 如果有多个前端域名，用逗号分隔
    EXTRA_ALLOWED_ORIGINS="https://app.example.com,https://m.example.com"
    ```
4.  **数据库同步**：
    - 在终端执行：`npx prisma migrate deploy` (确保表结构同步)。
5.  **启动服务**：
    - 宝塔 -> **网站** -> **Node项目** -> **添加Node项目**。
    - **项目目录**：选择上传的目录。
    - **启动选项**：`npm run start:prod` 或 `dist/src/main.js`。
    - **端口**：3000。
    - **绑定域名**：可以绑定 API 域名 (如 `api.example.com`)。
    - **反向代理**：如果不想直接暴露 3000，可以用 Nginx 反代（宝塔自动配置）。

### 第三步：前端服务器 (Server Web)
**目标**：部署 React 前端，通过 Nginx 转发 API 请求，支持多域名。

1.  **构建前端 (本地或任意服务器)**：
    - 在 `Index` 目录下，确保 `.env.production` 内容如下：
    ```env
    VITE_API_URL=/api  # 关键：使用相对路径，由 Nginx 负责转发
    ```
    - 执行 `npm run build`，生成 `dist` 目录。
2.  **部署代码**：
    - 将 `dist` 目录内的所有文件上传到 Server Web 的 `/www/wwwroot/pi-admin-frontend`。
3.  **配置 Nginx (宝塔网站)**：
    - 宝塔 -> **网站** -> **PHP/纯静态项目** -> **添加站点**。
    - **域名**：填写所有需要的前端域名（如 `www.example.com`, `app.example.com`, `landing.example.com`）。
    - **根目录**：指向上传的文件夹。
    - **伪静态 (解决路由 404)**：
    ```nginx
    location / {
      try_files $uri $uri/ /index.html;
    }
    ```
    - **反向代理 (关键)**：
    点击 **配置文件** 或 **反向代理**，添加 `/api` 的转发规则，将其指向 **Server API**。
    ```nginx
    # 将 /api 开头的请求转发给后端服务器
    location /api/ {
        # Server API 的内网或公网 IP + 端口
        proxy_pass http://Server_API_IP:3000/api/; 
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    ```
    *注意：如果 Server API 绑定了域名（如 api.example.com），`proxy_pass` 也可以填 `http://api.example.com/api/`。*

## 关键细节检查
1.  **安全组/防火墙**：
    - 确保 **Server DB** 放行 3306 (MySQL) 和 6379 (Redis) 给 **Server API**。
    - 确保 **Server API** 放行 3000 (或 Nginx 80/443) 给 **Server Web**。
    - **Server Web** 需开放 80/443 给全网访问。
2.  **CORS (跨域)**：
    - 虽然采用了 Nginx 反代 `/api`，前端浏览器实际上是向 `www.example.com/api` 发请求，属于**同域请求**，**不会**产生跨域问题！这是最推荐的方案。
    - 如果您的方案是前端直连 `api.example.com`，则需要在后端的 `.env` 中正确配置 `EXTRA_ALLOWED_ORIGINS`。

## 立即执行建议
如果您确认这个架构，我可以先帮您：
1.  **修改前端代码**：确认 `request.ts` 和 `.env.production` 适配 Nginx 反代模式（目前代码已经支持动态获取或 `/api` 默认值，无需大改）。
2.  **修改后端代码**：检查 CORS 配置确保万无一失。
3.  **打包前端**：生成 `dist` 包供您下载或部署。

是否需要我先帮您打包前端，并生成一份详细的 Nginx 配置文件供您复制？