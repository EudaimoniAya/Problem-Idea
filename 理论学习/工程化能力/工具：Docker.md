# 一、前置知识
大体概念在[[PostgreSQL和使用Docker完成pgvector扩展]]中提到，下面是一些拓展的点：
* OCI（Open Container Initiative，开放容器倡议）是一个由Linux基金会主导的项目，旨在指定容器技术的开放工业标准。它提倡：**让整个容器生态围绕一套公共的规范运转，避免被单一厂商（如 Docker）锁定**。OCI定义了三个关键规范：
  1. **OCI Runtime Specification（运行时规范）**：定义了如何解压容器镜像并启动一个隔离的进程（容器）。它规定了容器的配置格式（`config.json`）和运行环境（如命名空间、cgroups），以及生命周期操作（创建、启动、停止等）。任何遵循此规范的运行时（如 `runc`、`crun`、`youki`）都能运行任何符合 OCI 镜像规范的容器。Docker 早期使用自己的 libcontainer，后来将 `runc` 贡献给 OCI 并成为其参考实现。
  2. **OCI Image Specification（镜像规范）**：定义了**容器镜像的格式**。**镜像由一系列只读层（layers）组成**，每层是文件系统的变更集，并包含一个 JSON 配置文件（描述入口点、环境变量、暴露端口等）。这个规范让不同工具（如 Docker、Podman、Buildah、Skopeo）可以构建、传输、运行完全相同的镜像，保证了互操作性。（另外关于layers，在GHA的项目的`build-push`中，存在`cache-to: type=gha,mode=max`这个配置。其中`mode=max`就与layers相关，因为**在Docker构建中，每个`RUN`、`COPY`等指令都会生成一个只读层**，`mode=max`表示**缓存包括中间层的所有构建层，即镜像层**，这样后续构建时，即使代码变化导致某些层失效，之前**未变化的层仍能够从缓存中恢复**。而默认的`mode=min`只缓存最终镜像的层，复用率较低）
  3. **OCI Distribution Specification（分发规范）**：定义了镜像仓库（registry）与客户端之间的 API 协议，用于推送和拉取镜像。Docker Hub、Google Container Registry、Amazon ECR 等都遵循此规范，使得任何符合 OCI 的客户端都可以与任意合规的仓库交互。
* 容器的实现基于三大技术：用于隔离资源视图的Namespace、用于隔离资源使用的cgroup，这两点笔记中有分析。第三个技术就是Union Mount Filesystems（**overlayfs，联合挂载文件系统**），它能够将不同文件系统的文件或目录视为分支，并**合并成统一视图**。
  **镜像由多个只读层（layer）堆叠而成**，每一层代表Dockerfile中的一条指令（如`RUN`、`COPY`）；容器在镜像之上增加了一个可读写层（容器层），**所有对容器的修改（增删改查文件）都写在可读写层里，原镜像层保持不变**；最终联合文件系统将其”联合“挂载到一个目录下。
  这样让下层的镜像层可以被很多镜像共享，无需重复存储。因为只读层通过内容寻址（SHA-256）唯一标识（**和Git用SHA-1管理对象，Nix用SHA-256管理包是一种思路**）。所以当拉取一个新镜像是，Docker会检测本地是否存在哈希相同的层，存在则直接复用。比如很多不同镜像基于`alpine`对应的层，那么就会共享`alpine`的只读层
  OverlayFS是Linux内核的联合文件系统实现，它将目录分为三个关键部分：`Lower`、`Upper`、`Merged`。`Lower`对应底层的只读目录，对应镜像的只读层；`Upper`为上层可读写目录，记录容器层修改；`Merged`合并视图，容器进程看到最终的文件系统：
  ![[Pasted image 20260321165332.png]]
  示例：假设一个Docker镜像有两层：基础系统`lower1`和安装Python`lower2`，那么运行容器时，OverlayFS会将这两个只读层作为`lowerdir`，并创建一个空的`upperdir`叠加在上面，合并到`merged`。当要在容器内修改`/etc/hosts`，这个文件会从`lower2`复制到`upperdir`中修改，而原始镜像层不变。
