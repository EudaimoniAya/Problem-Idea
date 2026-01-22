# a. 理论学习
## 视频1：养活国内大半自研数据库团队？PostgreSQL是什么？架构是怎么样的？
https://www.bilibili.com/video/BV1CkCQBoEyp/?spm_id_from=333.337.search-card.all.click&vd_source=b0a6d2b11972b5dadba8ad8550b99d00
与MySQL快速易用的开发理念不同，PostgreSQL理念是希望打造出功能最丰富，最符合SQL标准的数据库。凭借其迭代速度，高可用性、高扩展性和自定义能力，在近几年的Stack Overflow调查中逐渐超越了MySQL，登顶了最受开发者欢迎的数据库。
考虑一个场景：在传统纯MySQL存储业务数据的场景下，随着业务需求增长，因为MySQL对地理计算支持很弱，所以引入了Redis的Geohash；要支持搜索商品，又因为MySQL的全文搜索功能太弱，所以引入了ElasticSearch（ES）；当需要进行跨平台商品比价，不同电商平台的数据字段各不相同，MySQL表结构太死板，于是将它们组织为JSON文档，引入了更灵活的MongoDB读写数据。原本简单的单体架构，就变成了MySQL+MongoDB+Redis+ES的复杂分布式系统，每个中间件都要运维监控，及其繁琐且易出错。
而PostgreSQL能够原生支持地理位置、JSON、数组、文本等复杂数据类型，在某些场景下可以完全替代MySQL、MongoDB等，且完全开源可商用。

类似于MySQL，PostgreSQL将数据组织成excel表的形式，每一行数据叫做tuple，每个表中有主键。在磁盘上，PostgreSQL的数据表在磁盘上是一个以数字命名且没有特定后缀的文件，称为堆表文件。其中的数字代表数据表在数据库中的ID，又称为relfileNode。随着堆表文件大到一定限度，就会自动分裂出小堆表文件。直接读写一个DB数据表的数据会很慢，所以MySQL和Pg会将数据表拆成一个个数据页，而MySQL使用B+树来进行查询，Pg则使用B-Link树查询（非子节点也同样有指向右兄弟的指针，方便叶分裂时直接跳到右节点查数据，无需返回父节点再到右节点，**是一种变种的B+树**）。而B-Link树只是一种其中的一种索引，如GIN进行全文搜索、GIST进行地理位置搜索、BRIN进行时序数据搜索等。

Pg通过提供多个后端进程（称为postgres）来为客户端应用提供数据读写服务，而这些后端进程又由一个主控进程（称为postmaster）来管理，主控进程监听客户端连接请求，有新连接需求就创建，并时刻监控各个后端进程的健康状态。另外，Pg为了避免每次从磁盘读数据页，所以给每个后端进程分配了缓存，但为了避免数据页在各进程中存在副本，所以为所有的后端进程建立了一个共享的内存区域，后端进程在之后每次查询时，先查找共享缓冲区，如果没有才去磁盘找。此时数据页都在共享内存或磁盘中，不在后端进程里，所以后端进程需要一个数据访问层（Access Methods）从共享内存或磁盘中获取数据。这里面和共享内存对接的模块叫Buffer Manager，和磁盘对接的模块叫做Storage Manager。

