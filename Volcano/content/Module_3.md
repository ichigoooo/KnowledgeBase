## 模块三：后端开发环境搭建 (以 Python FastAPI 和 PostgreSQL 为例)

在拥有了一台火山引擎云服务器之后，下一步就是在服务器上搭建我们的后端开发和运行环境。本模块将以 Python FastAPI 框架和 PostgreSQL 数据库为例，详细指导你完成环境配置。

**前提条件：**

*   你已经购买并成功启动了一台火山引擎云服务器实例 (推荐选择 Ubuntu Server 最新 LTS 版本)。
*   你已经可以通过 SSH 成功连接到你的云服务器。
*   你对 Linux 基本命令行操作有一定了解。

### 1. 服务器初始化与环境准备

连接到你的云服务器后，首先进行一些基础的初始化和准备工作。

#### 更新系统软件包

保持系统软件包最新是一个好习惯，有助于安全和获取最新特性。

```bash
# 更新软件包列表
sudo apt update

# 升级已安装的软件包
sudo apt upgrade -y
```
*   `sudo apt update`：从配置的软件源同步最新的软件包列表。
*   `sudo apt upgrade -y`：将所有已安装的软件包升级到最新版本。`-y` 参数会自动确认所有提示。

#### 安装常用工具

一些常用的命令行工具能让后续工作更便捷：

```bash
sudo apt install -y git vim curl wget unzip
```
*   `git`: 版本控制系统，用于管理你的代码。
*   `vim`: 强大的文本编辑器 (你也可以选择 `nano` 等其他编辑器)。
*   `curl` / `wget`: 用于从网络下载文件或测试网络连接。
*   `unzip`: 用于解压 `.zip` 文件。

### 2. Python 环境配置

FastAPI 是一个 Python 框架，所以我们需要在服务器上配置 Python 环境。

#### 安装 Python

大多数 Linux 发行版会预装 Python，但版本可能不是最新的。推荐使用 `pyenv` 来管理多个 Python 版本，这样更灵活。如果图简单，也可以直接使用系统自带的 Python3 (如果版本合适，例如 Python 3.7+)。

**方法一：使用系统自带 Python3 (如果版本 >= 3.7)**

检查当前 Python3 版本：
```bash
python3 --version
```
如果版本低于 3.7，或者你想使用特定版本的 Python，推荐使用 `pyenv`。

如果系统 Python3 版本满足要求 (FastAPI 推荐 Python 3.7+)，确保 `pip` (Python 包安装器) 也已安装：
```bash
sudo apt install -y python3-pip python3-venv
```

**方法二：使用 `pyenv` 安装和管理 Python 版本 (推荐)**

`pyenv` 可以让你轻松地在多个 Python 版本之间切换。

1.  **安装 `pyenv` 依赖包：**
    ```bash
    sudo apt update
    sudo apt install -y make build-essential libssl-dev zlib1g-dev \
    libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm \
    libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev
    ```

2.  **安装 `pyenv`：**
    推荐使用 `pyenv-installer`：
    ```bash
    curl https://pyenv.run | bash
    ```
    安装完成后，根据提示将 `pyenv` 初始化脚本添加到你的 shell 配置文件中 (通常是 `~/.bashrc` 或 `~/.zshrc`，如果你使用的是 zsh)。例如，对于 bash：
    ```bash
    echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
    echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
    echo 'eval "$(pyenv init -)"' >> ~/.bashrc
    ```
    然后使配置生效：
    ```bash
    source ~/.bashrc
    # 或者注销后重新登录
    ```

3.  **安装指定版本的 Python：**
    查看可安装的 Python 版本：
    ```bash
    pyenv install --list
    ```
    选择一个版本进行安装，例如 Python 3.10.12：
    ```bash
    pyenv install 3.10.12
    ```
    安装过程可能需要一些时间。

4.  **设置全局或局部 Python 版本：**
    *   设置全局默认 Python 版本：
        ```bash
        pyenv global 3.10.12
        ```
    *   在特定项目目录设置局部 Python 版本 (更推荐)：
        ```bash
        cd /path/to/your/project
        pyenv local 3.10.12
        ```
    验证 Python 版本：
    ```bash
    python --version
    pip --version
    ```

#### 虚拟环境的创建与使用 (`venv`)