* **Docker Desktop的架构【重要】**：
  ![[Pasted image 20260321172842.png]]
  **左侧为用户交互层**（Client），包含四个组件：
  ① **Docker CLI（`docker`）**：它用于接收用户输入的`docker`命令，**通过HTTP API将请求发送给Docker Daemon，它本身不执行容器操作**。其中，[[工具：GitHub Actions]]提到的Unix套接字`/var/run/docker.sock`就是两者通信的通道，里面传输的就是**HTTP协议**的请求和相应。在同一台机器通信时，使用Unix套接字比起TCP更轻量，且安全性更高（Unix套接字受文件系统权限控制，只有`root`或`docker`组的用户可以访问，而TCP端口容易暴露，需要额外认证）【图中CLI到Daemon的连线，背后就是`docker.sock`】
  ② Graphical User Interface（GUI）：提供可视化管理面板，方便查看容器、镜像、卷、网络状态，以及调整资源配额等，但是和自动化相比效率不高
  ③ Docker Credential Helpers：安全存储镜像仓库的登录凭证，如 Docker Hub 的用户名/密码或个人访问令牌
  ④ Extensions：扩展 Docker Desktop 的功能，例如集成日志查看器、性能监控、Kubernetes 仪表盘等
  **中间为核心运行环境**，Linux虚拟机：
  ① **Docker Daemon（`dockerd`）**：运行在Linux VM内部，负责接收来着CLI发来的用户命令、管理容器生命周期、镜像、网络等资源、与镜像仓库交互，拉取或推送镜像。
  ② 镜像与容器：镜像为只读模板，容器为镜像的运行实例，在其上增加一个可写层，并使用OverlayFS联合挂载，形成最终文件系统视图。
  ③ 内置K8s集群：为Docker的可选组件，运行在同一Linux VM中（通过`kubeadm`初始化），为开发人员提供本地K8s环境。Docker Desktop会启动一个单节点集群，并与Docker共享同一底层容器运行时。它与Docker能够并行存在，互不干扰，所以可以使用`kubectl`管理K8s，同时用`docker`管理容器
  **右侧为镜像仓库**，如Docker Hub、Amazon ECR等。用于存储和分发Docker镜像，遵循OCI发放规范。通过`docker pull`向某仓库请求镜像，拉取所有缺失层；通过`docker push`将本地镜像的层上传到仓库。
  在这过程中，CLI通过Docker Credential Helpers获取凭证，传递给Daemon，Daemon在请求仓库时附加认证头。
* 因为容器对文件的任何增删改查都保存在可读写层中，不会影响到下层的镜像，所以一旦容器停止并销毁，所有修改和数据会丢失。但是如果使用挂载（分为命名卷挂载和绑定挂载，细节见笔记），则可以让容器将需要持久化的数据：
  ![[Pasted image 20260321190904.png]]