> [!问题]
> **Q：** 这里的数据访问层，是原先带有缓存功能的后端进程为了避免数据页副本，而把缓存独立出来变为共享内存缓冲区之后，因为此时后端进程不再具有提取缓存功能而引入的。但其实它本来就应该存在一部分，在**抽象前后端进程有四个功能：负责与客户端对接、负责缓存、找不到缓存后去查磁盘、以及优化器执行器等中间件**，它本身就是具有查询磁盘的功能；而在抽象后，把缓存的责任独立出来，并将查共享内存的操作与查磁盘合并为数据访问层，此时**后端进程就只有三个功能了：负责与客户端对接、调用数据服务层、以及优化器执行器等中间件**。
> 这样的**先功能解耦**（缓存的独立），再**将独立的功能分层**（设置为共享内存），**确定依赖关系**（后端进程存取数据需要依赖于共享内存），最后**实现对接服务**（新增访问共享内存的进程，并与访问磁盘的进程合并为数据访问层）的思想。似乎在模型&仓储层中的优化中也有出现：原先ORM模型与数据库打交道，然后在原地配上增删改查很方便。但是为了方便维护，把模型层和仓储层解耦，模型定义提取出来，设立为模型层，操作模型的变为仓储层，它在模型层之上。而因为操纵ORM模型的仓储层功能被解耦，所以它要导入模型层的模型来实现功能。
> **A：** 这是一种“单一职责原则（SRP）”和“依赖倒置原则（DIP）”的体现。**每一层都建立在下一层提供的稳定抽象之上，使系统能够应对极其复杂的需求，每一层都只做一件事，并依赖于抽象（接口），而非具体实现**。

当进程多了以及缓存的引入，则需要考虑数据一致性的问题。为了避免内存数据意外丢失造成数据错误，Pg使用WAL（Write-Ahead Logging）日志机制，将事务中更新数据行的操作先写入到共享内存中的WAL缓冲区，再写进共享缓冲区中存在的数据页。事务提交时，将缓冲区内容刷入到磁盘的WAL文件中，这样崩溃重启后，通过WAL文件就能够重做数据，WAL文件的写入由后台进程（WAL Writer）完成。（这种使用日志记录操作的方法Redis、ES、MySQL等都有使用，方便服务重启后进行数据对账）在保证了数据一致性后，再定期由另外的后台进程（Checkpointer）负责将共享内存中缓冲区的数据页写入到磁盘，再配合平时连续异步写入脏页的后台进程（Background Writer）减少磁盘的IO压力。

客户端查询数据流程：客户端应用先与主控进程连接（无论读写），主控进程为该连接生成并分配一个专属的后端进程。然后用户将SQL语句发送到后端进程的解析器模块，解析器模块负责检查语法是否正确。之后由优化器选择应该使用的索引方式，生成执行计划转交给执行器去执行调用数据访问层函数的操作（查询共享内存和磁盘，或写入共享内存）完成数据读写。另外，为了支持扩展，优化器、执行器、数据访问层都被设计为开发接口。


## 视频2：为什么哪里都有Docker的身影？你真的了解它背后的容器技术吗？
https://www.bilibili.com/video/BV13GmDY9EtV/?spm_id_from=333.788.videopod.sections&vd_source=b0a6d2b11972b5dadba8ad8550b99d00&p=2
### 传统程序部署方式——存在依赖冲突
容器技术用于解决程序部署运行方面的问题，在容器出现之前，工业界直接将程序部署到物理服务器上运行，在这种情况下，主要会使用systemd类的守护类程序，把多个业务程序部署并管理起来，这样可以很方便地查看每个业务程序的运行情况。但这种方法**只管理了业务程序本身，而没有管理业务程序的依赖**。每个业务程序运行，必须得预先安装所有待部署程序的依赖。在多台机器规模化的情况下，往往通过下发部署脚本来实现，然而业务程序的依赖往往是复杂且易变的，同一程序的多个版本的依赖很可能不同，有时经常出现反复升降级、移除和新增依赖的情况，这使得单个业务程序依赖管理和安装非常繁琐复杂，另外还需考虑多个业务程序同时在一台机器上混合部署的情况，此时多个程序间的依赖可能还有冲突。
### 解决依赖冲突——虚拟机技术
程序的依赖和管理很复杂，因为程序的运行总是天生地和依赖、环境相耦合，如果把程序依赖和程序本身完全一体化，作为一个独立单元来管理就解决了依赖冲突问题。最简单的实现方式就是每个版本的程序独占一台机器，这台机器的依赖环境完全为这个版本的程序服务，需要安装新版本程序则需要将机器重置，然后安装此版本的程序所需要的依赖环境，而使用当时已经成熟的虚拟机技术可以让程序的管理能够满足上述的模式。**这样的本质就是把依赖环境和程序绑定，并打包到了一个虚拟机镜像中，然后以虚拟机作为程序管理部署的基本单位。**

