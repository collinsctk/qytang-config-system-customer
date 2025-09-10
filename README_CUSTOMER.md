# 网络设备批量配置管理系统 - 客户部署说明

本文件用于客户一键部署与授权。按“客户侧步骤”操作即可完成部署，无需手工执行 SQL。

## 1. 环境要求
- Rocky Linux 9（或任意 64 位 Linux，已安装 Docker 与 docker compose）
- 对外开放 8000 端口（Web 访问）

## 2. 授权与交付
授权与运行绑定到客户机器的 Machine ID。流程如下：

### 2.1 客户侧：上报 Machine ID（一次性）
```bash
python3 -m pip install -U 'pyarmor==8.5.12'
python3 -m pyarmor.cli.hdinfo | sed -n '1p' | sed "s/.*'\(.*\)'.*/\1/" | tee /tmp/machine_id.txt
cat /tmp/machine_id.txt   # 将这一行 Machine ID 发给供应商
```

### 2.2 供应商侧：生成并回传授权文件
- 供应商会基于 Machine ID 生成 `license/pyarmor.rkey` 并返回完整的交付目录（或仅返回该文件）。
- 若仅收到 `pyarmor.rkey`，请放置到本目录 `license/pyarmor.rkey`。

目录关键结构（示例）：
```
customer_deployment/
├── docker-compose.yml
├── initdb/
│   └── 00-init.sql        # 首次启动自动执行，创建表并写入 admin/admin
├── license/
│   └── pyarmor.rkey       # 授权运行密钥（供应商生成）
└── README_CUSTOMER.md
```

## 3. 一键启动
在 `customer_deployment` 目录运行：
```bash
docker compose up -d
docker compose ps
```
说明：
- 首次启动会自动创建数据库并初始化（由 `./initdb/00-init.sql` 完成）。
- 后端镜像固定为 `collinsctk/qytang-config-system-backend:1.0.1`，其余依赖镜像均使用固定版本标签。

## 4. 访问与默认账户
- Web: `http://<服务器IP>:8000`
- 默认管理员：`admin`
- 默认密码：`admin`

首次登录后请立即在系统中修改管理员密码。

## 5. 常见问题
### 5.1 首次登录失败（admin/admin 不生效）
可能之前已创建过数据库数据卷，未触发自动初始化。可按需重置（风险：清空数据库）：
```bash
docker compose down
docker volume rm customer_deployment_postgres_data || true
docker compose up -d
```

### 5.2 授权相关
- 确认 `license/pyarmor.rkey` 存在；该文件会被自动挂载为容器内 `/app/pyarmor.rkey`。
- 更新授权后重启相关服务：
```bash
docker compose restart backend celery_worker backup_scheduler
```

### 5.3 服务状态与日志
```bash
docker compose ps
docker compose logs -f backend
```

## 6. 升级后端
供应商发布新镜像标签后（示例：1.0.2），客户可按下述步骤升级：
```bash
docker compose pull backend celery_worker backup_scheduler
docker compose up -d backend celery_worker backup_scheduler
```

## 7. 安全性说明（源码保护）
后端镜像中仅包含经 PyArmor 保护后的产物，不包含明文源码。
可选自检命令：
```bash
# 查看产物结构
docker compose exec backend sh -lc 'cd /app && ls -lah'

# 抽样查看入口与模块（应为 PyArmor 引导/不可读内容，不是业务源码）
docker compose exec backend sh -lc 'head -n 60 /app/main.py | cat'
docker compose exec backend sh -lc 'for f in $(find /app/app -maxdepth 1 -type f -name "*.py" | head -n 3); do echo "== $f =="; head -n 30 "$f" | cat; done'
```

---
如需技术支持，请将 `docker compose ps` 与相关 `docker compose logs` 片段发送给供应商。