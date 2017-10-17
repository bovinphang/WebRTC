# CentOS 7 下安装WebRTC 

网上可以找到一堆WebRTC demo，在code.google.com上也能找到不少WebRTC应用项目的源码。有些demo是直接调用WebRTC API开发的，但大多数是调用上述两种WebRTC封装库开发的。由于WebRTC API的名称在不同浏览器及同一浏览器的不同版本之间存在差异，所以不是所有demo都能运行在所有浏览器上。Apprtc是一个使用WebRTC技术实现的一个在线聊天室，可以支持两个人进行视频通话，是一个不错的WebRTC demo。

运行AppRTC需要使用Google App Engine SDK for Python，因为这个项目的Web服务器是使用Google App Engine框架编写的，因此必须要安装。

同时还需要使用Grunt构建工具。

下面介绍如何在Centos 7上搭建AppRTC项目。

## 一、AppRTC的服务器组成

1. AppRTC 房间服务器  https://github.com/webrtc/apprtc
2. Collider - 信令服务器  上面 Github 工程里自带，在 src/collider 下
3. coTurn - 打洞(内网穿透)服务器   https://github.com/coturn/coturn
4. 需要自己实现coTurn连接信息接口，主要返回用户名、密码和turn配置信息，通常叫做TURN REST API,不实现这个接口的话AppRTCDemo连不上服务器，浏览器访问的话可以正常访问。

## 二、准备工作