但是虚拟机的部署较为臃肿和笨重：①镜像庞大，存储、传输、运行成本高；②启停速度慢，程序的部署效率无法做到很高；③虚拟化后的程序运行性能损耗和overhead大，机器性能被浪费。所以即使虚拟机部署能够解决依赖冲突的问题，但带来了一些新的问题，即程序部署方面仍存在很多优化空间。
### 容器技术

1. **解决镜像过大问题：** 虚拟机镜像过大是因为上面存储了OS运行所需要的所有文件，而对于同一系列的OS的不同版本、分支来说，往往存在大量相同的文件（动态链接库、内核、常用的工具程序等）。所以**像OS中的共享内存一样**，对于镜像中相同的文件只保存一份，为所有容器共享。当要修改时，就单独拷贝一份文件归属给发起修改的容器即可（Copy-on-Write思想）。**即通过“共享+按需复制”的智能机制，在保证隔离性的同时，极大地优化了资源使用效率。**
    
2. **解决启停速度问题：** 虚拟机启停速度慢是因为它把整个OS的启动流程都包含进去了，而把OS的启停过程去掉就能优化流程的冗余。而为了避免破坏OS隔离导致的虚拟性，容器自己实现了一个“轻量化的虚拟机”，把OS相关的启停开销给去掉，并且实现程序隔离。要实现一个程序的隔离，需要在：文件系统、进/线程管理、进/线程通信、网络、用户和权限以及硬件资源使用等方面进行隔离，那么它就能够作为一个单独的隔离空间（如**Linux上的cgroup和Namespaces技术**，**cgroups用来限制和隔离进程的资源使用**，可为每个容器设定CPU、内存、网络带宽等资源的使用上限，确保了一个容器的资源消耗不会影响宿主机或其他运行中的容器。**Namespaces用来隔离进程的资源视图，使得容器只能看到自己内部的进程ID、网络资源和文件目录，而看不到宿主机的**。），也就称作容器，这样只需启动程序进程本身，那么其启停速度就会比虚拟机快很多。**即提取出程序运行隔离的关键部分，进行按需隔离。且所有容器共享一个宿主机的Linux内核，这样容器就是一个被特殊配置过的进程（进程即容器），使得容器在保持足够隔离性的同时，获得了接近原生进程的性能和启动速度。**
    
3. **解决性能损耗问题：** 从软件角度说，虚拟机本质是一个完整的OS，多个OS运行在同一机器上，大量的重复冗余造成对各类硬件资源的使用损耗。而容器的进程隔离则解决了这一问题，因为**本质上所有容器内的进程都是直接共用内核**的。从硬件角度说，即硬件虚拟化的损耗，随着虚拟机技术发展成熟通用硬件层面（CPU、MEM）等的损耗基本上降得很低，但特殊硬件（GPU、各类IO设备等）的损耗仍然存在，但随着技术成熟，这一点也在不断地改进。

这三个问题的解决标志着容器技术的正式成熟。而对于基于Linux容器化技术的Docker，它就是将上述解决方案整合到了一起，并提供了一个用户友好的使用界面的软件系统。Docker依赖容器基础技术、联合文件系统和轻量化隔离能力，并在此之上制定和设计了容器镜像和运行的标准，提供了一套完整易用的工具，使得程序及其依赖绑定，作为独立单元来管理。**即软件产物标准化交付——把软件运行相关的所有东西，包括程序本身和程序依赖，打包为一个标准集装箱（镜像）作为软件部署运行的交付单元，从根本上全面变革了软件部署和分发的模式。**

