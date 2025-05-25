## 模块七：应用部署与运维 (火山引擎)

到目前为止，我们已经在本地开发环境中构建并测试了 FastAPI 应用。现在，是时候将它部署到火山引擎的云服务器上，使其能够为你的移动应用提供稳定可靠的后端服务了。本模块将指导你完成应用的生产环境部署，并介绍基本的运维知识。

### 1. 生产环境部署概述

与开发环境不同，生产环境对应用的稳定性、安全性、性能和可扩展性有更高的要求。关键组件包括：

*   **代码准备**：确保你的代码在版本控制系统 (如 Git) 中，并且有一个稳定的待部署版本。
*   **ASGI 服务器**：虽然 Uvicorn 可以单独运行 FastAPI 应用，但在生产环境中，通常会将其与 Gunicorn (或类似的进程管理器) 结合使用，以获得更好的并发处理能力和稳定性。
*   **反向代理服务器 (Nginx)**：处理来自客户端的 HTTP/HTTPS 请求，将其转发给 ASGI 服务器。Nginx 还可以负责 SSL/TLS 终止 (实现 HTTPS)、静态文件服务、负载均衡、请求限流等。
*   **进程管理器 (Systemd/Supervisor)**：确保你的应用在服务器重启或意外崩溃后能够自动重启，并管理应用的运行状态。
*   **数据库**：生产环境数据库需要考虑数据备份、安全、性能优化。
*   **HTTPS**：必须为你的 API启用 HTTPS 来加密客户端和服务器之间的通信。
*   **日志管理**：收集、存储和分析应用及服务器日志，用于监控和故障排查。
*   **监控与告警**：实时监控服务器资源使用情况 (CPU, 内存, 网络, 磁盘) 和应用性能，并在出现问题时发送告警。

### 2. 准备你的火山引擎 ECS

*   **服务器配置**：确保你的 ECS 实例配置 (CPU, 内存, 磁盘) 能够满足应用初期的负载需求。参考模块二的选择。
*   **操作系统**：Ubuntu Server LTS 仍然是推荐的选择。
*   **安全组**：
    *   开放 Nginx 将要监听的端口：HTTP (80) 和 HTTPS (443)。
    *   SSH (22) 端口应严格限制访问 IP。
    *   如果 Uvicorn/Gunicorn 在本地端口 (如 8000) 运行，该端口不应直接对公网开放，由 Nginx 进行代理。
    *   PostgreSQL (5432) 端口，如果数据库与应用在同一台服务器，且仅由应用本地访问，则不应向公网开放。
*   **域名解析**：如果你有域名，请将其解析到你的 ECS 实例的公网 IP 地址。

### 3. 部署 FastAPI 应用

假设你的 FastAPI 应用代码已经通过 Git 克隆到服务器的某个目录，例如 `/srv/my_fastapi_app`。

#### 创建和激活虚拟环境

在 `/srv/my_fastapi_app` 目录下为你的应用创建并激活虚拟环境：

```bash
cd /srv/my_fastapi_app
python3 -m venv env
source env/bin/activate
```

#### 安装依赖

在激活的虚拟环境中安装项目所需的依赖包。通常你会有一个 `requirements.txt` 文件。

```bash
pip install -r requirements.txt
# 确保 requirements.txt 包含：
# fastapi
# uvicorn[standard]
# gunicorn
# sqlalchemy[asyncio]
# asyncpg
# pydantic
# pydantic-settings
# python-jose[cryptography]
# passlib[bcrypt]
# python-multipart
# alembic
# psycopg2-binary # (Alembic可能需要)
```

#### 运行数据库迁移 (Alembic)

在部署新版本代码或首次部署时，确保数据库模式是最新的：

```bash
# 确保 alembic.ini 中的 sqlalchemy.url 指向正确的生产数据库
# 确保 alembic/env.py 中的 target_metadata 指向正确的 Base.metadata
alembic upgrade head
```

#### 配置 Gunicorn 作为 ASGI 进程管理器

