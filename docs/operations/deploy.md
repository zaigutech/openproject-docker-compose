# fork 的部署流程

OpenProject 接入公用基础设施：数据库用 [middleware](../../../middleware/README.md) 的 PostgreSQL，生产环境的
HTTPS 和域名由 [caddy](../../../caddy/Caddyfile) 处理（`op.zaigu.com`）。

| 文件 | 作用 |
|---|---|
| `docker-compose.yml` | 基于上游修改，只放开发和生产共用的配置 |
| `docker-compose.override.yml` | 开发环境，`docker compose` 自动合并。上游的 `.gitignore` 忽略了它，首次提交要用 `git add -f` |
| `docker-compose.prod.yml` | 生产环境，用 `-f` 显式指定 |
| `.env` | 本机配置和密钥，不提交，开发和生产都需要 |

`.env.example`、上游 README 的启动步骤和 `docker-compose.control.yml` 都是原样保留的，部署按这份文档操作。
`.env.example` 里的 `DATABASE_URL`、`PGDATA`、`POSTGRES_VERSION` 已不起作用。

## 对上游 docker-compose.yml 的改动

- 删除自带的 `db` 和 `proxy`，所有服务接入 `zaigu_network`。生产环境由公用 Caddy 分别转发主应用和 `/hocuspocus`。
- 数据库密码读取 `middleware/secrets/openproject_password`，由 entrypoint 拼成 `DATABASE_URL`，不用写进 `.env`。
- 服务名加 `openproject-` 前缀，`web` 这样的名字会和其他项目在共享网络上冲突。`hocuspocus` 和 `autoheal` 保持原名：上游经常
  升级紧挨着 `hocuspocus` 服务名的 `image` 行，改名每次合并都会冲突。
- `web`、`worker`、`cron` 等 `seeder` 成功退出后才启动，否则它们会和 seeder 抢着建表，导致首次初始化失败。

合并上游后，留意上游是否新增了服务或引用了旧服务名。

## 本地开发

开发环境的配置都在 `docker-compose.override.yml`：HTTP，主应用绑定 `127.0.0.1:8090`（EOS 占用了 8080），协作服务绑定 `127.0.0.1:1234`。

先启动 middleware，它会创建 `zaigu_network` 和数据库。按 middleware 的 README 生成 `secrets/` 时已包含
`openproject_password`，`scripts/init-openproject.sh` 会在 `/mnt/db` 为空时创建用户 `openproject` 和数据库 `openproject_db`。

然后准备 `.env`。OpenProject 会拒绝默认的 `SECRET_KEY_BASE`，没有配置就无法启动：

```bash
cp .env.example .env
```

```text
SECRET_KEY_BASE=<openssl rand -hex 32>
COLLABORATIVE_SERVER_SECRET=<openssl rand -hex 32>
OPDATA="/mnt/data/openproject/assets"
```

`.env.example` 里的 `OPENPROJECT_HTTPS`、`OPENPROJECT_HOST__NAME`、`PORT` 和 `COLLABORATIVE_SERVER_URL` 在开发环境由 override 覆盖或不再使用。
`OPDATA` 目录的属主要是 UID 1000，见下面生产环境第 3 步。

```bash
docker compose up -d
```

等 `openproject-seeder` 完成数据库迁移后，访问 http://localhost:8090 ，用 `admin` / `admin` 登录。

如果 `/mnt/db` 已经初始化过，脚本不会再执行。这时在 middleware 目录下手动建库（`:'pw'` 由 psql 负责转义）：

```bash
docker compose exec -T db psql -U postgres -v pw="$(cat secrets/openproject_password)" <<'SQL'
CREATE USER openproject WITH PASSWORD :'pw';
CREATE DATABASE openproject_db OWNER openproject;
SQL
```

## 生产环境

1. 启动 middleware 和 caddy，给 `op.zaigu.com` 加 DNS 解析。
2. 在服务器上准备 `.env`：

   ```text
   SECRET_KEY_BASE=<openssl rand -hex 32>
   COLLABORATIVE_SERVER_SECRET=<openssl rand -hex 32>
   OPDATA="/mnt/data/openproject/assets"
   ```

   `SECRET_KEY_BASE` 改了以后所有会话和 2FA 记住状态都会失效，不要随意更换。

3. 准备附件目录，属主要是 UID 1000：

   ```bash
   sudo mkdir -p /mnt/data/openproject/assets
   sudo chown -R 1000:1000 /mnt/data/openproject/assets
   ```

4. 启动。所有 `docker compose` 命令都要加这两个 `-f`：

   ```bash
   docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --pull always
   ```

5. 用 `admin` / `admin` 登录，并立即修改密码。

## 备份

上游的 `docker-compose.control.yml` 依赖已删除的 `db`，不能再用。数据库在 middleware 目录下导出，附件单独打包：

```bash
docker compose exec -T db pg_dump -U postgres openproject_db > openproject_db.sql
```

```bash
tar czf openproject-assets.tar.gz -C /mnt/data/openproject/assets .
```

## 升级

```bash
git fetch upstream && git merge upstream/stable/17
```

备份后，重新执行启动命令。