1. AppRTC 依赖 [Google App Engine SDK for Python](https://cloud.google.com/appengine/downloads#Google_App_Engine_SDK_for_Python)离线版本,[Node.js](https://nodejs.org) 和 [Grunt](http://gruntjs.com/)。
   GAE SDK 安装很简单：下载、解压、添加到PATH环境变量即可完成。（谷歌已经关闭新应用的申请，不过好像不影响使用）
   grunt，是 Node.js 下的一套任务执行系统，经过 Gruntfile.js 的配置，可以做很多事情。首先安装 Node.js。使用 nvm 可以很方便的为自己的 Linux 账户安装并设置好 Node.js。（而后，你可以选择安装 cnpm，这样就可以使用国内的缓存节点，比npm install 命令会快许多，如果你只用这一次 grunt，那么不装这个也是可以的。）接下来只需要执行一个 npm install -g grunt-cli 即可安装好 grunt。
2. Collider 依赖 golang。尿性的问题来了：墙……安装 golang 和日后使用 golang 所需的包，几乎都要翻墙。所以这里解决掉墙这个问题后，再使用 gvm 安装 golang 即可完成我们的准备工作。

## 三、安装Google Engine SDK for Python

官网地址：https://cloud.google.com/appengine/downloads#Google_App_Engine_SDK_for_Python

Google Engine SDK for Python 使用的是Python 2.7。因此在安装Google Engine SDK for Python之前，我们必须安装Python 2.7。

##### 安装Python 2.7 （因CentOS 7 已预装 Python 2.7.5，可略过此步骤）

1. 安装Python 2.7.12:

  ```shell
   [root@localhost src]# wget wget https://www.python.org/ftp/python/2.7.12/Python-2.7.12.tgz
   [root@localhost src]# tar -zxvf Python-2.7.12.tgz
   [root@localhost src]# cd Python-2.7.12
   [root@localhost Python-2.7.12]# ./configure
   [root@localhost Python-2.7.12]# make
   [root@localhost Python-2.7.12]# make install
  ```

2. 配置Python 2.7:

  建立软连接，使系统默认的 python指向 python2.7：

  ```shell
  [root@localhost Python-2.7.12]# mv /usr/bin/python /usr/bin/python2.6.6
  [root@localhost Python-2.7.12]# ln -s /usr/local/bin/python2.7 /usr/bin/python
  ```

3. 系统 Python 软链接指向 Python2.7 版本后，yum不能正常工作，我们需要指定 yum 的Python版本

   ```shell
    [root@localhost Python-2.7.12]# vi /usr/bin/yum
   ```

   将文件头部的：
   ```shell
   #!/usr/bin/python
   ```
   改成：
   ```shell
   #!/usr/bin/python2.6.6
   ```
##### 安装Google Engine SDK for Python

1. 下载Google App Engine SDK for Python源码:
   ```shell
   [root@localhost src]# wget https://storage.googleapis.com/appengine-sdks/featured/google_appengine_1.9.40.zip
   ```

2. 解压:
   ```shell
   [root@localhost src]# yum install -y unzip zip #若未装unzip和zip
   [root@localhost src]# unzip google_appengine_1.9.40.zip -d /usr/local/
   ```

3. 将解压后的目录添加到环境变量中:

   ```shell
   [root@localhost src]# vi /etc/profile
   ```

   将其打开把:

   ```shell
   export PATH=$PATH:/usr/local/google_appengine
   ```
   这句放在“export PATH USER LOGNAME MAIL HOSTNAME HISTSIZE INPUTRC”的上一行。

   要立即生效环境配置，不需要重启，用下命令：
   ```shell
   [root@localhost src]# . /etc/profile
   ```
   配置成功之后我们就可以在命令行中使用dev_appserver.py了

## 四、安装NodeJS

```shell
[root@localhost src]# wget https://nodejs.org/dist/v8.7.0/node-v8.7.0.tar.gz
[root@localhost src]# tar -zxvf node-v8.7.0.tar.gz
[root@localhost src]# cd node-v8.7.0
[root@localhost node-v8.7.0]# ./configure
[root@localhost node-v8.7.0]# make
[root@localhost node-v8.7.0]# make install
[root@localhost node-v8.7.0]# ln -fs /usr/local/bin/node /sbin/node #让所有用户都可用node
[root@localhost node-v8.7.0]# node -v #测试node是否安装成功
```
  或使用yum安装：
   ```shell
[root@localhost src]# yum install -y nodejs
   ```

## 五、安装Grunt

##### 安装CNPM

在中国大陆的话，因为网络原因，我们可能需要使用cnpm：

```shell
[root@localhost src]# npm install -g cnpm --registry=https://registry.npm.taobao.org
```

##### 安装Grunt
```shell
[root@localhost src]# npm -g install grunt-cli
```
如果在中国大陆不能正常安装的话，尝试使用cnpm安装，即：
```shell
[root@localhost src]# cnpm -g install grunt-cli
```
## 六、Open-JDK
Apprtc这个项目还需要JAVA环境，因此我们还需要配置一下Java环境。这里我使用的是Open-JDK

官网:http://openjdk.java.net/

1. 安装Open-JDK：

```shell
[root@localhost src]# yum install java-1.8.0-openjdk
```

2. 配置环境变量:

Redhat公司提供的OpenJDK环境变量配置参考文章:<https://access.redhat.com/documentation/en-US/JBoss_Enterprise_Application_Platform/6/html/Installation_Guide/Install_OpenJDK_on_Red_Hat_Enterprise_Linux.html>

```shell
[root@localhost src]# vi ~/.bashrc
```

将其打开把:

```shell
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.144-0.b01.el7_4.x86_64/jre
```

这句放在最后一行。

要立即生效环境配置，不需要重启，用下命令：

```shell
[root@localhost src]# source  ~/.bashrc
```

## 七、安装AppRTC
##### 安装git（若已安装，可略过此步骤）

```shell
[root@localhost src]# yum install git
[root@localhost src]# git --version  #测试git是否安装成功
```
或通过编译安装：
```shell
[root@localhost src]# wget https://git-core.googlecode.com/files/git-1.9.0.tar.gz
[root@localhost src]# tar -zxvf git-1.9.0.tar.gz
[root@localhost src]# cd git-1.9.0
[root@localhost git-1.9.0]# ./configure --prefix=/usr/local/git
[root@localhost git-1.9.0]# make
[root@localhost git-1.9.0]# make install
[root@localhost git-1.9.0]# git --version  #测试git是否安装成功
```
添加git到环境变量：
```shell
[root@localhost src]# vi /etc/profile
```
将其打开把：
```shell
export PATH=$PATH:/usr/local/git/bin:/usr/local/git/libexec/git-core
```
这句放在“export PATH USER LOGNAME MAIL HOSTNAME HISTSIZE INPUTRC”的上一行。
（因为bin目录只有4个命令，其它的几十个命令在libexec/git-core目录下，所以，在PATH搜索路径下，也要加上才能找到）
立即生效环境配置，不需要重启，用下命令：
```shell
[root@localhost src]# . /etc/profile
```
##### 获取 AppRTC 源码并安装项目依赖：
```shell
[root@localhost src]# git clone https://github.com/webrtc/apprtc.git
[root@localhost src]# cd apprtc
[root@localhost apprtc]# npm install
```

##### 构建Apprtc项目：

```shell
[root@localhost apprtc]# grunt build
```

常见错误：requests模块不存在

```shell
Running "shell:buildAppEnginePackage" (shell) task
Traceback (most recent call last):
  File "./build/build_app_engine_package.py", line 12, in <module>
    import requests
ImportError: No module named requests
Warning: Command failed: python ./build/build_app_engine_package.py src out/app_engine
Traceback (most recent call last):
  File "./build/build_app_engine_package.py", line 12, in <module>
    import requests
ImportError: No module named requests
 Use --force to continue.

Aborted due to warnings.
```

解决方法：通过python的包管理工具pip来安装requests

##### 安装pip

pip类似RedHat里面的yum，安装Python包非常方便。

1. 下载并安装setuptools：
```shell
[root@localhost src]# wget --no-check-certificate https://bootstrap.pypa.io/ez_setup.py
[root@localhost src]# python ez_setup.py --insecure
```
2. 安装pip:
  到[python官网](https://pypi.python.org/pypi/pip)下载pip安装包安装

```shell
[root@localhost src]# wget https://pypi.python.org/packages/11/b6/abcb525026a4be042b486df43905d6893fb04f05aac21c32c638e939e447/pip-9.0.1.tar.gz#md5=35f01da33009719497f01a4ba69d63c9
[root@localhost src]# tar -zxvf pip-9.0.1.tar.gz
[root@localhost src]# cd pip-9.0.1
[root@localhost pip-9.0.1]# sudo python setup.py install
[root@localhost pip-9.0.1]# pip -V #测试pip是否安装成功
```

##### 安装requests模块

```shell
[root@localhost src]# pip install requests
```

##### 再次构建Apprtc项目：

```shell
[root@localhost src]# cd apprtc
[root@localhost apprtc]# grunt build
```

##### 运行Apprtc

```shell
[root@localhost src]# dev_appserver.py /usr/local/src/apprtc/src/app_engine
```

dev_appserver.py 是/usr/local/google_appengine目录下的文件，已经配置在环境变量中。

如果要外网访问，加上host和端口，如：dev_appserver.py --host 121.40.28.178 --port 80 --admin_host 121.40.28.178  /usr/local/src/apprtc/src/app_engine

参照上面的配置，在浏览器中打开http://121.40.28.178即可访问 。

注：需检查80、8080和8000端口是否已开启，若未开启，则开启之：

```shell
[root@localhost src]# firewall-cmd --zone=public --add-port=80/tcp --permanent
[root@localhost src]# firewall-cmd --zone=public --add-port=8080/tcp --permanent
[root@localhost src]# firewall-cmd --zone=public --add-port=8000/tcp --permanent
[root@localhost src]# firewall-cmd --reload   #重启防火墙，使开启的端口生效
[root@localhost src]# firewall-cmd --permanent --query-port=80/tcp  #查询是否已开启的80端口
```

## 八、安装Collider

Collider是Google Chrome WebRTC项目里提供的用GO语言编写的基于WebSocket的信令服务器，也是Apprtc这个项目配套的一个信令服务器。在我们的Apprtc项目中就已经携带了它的源码。在安装collider之前，我们必须先安装Golang。

##### 安装Golang

go标准包安装：

```
[root@localhost src]# wget https://studygolang.com/dl/golang/go1.9.1.linux-amd64.tar.gz
[root@localhost src]# tar -zxvf go1.9.1.linux-amd64.tar.gz -C /usr/local
[root@localhost src]# mkdir -p /usr/local/gopath
```

添加go到环境变量：

```shell
[root@localhost src]# vi /etc/profile
```

将其打开把：

```shell
export GOROOT=/usr/local/go
export GOPATH=/usr/local/gopath
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```

这句放在“export PATH USER LOGNAME MAIL HOSTNAME HISTSIZE INPUTRC”的上一行。
立即生效环境配置，不需要重启，用下命令：

```shell
[root@localhost src]# . /etc/profile
[root@localhost src]# go version #测试go是否安装成功
```

注：GOPATH环境变量用来指定你的工作目录。当我们在开发Golang项目的时候需要指定

> The GOPATH environment variable specifies the location of your workspace. It is likely the only environment variable you'll need to set when developing Go code.
> GOPATH环境变量用来指定除GOROOT之外包含的Go项目源代码和二进制文件的目录。go install和go工具都会用到GOPATH,作为编译后二进制文件的存放目的地和import包时的搜索路径。

GOPATH是一个路径列表，也就是可以同时指定多个目录。多个目录在Mac和Linux下通过”:”分割；Windows下通过”;”分割。注意，大部分情况下会是第一个路径优先。

$GOPATH 目录约定有三个子目录：

- src 存放源代码（比如：.go .c .h .s等）
- pkg 编译后生成的文件（比如：.a）
- bin 编译后生成的可执行文件（为了方便，可以把此目录加入到 $PATH 变量中，如果有多个gopath，那么使用${GOPATH//://bin:}/bin添加所有的bin目录）

或yum安装：

```shell
[root@localhost src]# yum remove golang
[root@localhost src]# yum -y install golang
[root@localhost src]# go version #测试go是否安装成功
```

##### 构建

1. Checkout  `apprtc` 库

   ```shell
   [root@localhost src]# git clone https://github.com/webrtc/apprtc.git
   ```

2. 创建 $GOPATH 的src目录

   E.g. 

   ```shell
   [root@localhost src]# export GOPATH=/usr/local/gopath
   [root@localhost src]# mkdir $GOPATH/src
   ```

3. 将collider的源码目录链接到 `$GOPATH/src`

   ```shell
   [root@localhost src]# ln -s /usr/local/src/apprtc/src/collider/collider $GOPATH/src
   [root@localhost src]# ln -s /usr/local/src/apprtc/src/collider/collidermain $GOPATH/src
   [root@localhost src]# ln -s /usr/local/src/apprtc/src/collider/collidertest $GOPATH/src
   ```

4. 安装collidermain项目依赖

   ```shell
   [root@localhost src]# go get -v collidermain
   ```
   报错：
   ```shell
   Fetching https://golang.org/x/net/websocket?go-get=1
   https fetch failed: Get https://golang.org/x/net/websocket?go-get=1: dial tcp 216.239.37.1:443: i/o timeout
   package golang.org/x/net/websocket: unrecognized import path "golang.org/x/net/websocket" (https fetch: Get https://golang.org/x/net/websocket?go-get=1: dial tcp 216.239.37.1:443: i/o timeout)
   ```
   原因：因为golang.org被墙，国内使用 go get 安装 golang 官方包会失败。

   不翻墙的情况下怎么解决这个问题？其实 golang 在 github 上建立了一个[镜像库](https://github.com/golang)，如 <https://github.com/golang/net> 即是 <https://golang.org/x/net> 的镜像库。

   获取 golang.org/x/net 包，其实只需要以下步骤：

   ```shell
   [root@localhost src]# mkdir -p $GOPATH/src/golang.org/x
   [root@localhost src]# cd $GOPATH/src/golang.org/x
   [root@localhost src]# git clone https://github.com/golang/net.git
   [root@localhost src]# go install net
   ```

   安装其它依赖类似。

   如果上面命令仍失败,那么则可用下面这个麻烦的方法安装GO环境的websocket包：

   ```shell
   [root@localhost src]# cd $GOPATH/src
   [root@localhost src]# wget http://www.golangtc.com/static/download/packages/golang.org.x.net.tar.gz
   [root@localhost src]# tar -zxvf golang.org.x.net.tar.gz
   [root@localhost src]# go install golang.org/x/net/websocket/
   ```

   这样collidermain的项目依赖就安装完了。

5. 安装 `collidermain`

   ```shell
   [root@localhost src]# go install collidermain
   [root@localhost src]# collidermain --help #测试collidermain是否安装成功
   ```

##### 运行（启动colider）

```shell
[root@localhost src]# $GOPATH/bin/collidermain -port=8089 -tls=true
```

##### 测试

```shell
[root@localhost src]# go test collider
```

##### 部署

1. 修改房间服务器IP。即修改 [roomSrv](https://github.com/webrtc/apprtc/blob/master/src/collider/collidermain/main.go#L16) 为 你的apprtc服务器实例， e.g.

```shell
[root@localhost src]# vi /usr/local/src/apprtc/src/collider/collidermain/main.go
```

找开修改下面一行

```go
var roomSrv = flag.String("room-server", "https://192.168.9.223", "The origin of the room server")
```
2. 然后重复上面构建部分的步骤5。

### Install Collider

1. Login on the machine that is going to run Collider.
2. Create a Collider directory, this guide assumes it's created in the root (`/collider`).
3. Create a certificate directory, this guide assumes it's created in the root (`/cert`).
4. Copy `$GOPATH/bin/collidermain ` from your development machine to the `/collider` directory on your Collider machine.

### Certificates

If you are deploying this in production, you should use certificates so that you can use secure websockets. Place the `cert.pem` and `key.pem` files in `/cert/`. E.g. `/cert/cert.pem` and `/cert/key.pem`

### Auto restart

1\. Add a `/collider/start.sh` file:

```bash
#!/bin/sh -
/collider/collidermain 2>> /collider/collider.log
```

2\. Make it executable by running `chmod 744 start.sh`.

#### If using inittab otherwise jump to step 5:

3\. Add the following line to `/etc/inittab` to allow automatic restart of the Collider process (make sure to either add `coll` as an user or replace it below with the user that should run collider):

```bash
coll:2:respawn:/collider/start.sh
```

4\. Run `init q` to apply the inittab change without rebooting.

#### If using systemd:

5\. Create a service by doing `sudo nano /lib/systemd/system/collider.service` and adding the following:

```
[Unit]
Description=AppRTC signalling server (Collider)
 
[Service]
ExecStart=/collider/start.sh
StandardOutput=null
 
[Install]
WantedBy=multi-user.target
Alias=collider.service
```

6\. Enable the service: `sudo systemctl enable collider.service`

7\. Verify it's up and running: `sudo systemctl status collider.service`

#### Rotating Logs

To enable rotation of the `/collider/collider.log` file add the following contents to the `/etc/logrotate.d/collider` file:

```
/collider/collider.log {
    daily
    compress
    copytruncate
    dateext
    missingok
    notifempty
    rotate 10
    sharedscripts
}
```

The log is rotated daily and removed after 10 days. Archived logs are in `/collider`.