为每个 Python 项目使用独立的虚拟环境是一个非常好的实践，它可以隔离项目依赖，避免不同项目之间的包版本冲突。

1.  **创建虚拟环境：**
    进入你的项目目录 (如果还没有，可以创建一个)：
    ```bash
    mkdir ~/my_fastapi_app
    cd ~/my_fastapi_app
    ```
    使用 `venv` 模块创建名为 `env` (或其他你喜欢的名字) 的虚拟环境：
    ```bash
    python3 -m venv env
    # 或者，如果你使用 pyenv 管理的 python
    # python -m venv env
    ```

2.  **激活虚拟环境：**
    ```bash
    source env/bin/activate
    ```
    激活后，你的命令行提示符通常会显示虚拟环境的名称，例如 `(env) user@hostname:~$`。此时，通过 `pip` 安装的包都将安装在这个虚拟环境中。

3.  **退出虚拟环境：**
    ```bash
    deactivate
    ```

### 3. FastAPI 框架安装与项目创建

确保你已经激活了项目的虚拟环境。

#### 通过 `pip` 安装 FastAPI 和 Uvicorn

*   **FastAPI**: 核心框架。
*   **Uvicorn**: 一个 ASGI (Asynchronous Server Gateway Interface) 服务器，用于运行 FastAPI 应用。

```bash
pip install fastapi uvicorn[standard]
```
`uvicorn[standard]` 会安装 Uvicorn 并包含一些推荐的依赖，如 `websockets` 和 `httptools`，以获得更好的性能。

#### 创建 FastAPI 项目基本结构

在你的项目目录 (`~/my_fastapi_app`) 下，创建一个主应用文件，例如 `main.py`：

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def read_root():
    return {"message": "你好，世界！来自 FastAPI 和火山引擎！"}

@app.get("/items/{item_id}")
async def read_item(item_id: int, q: str | None = None):
    return {"item_id": item_id, "q": q}
```

这个简单的 FastAPI 应用定义了两个路由：
*   `/`: 返回一个 JSON 问候消息。
*   `/items/{item_id}`: 接受一个路径参数 `item_id` 和一个可选的查询参数 `q`。

#### 运行开发服务器 (Uvicorn)

在 `main.py` 所在的目录，运行 Uvicorn：

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```
*   `main:app`: `main` 指的是 `main.py` 文件，`app` 指的是在 `main.py` 中创建的 FastAPI 实例 `app = FastAPI()`。
*   `--reload`: 开发模式下很有用，当代码发生改变时，服务器会自动重启。生产环境不要用。
*   `--host 0.0.0.0`: 使服务器监听所有可用的 IP 地址，这样你才能通过服务器的公网 IP 访问它。
*   `--port 8000`: 指定监听的端口号。

现在，如果你的火山引擎服务器安全组已经为端口 8000 配置了允许入站的规则 (源 IP 设置为 `0.0.0.0/0` 或你的 IP)，你应该可以通过浏览器访问：
`http://<你的服务器公网IP>:8000`  -> 你会看到 `{"message":"你好，世界！来自 FastAPI 和火山引擎！"}`
`http://<你的服务器公网IP>:8000/items/5?q=somequery` -> 你会看到 `{"item_id":5,"q":"somequery"}`
`http://<你的服务器公网IP>:8000/docs` -> 你会看到自动生成的 Swagger UI API 文档。
`http://<你的服务器公网IP>:8000/redoc` -> 你会看到自动生成的 ReDoc API 文档。

按 `Ctrl+C` 可以停止 Uvicorn 开发服务器。

### 4. Web 服务器配置 (Nginx) - 用于生产环境

Uvicorn 自带的开发服务器不适合直接用于生产环境。在生产环境中，我们通常会使用更强大的 Web 服务器，如 Nginx 或 Apache，作为反向代理。Nginx 非常流行且性能优越。

Nginx 会处理传入的 HTTP 请求，并将它们转发给在本地运行的 Uvicorn (或其他 ASGI/WSGI 服务器如 Gunicorn)。Nginx 还可以负责处理 HTTPS (SSL/TLS 加密)、静态文件服务、请求限流、负载均衡等。

#### 安装 Nginx

```bash
sudo apt update
sudo apt install -y nginx
```

#### 启动并检查 Nginx 状态

