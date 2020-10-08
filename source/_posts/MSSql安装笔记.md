---
title: MSSQL在Ubuntu上的安装笔记
date: 2020-09-14 23:00:00
tags: 
	- C#
	- MSSQL
	- Ubuntu
	- 个人笔记
categories:
	- 个人笔记
---

##### 以下文档均摘抄于微软官方文档(https://docs.microsoft.com/zh-cn/sql/linux)

## 1、Ubuntu使用普通方式安装

### 1.1 安装

1. 导入公共存储库GPG密钥

   ```bash
   sudo add-apt-repository "$(wget -qO- https://packages.microsoft.com/config/ubuntu/18.04/mssql-server-2019.list)"
   ```

   

2. 为 SQL Server 2019 注册 Microsoft SQL Server Ubuntu 存储库

   ```bash
   sudo apt-get update
   sudo apt-get install -y mssql-server
   ```

   

3. 运行以下命令以安装 SQL Serve

   ```bash
   sudo apt-get update
   sudo apt-get install -y mssql-server
   ```

   

4. 包安装完成后，运行 `mssql-conf setup`，按照提示设置 SA 密码并选择版本

   ```bash
   sudo /opt/mssql/bin/mssql-conf setup
   ```

   

5. 完成配置后，验证服务是否正在运行

   ```bash
   systemctl status mssql-server --no-pager
   ```

   

6. 如果计划远程连接，可能还需要在防火墙上打开 SQL Server TCP 端口（默认值为 1433）

### 1.2、管理SQL Service服务

1. 启动服务

   ```bash
   sudo systemctl start mssql-server
   ```

   

2. 关闭服务

   ```bash
   sudo systemctl stop mssql-server
   ```

   

3. 重启服务

   ```bash
   sudo systemctl restart mssql-server
   ```



***

## 2、Ubuntu使用Docker方式安装

1. 从 Microsoft 容器注册表中拉取 SQL Server 2019 Linux 容器映像

   ```bash
   sudo docker pull mcr.microsoft.com/mssql/server:2019-CU5-ubuntu-18.04
   ```

   

2. 要使用 Docker 运行容器映像，可以从 Bash Shell (Linux/macOS) 或提升的 PowerShell 命令提示符使用以下命令

   **注意：以下命令中的`<YourStrong@Passw0rd>` 即为设置的密码，包括`<` 和 `>` 。**

   ```bash
   sudo docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=<YourStrong@Passw0rd>" \
      -p 1433:1433 --name sql1 \
      -d mcr.microsoft.com/mssql/server:2019-CU5-ubuntu-18.04
   ```

   

3. 要查看 Docker 容器，请使用 `docker ps` 命令

   ```bash
   sudo docker ps -a
   ```

4. 更改SA账户密码

   **注意：以下命令中的`<YourNewStrong@Passw0rd>` 即为设置的密码，包括`<` 和 `>` 。**

   ```bash
   sudo docker exec -it sql1 /opt/mssql-tools/bin/sqlcmd \
      -S localhost -U SA -P "<YourStrong@Passw0rd>" \
      -Q 'ALTER LOGIN SA WITH PASSWORD="<YourNewStrong@Passw0rd>"'
   ```

   