虚拟机镜像 VS 容器镜像：**“镜像”是一种静态的概念，是一个模板，可以把镜像类比成软件安装包，而虚拟机/容器可以看成是安装好的软件**。所以两者**同样是一个静态的、不变的、对程序和其相关环境的快照**，但是虚拟机镜像需要保存整个的环境（甚至包括OS），而容器镜像需要保存的只是单个独立运行单元即可（**对于公共应用则需要宿主OS提供**）。所以虚拟机镜像就像哈尔的移动城堡，把整个生态系统包括起来，提供了绝对的控制权和隔离性；而容器镜像就像飞屋环游记，仅转移独立维持运作的单间，至于必要的水源食物则需就地取用（宿主OS提供的必备功能），即带着所有必要的依赖，可以在任何提供基础资源（内核）的“土地”上立即运行。

## 视频3：40分钟的Docker实战攻略，一期视频精通Docker
https://www.bilibili.com/video/BV1THKyzBER6/?spm_id_from=333.788.videopod.sections&vd_source=b0a6d2b11972b5dadba8ad8550b99d00
### 一、Docker容器创建与删除
#### 1. `docker pull`
`docker pull`命令**用来从仓库下载镜像**，**如`docker pull docker.io/library/nginx:latest`**，可以看到一个镜像有四部分内容：
* `docker.io`：第一部分为registry，称为仓库地址或注册表，这里表示**Docker仓库地址/注册表**，这里的`docker.io`表示这是Docker Hub的官方仓库，而对于官方仓库，可以省略仓库地址。
* `library`：第二部分是**命名空间（作者名）**，因为Docker Hub是公共仓库，每个人都可以上传自己的镜像，而`library`是Docker官方仓库的命名空间，这个空间下所有镜像都由官方管理。如果一个镜像属于官方的命名空间（即开发者为官方）那么也可以省略。
* `nginx`：第三部分为**镜像名**。
* `:latest`：是**Docker镜像的标签名（版本号）**，可以指定下载一个特定的版本，如`:latest`表示最新版本（不写标签名默认为最新），或`:1.28.0`表示特定版本。
所以示例中的镜像`docker pull docker.io/library/nginx:latest`可以简写为：`docker pull nginx`，它表示从Docker官方仓库的官方命名空间里面，下载最新版的Nginx Docker镜像。再比如`docker pull docker.n8n.io/n8nio/n8n`，表示从n8n的私有仓库`docker.n8n.io`里面，下载作者名为`n8nio`的名为n8n的镜像。
所以**镜像指定的前三个部分表示了一个镜像的不同版本，它称为`repository`镜像库**。比如Docker Hub是一个registry，在上面搜索Nginx后返回的结果就是一个repository，里面有同一镜像的不同版本。
另外`dcoker pull --platform=xxxxxx nginx`，表示拉取特定CPU架构的镜像（大部分情况不需要关注，因为Docker会自动选择最适合宿主机CPU架构的镜像进行安装）
##### 其他镜像相关命令
* `docker images`可以**列出所有下载过的Docker镜像**。
```bash
eudaimoniaya@XXX:/mnt/c/Users/31084$ sudo docker images
[sudo] password for eudaimoniaya:
															 i Info →   U  In Use
e
IMAGE                             ID             DISK USAGE   CONTENT SIZE   EXTRA
ankane/pgvector:latest            956744bd14e9        628MB          160MB    U
docker.n8n.io/n8nio/n8n:latest    0a65e6e5995c       1.68GB          278MB    U
docker/welcome-to-docker:latest   c4d56c24da4f       22.2MB         6.03MB
nginx:latest                      553f64aecdc3        225MB         59.8MB
```
* `docker rmi <imagename>`用于**删除镜像**。
#### 2. `docker run`
`docker run`命令能够**使用镜像创建并运行容器**，就像使用一个模具制作一个糕点。如果本地不存在镜像，就会自动拉取一份并创建容器，因此**可以直接使用`docker run`，而不用事先使用`docker pull`**。
##### `docker run`的重要参数
* `-d`：表示**分离模式（detached mode），让容器在后台执行，不会阻塞当前窗口**（`docker run`命令会导致命令行的持续占用，**通常都要加上这个参数**）。运行后会返回一个容器ID，后续的容器日志不会打印在控制台。
* `-p`：表示**端口映射**，**每个Docker容器都运行在一个独立的虚拟环境里面，容器的网络与宿主机是隔离的**。所以默认状况下，并不能直接从宿主机访问到Docker的内部网络，所以需要将内外网络的端口进行映射：**冒号前是宿主机端口，冒号后是容器内端口**。如`-p 80:80`就是将宿主机的80端口信息转发到容器内的80端口进行处理。
* `-v`：表示**目录映射**，v（volume）表示挂载卷。它**将宿主机和容器的目录联系到了一起，实现了容器数据持久化保存**，因为当删除容器时，容器内的数据会被同时删除，如果使用了挂载卷，容器内对应目录的数据就会保存在宿主机的对应目录里面，这样删除容器后保证了数据不被删除。
  在使用-v时，宿主机的目录会（暂时）覆盖掉容器内的目录，所以如果宿主机目录是空的，那么执行命令`docker run -d -p 80:80 -v /website/html:/usr/share/nginx/html nginx`后访问`localhost:80`会出现403 Forbidden错误。这种挂载方式叫做**绑定挂载（-v 宿主机目录:容器内目录）**。
  如`docker run -d -p 80:80 -v /website/html:/usr/share/nginx/html nginx`，其中容器的目录`/usr/share/nginx/html`一般是Nginx存放静态网页的目录，另外进行了端口映射，访问宿主机的80端口相当于访问容器的80端口。
  而另一种挂载则是让Docker创建一个存储空间（命名卷），这种方式叫做**命名卷挂载（-v 卷的名字:容器内目录）**，具体命令为：