Gunicorn 是一个成熟的 Python WSGI HTTP 服务器，但它也可以通过 Uvicorn worker 来运行 ASGI 应用 (如 FastAPI)。

1.  **测试 Gunicorn 运行**:
    在你的项目目录 (`/srv/my_fastapi_app`) 中，尝试用 Gunicorn 启动应用：
    ```bash
    # 假设你的 FastAPI app 实例在 my_fastapi_app/main.py 中的 app 对象
    # 例如，你的项目结构可能是：
    # /srv/my_fastapi_app/
    #   env/
    #   my_fastapi_app/  (Python 包)
    #     __init__.py
    #     main.py        (包含 app = FastAPI())
    #     models.py
    #     schemas.py
    #     crud.py
    #     auth.py
    #     database.py
    #     config.py
    #     routers/
    #   alembic/
    #   alembic.ini
    #   requirements.txt

    # 确保你在 /srv/my_fastapi_app 目录下，并且虚拟环境已激活
    gunicorn -w 4 -k uvicorn.workers.UvicornWorker my_fastapi_app.main:app -b 127.0.0.1:8000
    ```
    *   `-w 4`: 指定 worker 进程的数量。一个常见的建议是 `(2 * CPU核心数) + 1`。你需要根据你的服务器配置和应用负载进行调整。
    *   `-k uvicorn.workers.UvicornWorker`: 指定使用 Uvicorn worker 来处理 ASGI 应用。
    *   `my_fastapi_app.main:app`: 指向你的 FastAPI 应用实例。`my_fastapi_app.main` 是 Python 模块路径，`app` 是在该模块中定义的 FastAPI 实例。
    *   `-b 127.0.0.1:8000`: Gunicorn 绑定的地址和端口。我们将其绑定到本地回环地址，因为 Nginx 将作为外部访问的入口。

    按 `Ctrl+C` 停止 Gunicorn。

#### 配置 Nginx 作为反向代理

我们在模块三中已经安装并基本配置了 Nginx。现在，我们需要为生产环境的 FastAPI 应用更新 Nginx 配置。

1.  **创建/编辑 Nginx 配置文件**:
    ```bash
    sudo vim /etc/nginx/sites-available/my_fastapi_app_prod
    ```
    填入以下内容 (替换 `your_domain.com` 和其他占位符)：
    ```nginx
    server {
        listen 80;
        server_name your_domain.com www.your_domain.com your_server_public_ip; # 替换为你的域名或IP

        # 用于 Let's Encrypt 验证 (如果使用 Certbot)
        location ~ /.well-known/acme-challenge {
            allow all;
            root /var/www/html; # 或者 Certbot 指定的其他路径
        }

        location / {
            proxy_pass http://127.0.0.1:8000; # Gunicorn 监听的地址和端口
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade; # 用于 WebSocket (如果你的应用使用)
            proxy_set_header Connection "upgrade";  # 用于 WebSocket
            proxy_http_version 1.1;
            proxy_buffering off; # 对于流式响应或SSE有好处
            proxy_read_timeout 300s; # 增加超时时间，以防长轮询或大文件上传
            proxy_send_timeout 300s;
        }

        # 可选：静态文件服务 (如果你的 FastAPI 应用有 /static 目录)
        # location /static {
        #     alias /srv/my_fastapi_app/static; # 确保路径正确
        #     expires 7d;
        # }

        # 错误日志和访问日志 (可以自定义路径)
        access_log /var/log/nginx/my_fastapi_app.access.log;
        error_log /var/log/nginx/my_fastapi_app.error.log;
    }
    ```

2.  **启用站点并测试配置**:
    ```bash
    sudo ln -s /etc/nginx/sites-available/my_fastapi_app_prod /etc/nginx/sites-enabled/
    # 如果存在默认配置且冲突，可以移除： sudo rm /etc/nginx/sites-enabled/default
    sudo nginx -t
    ```
    如果测试成功，则重启 Nginx:
    ```bash
    sudo systemctl restart nginx
    ```