```bash
sudo systemctl start nginx
sudo systemctl enable nginx # 设置开机自启
sudo systemctl status nginx
```
如果 Nginx 成功启动，你应该能看到 "active (running)" 的状态。
此时，如果你的安全组开放了 80 端口，访问 `http://<你的服务器公网IP>` 应该能看到 Nginx 的欢迎页面。

#### 配置 Nginx 作为 FastAPI 的反向代理

1.  **为你的应用创建一个 Nginx 配置文件：**
    我们将在 `/etc/nginx/sites-available/` 目录下创建一个新的配置文件。假设你的域名是 `your_domain.com` (如果没有域名，你可以暂时使用服务器的公网 IP，但为了后续 HTTPS 配置，建议使用域名)。

    ```bash
    sudo vim /etc/nginx/sites-available/my_fastapi_app
    ```
    或者使用 `nano`：
    ```bash
    sudo nano /etc/nginx/sites-available/my_fastapi_app
    ```

2.  **编辑配置文件：**
    将以下内容粘贴到文件中，并根据你的情况修改：

    ```nginx
    server {
        listen 80;
        server_name your_domain.com www.your_domain.com your_server_public_ip; # 替换为你的域名或IP

        location / {
            proxy_pass http://127.0.0.1:8000; # 假设 Uvicorn 运行在本地的 8000 端口
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # 可选：配置静态文件服务 (如果你的 FastAPI 应用有静态文件)
        # location /static {
        #     alias /path/to/your/project/static;
        # }
    }
    ```
    *   `listen 80;`: 监听 HTTP 80 端口。
    *   `server_name`: 你的域名或服务器 IP。如果你有多个域名指向这个应用，可以都列出来。
    *   `location / { ... }`: 定义了如何处理根路径 (`/`) 的请求。
    *   `proxy_pass http://127.0.0.1:8000;`: 将请求转发给运行在本地 `127.0.0.1` (localhost) 的 `8000` 端口的 Uvicorn 服务。
    *   `proxy_set_header ...`: 这些头部信息有助于后端应用获取真实的客户端信息。

3.  **创建符号链接到 `sites-enabled` 目录：**
    Nginx 会加载 `sites-enabled` 目录下的配置文件。

    ```bash
    sudo ln -s /etc/nginx/sites-available/my_fastapi_app /etc/nginx/sites-enabled/
    ```
    为了避免与默认配置冲突，可以考虑移除默认的配置链接 (如果存在且你不需要它)：
    ```bash
    # 检查 sites-enabled 目录
    ls /etc/nginx/sites-enabled/
    # 如果有 default，并且你不需要它，可以移除
    # sudo rm /etc/nginx/sites-enabled/default
    ```

4.  **测试 Nginx 配置并重启：**
    ```bash
    sudo nginx -t
    ```
    如果显示 `syntax is ok` 和 `test is successful`，则配置无误。
    然后重启 Nginx 使配置生效：
    ```bash
    sudo systemctl restart nginx
    ```

现在，当你的 Uvicorn 服务在 `127.0.0.1:8000` 运行时，访问 `http://your_domain.com` (或你的服务器公网 IP) 应该会通过 Nginx 代理到你的 FastAPI 应用。注意，安全组需要开放 80 端口给 Nginx，而 8000 端口可以只对服务器本地 (127.0.0.1) 开放，或者从安全组中移除对公网的直接访问规则 (如果之前为 Uvicorn 开发服务器设置过)。

### 5. 数据库选择与安装 (聚焦 PostgreSQL)

你已经确定使用 PostgreSQL 作为数据库。你可以选择在云服务器上自行安装和配置 PostgreSQL，也可以考虑使用火山引擎提供的 RDS for PostgreSQL（关系型数据库服务）。

#### PostgreSQL 简介与特性

*   **强大的开源对象关系型数据库系统**：以其可靠性、功能健壮性和性能而闻名。
*   **ACID 兼容**：保证事务的原子性、一致性、隔离性和持久性。
*   **高度可扩展**：支持多种索引类型、全文搜索、JSONB 数据类型（非常适合存储半结构化数据）、地理空间数据 (通过 PostGIS 扩展) 等。
*   **丰富的生态系统**：拥有大量的客户端库、工具和社区支持。
*   **并发控制**：采用多版本并发控制 (MVCC)，读操作不阻塞写操作，写操作不阻塞读操作，提高了并发性能。