```bash
# 命名卷挂载格式
docker volume create <卷名>
# 列出所有创建过的卷
docker volume list
# 删除命名卷格式
docker volume rm <卷名>
# 删除所有没有任何容器在使用的卷
docker volume prune -a


# 示例
docker volume create nginx_html
docker run -d -p 81:80 -v nginx_html:/usr/share/nginx/html nginx

# 查看命名卷挂载的卷信息
docker volume inspect nginx_html
```
   
> [!NOTE]
> **实践中遇到的错误：把/usr错输为/user。**
> 
> **容器目录的`/usr`是指Unix System Resources（Unix系统资源）**，在Linux系统中，`/user`这个目录通常不存在。当发现结果不对时，可以检查挂载是否正确，具体检查方法是进入容器内部的目录，检查文件列表中是否存在宿主机目录下的文件。
> 
> 在使用`-v /宿主机路径:/容器路径时`，若容器路径不存在，则Docker会自动创建该目录。在错误输入`-v ...:/user/share/nginx/html`时，Docker在容器中自动创建了该目录，并把宿主机目录挂载到了新创建的目录，所以能够在容器内看到文件正确挂载。但是为什么即使输入错误的容器目录，最后还是能出现nginx网页呢？因为在容器内部使用的是包含默认网页的`/usr/share/nginx/html`，而Nginx的工作流程为：
> 
> 1. Nginx启动时读取配置文件：`/etc/nginx/conf.d/default.conf`
> 2. 配置中明确指定：`root/usr/share/nginx/html`
> 3. Nginx只认这个配置的路径，完全不知道`/user`目录的存在
> 4. 所以网页请求都被指向`/usr/share/nginx/html`下的文件
> 
> 所以导致了即使挂载的容器目录出错，但是对于网页请求，Nginx是按照配置文件中规定的去执行操作（启动自己的默认页面）。所以挂载错了地方，导致容器自己`/usr`目录下的网页没被覆盖能正常显示。

