# Docker

## 基本概念

Docker 包括三个基本概念

- 镜像（Image）

    Docker 镜像 是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像 不包含 任何动态数据，其内容在构建之后也不会被改变。

- 容器（Container）

    容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的命名空间。

- 仓库（Repository）

    如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务。一个 Docker Registry 中可以包含多个仓库；每个仓库可以包含多个 标签（Tag）；每个标签对应一个镜像。

## 使用镜像

### 获取镜像

从 Docker 镜像仓库获取镜像的命令是 docker pull。其命令格式为：

``` shell
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

具体的选项可以通过 docker pull --help 命令看到，这里我们说一下镜像名称的格式。

Docker 镜像仓库地址：地址的格式一般是 <域名/IP>[:端口号]。默认地址是 Docker Hub(docker.io)。
仓库名：<用户名>/<软件名>。对于 Docker Hub，如果不给出用户名，则默认为 library 即官方镜像。

### 列出镜像

列出已经下载下来的镜像：

```shell
docker image ls
```

#### 镜像体积

使用命令便捷的查看镜像、容器、数据卷所占用的空间。

```shell
docker system df
```

#### 虚悬镜像

由于新旧镜像同名，旧镜像名称被取消，从而出现仓库名、标签均为 <none> 的镜像。这类无标签镜像称为虚悬镜像（ dangling ）。

可以用下面的命令专门显示这类镜像：

```shell
docker image ls -f dangling=true
```

一般来说，虚悬镜像已经失去了存在的价值，可以随意删除的，可以用下面的命令删除：

```shell
docker image prune
```

#### 中间层镜像

为了加速镜像构建、重复利用资源，Docker 会利用中间层镜像。默认的 `docker image ls` 列表中只会显示顶层镜像

显示包括中间层镜像在内的所有镜像的话，需要加 -a 参数。

```shell
docker image ls -a
```

#### 列出部分镜像

- 根据仓库名列出镜像

    ```shell
    docker image ls ubuntu
    ```

- 列出特定的某个镜像，也就是说指定仓库名和标签

    ```shell
    docker image ls ubuntu:18.04
    ```

- docker image ls 支持强大的过滤器参数 --filter （ -f ）。

    - 看到在 mongo:3.2 之后建立的镜像，可以用下面的命令：

    ```shell
    docker image ls -f since=mongo:3.2
    ```

    - 查看某个位置之前的镜像也可以，把 since 换成 before 即可。

- 如果镜像构建时，定义了 LABEL，还可以通过 LABEL 来过滤。

    ```shell
    docker image ls -f label=com.example.version=0.1
    ```

#### 以特定格式显示

- 只将所有的虚悬镜像的 ID 列出来使用 -q 参数。

    ```shell
    docker image ls -q
    ```

--filter 配合 -q 产生出指定范围的 ID 列表，然后送给另一个 docker 命令作为参数，从而针对这组实体成批的进行某种操作的做法在 Docker 命令行使用过程中非常常见。

- 直接列出镜像结果，并且只包含镜像 ID 和仓库名：

    ```shell
    docker image ls --format "{{.ID}}: {{.Repository}}"
    ```

- 以表格等距显示，有标题行，和默认一样，但自己定义列：

    ```shell
    docker image ls --format "table {{.ID}}\t{{.Repository}}\t{{.Tag}}"
    ```

### 删除本地镜像

如果要删除本地的镜像，可以使用 docker image rm 命令，其格式为：

```shell
docker image rm [选项] <镜像1> [<镜像2> ...]
```

- 删除所有的镜像

    ```shell
    docker rmi $(docker images -q)
    ```

- 用 ID删除镜像。

    ```shell
    docker image rm dockerId
    ```

- 用镜像名，也就是 <仓库名>:<标签>，来删除镜像。

    ```shell
    docker image rm centos
    ```

- 更精确的是使用镜像摘要删除镜像。

    ```shell
    docker image ls --digests  //获取镜像摘要
    docker image rm 摘要
    ```

#### Untagged 和 Deleted

删除行为分为两类，一类是 Untagged，另一类是 Deleted。

镜像的唯一标识是其 ID 和摘要，而一个镜像可以有多个标签。因此当删除镜像的时候，实际上是在要求删除某个标签的镜像。所以首先需要做的是将满足我们要求的所有镜像标签都取消，这就是我们看到的 Untagged 的信息。因为一个镜像可以对应多个标签，因此当我们删除了所指定的标签后，可能还有别的标签指向了这个镜像，如果是这种情况，那么 Delete 行为就不会发生。所以并非所有的 docker image rm 都会产生删除镜像的行为，有可能仅仅是取消了某个标签而已。

#### 用 docker image ls 命令来配合

可以使用 docker image ls -q 来配合使用 docker image rm，这样可以成批的删除希望删除的镜像。

比如，我们需要删除所有仓库名为 redis 的镜像：

```shell
docker image rm $(docker image ls -q redis)
```

或者删除所有在 mongo:3.2 之前的镜像：

```shell
docker image rm $(docker image ls -q -f before=mongo:3.2)
```

### 利用 commit 理解镜像构成

当我们运行一个容器的时候（如果不使用卷的话），我们做的任何文件修改都会被记录于容器存储层里。而 Docker 提供了一个 docker commit 命令，可以将容器的存储层保存下来成为镜像。换句话说，就是在原有镜像的基础上，再叠加上容器的存储层，并构成新的镜像。以后我们运行这个新镜像的时候，就会拥有原有容器最后的文件变化。

docker commit 的语法格式为：

```shell
docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]
```

例如，可以用下面的命令将容器保存为镜像：

```shell
docker commit -a "Hakurei Yan <hakurei.yan@maiscrm.com>" -m "修改了默认网页" 容器ID或容器名 <仓库名>:<标签>
```

--author 是指定修改的作者， --message 则是记录本次修改的内容。这些信息可以省略留空。

可以用如下命令具体查看镜像内的历史记录

```shell
docker history 镜像
```

## 操作容器

容器是独立运行的一个或一组应用，以及它们的运行态环境。

### 启动容器

启动容器有两种方式，一种是基于镜像新建一个容器并启动，另外一个是将在终止状态的容器重新启动。

#### 新建并启动

所需要的命令主要为 `docker run`。

以这个镜像为基础启动并运行一个容器。如果要启动里面的 bash 并且进行交互式操作的话，可以执行下面的命令。

```shell
docker run -it --rm ubuntu:18.04 bash
```

- docker run 是运行容器的命令。
- -t 选项让Docker分配一个伪终端并绑定到容器的标准输入上
- -i 让容器的标准输入保持打开，保持交互模式。
- --rm：这个参数是说容器退出后随之将其删除。
- ubuntu:18.04：这是指用 ubuntu:18.04 镜像为基础来启动容器。
- bash：放在镜像名后的是命令，这里希望有个交互式 Shell，因此用的是 bash。

当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：

- 检查本地是否存在指定的镜像，不存在就从 registry 下载。
- 利用镜像创建并启动一个容器。
- 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层。
- 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去。
- 从地址池配置一个 ip 地址给容器。
- 执行用户指定的应用程序。
- 执行完毕后容器被终止。

#### 启动已终止容器

```shell
docker start [OPTIONS] container
```

### 后台运行

需要让 Docker 在后台运行而不是直接把执行命令的结果输出在当前宿主机下。此时，可以通过添加 -d 参数来实现。

```shell
docker run -d ubuntu:18.04 /bin/sh -c "while true; do echo hello world; sleep 1; done"
```

此时容器会在后台运行并不会把输出的结果 (STDOUT) 打印到宿主机上面(输出结果可以用 docker logs 查看)。

使用 -d 参数启动后会返回一个唯一的 id

可以通过 docker container ls 命令来查看所有容器信息。

```shell
docker container ls [OPTIONS]
```

要获取容器的输出信息，可以通过 docker container logs 命令。

```shell
docker container logs [OPTIONS] CONTAINER
```

### 终止容器

- 终止一个运行中的容器。

    ```shell
    docker stop CONTAINER
    ```

     此外，当 Docker 容器中指定的应用终结时，容器也自动终止。

- 终止所有的容器。

    ```shell
    docker stop $(docker ps -aq)
    ```

- 重新启动处于终止状态的容器。

    ```shell
    docker start CONTAINER
    ```

- 终止一个运行态的容器，然后再重新启动它。

    ```shell
    docker restart CONTAINER
    ```

### 进入容器

在使用 -d 参数时，容器启动后会进入后台。某些时候需要进入容器进行操作，包括使用 `docker attach` 命令或 `docker exec` 命令，推荐使用 `docker exec` 命令。

- attach 命令

    ```shell
    docker run -dit ubuntu  // 启动一个容器
    docker container ls     // 获取容器的 ID
    docker attach 243c      // attach 容器 ID 即可对容器输入命令
    ```

    注意：如果从这个 stdin 中 exit，会导致容器的停止。

- exec 命令

    ```shell
    docker run -dit ubuntu
    docker container ls
    docker exec -it 69d1 bash
    ```

    如果从这个 stdin 中 exit，不会导致容器的停止。这就是为什么推荐使用 docker exec 的原因。

### 导出和导入容器

- 导出容器
    如果要导出本地某个容器，可以使用 `docker export` 命令。

    ```shell
    docker container ls -a    // 获取容器的 ID
    docker export 7691a814370e > ubuntu.tar
    ```

    这样将导出容器快照到本地文件。

- 导入容器快照

    可以使用 docker import 从容器快照文件中再导入为镜像，例如

    ```shell
    cat ubuntu.tar | docker import - test/ubuntu:v1.0
    ```

    此外，也可以通过指定 URL 或者某个目录来导入，例如

    ```shell
    docker import <http://example.com/exampleimage.tgz> example/imagerepo
    ```

注：用户既可以使用 docker load 来导入镜像存储文件到本地镜像库，也可以使用 docker import 来导入一个容器快照到本地镜像库。这两者的区别在于容器快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），而镜像存储文件将保存完整记录，体积也要大。此外，从容器快照文件导入时可以重新指定标签等元数据信息。

### 删除容器

- 删除一个处于终止状态的容器。

    ```shell
    docker container rm CONTAINER
    ```

- 如果要删除一个运行中的容器，可以添加 -f 参数。Docker 会发送 SIGKILL 信号给容器。

    ```shell
    docker container rm -f CONTAINER
    ```

- 删除所有的容器

    ```shell
    docker container rm $(docker ps -aq)
    ```

- 删除所有处于终止状态的容器

    ```shell
    docker container prune
    ```

## Docker 仓库

### Docker Hub

目前 Docker 官方维护了一个公共仓库 Docker Hub，其中已经包括了数量超过 2,650,000 的镜像。大部分需求都可以通过在 Docker Hub 中直接下载镜像来实现。

- 注册
    你可以在 <https://hub.docker.com> 免费注册一个 Docker 账号。

- 登录
    可以通过执行 docker login 命令交互式的输入用户名及密码来完成在命令行界面登录 Docker Hub。通过 docker logout 退出登录。

- 拉取镜像
    你可以通过 docker search 命令来查找官方仓库中的镜像，并利用 docker pull 命令来将它下载到本地。

    ```shell
    docker search centos
    ```

    在查找的时候通过 --filter=stars=N 参数可以指定仅显示收藏数量为 N 以上的镜像。

- 推送镜像

    用户可以在登录后通过 docker push 命令来将自己的镜像推送到 Docker Hub。

    ```shell
    docker tag ubuntu:18.04 username/ubuntu:18.04     //  username 替换为你的 Docker 账号用户名
    docker image ls
    docker push username/ubuntu:18.04
    ```

- 自动构建

    自动构建（Automated Builds）可以自动触发构建镜像，方便升级镜像。自动构建允许用户通过 Docker Hub 指定跟踪一个目标网站（支持 GitHub 或 BitBucket ）上的项目，一旦项目发生新的提交或者创建了新的标签，Docker Hub 会自动构建镜像并推送到 Docker Hub 中。

    要配置自动构建，包括如下的步骤：

    - 登录 Docker Hub；
    - 在 Docker Hub 点击右上角头像，在账号设置（Account Settings）中关联（Linked Accounts）目标网站；
    - 在 Docker Hub 中新建或选择已有的仓库，在 Builds 选项卡中选择 Configure Automated Builds；
    - 选取一个目标网站中的项目（需要含 Dockerfile）和分支；
    - 指定 Dockerfile 的位置，并保存。
    - 之后，可以在 Docker Hub 的仓库页面的 Timeline 选项卡中查看每次构建的状态。

### 私有仓库

有时候使用 Docker Hub 这样的公共仓库可能不方便，用户可以创建一个本地仓库供私人使用。

docker-registry 是官方提供的工具，可以用于构建私有的镜像仓库。

```shell
docker run -d -p 5000:5000 --restart=always --name registry registry
```

例如：

```shell
    docker run -d -p 5000:5000 -v /home/user1/storage:/var/lib/registry --name registry registry:2
