# 一、Task
Task是一个用Go编写的**任务运行工具**，可以理解为Make的现代替代品。它的核心作用是**把项目中常用的复杂命令（比如“启动数据库”、"运行测试"、"构建镜像"）封装成简单、好记的任务名称**。
这样未来项目开源时，**Devbox负责环境一致、Task负责操作统一，两者结合实现“开箱即用”**。只需运行`task --list`就知道所有操作；只需要运行`task test`就自动拉起数据库、运行迁移、执行测试，无需记忆`docker run ...`的长命令，也不用担心忘记启动数据库而测试失败，因为可以事先设置`deps`让Task自动处理依赖。
![[Pasted image 20260317150017.png]]
# 二、Taskfile
Task的配置文件是`Taskfile`，它本质上是一个YAML文件，即`taskfile.yaml`。但是它在YAML之上定义了一套专用字段和结构，用来描述任务的名称、命令、依赖关系、变量等。
## 2.1 Taskfile创建
在安装好`go-task`工具后，在`bash`终端执行`task --init`即可在当前目录创建一个初始的Taskfile：
```yaml
version: '3'

vars:
  GREETING: Hello, World!

tasks:
  default:
    cmds:
      - echo "{{.GREETING}}"
    silent: true
```
执行`task default`或`task`（省略任务名自动运行default）即可执行命令`echo "{{.GREETING}}"`，把`vars`定义的变量打到控制台：
![[Pasted image 20260318043839.png]]
## 2.2 Taskfile核心语法
一个`Taskfile`文件的基本结构为：
```yaml
version: '3'           # Taskfile 版本号

vars:                  # 全局变量
  APP_NAME: myapp
  IMAGE_REPO: docker.io/username

tasks:
  task-name:           # 任务名称
    desc: 任务描述      # 会显示在 task --list 中
    deps:              # 依赖的其他任务
      - dep-task1
      - dep-task2
    cmds:              # 要执行的命令列表
      - echo "Start building"
      - go build -o {{.APP_NAME}} main.go
    status:
      - test -f app
    dir: ./subdir      # 指定命令执行目录
    silent: true       # 不打印命令本身，只打印输出
```
* `vars`：Task使用Go模板语法，使用`{{.VAR_NAME}}`引用变量
* `deps`：**如果任务依赖其他任务，那么依赖任务会先执行**。依赖关系既可以像YAML语法那样使用`-`表示（块样式），也可以用JSON列表表示方法（流样式），所以上面的`deps:`部分等价于`deps: [dep-task1, dep-task2]`
* `status`：**在执行任务前先检查“任务是否已经完成”**。如果`status`定义的命令都执行成功（返回退出码`0`），Task就会认为该任务已完成，从而跳过`cmds`中定义的主要命令。比如示例中表示执行命令前先检查当前目录下是否存在名为`app`的文件，只有找到了才会跳过。**它一般用于检查构建产物、检查Docker镜像是否存在、检查服务是否在执行、检查依赖项是否存在等生成静态产物的任务，而对于每次运行都要重新执行的任务（如启动服务）不应该使用它**。
  另外，`status`也支持列表。且通常会将输出重定向到`/dev/null`（如`pip show requests > /dev/null 2>&1`，把错误信息也重定向到标准输出），避免污染终端，只保留退出码。
## 2.3 命名空间与Include
**Task默认只读取当前目录下的`Taskfile.yaml`（不会自动向上查找或合并其他目录）**，所以在不同目录下执行`task --list`看到的任务列表不同，这让每个微服务可以不受其他服务干扰。
但是Task又提供了`includes`字段，能够将其他的Taskfile定义的任务包含，**一般在项目根目录中就会这样使用`include`，把微服务的所有任务都统一管理**，比如在项目根目录的Taskfile中有：
```yaml
includes:
  api-golang:
    taskfile: ./services/go/api-golang/Taskfile.yaml
    dir: ./services/go/api-golang

  api-node:
    taskfile: ./services/node/api-node/Taskfile.yaml
    dir: ./services/node/api-node

  client-react:
    taskfile: ./services/react/client-react/Taskfile.yaml
    dir: ./services/react/client-react

  load-generator-python:
    taskfile: ./services/python/load-generator-python/Taskfile.yaml
    dir: ./services/python/load-generator-python

  api-golang-migrator:
    taskfile: ./services/other/api-golang-migrator/Taskfile.yaml
    dir: ./services/other/api-golang-migrator
```
这样在项目根目录下执行`task --list`就能看到所有的任务，并且**同名任务能够通过命名空间隔离**，比如`api-golang:test`，`api-golang`就是自己定义的微服务名，`:`用于命名空间分组，`test`就是任务名。