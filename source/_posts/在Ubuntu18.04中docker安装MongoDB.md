---
title: 在Ubuntu18.04中使用docker安装MongoDB数据库
date: 2020-04-19 15:00:00
tags:
	- 个人笔记
	- Ubuntu
	- MongoDb
	- Docker
categories:
	- 个人笔记
	- MongoDb
---

### 1.获取MongoDB镜像

在终端中输入 

```
docker pull mongo
```

拉取最新的MongoDB镜像

当然我们也可以打开docker官方镜像站点 [https://hub.docker.com/_/mongo?tab=tags&page=1]( https://hub.docker.com/_/mongo?tab=tags&page=1) 自由选择所需的版本，比如在终端中输入 `docker pull mongo:4.2.5`  可下载4.2.5版本的镜像

### 2.运行容器

在终端中输入以下命令即可运行Mongo容器

```
docker run -itd --name mongo -p 27017:27017 mongo --auth
```

 `--auth` 参数表示需要密码才能访问容器，接下来我们开始设置MongoDB的用户及密码

### 3.MongoDB添加用户和设置密码

在终端中执行以下命令在docker容器中和mongo交互

```
docker exec -it mongo mongo admin
```

然后我们创建用户，账号为admin，密码为qwer1234

```
db.createUser({ user:'admin',pwd:'qwer1234',roles:[ { role:'userAdminAnyDatabase', db: 'admin'}]});
```

接下来启用认证

```
db.auth('admin', 'qwer1234')
```

### 4.连接MongoDB

我使用的是 `Robo 3T`  这个可视化客户端来连接MongoDB，这个软件有Windows版本和Mac版本，挺好用的，而且还是开源免费的。

* 打开 `Robo 3T` 后，点击 `File` -> `Connect`  在弹出的窗口中点击右键，选择 `Add` ，在 `Connection` 选项卡下面的 `Address` 处填写MongoDB的服务器IP地址
* 接下来点击 `Authentication` 选项卡填写认证信息，先勾选上 `Perform authentication` ，然后在 `User Name` 处填写账号，在 `Password` 处填写密码，在保存之前可以先点击一下左下角的 `Test` 测试下能否连接上MongoDB
* 点击 `save` 保存按钮后会自动回到连接列表，此时我们可以看到连接列表中已经有了刚刚添加好的连接，双击它就可以打开MongoDB了