* **镜像层叠加**：在执行下列代码时
```bash
# 交互式地运行Ubuntu容器，一旦停止即销毁
docker run --interactive --tty --rm ubuntu:22.04

# ping baidu.com会失败，因为没有ping命令
ping baidu.com -c 1 

# 安装ping
apt update
apt install iputils-ping --yes

ping baidu.com -c 1 # 这次则会成功
exit
```
但是如果再运行一个容器执行刚才的`ping`命令，则需要重复一遍刚才的依赖下载，所以可以这样**在当前镜像上再叠加一层镜像**（`RUN`也是一层）：
```bash
# Build a container image with ubuntu image as base and ping installed
docker build --tag my-ubuntu-image -<<EOF
FROM ubuntu:22.04
RUN apt update && apt install iputils-ping --yes
EOF

# 再运行该镜像构建容器，则会自带新版apt和ping工具
docker run -it --rm my-ubuntu-image
ping baidu.com -c 1
```
镜像应该是要将运行时依赖”固化“进镜像，如软件包、代码、配置模板这些；而**环境相关的配置应该用环境变量注入**，而不应该写进镜像里，**这样同一份镜像可以不加改动地在开发、测试、生产环境运行，只需注入不同的环境变量**。
![[Pasted image 20260321203411.png]]
* **查看命名卷下持久数据的物理位置**：通过挂载卷可以实现容器内部数据的持久化，例如执行下面代码可以验证：
```bash
docker volume create my-volume
docker run -it --rm --mount \
    source=my-volume,destination=/mydata/ \
    ubuntu:22.04
echo "Hello, linux." > /my-data/hello.txt
exit
# 新创建容器再从该目录查看
docker run -it --rm --mount \
    source=my-volume,destination=/mydata/ \
    ubuntu:22.04
cat my-data/hello.txt
exit
```
挂载目录中的文件应该能在容器被销毁后仍能保留：
![[Pasted image 20260321212154.png]]
可见挂载的目录必然存放在磁盘的某处。可以通过容器`justioncormack/nsenter1`进入Docker Desktop的Linux虚拟机：
```bash
docker run -it --rm --privileged --pid=host justincormack/nsenter1@sha256:5af0be5e42ebd55eea2c593e4622f810065c3f45bb805eaacf43f08f3d06ffd8

# 查找挂载目录/var/lib/docker/volumes
ls /var/lib/docker/volumes/my-volume/_data
cat /var/lib/docker/volumes/my-volume/_data/hello.txt 
# 可以看到"Hello, linux."
```
![[Pasted image 20260321212524.png]]
另外，对于数据库场景，绑定挂载可以实现让容器内的数据库服务使用本机的配置文件（如`postgresql.conf`、`redis.conf`等，使用`-c 'config_file=/etc/postgresql/postgresql.conf'`实现配置注入），从而无需重新构建镜像也可以调整数据库参数，只需更换配置文件即可让同一镜像服务于各种环境，还可以让配置文件也纳入Git管理，和代码一起演进。