#### 配置 HTTPS (使用 Let's Encrypt 和 Certbot)

为你的 API 启用 HTTPS 至关重要。

1.  **安装 Certbot**:
    Certbot 是 Let's Encrypt 官方推荐的客户端，用于自动获取和续订 SSL/TLS 证书。
    ```bash
    sudo apt update
    sudo apt install -y certbot python3-certbot-nginx
    ```

2.  **获取 SSL 证书**:
    确保你的 Nginx 配置中 `server_name` 已正确设置，并且域名已解析到服务器 IP。
    ```bash
    sudo certbot --nginx -d your_domain.com -d www.your_domain.com # 根据你的域名修改
    ```
    Certbot 会引导你完成证书获取过程，并自动修改你的 Nginx 配置文件以启用 HTTPS (通常会创建一个新的 `server` 块监听 443 端口，并处理 SSL 相关配置，同时将 HTTP 请求重定向到 HTTPS)。

3.  **验证自动续订**:
    Let's Encrypt 证书有效期为 90 天。Certbot 通常会自动设置一个 cron job 或 systemd timer 来定期续订证书。
    ```bash
    sudo certbot renew --dry-run
    ```
    如果测试成功，说明自动续订已正确配置。

#### 使用 Systemd 管理 Gunicorn 进程

Systemd 是 Linux 系统中常用的服务管理器，可以确保你的 Gunicorn 进程在服务器启动时自动运行，并在意外退出时自动重启。

1.  **创建 Systemd 服务文件**:
    ```bash
    sudo vim /etc/systemd/system/my_fastapi_app.service
    ```
    填入以下内容 (根据你的实际路径和用户名修改)：
    ```ini
    [Unit]
    Description=My FastAPI Application managed by Gunicorn
    After=network.target # 确保网络可用后再启动
    # 如果依赖数据库服务，可以添加： Wants=postgresql.service After=postgresql.service

    [Service]
    User=ubuntu # 运行 Gunicorn 的用户 (例如 'ubuntu' 或你创建的专用用户)
    Group=www-data # 可选，运行 Gunicorn 的用户组 (例如 'www-data' 或用户同名组)
    WorkingDirectory=/srv/my_fastapi_app # 你的项目根目录
    Environment="PATH=/srv/my_fastapi_app/env/bin" # 指定虚拟环境的路径
    # Environment="PYTHONUNBUFFERED=1" # 确保Python输出不被缓冲，日志直接可见
    # Environment="ENV_VAR_NAME=value" # 如果需要设置其他环境变量

    ExecStart=/srv/my_fastapi_app/env/bin/gunicorn \
        --workers 4 \
        --worker-class uvicorn.workers.UvicornWorker \
        --bind 127.0.0.1:8000 \
        my_fastapi_app.main:app
    
    Restart=always # 进程退出时总是重启
    RestartSec=5s    # 重启前等待5秒
    StandardOutput=append:/srv/my_fastapi_app/logs/gunicorn_access.log # Gunicorn 标准输出日志
    StandardError=append:/srv/my_fastapi_app/logs/gunicorn_error.log  # Gunicorn 错误输出日志
    SyslogIdentifier=my_fastapi_app

    [Install]
    WantedBy=multi-user.target
    ```
    *   确保 `/srv/my_fastapi_app/logs/` 目录存在并且运行 Gunicorn 的用户有写入权限：
        ```bash
        sudo mkdir -p /srv/my_fastapi_app/logs
        sudo chown ubuntu:www-data /srv/my_fastapi_app/logs # 替换用户和组
        ```
    *   `User` 和 `Group`: 推荐创建一个非 root 的专用用户来运行你的应用。
    *   `WorkingDirectory`: Gunicorn 的工作目录。
    *   `Environment="PATH=..."`: 非常重要，确保 Gunicorn 使用虚拟环境中的 Python 和依赖。
    *   `ExecStart`: 启动 Gunicorn 的命令，与之前测试的一致。确保路径正确。
    *   `Restart=always`: 保证服务的高可用性。

