# 一、基于平台的持续测试方案
CI/CD：持续集成/持续部署

持续测试：将新的代码、新的测试用例，一起执行测试
持续集成：将新的代码加入旧的代码，一起编译
持续部署：将编译结果部署生产环境

 ① 公有云DevOps平台：Gitlab Runner / Github Action
* 使用YAML编写流程脚本
* 主服务器在国外
* 公司也在国外，没有发票不好报销
② 自建CI/CD平台：BuildBot（Python写的） / Jenkins（Java写的，主干很简单，基本靠开源插件扩展）
* 灵活搭配，可以执行任意的代码、命令
* 完全私有，容易得到公司认可
* 开源软件，缺少配套的技术支持
③ 企业自研测试平台
* 灵活搭配，可以执行任意的代码、命令
* 完全私有，容易得到公司认可
* 自研软件，需要更强的技术支持

# 二、搭建Jenkins集成环境
## 1. 准备环境：
JDK 17+、Jenkins.war包
```cmd
java --version
java -jar jenkins.war
```
注：JAR和WAR都是Java的压缩包，其中：
* JAR（Java Archive）为通用存档格式，用于打包Java类库、工具或独立应用。如果打包为**可执行JAR**，内部`META-INF/MANIFEST.MF`文件中会**指定一个`Main-Class`入口**，告诉JVM启动时先执行这个类的`main`方法，所以可以用`java -jar xxx.jar`运行它。
* WAR（Web Application Archive）是专门为Web应用设计的打包格式，必须包含`WEB-INF/`文件夹。它的设计目标是**部署到外部的Servlet容器**（如Tomcat、Jetty）中，**由容器负责解析、加载并处理HTTP请求**。
**传统的WAR包本身没有`main`方法，不能独立运行**。但是Jenkins的`jenkins.war`虽然符合WAR的目录结构（包含`WEB-INF/`）但它的内部同时打包了一个嵌入式的Servlet容器，如下图可以看到jenkins运行的地方是**本机的**8080端口，证明了这个`.war`包非常规，而是内置了一个Servlet服务器。
![[Pasted image 20260308030205.png]]
当执行该命令时，Java就会读取WAR包的`META-INF/MANIFEST.MF`文件，该文件中指定了`Main-Class`，指向了Jenkins自己的启动器。这个启动器会：
1. 启动内嵌的 Jetty 容器。
2. 将 Jenkins 自身作为一个 Web 应用部署到这个内嵌容器上。
3. 开始监听 8080 端口。
![[Pasted image 20260308025635.png]]
访问`http://127.0.0.1:8080`：
![[Pasted image 20260308030710.png]]
## 2. 配置代理

![[Pasted image 20260308034449.png]]
### 3. 安装插件
安装以下必备的插件：
* git
* allure
### 4. 配置插件
* git：命令行可以正常使用即可
* allure：自动安装或手动配置安装目录
### 5. 验证插件
* git：能够获取github仓库内容，可以分辨**正确、错误**的git地址。基于这个特性，提供正确、错误两个地址即可。
![[Pasted image 20260308042749.png]]
* allure：**能够生成HTML报告**，但是在配置界面没有allure显示，因为插件有bug：
![[Pasted image 20260308043613.png|340]]
此时需要下载旧版本的allure插件再本地导入，下载地址为：https://mirrors.aliyun.com/jenkins/plugins/allure-jenkins-plugin/2.30.3/
导入后部署重启，下次创建新项目时就可以选择生成allure报告了
![[Pasted image 20260308044652.png]]
进入创建的项目，点击`Build Now`先完成自动化操作后，Allure就会生成报告：
![[Pasted image 20260308045554.png]]
![[Pasted image 20260308045610.png]]
以上表示git和allure插件都可以正常工作。
## 三、准备命令行测试框架

pytest测试框架，都支持命令行：
* unittest
* pytest
自行封装的测试框架，也要支持命令行


框架执行测试的命令：
```
sanmu run
```
jenkins通过该命令来实现对框架用例的执行，而jenkins可以实现**根据git仓库的内容变化**，自动地执行测试用例，并生成测试报告，实现项目的持续测试。此时就需要连接git仓库了：
![[Pasted image 20260308075527.png]]
指定仓库后，需要指定触发器：
![[Pasted image 20260308075752.png]]




# 测试进阶路线（参考）：
* 软件测试零基础到高薪系列课（功能、金融项目、Linux、数据库测试）
* 精通接口测试工具系列课（Jmeter，Postman，Fiddler，ApiFox，微服务架构Dubbo协议、基于Flask的Mock Server接口服务测试）
* Python全栈自动化系列课，零代码极限封装（零代码接口、WEB、APP自动化测试）
* Java全栈自动化进阶系列课
* 全栈性能测试专家系列课（Jmeter性能压测、Grafana+Prometheus监控、全方位性能瓶颈分析、调优、全链路性能压测、分析以及调优）【可深入】
* 全栈测试开发平台系列课（Python工程高阶编程、Pytest高阶插件Hook定制开发、Django4+Vue3前后端分离）
* 全栈安全测试渗透测试系列课（BurpSuite+Appscan安全工具、安全测试法规、体系以及常用工具、XSS、DVWA测试、CSRF攻击以及DOM漏洞攻击、SQL注入、SQLMap）
* 云原生（Cloud Native）系列课（git、github、gitee、gitlab版本控制、jenkins持续集成、pipeline流水线、容器化部署、容器编排、K8S集成技术）【可深入】
-------------------专业领域---------------
* 车载测试专项业务系列课（CANoe、CAPL、UDS、ADAS、HIL、OTA、仪表盘、座舱中控）
* 银行测试专项业务系列课（公共、贷款、信用、理财、网银）
* 大数据测试专项业务系列课（数据仓库、数据分层、数据优化）
* 跨平台自动化测试工具（HttpRunner、RF两大框架工具）






