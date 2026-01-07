# 分布式部署方案 (宝塔面板版)

根据您的需求，我们将采用 **3台服务器** 进行分布式部署，并使用宝塔面板进行管理。

## 服务器规划
| 服务器角色   | 最低配置  | 部署内容                  | 说明                  |
| :------ | :---- | :-------------------- | :------------------ |
| **Web** | 2核 4G | 前端静态资源 (Nginx)        | 负责静态页面分发、反向代理、多域名接入 |
| **API** | 4核 8G | 后端服务 (NestJS + PM2)   | 负责业务逻辑、API 处理       |
| **DB**  | 4核 8G | MySQL 8.0 + Redis 7.0 | 负责数据存储，需开启远程访问      |

## 详细部署步骤

### 第一步：数据库服务器（DB）

**目标**：安装数据库并开启远程访问，供 API 连接。

1.  **安装软件**：

    - 登录宝塔面板 -> **软件商店**。

    - 安装 **MySQL 8.0** 和 **Redis 7**

2.  **配置 MySQL 远程访问**：

    - 宝塔 -> **数据库** -> **root密码** (记下密码)。

    - 点击 **权限** (或在“安全”页面放行 3306 端口)。

    - 建议：仅允许 **Server API 的 IP 地址** 访问 3306 端口（更安全），或者为了测试方便暂时允许所有 IP。

    - **导入数据**：将现有服务器的数据库导出为 SQL，在此处导入。

3.  **配置 Redis 远程访问**：

    - 宝塔 -> **软件商店** -> Redis -> **设置**。

    - **配置文件**：找到 `bind 127.0.0.1`，改为 `# bind 127.0.0.1` (注释掉) 或 `bind 0.0.0.0`。

    - **requirepass**：设置一个复杂的密码。

    - **安全**：在宝塔“安全”页面放行 6379 端口。

    - 重启 Redis 服务。
   
### 第二步：后端服务器 (Server API)

**目标**：部署 NestJS 后端，连接远程数据库。

1.  **环境准备**：

    - 宝塔 -> **网站** -> **Node项目** -> **Node版本管理器** -> 安装 **Node v20+** (建议 v20 或 v22)。

    - 安装 **PM2** (通常 Node 管理器会自带，或手动 `npm install -g pm2`)。

2.  **部署代码**：

    - 将 `Admin` 文件夹上传到服务器（如 `/www/wwwroot/pi-admin-backend`）。

    - 进入目录，删除 `node_modules` (如果以前有)，重新运行 `npm install --production`。

3.  **配置环境变量 (.env)**：

    - 修改 `.env` 文件

5.  **启动服务**：

    - 宝塔 -> **网站** -> **Node项目** -> **添加Node项目**。

    - **项目目录**：选择上传的目录。

    - **启动选项**：`npm run start:prod` 或 `dist/src/main.js`。

    - **端口**：3000。

    - **绑定域名**：可以绑定 API 域名 (如 `api.example.com`)。

    - **反向代理**：如果不想直接暴露 3000，可以用 Nginx 反代（宝塔自动配置）。

### 第三步：前端服务器 (Server Web)

**目标**：部署 React 前端，通过 Nginx 转发 API 请求，支持多域名。

1.  **构建前端 (本地或任意服务器)**：

    - 在 `Index` 目录下，确保 `.env.production` 内容如下：

    ```env

    VITE_API_URL=/api  # 关键：使用相对路径，由 Nginx 负责转发

    ```
    - 执行 `npm run build`，生成 `dist` 目录。

2.  **部署代码**：

    - 将 `dist` 目录内的所有文件上传到 Server Web 的 `/www/wwwroot/pi-admin-frontend`。

3.  **配置 Nginx (宝塔网站)**：

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