* `-e`：用于**往容器里面传递环境变量**，例如使用MongoDB容器，而使用MongoDB服务时一般需要设置账号名和密码。所以可以使用`-e`向里面传递账号密码：
```bash
sudo docker run -d \
-p 27017:2017 \
-e MONGO_INITDB_ROOT_USERNAME=aya \
-e MONGO_INITDB_ROOT_PASSWORD=111 \
mongo
```
* `--name <container_name>`：用于**给容器起一个自定义的名字**（它在宿主机上必须是唯一的），它与容器ID等价。
* `-it`：用于**让控制台进入容器进行交互**。
* `--rm`：用于设置**当容器停止时就将它删除掉**。
* `--restart`：用来**配置容器在停止时的重启策略**，如`--restart always`表示只要容器停止就立即重启；`--restart unless-stopped`表示除了手动停止的容器，只要容器停止就立即重启。
* `--network`：用于**设置容器所在的子网**。
##### 其他容器相关命令
* `docker ps`用于**查看所有运行中的容器**，`ps`即 process status。该指令只能看到运行中的容器，如果需要查看所有容器，应该**添加参数`-a`表示查看全部**（all）。
* `docker rm <imageID>`用于**删除容器**，但无法删除正在运行的。如果要强制删除正在运行的容器，应该**添加参数`-f`表示强制删除**（force）。
### 二、Docker容器调试

`docker run`命令时从镜像创建并且运行容器。这意味着每次执行`docker run`都会创建一个全新的容器，如果不想创建新容器，只想对容器进行启停，则可以使用`start`和`stop`命令。
```bash
 docker start <容器ID>/<容器名>  
 docker stop <容器ID>/<容器名>
```
通过`start`开启的容器，各种变量和参数都保留，不用再填一遍。另外如果忘记容器启动时填的参数是什么，可以使用`inspect`命令，输出结果无需深究，交给AI分析即可。
```bash
 # 查看容器信息  
 docker inspect <容器ID>/<容器名>  
 # 查看容器日志（如果加参数-f，表示--follow，即追踪输出，滚动查看日志）  
 docker logs <容器ID>/<容器名>
```
`docker create`命令与`docker run`命令非常类似，唯一一点区别就是，`create`命令只创建容器但不立即启动，如果需要启动，还需要输入`start`命令。

Docker容器技术基于Linux原生的两大技术：Namespaces和cgroups，基于这两个技术使得容器本质上还是一个特殊的进程，不过当进入容器内部后，容器内部看起来就像一个独立的OS，每个Docker容器都是一个独立的运行环境，每一个容器都表现得像一个独立的Linux系统。`exec`命令可以在容器的内部执行Linux命令。
```bash
 # 格式  
 docker exec <容器ID> linux命令  
 ​  
 # 示例1:显示容器内部的进程情况  
 docker exec c29afbe41ce1 ps -ef  
 # 示例2:进入一个正在运行的Docker容器内部，获得一个交互式命令行环境  
 docker exec -it c29afbe41ce1 /bin/sh
```
### 三、构建和推送镜像
容器是用镜像这个模具生产出来的糕点，镜像是制作糕点的模具，而Dockerfile就是制作模具的图纸，它是一个列出了如何构建镜像的文件。
`requirements.txt`
```text
fastapi
uvicon
```
`main.py`
```python
from fastapi import
import uvicorn
app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```
如果只需在本地运行以程序，则只需要三步：① 给电脑安装python环境；② 安装python依赖，使用命令`pip install -r requirements.txt`；③ 启动项目，使用命令`python main.py`。然后浏览器访问localhost:8000就可以看得到页面上的{"Hello": "World"}。接下来把这个程序打包成镜像：