# 二、镜像的构建
Dockerflie是一个文本文件，其中包含一系列指令（如 `FROM`、`RUN`、`COPY` 等），用于定义如何从基础镜像一步步构建出最终的应用镜像。每条指令都会在镜像中创建一个新层，这些层会被缓存和复用，从而加速后续构建。
在执行`docker build`命令时，需要指定一个**构建上下文（Build Context）**，里面包含所有的项目源代码，通常是一个本地目录的路径，或一个Git仓库URL：
```bash
docker build -t myapp:[版本号] .
```
这里`.`就是当前目录，它作为构建上下文被发送给Docker守护进程。对于不应该被构建的项目文件，比如本地安装了Node模块的话，应该把该模块忽略掉，以确保是docker提供依赖，而非源代码中自行提供，使用`.dockerignore`忽略不需要构建的文件
![[Pasted image 20260323180334.png]]
## 编写Dockerfile
### 迭代0：基础构建镜像
在Ubuntu基础镜像中使用`apt`从官方源安装`node.js`，非常低效且不专业，专业的Docker实践中几乎不会用这种方法：
```Dockerfile
FROM ubuntu

RUN sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list && \

    apt update && apt install nodejs npm -y

COPY . .

RUN npm install

CMD [ "npm", "run", "dev" ]
```
最终构建镜像消耗了十多分钟，且镜像也很大：
![[Pasted image 20260324040906.png]]
### 迭代1：下载官方镜像减少耗时
从Ubuntu上安装Node.js及其依赖（如gcc、make、python3等）会额外增加数百MB，最终镜像往往超过1GB。但是官方的Node镜像只有约200-300MB，只包含运行Node应用必须的组件：
```Dockerfile
FROM node

COPY . .
  
RUN npm install

CMD [ "npm", "run", "dev" ]
```
这里的镜像是1.71GB（因为没有指定版本，默认下载内置了大量工具的`latest`），但是耗时却少了很多。因为之前的做法（Ubuntu+apt）每次构建都需要拉取Ubuntu基础镜像、更新软件源列表、解析依赖、下载Node.js及其所有依赖包。而官方Node镜像已经包含了Node.js运行环境和npm，使用`FROM node`可以在拉取镜像时**只需一次性下载其预构建的层，而避免apt的长耗时安装**。
![[Pasted image 20260324043128.png]]
### 迭代2：选定Node镜像版本减小镜像大小
```Dockerfile
FROM node:19.6-alpine

COPY . .
  
RUN npm install

CMD [ "npm", "run", "dev" ]
```
最后镜像大小减小到了300MB左右，构建时间进一步减少到40秒：
![[Pasted image 20260324044727.png]]
### 迭代3：利用Docker层缓存机制加速构建
Docker 在构建镜像时，**每一条指令都会生成一个缓存层**（层缓存机制）。当再次构建时，**如果当前指令和上下文都没有变化，Docker 会直接复用之前的缓存层**，从而极大加速构建。
在之前的Dockerfile中，复制整个项目的源代码指令`COPY . .`在`RUN npm install`前面，**导致一旦任何源代码文件（哪怕是注释）发生变化，`COPY . .`这一层就会失效，导致其后的`RUN npm install`层也必须重新执行**。
所以可以先让依赖列表`package.json`和锁文件`package-lock.json`先复制，在执行完`RUN`之后再完整地拷贝，这样只有在依赖变更之后才会让缓存无效：
```Dockerfile
FROM node:19.6-alpine

COPY package*.json ./

RUN npm install

COPY . .

CMD [ "npm", "run", "dev" ]
```
这种技巧核心是：**把稳定、不易变化的依赖安装步骤尽量前置，把频繁变动的源代码复制步骤后置**，从而最大限度地复用构建缓存。这里利用了缓存，让构建时间缩短到20秒：
![[Pasted image 20260324050545.png]]
### 迭代4：指定工作目录
通过`WORKDIR`字段**为后续的指令设置工作目录**，它相当于让后续所有命令都`cd`到指定目录，可以在一个干净的目录下管理项目文件，且不用写绝对路径，几乎所有生产级Dockerfile都会用到。如果不添加它，那么`COPY . .`会把文件复制到镜像的**根目录**`/`，这会让根目录混乱，并且之后运行`npm install`或启动应用时，都需要使用绝对路径，如`/index.js`。
```Dockerfile
FROM node:19.6-alpine

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install

COPY . .

CMD [ "node", "index.js" ]
```
如果在生产环境下，直接运行`node`比通过`npm`启动更轻量，无需启动npm进程和额外脚本解析。所以让容器启动时执行的指令从`npm run dev`变成了`node index.js`
### 迭代5：容器安全
默认情况下，容器内进程以`root`用户运行，但是如果应用存在漏洞，攻击者可能利用root权限破坏宿主机，所以Docker官方建议：**生产环境中，永远不要以`root`用户运行应用**。使用 `USER node` 将容器启动后的用户切换为 **`node`**（该用户是官方 Node 镜像中预置的非特权用户），并将该项目的所有者设置为`node`用户和`node`组：
```Dockerfile
FROM node:19.6-alpine

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install

USER node

COPY --chown=node:node . .

CMD [ "node", "index.js" ]
```
### 迭代6：生产环境优化
`ENV NODE_ENV production`设置环境变量，许多Node.js库会检测该变量，从而启用生产模式（如禁用详细错误堆栈、启用缓存等）`npm ci`会默认读取该变量，对于生产环境它会自动跳过`devDependencies`，而只安装生产依赖（后面可以再加上`--only=production` 显式指定）
```Dockerfile
FROM node:19.6-alpine
  
WORKDIR /usr/src/app
  
ENV NODE_ENV production
  
COPY package*.json ./
  
RUN npm ci --only=production
  
USER node
  
COPY --chown=node:node . .

# 提示信息，并不会对容器行为有什么实际改变
EXPOSE 3000
  
CMD [ "node", "index.js" ]
```
### 迭代7：使用BuildKit缓存挂载
传统Docker构建时，每个 `RUN` 指令会生成一个只读层，如果该指令的命令及上下文没有变化，Docker 会直接复用之前的层，从而跳过该步骤（层缓存机制），对于`npm install`能够加快镜像构建速度。
但是`npm ci`每次执行**仍然会从网络上下载所有依赖包**，因为Docker缓存的是指令执行的结果，而不是网络请求的中间产物。所以即使 `RUN` 指令本身被缓存，但 `npm ci` 下载的依赖包仍然需要重新下载，只是被保存在这一层中。
而BuildKit是Docker的新一代构建引擎，提供了多种增强功能（从Docker 23.0+开始它就是Docker的默认构建引擎，无需额外配置。如果需要显示启用，可以设置环境变量`DOCKER_BUILDKIT=1 docker build -t myapp .`）。如`--mount=type=cache`允许在构建镜像时，**将宿主机的目录（或一个共享的卷）挂载到容器中**，且这个挂载点在多次构建之间**持久化**，基本用法为：
```Dockerfile
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production
```
因为`npm ci`**默认会将下载的包存储在`~/.npm`缓存目录中**，如果该目录被持久化（将缓存挂载到`target=/root/.npm`），那么下次构建时，npm会直接从本地缓存中读取，而无需重新从网络下载。如果需要更改缓存位置，也可以使用`npm set cache <path>`命令：
```Dockerfile
FROM node:19.6-alpine
  
WORKDIR /usr/src/app
  
# 使用新版环境变量设置格式
ENV NODE_ENV=production
  
COPY package*.json ./
  
RUN --mount=type=cache,target=/usr/src/app/.npm \
  npm set cache /usr/src/app/.npm && \
  npm ci --only=production
  
USER node
  
COPY --chown=node:node . .

# 提示信息，并不会对容器行为有什么实际改变
EXPOSE 3000
  
CMD [ "node", "index.js" ]
```
### 多阶段构建
多阶段构建可以实现在单个Docker文件中，多个独立阶段构建独立的容器镜像。**对于像Go这种需要被编译的语言非常有用**，因为在构建阶段，会使用大量的依赖包构建代码。但是最终编译结果是二进制文件，其所有依赖关系都可以静态链接，因此最终的镜像中可以不需要那些golang工具链**
```Dockerfile
FROM golang:1.19-buster AS build

WORKDIR /app

# 新加一个非根用户
RUN useradd -u 1001 nonroot

COPY go.mod go.sum ./

RUN --mount=type=cache,target=/go/pkg/mod \
  --mount=type=cache,target=/root/.cache/go-build \
  go mod download

COPY . .

RUN go build \
  -ldflags="-linkmode external -extldflags -static" \
  -tags netgo \
  -o api-golang

FROM scratch

ENV GIN_MODE=release

COPY --from=build /etc/passws /etc/passwd

COPY --from=build /app/api-golang api-golang

USER nonroot

EXPOSE 8080

CMD ["/api-golang"]
```
其中`scratch`是一个特殊的官方镜像，它除了一个有效的根文件系统外什么都没有，所以它就会被用来作为最后的镜像，最终只有16.8MB：
![[Pasted image 20260324064649.png]]
### 总结
Dockerfile编写准则：先构建成功，再考虑安全和性能
![[Pasted image 20260324070654.png]]![[Pasted image 20260324070856.png]]
基础镜像选择：
![[Pasted image 20260324071013.png]]
其他功能：
![[Pasted image 20260324071410.png]]
