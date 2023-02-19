# Docker

SECoder 平台通过构建 Docker 镜像并运行容器进行部署。Docker 是一个应用程序容器化环境，通过将程序隔离在不同的容器中运行来提供一致的运行环境，降低运维成本。

Docker 可以理解为程序级别的虚拟化，类比于虚拟机，但 Docker 容器要轻量得多，几乎不带来性能损耗。一个 Docker 容器包含了除了操作系统内核之外的整个文件系统，包括依赖的库以及应用程序本身。

在进行部署之前，我们首先需要将程序运行所需的环境构建为一个**镜像 (Image)**。一个镜像包含与主机隔离的属于自己的文件系统。我们通过构建镜像将程序运行所需的环境放入这个文件系统中，之后基于镜像运行一个**容器 (Container)**。镜像与容器的关系类似于程序 (Program) 与进程 (Process) 的关系，即镜像是静态存在，而容器是基于镜像运行的动态存在。

!!! note "容器的易失性"

    容器的文件系统是一个临时文件系统，因此容器重启时其中的所有数据将会丢失。我们将在后续的文档介绍如何通过配置项和持久存储来使得数据在不同容器之间保持。

## Dockerfile

我们通过 Dockerfile 文件指示构建镜像时需要执行的操作。以下是一个示例 Dockerfile：

```dockerfile
FROM python:3.9

ENV HOME=/opt/app

WORKDIR $HOME

COPY requirements.txt .

RUN pip install -i https://pypi.tuna.tsinghua.edu.cn/simple -r requirements.txt

COPY src $HOME

EXPOSE 80

CMD ["python3", "main.py"]
```

我们将会通过逐句讲解此 Dockerfile 中每条指令的作用来让你对 Docker 镜像的构建过程有一个快速了解。

```dockerfile
FROM python:3.9
```