2.  **重新加载 Systemd 并启动/启用服务**:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl start my_fastapi_app
    sudo systemctl enable my_fastapi_app # 设置开机自启
    ```

3.  **检查服务状态和日志**:
    ```bash
    sudo systemctl status my_fastapi_app
    journalctl -u my_fastapi_app -f # 查看实时日志
    cat /srv/my_fastapi_app/logs/gunicorn_error.log # 查看 Gunicorn 错误日志
    ```

### 4. 应用日志管理

FastAPI 应用本身的日志记录非常重要。

*   **FastAPI 日志配置**：你可以使用 Python 内置的 `logging` 模块来配置 FastAPI 的日志记录。FastAPI (实际上是 Uvicorn) 会使用这个配置。
    在 `main.py` 或一个单独的日志配置文件中进行配置：
    ```python
    # main.py (或 log_config.py)
    import logging
    from logging.config import dictConfig

    # 简单的日志配置示例
    LOGGING_CONFIG = {
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": {
            "default": {
                "()": "uvicorn.logging.DefaultFormatter",
                "fmt": "%(levelprefix)s %(asctime)s [%(name)s] %(message)s",
                "datefmt": "%Y-%m-%d %H:%M:%S",
            },
            "access": {
                "()": "uvicorn.logging.AccessFormatter",
                "fmt": '%(levelprefix)s %(asctime)s [%(name)s] %(client_addr)s - "%(request_line)s" %(status_code)s',
                "datefmt": "%Y-%m-%d %H:%M:%S",
            }
        },
        "handlers": {
            "default": {
                "formatter": "default",
                "class": "logging.StreamHandler",
                "stream": "ext://sys.stdout", # 或者配置为写入文件
            },
            "access": {
                "formatter": "access",
                "class": "logging.StreamHandler",
                "stream": "ext://sys.stdout",
            },
            # "file_handler": { # 示例：写入到文件
            #     "formatter": "default",
            #     "class": "logging.handlers.RotatingFileHandler",
            #     "filename": "/srv/my_fastapi_app/logs/app.log",
            #     "maxBytes": 1024 * 1024 * 100,  # 100 MB
            #     "backupCount": 5,
            #     "encoding": "utf8",
            # },
        },
        "loggers": {
            "uvicorn": {"handlers": ["default"], "level": "INFO", "propagate": False},
            "uvicorn.error": {"level": "INFO"},
            "uvicorn.access": {"handlers": ["access"], "level": "INFO", "propagate": False},
            # "my_app_logger": { # 你自定义的 logger
            #     "handlers": ["default", "file_handler"],
            #     "level": "DEBUG", 
            #     "propagate": False,
            # },
        },
        # "root": { # 根logger，捕获所有未明确配置的loggers
        #     "handlers": ["default", "file_handler"],
        #     "level": "INFO",
        # },
    }

    # dictConfig(LOGGING_CONFIG) # 在 FastAPI app 创建之前应用

    # app = FastAPI(log_config=LOGGING_CONFIG) # 或者直接在FastAPI实例化时传入
    ```
    如果配置了日志输出到文件，确保 Gunicorn 的 Systemd 服务配置中 `StandardOutput` 和 `StandardError` 指向其他文件或 `/dev/null`，以避免日志重复或冲突。
    **注意**：当 Gunicorn 与 Uvicorn worker 一起使用时，Uvicorn 的日志配置通常会生效。你可以通过 Uvicorn 的命令行参数 (在 Gunicorn `ExecStart` 中) 或通过 FastAPI 的 `log_config` 参数来影响日志。

*   **Nginx 日志**：已在 Nginx 配置文件中指定了 `access_log` 和 `error_log` 的路径。
*   **Systemd 日志**：`journalctl -u my_fastapi_app` 可以查看由 Systemd 捕获的 Gunicorn 进程的 stdout/stderr。
*   **火山引擎日志服务**：对于更集中的日志管理和分析，可以考虑使用火山引擎提供的日志服务 (如果可用)，将应用日志、Nginx 日志等推送到该服务。

### 5. 数据库运维

*   **备份**：定期备份你的 PostgreSQL 数据库至关重要。
    *   **`pg_dump`**：用于创建数据库的逻辑备份。
        ```bash
        sudo -u postgres pg_dump mydb > mydb_backup_$(date +%Y%m%d_%H%M%S).sql
        ```
    *   可以设置一个 cron job 来自动执行备份，并将备份文件存储到安全的位置 (例如火山引擎对象存储 TOS)。
    *   考虑备份策略 (完整备份、增量备份、备份频率、保留周期)。
*   **安全**：
    *   确保 PostgreSQL 用户密码强壮。
    *   限制网络访问 (通过 `pg_hba.conf` 和 ECS 安全组)。
    *   定期更新 PostgreSQL。
*   **监控**：监控数据库的性能指标 (连接数、查询延迟、磁盘空间等)。火山引擎的云监控服务可能支持对自建数据库进行部分监控，或者你可以使用 `pgAdmin` 等工具。
*   **火山引擎 RDS for PostgreSQL**：如果运维自建数据库的负担过重，或者对高可用性、自动备份、轻松扩展有更高要求，可以考虑迁移到火山引擎的 RDS for PostgreSQL 服务。

### 6. 监控与告警 (火山引擎)

火山引擎提供云监控服务 (CloudMonitor)，你可以用它来：

*   **监控 ECS 实例**：CPU 使用率、内存使用率、磁盘 I/O、网络流量等。
*   **设置告警规则**：当某个指标超过阈值时 (例如 CPU 使用率持续高于 80%)，自动发送告警通知 (短信、邮件等)。
*   **自定义监控**：如果需要监控应用特定的指标 (例如 API 错误率、平均响应时间)，你可能需要将这些指标上报给云监控服务或使用第三方监控工具 (如 Prometheus + Grafana)。

### 7. 持续集成/持续部署 (CI/CD) - 简介

对于更自动化的部署流程，可以引入 CI/CD。

*   **持续集成 (CI)**：代码变更后自动运行测试、构建应用。
*   **持续部署 (CD)**：构建成功后自动将应用部署到服务器。
*   **工具**：火山引擎可能提供相关的 DevOps 工具。通用的工具有 Jenkins, GitLab CI/CD, GitHub Actions 等。
*   **简单脚本**：对于个人项目，一个简单的部署脚本 (例如，`git pull` -> 激活虚拟环境 -> 安装依赖 -> 数据库迁移 -> 重启 Gunicorn 服务) 可能就足够了。

### 8. 成本优化与弹性

*   **选择合适的 ECS 规格**：根据实际负载选择，避免资源浪费。火山引擎支持实例规格的升降级。
*   **计费模式**：对于稳定负载，包年包月可能更经济；对于波动负载，按量付费结合弹性伸缩更优。
*   **弹性伸缩 (Auto Scaling)**：如果你的应用负载有明显峰谷，可以配置弹性伸缩服务，根据 CPU 使用率等指标自动增减 ECS 实例数量。这需要你的应用是无状态的，或者状态能够被共享 (例如通过 Redis, 数据库)。
*   **负载均衡 (CLB)**：当使用多个 ECS 实例时，需要使用负载均衡器将流量分发到各个实例。
*   **对象存储 (TOS)**：用于存储静态文件 (如用户上传的图片、视频、备份文件)，成本通常低于 ECS 云硬盘，并且可以结合 CDN 使用。

### 小结

将 FastAPI 应用部署到生产环境涉及多个步骤，从准备服务器、配置 ASGI 服务器和反向代理，到设置 HTTPS、管理进程和处理日志。火山引擎提供了必要的基础设施和服务来支持这些任务。

记住，部署不是一次性的事情，持续的监控、维护和优化是确保应用稳定运行的关键。随着应用的发展，你可能还需要引入更高级的部署和运维技术。

--- 