```

参数说明如下：
-d：后台运行。

-p：主机的 5000 port mapping 到 container 的 5000 port。

-v：將主机的档案路径 mapping 到 container 里面的档案路径。

--name：设定 docker container 的名称。

### 在私有仓库上传、搜索、下载镜像

创建好私有仓库之后，就可以使用 docker tag 来标记一个镜像，然后推送它到仓库。例如私有仓库地址为 127.0.0.1:5000。

使用 docker tag 将 ubuntu:latest 这个镜像标记为 127.0.0.1:5000/ubuntu:latest。

格式为 docker tag IMAGE[:TAG] [REGISTRY_HOST[:REGISTRY_PORT]/]REPOSITORY[:TAG]。

```shell
docker tag ubuntu:latest 127.0.0.1:5000/ubuntu:latest
```

使用 docker push 上传标记的镜像。

```shell
docker push 127.0.0.1:5000/ubuntu:latest
```

用 curl 查看仓库中的镜像。

```shell
curl 127.0.0.1:5000/v2/_catalog
```

### 配置非 https 仓库地址

如果想让本网段的其他主机也能把镜像推送到私有仓库，就得把例如 192.168.199.100:5000 这样的内网地址作为私有仓库地址，这时会发现无法成功推送镜像。

这是因为 Docker 默认不允许非 HTTPS 方式推送镜像。我们可以通过 Docker 的配置选项来取消这个限制。

- Ubuntu 16.04+, Debian 8+, centos 7。

    对于使用 systemd 的系统，请在 /etc/docker/daemon.json 中写入如下内容（如果文件不存在请新建该文件）。

    ```json
    {
      "registry-mirror": [
        "https://hub-mirror.c.163.com",
        "https://mirror.baidubce.com"
      ],
      "insecure-registries": [
        "192.168.199.100:5000"
      ]
    }
    ```

    注意：该文件必须符合 json 规范，否则 Docker 将不能启动。

- 其他

对于 Docker Desktop for Windows 、 Docker Desktop for Mac 在设置中的 Docker Engine 中进行编辑 ，增加和上边一样的字符串即可。

## 数据管理

在容器中管理数据主要有两种方式：

- 数据卷（Volumes）。
- 挂载主机目录 (Bind mounts)。

### 数据卷

数据卷是一个可供一个或多个容器使用的特殊目录，它绕过 UFS。

- 特性：
    - 数据卷可以在容器之间共享和重用。
    - 对数据卷的修改会立马生效。
    - 对数据卷的更新，不会影响镜像。
    - 数据卷默认会一直存在，即使容器被删除。
- 创建一个数据卷。

    ```shell
    docker volume create my-vol
    ```

- 查看所有的数据卷。

    ```shell
    docker volume ls
    ```

- 查看指定数据卷的信息。

    ```shell
    docker volume inspect my-vol
    ```

- 启动一个挂载数据卷的容器。

    在用 docker run 命令的时候，使用 --mount 标记来将数据卷挂载到容器里。在一次 `docker run` 中可以挂载多个数据卷。

    下面创建一个名为 web 的容器，并加载一个数据卷到容器的 /usr/share/nginx/html 目录。

    ```shell
    docker run -d -P --name web -v my-vol:/usr/share/nginx/html --mount source=my-vol,target=/usr/share/nginx/html nginx:alpine
    ```

- 查看数据卷的具体信息,

    在主机里使用以下命令可以查看 web 容器的信息

    ```shell
    docker inspect web
    ```

    数据卷 信息在 "Mounts" Key 下面

- 删除数据卷

    ```shell
    docker volume rm my-vol
    ```

    数据卷 是被设计用来持久化数据的，它的生命周期独立于容器，Docker 不会在容器被删除后自动删除数据卷，并且也不存在垃圾回收这样的机制来处理没有任何容器引用的数据卷。如果需要在删除容器的同时移除数据卷。可以在删除容器的时候使用 `docker rm -v` 这个命令。

- 删除所有无主的数据卷。

    ```shell
    docker volume prune
    ```

### 挂载主机目录

- 挂载一个主机目录作为数据卷

    使用 --mount 标记可以指定挂载一个本地主机的目录到容器中去。

    ```shell
    docker run -d -P --name web --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html nginx:alpine
    ```

    上面的命令加载主机的 /src/webapp 目录到容器的 /usr/share/nginx/html目录。这个功能在进行测试的时候十分方便，比如用户可以放置一些程序到本地目录中，来查看容器是否正常工作。本地目录的路径必须是绝对路径，以前使用 -v 参数时如果本地目录不存在 Docker 会自动为你创建一个文件夹，现在使用 --mount 参数时如果本地目录不存在，Docker 会报错。

- 读写权限。

    Docker 挂载主机目录的默认权限是读写，用户也可以通过增加 readonly 指定为 只读。

    ```shell
    docker run -d -P --name web --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html,readonly nginx:alpine
    ```

- 挂载一个本地主机文件作为数据卷

    --mount 标记也可以从主机挂载单个文件到容器中

    ```shell
    docker run --rm -it --mount type=bind,source=$HOME/.bash_history,target=/root/.bash_history ubuntu:18.04 bash
    ```

## Docker 网络功能

Docker 允许通过外部访问容器或容器互联的方式来提供网络服务。

### 外部访问容器

容器中可以运行一些网络应用，要让外部也可以访问这些应用，可以通过 -P 或 -p 参数来指定端口映射。

当使用 -P 标记时，Docker 会随机映射一个端口到内部容器开放的网络端口。

```shell
docker run -d -P nginx:alpine
```

通过 docker logs 命令来查看访问记录。

```shell
docker logs fa
```

-p 则可以指定要映射的端口，并且，在一个指定端口上只可以绑定一个容器。支持的格式有 ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort。

- 映射所有接口地址

    使用 hostPort:containerPort 格式本地的 80 端口映射到容器的 80 端口，可以执行

    ```shell
    docker run -d -p 80:80 nginx:alpine
    ```

    此时默认会绑定本地所有接口上的所有地址。

- 映射到指定地址的指定端口

    可以使用 ip:hostPort:containerPort 格式指定映射使用一个特定地址。

    ```shell
    docker run -d -p 127.0.0.1:80:80 nginx:alpine
    ```

- 映射到指定地址的任意端口

    使用 ip::containerPort 绑定 localhost 的任意端口到容器的 80 端口，本地主机会自动分配一个端口。

    ```shell
    docker run -d -p 127.0.0.1::80 nginx:alpine
    ```

    还可以使用 udp 标记来指定 udp 端口

    ```shell
    docker run -d -p 127.0.0.1:80:80/udp nginx:alpine
    ```

- 查看映射端口配置

    使用 docker port 来查看当前映射的端口配置，也可以查看到绑定的地址

    ```shell
    docker port ContainerId 80
    ```

    容器有自己的内部网络和 ip 地址（使用 docker inspect 查看，Docker 还可以有一个可变的网络配置）。

- -p 标记可以多次使用来绑定多个端口。

    ```shell
    docker run -d -p 80:80 -p 443:443 nginx:alpine
    ```

### 容器互联

- 新建网络
    下面先创建一个新的 Docker 网络。

    ```shell
    docker network create -d bridge my-net
    ```

    -d 参数指定 Docker 网络类型，有 bridge overlay。其中 overlay 网络类型用于 Swarm mode。

- 连接容器

    运行一个容器并连接到新建的 my-net 网络

    ```shell
    docker run -it --rm --name busybox1 --network my-net busybox sh
    ```

    打开新的终端，再运行一个容器并加入到 my-net 网络

    ```shell
    docker run -it --rm --name busybox2 --network my-net busybox sh
    ```

- 通过 ping 来证明 busybox1 容器和 busybox2 容器建立了互联关系。

    在 busybox1 容器输入以下命令

    ```shell
    ping busybox2
    ```

    在 busybox2 容器执行 ping busybox1，也会成功连接到。

    ```
    ping busybox1
    ```