#### 方案一：在火山引擎服务器上自行安装和配置 PostgreSQL

对于个人开发者初期，或希望完全掌控数据库环境并降低直接成本，自行安装是一个不错的选择。

1.  **安装 PostgreSQL：**
    Ubuntu 的官方仓库通常包含 PostgreSQL。
    ```bash
    sudo apt update
    sudo apt install -y postgresql postgresql-contrib
    ```
    `postgresql-contrib` 包含一些额外的工具和功能。

2.  **检查安装和状态：**
    安装完成后，PostgreSQL 服务通常会自动启动。
    ```bash
    sudo systemctl status postgresql
    ```
    你应该能看到 "active (running)"。

3.  **PostgreSQL 用户和数据库基础：**
    *   **`postgres` 操作系统用户**：安装 PostgreSQL 时，会自动创建一个名为 `postgres` 的 Linux 系统用户。很多 PostgreSQL 的管理操作需要切换到这个用户。
    *   **`postgres` 数据库用户 (超级用户)**：PostgreSQL 内部也有一个默认的数据库超级用户，也叫 `postgres`。
    *   **默认访问方式**：PostgreSQL 默认使用 "ident" 认证方式进行本地连接，这意味着它会尝试将连接的 Linux 用户名映射到同名的 PostgreSQL 用户名。

4.  **访问 PostgreSQL 命令行 (`psql`)：**
    切换到 `postgres` Linux 用户，然后运行 `psql`：
    ```bash
    sudo -i -u postgres
    psql
    ```
    你会进入 `psql` 命令行界面，提示符通常是 `postgres=#`。
    输入 `\q` 然后 `exit` 可以退出 `psql` 和 `postgres` 用户会话。

5.  **创建数据库用户和数据库：**
    为你的 FastAPI 应用创建一个专用的数据库用户和数据库通常是一个好习惯。

    *   **创建用户 (例如 `myuser`) 并设置密码：**
        在 `psql` 中执行：
        ```sql
        CREATE USER myuser WITH PASSWORD 'your_secure_password';
        ```
        **请务必替换 `'your_secure_password'` 为一个强密码。**

    *   **创建数据库 (例如 `mydb`) 并指定所有者：**
        在 `psql` 中执行：
        ```sql
        CREATE DATABASE mydb OWNER myuser;
        ```

    *   **授予用户在新数据库上的所有权限 (可选，但通常需要)：**
        在 `psql` 中执行：
        ```sql
        GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
        ```
        如果你的应用还需要创建表等操作，这是必要的。

6.  **配置 PostgreSQL 允许远程连接 (如果需要)：**
    默认情况下，PostgreSQL 可能只允许来自 `localhost` 的连接。如果你的 FastAPI 应用和 PostgreSQL 在同一台服务器上，并且 FastAPI 通过 `localhost` (或 `127.0.0.1`) 连接数据库，则不需要修改此项。

    但如果你需要从其他机器 (例如你的本地开发机) 连接到这个 PostgreSQL 服务器，或者将来你的应用部署在不同的服务器上，就需要配置远程访问：

    *   **修改 `postgresql.conf` 文件：**
        该文件通常位于 `/etc/postgresql/<version>/main/postgresql.conf` (例如 `/etc/postgresql/14/main/postgresql.conf`，具体版本号可能不同)。
        ```bash
        sudo vim /etc/postgresql/14/main/postgresql.conf # 用你的版本号替换
        ```
        找到 `listen_addresses` 配置项，默认可能是 `listen_addresses = 'localhost'`。
        将其修改为 `listen_addresses = '*'` (监听所有 IP 地址) 或特定 IP 地址。
        如果你只想允许特定 IP 连接，可以设置为 `'localhost, your_other_server_ip'`。

    *   **修改 `pg_hba.conf` 文件 (Host-Based Authentication)：**
        该文件通常位于同一目录下，例如 `/etc/postgresql/14/main/pg_hba.conf`。
        这个文件控制哪些主机允许连接，使用哪个用户，以及认证方法。
        你需要添加一行来允许你的用户从特定 IP (或所有 IP) 通过密码认证 (`md5` 或 `scram-sha-256` 更安全) 连接到你的数据库。

        例如，允许 `myuser` 从任何 IP 连接到 `mydb` 数据库，使用密码认证：
        ```
        # TYPE  DATABASE        USER            ADDRESS                 METHOD
        host    mydb            myuser          0.0.0.0/0               md5
        ```
        或者，只允许从特定 IP `1.2.3.4` 连接：
        ```
        host    mydb            myuser          1.2.3.4/32              md5
        ```
        **注意：`0.0.0.0/0` 意味着允许任何 IP 地址连接，这有安全风险。请确保你的服务器安全组规则和数据库用户密码足够强壮。对于生产环境，最好限制具体的 IP 地址段。**

    *   **重启 PostgreSQL 服务使配置生效：**
        ```bash
        sudo systemctl restart postgresql
        ```

    *   **安全组**：确保你的火山引擎服务器安全组允许 TCP 端口 5432 的入站连接 (如果需要公网访问)。如前所述，源 IP 应尽可能限制。

