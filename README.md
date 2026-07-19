# RMMT

Roommate Matcher（舍友匹配系统）Docker 一键部署仓库。

本仓库通过 **Git Submodule** 引用三个子项目的 `React` 分支：

| 目录 | 仓库 | 分支 |
|------|------|------|
| `RMMT-API/` | [ZhiXin-College/RMMT-API](https://github.com/ZhiXin-College/RMMT-API) | `React` |
| `RMMT-Admin/` | [ZhiXin-College/RMMT-Admin](https://github.com/ZhiXin-College/RMMT-Admin) | `React` |
| `RMMT-Student/` | [ZhiXin-College/RMMT-Student](https://github.com/ZhiXin-College/RMMT-Student) | `React` |

统一入口由 `docker-compose.yml` + `deploy/nginx/` 反向代理提供。

---

## 架构概览

```
浏览器
  ├─ /           → rmmt-student
  ├─ /admin/     → rmmt-admin
  ├─ /api/       → rmmt-api
  └─ /static/    → rmmt-api（上传的 logo / 背景 / 头像）

rmmt-api-task    → 后台匹配算分（MATCHING_ALGORITHM=v2）
mysql            → 业务数据
```

上传文件持久化在宿主机：`/var/lib/rmmt/uploads` → 容器 `/app/static/uploads`。

---

## 前置要求

- Docker Engine + Docker Compose v2
- 能访问 GitHub（SSH：`git@github.com:...`）
- 建议内存 ≥ 4GB（匹配任务容器限制约 2GB，可配合 swap）

---

## 1. 克隆（含 Submodule）

```bash
git clone --recurse-submodules -b main git@github.com:ZhiXin-College/RMMT.git
cd RMMT
```

若已克隆但 submodule 为空：

```bash
git submodule update --init --recursive
```

各 submodule 跟踪 **`React` 分支**。更新到远端最新代码：

```bash
git submodule update --remote --merge
# 或进入子目录：
#   cd RMMT-API && git checkout React && git pull
```

---

## 2. 配置环境变量

```bash
cp .env.example .env
# 编辑 .env，至少修改密码与 JWT_SECRET
```

字段说明见 [`.env.example`](./.env.example)。**不要**把填好密钥的 `.env` 提交进 Git。

常用项：

| 变量 | 含义 |
|------|------|
| `MYSQL_ROOT_PASSWORD` | MySQL root 密码（API 容器也用它连库） |
| `DB_NAME` / `DB_USER` / `DB_PASSWORD` | 业务库名与应用用户 |
| `JWT_SECRET` | 登录 Token 签名密钥 |
| `AI_API_KEY` 等 | 可选，AI 舍友搜索 |
| `PMA_ABSOLUTE_URI` | 仅在启用 phpMyAdmin 时需要 |

---

## 3. 创建上传目录

```bash
sudo mkdir -p /var/lib/rmmt/uploads/{system_style,student_avatar}
sudo chmod -R 755 /var/lib/rmmt
```

---

## 4. 启动全部服务

```bash
docker compose up -d --build
```

查看状态：

```bash
docker compose ps
docker compose logs -f
# 或 lazydocker
```

首次构建可能较久（尤其 `rmmt-api-task` 拉模型缓存）。

---

## 5. 访问入口

假设主机公网 IP 为 `YOUR_IP`，反向代理监听 **80**：

| URL | 服务 |
|-----|------|
| `http://YOUR_IP/` | 学生端 |
| `http://YOUR_IP/admin/` | 管理端 |
| `http://YOUR_IP/api/` | API |
| `http://YOUR_IP/static/...` | 静态上传资源 |

本机调试也可：`http://127.0.0.1/` 。

管理端构建时使用 `VITE_BASE=/admin/`，必须通过 `/admin/` 路径访问。

---

## 常用运维

### 只重建前端 / API

```bash
docker compose up -d --build rmmt-api rmmt-student rmmt-admin
```

### 匹配算分任务

默认已在 compose 中定义。单独重建/看日志：

```bash
docker compose up -d --build rmmt-api-task
docker compose logs -f rmmt-api-task
```

### phpMyAdmin（默认关闭，勿对公网暴露）

```bash
docker compose --profile db-ui up -d phpmyadmin
# 浏览器打开 http://127.0.0.1:8081/
# 用户 root，密码为 .env 中 MYSQL_ROOT_PASSWORD
```

### 停止

```bash
docker compose down
# 保留数据卷；若要清空数据库（危险）：
# docker compose down -v
```

---

## 管理端样式相关

登录管理端 → **系统设置 → 系统样式**：

- **主题色**：学生端按钮/链接/强调色及浅深梯度  
- **背景颜色**：学生端页面底色（与主题色独立）  
- **Logo / 登录背景**：上传后落在 `/var/lib/rmmt/uploads`，重启容器不丢失  

键值设置中的 **学生端系统名称** 会同步到登录页大标题与登录后左上角文案。

---

## 目录结构

```
RMMT/
├── README.md
├── .env.example
├── .gitignore
├── docker-compose.yml
├── deploy/
│   ├── README.md
│   └── nginx/
│       ├── nginx.conf
│       └── default.conf
├── RMMT-API/          # submodule → React
├── RMMT-Admin/        # submodule → React
└── RMMT-Student/      # submodule → React
```

---

## 故障排查

1. **`/admin/` 404**：确认 `rmmt-admin` 健康，且用的是带 `/admin/` 前缀的地址。  
2. **Logo / 头像重启后消失**：确认宿主机挂载目录存在，且 `docker compose` 中 `rmmt-api` 有  
   `/var/lib/rmmt/uploads:/app/static/uploads`。  
3. **API 起不来**：`docker compose logs rmmt-api`，检查 MySQL 是否 healthy、`.env` 密码是否一致。  
4. **Submodule 目录为空**：执行 `git submodule update --init --recursive`。  
5. **内存不足导致 task 被杀**：加大 swap，或暂时 `docker compose stop rmmt-api-task`。

---

## License

与各 submodule 仓库保持一致；使用前请确认组织内部权限与部署规范。
