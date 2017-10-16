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
添加git的环境变量：
```shell
[root@localhost src]# vi /etc/profile
```
在最后加上如下一行代码：
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
[root@localhost src]# npm install
```







# Collider

A websocket-based signaling server in Go.

## Building

1. Install the Go tools and workspaces as documented at http://golang.org/doc/install and http://golang.org/doc/code.html

2. Checkout the `apprtc` repository

   ```
    git clone https://github.com/webrtc/apprtc.git
   ```

3. Make sure to set the $GOPATH according to the Go instructions in step 1

   E.g. `export GOPATH=$HOME/goWorkspace/`
   `mkdir $GOPATH/src`

4. Link the collider directories into `$GOPATH/src`

   ```
    ln -s `pwd`/apprtc/src/collider/collider $GOPATH/src
    ln -s `pwd`/apprtc/src/collider/collidermain $GOPATH/src
    ln -s `pwd`/apprtc/src/collider/collidertest $GOPATH/src
   ```

5. Install dependencies

   ```
    go get collidermain
   ```

6. Install `collidermain`

   ```
    go install collidermain
   ```

## Running

```
$GOPATH/bin/collidermain -port=8089 -tls=true
```

## Testing

```
go test collider
```

## Deployment

These instructions assume you are using Debian 7/8 and Go 1.6.3.

1. Change [roomSrv](https://github.com/webrtc/apprtc/blob/master/src/collider/collidermain/main.go#L16) to your AppRTC server instance e.g.

```go
var roomSrv = flag.String("room-server", "https://your.apprtc.server", "The origin of the room server")
```

1. Then repeat step 6 in the Building section.

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