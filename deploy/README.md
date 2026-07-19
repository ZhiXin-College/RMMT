# RMMT Docker 集群（补充说明）

主部署文档见仓库根目录 [README.md](../README.md)。

## 快速启动

```bash
sudo mkdir -p /var/lib/rmmt/uploads/{system_style,student_avatar}
sudo chmod -R 755 /var/lib/rmmt

cp .env.example .env   # 在仓库根目录
docker compose up -d --build
```

上传文件保存在宿主机 `/var/lib/rmmt/uploads`，挂载进 `rmmt-api` 的 `/app/static/uploads`，重建/重启容器不会丢失。

## 访问入口（反向代理 :80）

| 入口 | 目标容器 |
|------|----------|
| http://\<公网IP\>/ | rmmt-student |
| http://\<公网IP\>/admin/ | rmmt-admin |
| http://\<公网IP\>/api/ | rmmt-api |
| http://\<公网IP\>/static/ | rmmt-api 静态资源 |

管理端以路径前缀 `/admin/` 挂载，构建时使用 `VITE_BASE=/admin/`。

phpMyAdmin 默认不对公网暴露；需要时：

```bash
docker compose --profile db-ui up -d phpmyadmin
```

本机访问 `http://127.0.0.1:8081/`，用户 `root`，密码见 `.env` 中 `MYSQL_ROOT_PASSWORD`。

## 监测

```bash
lazydocker
# 或
docker compose ps
docker compose logs -f
```

## 匹配算分任务

```bash
docker compose up -d --build rmmt-api-task
docker compose logs -f rmmt-api-task
```

- 算法：`MATCHING_ALGORITHM=v2`
- 内存限制：2GB（另可用 host swap）
- 日志卷：`rmmt_task_log`
