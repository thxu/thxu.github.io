---
title: Ubuntu永久开放指定端口
date: 2020-04-19 15:00:00
tags:
	- 个人笔记
	- Ubuntu
	- 端口开放
categories:
	- 个人笔记
	- Ubuntu
---

### 1.查看已经开启的端口

```bash
sudo ufw status
```

### 2.打开3306端口

```bash
sudo ufw allow 3306
```

### 3.开启防火墙

```bash
sudo ufw enable
```

### 4.重启防火墙

```bash
sudo ufw reload
```

执行完上述命令后，就开放了3306端口了，而且重启服务器后也不影响。