指定当前镜像基于什么镜像进行构建。你可以在 [Docker Hub](https://hub.docker.com) 上找到各种各样的镜像。像 Docker Hub 这样的网站被称为 Image Registry，在配置 CI 时我们将会看到除了默认的 Docker Hub 之外，也可以指定其他 registry 作为镜像来源，例如 SECoder 自己的 registry。

一般来说，一个镜像只包含能够满足功能需求的最小环境。例如，我们在这里所使用的 `python` 镜像就包含了能够运行 Python 的最小环境。这样做的好处是使镜像充分精简，同时允许通过基于已有镜像构建新镜像的方法获得所需的功能。

在镜像名后可以加上以冒号分隔的标签。标签一般指定镜像的版本，例如这里是 `3.9`，代表此镜像的 Python 版本为 3.9。如果不指定标签，默认获取标签为 `latest` 的版本。

!!! note "本课程中你也许会需要的镜像"

    这里列出几个在本课程中也许用得到的镜像，你可以基于它们构建自己的镜像。

    |镜像|说明|
    |-|-|
    |[ubuntu](https://hub.docker.com/_/ubuntu)|Ubuntu 镜像，为程序提供基础运行环境|
    |[gcc](https://hub.docker.com/_/gcc)|GCC 镜像，用于 C/C++ 程序的编译|
    |[python](https://hub.docker.com/_/python)|Python 镜像，用于基于 Python 的后端等|
    |[node](https://hub.docker.com/_/node)|Node.js 镜像，用于前端网站文件的生成|
    |[nginx](https://hub.docker.com/_/nginx)|nginx 镜像，用于前端网站的反向代理|
    |[mysql](https://hub.docker.com/_/mysql)|MySQL 镜像，用于数据库容器|
    |[postgres](https://hub.docker.com/_/postgres)|PostgreSQL 镜像，用于数据库容器|

```dockerfile
ENV HOME=/opt/app
```

设定一个环境变量。注意环境变量不仅可以在后续指令中使用，也会保留到容器运行时。

```dockerfile
WORKDIR $HOME
```

指定工作目录。注意这一工作目录指在镜像的文件系统中的工作目录，而非本机的文件系统。这一工作目录不仅会作为之后 Dockerfile 指令的工作目录，也会作为运行时的初始工作目录。如不指定，默认工作目录为根目录 `/`。在这里我们用 `$HOME` 引用了我们之前声明的环境变量。

```dockerfile
COPY requirements.txt .
```

将本机文件系统的文件拷贝到镜像文件系统中。这里我们将本机文件系统中，工作目录 (即 Dockerfile 所在的目录) 下的 `requirements.txt` 复制到镜像文件系统中的 `.` 即工作目录，这里是 `/opt/app`。

!!! note "requirements.txt"

    将 Python 程序需要的第三方库的清单列入 `requirements.txt` 是一种常见做法。以下是一个示例 `requirements.txt`：

    ```
    django
    requests==2.28.1
    ```

    这表明程序需要 `django` 以及版本为 `2.28.1` 的 `requests` 库来运行。

```dockerfile
RUN pip install -i https://pypi.tuna.tsinghua.edu.cn/simple -r requirements.txt
```

`RUN` 指令在镜像环境中执行一段命令。在这里，我们在镜像中通过 `pip` 安装 `requirements.txt` 指定的第三方库，其中 `-i` 选项将索引源更换到 TUNA 以加快下载速度。`RUN` 指令的影响将会在镜像中保留。因此，最终构建的镜像将包含有我们安装的第三方库。

`RUN` 是一个构建时的指令，这意味着运行容器时将不会重新执行这段命令。

```dockerfile
COPY src $HOME
```

将主机文件系统中的 `src` 目录下的文件复制到镜像文件系统的 `$HOME` 目录。

!!! warning

    目录本身不会被复制，只有目录下的文件会被复制。

!!! note "`.dockerignore`"

    严格来说，`COPY` 指令其实是从 Docker 的**构建上下文** (Build Context) 复制文件的。在构建过程开始前，当前目录下的文件将会被复制到构建上下文中，因此表现上 `COPY` 指令就如同是在从主机文件系统中复制文件一样。

    你可以通过 `Dockerfile` 所在目录下的 `.dockerignore` 文件来控制哪些文件应该被排除在构建上下文之外。该文件的格式类似于 `.gitignore`，但有一些细微的差别，例如 `.dockerignore` 中出现单独的文件名只会忽略工作目录顶层的该文件，而非 `.gitignore` 那样忽略任意层级下该文件名的文件。

    一般来说，我们会把 `Dockerfile`、`.git` 等容器运行不需要的文件忽略掉，这样一方面可以让镜像更加精简，另一方面可以使得 Docker 更好地利用缓存机制加速构建。后一点是因为 Docker 会在可能时利用缓存来省略某些步骤的执行，但一旦某处的 `COPY` 指令包含了被修改的文件，该步骤和此后的步骤势必只能重新执行。忽略 `.git` 这类常常被修改但实际上不影响容器构建过程的文件之后，对它们的修改将不会重新触发 `COPY` 指令的执行，Docker 就能利用已经存在的缓存来加速构建了。

我们通过将程序复制到镜像中来在容器运行时执行程序。对于 Python 这样的解释型语言，我们将源文件复制到镜像中即可。

```dockerfile
EXPOSE 80
```

暴露 80 端口。Docker 容器的网络环境与外界同样是隔离的，因此如果需要让外界能够向容器发送网络请求，就需要指定需要暴露的端口。

```dockerfile
CMD ["python3", "main.py"]
```

指定容器运行时执行的命令。注意，与 `RUN` 指令不同，容器构建时不会执行这段命令，只有容器启动时才会。注意此指令的格式，需要将每个参数分隔为单独字符串并在最外层加方括号。这里的这条指令表明基于该镜像的容器运行时将执行 `python3 main.py`。典型地，这条指令用于启动我们的程序。不过，如果程序运行步骤比较复杂，我们一般会选择将程序运行逻辑写为一个 shell 脚本，而在 Dockerfile 中使用 `CMD ["sh", "run.sh"]` 之类的指令来执行这个脚本。

!!! warning "不要将开发配置作为正式部署"

    开发环境的配置往往没有针对部署场景进行安全性和性能方面的检查，因此请不要在正式部署中使用开发配置。例如，如果你直接使用 `python3 manage.py runserver` 作为容器的运行命令，这将启动一个开发服务器，它并没有针对高并发场景进行优化。同时，若你没有在 Django 项目的 `settings.py` 中将 `DEBUG` 设置为 `False`，在发生异常时程序的调用栈将会直接展示给用户，这将带来极大的危险性。

    正确的部署方式可以参考你所使用的框架的文档。例如，[How to deploy Django](https://docs.djangoproject.com/en/4.1/howto/deployment/) 文档说明了应该如何部署 Django，包括使用 Gunicorn 和 uWSGI 等方式。

---

到此，该镜像的构建过程就结束了，之后我们就可以使用 SECoder 的 deployer 工具基于构建的镜像运行一个容器。

## 多阶段构建

前面我们提到，Python 等解释型语言只需要将源文件直接复制到镜像中，在容器运行时解释运行源文件即可。但对于 C++/Go/Rust 等编译型语言来说，我们通常不会在容器运行时再进行编译，而是在镜像构建时就执行编译，在容器运行时直接执行可执行文件即可。但是，我们通常希望镜像尽可能精简，为此我们希望最终的镜像只包含可执行文件，不包含源文件。我们可以使用多阶段构建来实现这一点。

结合本课程的特点，下面我们以前端镜像的构建为例说明多阶段构建。前端的部署过程与编译型语言有些类似：我们首先生成网站所提供的静态文件，然后再通过 nginx 反向代理使网站向访问者提供这些文件。

```dockerfile
# Stage 0: build
FROM node:18 AS build

ENV FRONTEND=/opt/frontend

WORKDIR $FRONTEND

RUN npm config set registry https://registry.npmmirror.com

COPY . .

RUN npm install

RUN npm run build

# Stage 1
FROM nginx:1.22

ENV HOME=/opt/app

WORKDIR $HOME

COPY --from=build /opt/frontend/dist dist

COPY nginx /etc/nginx/conf.d

EXPOSE 80
```

可以看到，现在我们的构建过程被 `FROM` 指令分为两个阶段，其中我们把第一个阶段命名为 `build`。 (`#` 开头的行只是注释。) 在第一个阶段，我们首先将 `npm` 源更换为 [npmmirror](https://npmmirror.com) 以加速下载，然后将主机文件系统下当前目录的文件全部复制到镜像文件系统中，再使用 `npm install` 安装需要的第三方包，最后使用 `npm run build` 生成站点的静态文件。我们的网站只需要这些文件就能提供服务，不需要我们下载的第三方包以及整个工具链。因此，我们接下来在下一个阶段构建我们真正使用的镜像。

!!! note "主机文件系统"

    在 GitLab 平台上执行 CI/CD 时，这里的主机文件系统实际上是我们的仓库，因此 `COPY . .` 实际上是将仓库中的所有文件复制到了镜像的工作目录。因此，不用担心庞大的 `node_modules` 会带来什么负担。

    > 如果你没有忘了 `.gitignore` 它的话。相信我，你不会忘的。

要向外界提供我们的静态文件，我们一般会使用 nginx 进行反向代理，根据请求向客户提供页面文件。

!!! note "nginx"

    我们不会过多介绍 nginx 的用法，你可以通过 [nginx 文档](https://nginx.org/en/docs/) 或网络上的其他资源进行学习。对于简单的前端页面部署，我们提供一个样例 nginx 配置，它应该能够满足多数需求。我们在仓库的 `nginx` 目录下放置名为 `default.conf` 的以下配置文件：

    ```nginx
    server {
        listen 80;
        root /opt/app/dist;

        location / {
            try_files $uri $uri/ /index.html;
        }

        location /api/ {
            proxy_pass https://your.backend.url/;
        }
    }
    ```

    它表明监听 80 端口，以 `/opt/app/dist` 目录作为根目录，在访问任意 URL 时首先尝试查找该 URI 的文件，再查找该 URI 的目录，最后回退到 `/index.html`。这意味着对特定资源的请求将会直接返回该资源，而对于其他情况将会转交给前端框架，由前端框架处理路由。

    第二个 `location` 部分适用于需要前端服务器重写 API 请求的情况，这么做会将到前端 `/api/` 路径下的请求转发到你的后端地址 `https://your.backend.url/`。如果没有需要前端重写 API 请求的需求，则不需要这一部分。

    请注意在 `proxy_pass` 的 URL 末尾需要加 `/`。如果你对原因感到好奇，可以查阅相关资料了解 `location` 和 `proxy_pass` 的工作方式。

我们基于 `nginx` 镜像构建我们这一阶段的镜像。不同构建阶段的镜像是互相独立的，不共享文件系统。因此，我们在希望从之前的阶段复制文件时，需要使用 `COPY --from={stage}` 指令，其中 `{stage}` 为阶段名称或编号。在这里，我们将前一阶段生成的位于 `/opt/frontend/dist` 目录下的静态文件复制到当前镜像的 `dist` 目录，再将主机文件系统 `nginx` 目录下的配置文件复制到镜像中 nginx 的配置目录 `/etc/nginx/conf.d`。最后，我们暴露 80 端口。由于没有显式指定 `CMD` 指令，这一镜像将使用 `nginx` 镜像的 `CMD` 指令，运行 nginx 服务器。

## 本地运行容器

在安装 [Docker](https://www.docker.com) 后，你可以通过在本地构建镜像和运行容器来进行调试。

!!! note "Docker Desktop"

    在 Windows 和 macOS 下，你需要通过 [Docker Desktop](https://www.docker.com/products/docker-desktop/) 来使用 Docker。不过，在 Windows 下更推荐的方法是通过 WSL 来使用 Docker。

### 构建镜像

在包含 Dockerfile 的目录下，执行 `docker build . --tag {name}:{tag}` 即可构建镜像，其中 `{name}` 为镜像名，`{tag}` 为标签。

### 运行容器

执行 `docker run -it --rm {name}:{tag}` 来基于指定镜像运行一个容器，镜像既可以是本地构建的也可以是在 Docker Hub 或其他 registry 中托管的。其中 `-i` 选项指定交互模式，`-t` 指定分配伪终端，`--rm` 指定运行结束后移除容器。这些选项在调试时会比较有用。

你可以可选地在最后指定容器运行时执行的命令，这将会覆盖镜像中使用 `CMD` 指定的命令。例如，你可以通过 `docker run -it --rm nginx:1.22 bash` 来启动一个运行 `bash` 的 `nginx` 容器，这将使你能够探索 `nginx` 容器的内部结构并允许你在容器内部运行命令。

你还可以指定 `-p [hostPort]:{containerPort}` 来将本机端口 `hostPort` 映射到容器暴露的端口 `containerPort`。

!!! note "容器的退出与移除"

    Docker 容器退出后并不会自动被移除，将会继续留存并占用空间，因此我们在调试时通过 `--rm` 来让容器退出后自动移除。

    我们也可以使用 `docker ps --all` 命令来查看所有容器 (包括已退出的)，并使用 `docker rm` 命令来移除不再需要的容器。

## 参考资料

以上我们简要介绍了 Docker 的使用，包括如何通过 Dockerfile 来声明镜像的构建步骤以及如何在本地构建镜像和运行容器等等。

如果希望更加深入地学习 Docker 的高级用法以及工作原理等，可以参考 [2022 酒井科协暑培 Docker 文档](https://liang2kl.github.io/2022-summer-training-docker-tutorial)和 [Docker 官方文档](https://docs.docker.com)。

在本课程提供的 [样例项目](https://git.tsinghua.edu.cn/SEG/example) 仓库中也可以找到几种常见项目框架的 Dockerfile，可供配置部署时参考。需要注意的是，这些样例项目都较为老旧，请在参考时注意版本和兼容性等问题。