#### 方案二：探讨火山引擎 RDS for PostgreSQL 服务的优缺点及适用场景

火山引擎提供托管的关系型数据库服务 (RDS)，其中包括 RDS for PostgreSQL。

*   **优点：**
    *   **易于管理和维护**：火山引擎负责数据库的安装、补丁、备份、恢复、监控等日常运维工作，让你更专注于应用开发。
    *   **高可用性**：通常提供主备架构或集群架构，具备自动故障转移能力，比自建数据库更容易实现高可用。
    *   **弹性伸缩**：可以根据需求方便地调整数据库实例的规格 (CPU, 内存, 存储)。
    *   **安全性**：提供多层安全防护，如网络隔离、访问控制、数据加密等。
    *   **性能优化**：针对云环境进行了优化。
    *   **备份与恢复**：提供自动备份和按时间点恢复功能。

*   **缺点：**
    *   **成本**：通常比在 ECS 上自建数据库的直接成本要高。
    *   **灵活性限制**：对数据库的某些高级配置或操作系统层面的访问可能会受限。
    *   **特定版本支持**：可能不会立即支持最新的 PostgreSQL 版本。

*   **适用场景 (个人开发者)：**
    *   **初期/学习阶段**：如果对数据库运维不熟悉，或者希望快速搭建并运行，自建 PostgreSQL 是一个很好的起点，成本也低。
    *   **应用增长/对可靠性要求提高**：当你的应用用户量增长，数据变得越来越重要，或者你没有足够的时间和精力去维护数据库时，迁移到 RDS for PostgreSQL 是一个值得考虑的选择。它可以显著降低运维负担，并提供更好的数据可靠性和可用性保障。
    *   **预算考量**：你需要权衡 RDS 带来的便利性和增加的成本。

对于本教程，我们将主要基于**在 ECS 上自行安装 PostgreSQL** 的方式进行后续演示，因为这更符合个人开发者初期低成本和完全控制的需求。但了解 RDS 的选项也很重要。

#### 数据库连接与基本操作 (psql)

我们已经在前面接触了 `psql`。这里再补充一些：

*   连接到特定数据库：
    ```bash
    sudo -u postgres psql -d mydb
    ```
*   常用 `psql` 元命令：
    *   `\l`: 列出所有数据库。
    *   `\c dbname`: 连接到另一个数据库。
    *   `\dt`: 列出当前数据库中的所有表。
    *   `\d table_name`: 显示表结构。
    *   `\du`: 列出所有用户 (角色)。
    *   `\conninfo`: 显示当前连接信息。
    *   `\q`: 退出 `psql`。
    *   `\?`: 查看帮助。

### 小结

到目前为止，我们已经在火山引擎云服务器上：
1.  完成了服务器的基本初始化。
2.  配置了 Python 开发环境，并创建了虚拟环境。
3.  安装了 FastAPI 和 Uvicorn，并运行了一个简单的 FastAPI 应用。
4.  安装并配置了 Nginx 作为反向代理 (为生产环境做准备)。
5.  安装并初步配置了 PostgreSQL 数据库服务器，创建了用户和数据库。

这些是构建后端服务的基础。在接下来的模块中，我们将深入 FastAPI 的使用，并学习如何将它与 PostgreSQL 数据库集成。

--- 