Dockerfile（D要大写，且Dockerfile文件无后缀名）
```dockerfile
# Dockerfile文件第一行都是FROM，选择一个基础镜像，即正在打包的镜像是从哪个镜像基础上构建的。这里选用的是Linux发行的内置python3.13的镜像
FROM python:3.13-slim

# WORKDIR类似于cd，切换到镜像内的一个目录，作为工作目录，之后的命令都是在这个工作目录下面执行的
WORKDIR /app

# COPY将代码文件拷贝到镜像内的工作目录，后面的两个点分别表示：本电脑的当前目录、镜像内当前的工作目录（即WORKDIR的/app）
COPY . .

# 安装依赖，RUN表示命令需要在镜像内部执行
RUN pip install -r requirements.txt

# 声明镜像提供服务的端口，该字段非必要，只是给其他使用镜像的人一个提示。即使不写也可以，实际使用时还是以-p参数为准
EXPOSE 8000

# CMD指容器启动时的默认执行命令（最好写成数组形式，中间不要空格），也就是每当容器启动时，容器内部会自动执行这个命令，一个Dockerfile文件中只能有一个CMD。对于该例，这使得容器内有一个python程序在运行。
# 与CMD类似的是ENTRYPOINT，具有更高的优先级，因此不容易被覆盖
CMD ["python3"， "main.py"]
```
准备好Dockerfile文件后，就可以在当前目录的终端构建镜像：
```bash
# -t后接自己取的名字，后面的.表示在当前文件夹内构建
docker build -t docker_test .
```
构建好这个镜像之后，同样可以像其他镜像一样使用run命令，创造基于该镜像的容器后运行。而且还可以推送到Docker Hub上面：docker login => 填写验证码 => 登录成功 => 加上用户名后再构建镜像（必须在镜像前加上用户名）=> push命令推送到Docker Hub。
```bash
docker build -t username/docker_test
docker push username/docker_test
```
### 四、Docker网络
Docker网络默认是bridge（桥接模式），所有的容器默认都连接到这个网络，每个容器都分配了一个内部IP地址（一般是`172.17`开头的），**在这个内部子网中，容器可以通过内部IP地址相互访问，但容器网络与宿主机网络是隔离的**。可以使用`docker network create`命令创建子网，并指定容器加入不同子网，使得**同子网的容器可通信**（且借助于Docker内部的DNS机制，可以**不用IP地址，直接用容器名访问**），跨子网的不可通信。
另一种常见的网络模式为host模式，Docker容器直接共享宿主机的网络，容器直接使用宿主机的IP地址，且无需`-p`参数进行端口映射。最后一种网络模式是none模式，即不联网。
另外可以使用`docker network list`列出所有的Docker网络，使用`docker network rm`删除自定义Docker网络。
### 五、Docker Compose
有些时候一个完整应用可能是很多部分组成的，比如前端、后端、数据库等，如果把它们一起打包成一个容器，那么只要一个模块出现了故障整个容器可能会崩溃。并且可伸缩性差，如果想要给系统扩容，只能把整个大容器再复制一份，而做不到针对某个模块的精准性扩容。
对于这种多应用场景的最佳实践是把每一个模块都打包成一个独立的容器，但是如果多次使用`docker run`命令再把子网关系配置好又很麻烦且易出错。此时可以使用Docker Compose容器编排技术，它使用`.yml`文件管理多个容器，里面列出了容器之间是如何创建以及如何协同工作的。
对于一个场景：避免MongoDB数据库直接暴露，而是使用mongo-express提供交互服务，利用子网可以实现这一点。让mongo容器不进行端口映射，让mongo-express容器包含一个提供服务的IP地址/容器名，再将两个容器放进同一个子网。所以分别需要执行两个`docker run`命令：
```bash
docker network create network1

docker run -d \
--name my_mongodb \
-e MONGO_INITDB_ROOT_USERNAME=name \
-e MONGO_INITDB_ROOT_PASSWORD=pass \
-v /my/datadir:/data/db \
--network natwork1 \
mongo

docker run -d \
--name my_mongodb_express \
-p 8081:8081 \
-e ME_CONFIG_MONGODB_SERVER=my_mongodb \
-e ME_CONFIG_MONGODB_ADMINUSERNAME=name \
-e ME_CONFIG_MONGODB_ADMINPASSWORD=pass \
--network natwork1 \
mongo-express
```
对应的`docker-compose.yml`文件：
```yaml
%% services是最顶级的配置项，用于定义和配置需要运行的所有容器服务 %%
services:
	my_mongodb:
		image: mongo
		environment:
			MONGO_INITDB_ROOT_USERNAME=name
			MONGO_INITDB_ROOT_PASSWORD=pass
		volumes:
			- /my/datadir:/data/db
			  
	my_mongodb_express:
		images: mongo-express
		ports:
			- 8081:8081
		environment:
			ME_CONFIG_MONGODB_SERVER=my_mongodb
			ME_CONFIG_MONGODB_ADMINUSERNAME=name
			ME_CONFIG_MONGODB_ADMINPASSWORD=pass
		depends_on:
			- my_mongodb
```
这里无需像上一种方法那样自定义子网`network1`，因为Docker会为每一个compose文件自动创建一个子网，**同一文件中定义的所有容器都会自动加入同一个子网**。
另外这种方法还可以自定义容器的启动顺序，比如可以在`my_mongodb_express:`下面加入`depends_on:`字段来规定依赖。

# b. 使用Docker完成`pgvector`扩展
使用Docker完成`pgvector`扩展意味着需要把PostgreSQL服务放在Docker容器中执行，此时可以使用官方提供的一个包含`pgvector`扩展的镜像来创建并运行一个容器：
```bash
docker run -d \
  --name postgres-pgvector \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=mydatabase \
  -p 55432:5432 \
  ankane/pgvector:latest
```
此时就会生成一个名字为`postgres-pgvector`的容器，它占用宿主机的55432端口（为了和本地PostgreSQL端口区别开）。此时在项目中需要在`operations/config.py`的类`DatabaseSettings`中添加`product_vector_db`相关字段，其中端口应该为55432。所以它和使用5432端口的本地PostgreSQL（存储agent上下文）不相关。所以它在pgAdmin4中可以对应两个不同的服务器：`PostgreSQL_pgvector`和`PostgreSQL_local`，里面分别有商品向量模型实例和agent上下文记录的数据库。

> [!问题]
> **Q：** 在pgAdmin4中，服务器能够仅通过IP地址和端口注册，但它是什么，有什么作用？在Navicat中，我可以很简单地为上面这两个数据库创建不同的连接，为什么Navicat更倾向于提供具体数据库管理的服务？在pgAdmin4中能够在某个服务器中创建多个数据库，比如在`PostgreSQL_pgvector`中新建了`test_db`，那么在pgAdmin4中创建的数据库不在容器的定义内，就像编外人员一样。这种在可视化界面新增的数据库，在这种一切使用文件规范的开发环境下，有什么意义吗？
> **A：** pgAdmin4中的“服务器”是指**PostgreSQL实例（或集群），并不是指一个具体的数据库**。它相当于一栋公寓楼的管理系统；它管理的实例是整栋楼及设施；而数据库则是里面的套房；模式是套房里的房间；表是房间中的家具。它**只需要IP地址和端口就能注册，所以它对应的应该是一个PostgreSQL服务进程的远程连接入口（远程控制台）**，它可以为每一个服务提供方（多个容器端口不同）列出它所有的数据库。**Navicat为了让开发者快速操作，将这个概念隐藏起来**，倾向于提供高效的具体数据操作工具，而PostgreSQL则适合DBA，显示了所有数据库的关系。
> 在pgAdmin4这种可视化工具中创建表，与在用户操作界面创建表是等价的，本质都是数据库进程提供的服务。这些表都同一视为“用户创建”，并非编外人员，它们可以很灵活地**在开发测试时随时创建销毁**。

选择容器对应的`PostgreSQL_pgvector`服务器，找到`mydatabase`数据库，选择`mydatabase` -> 右键`扩展` -> `创建 扩展` -> 名称选择`vector` -> 保存。这样就使用Docker完成了PostgreSQL的`pgvector`扩展。
