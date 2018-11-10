# 使用 MySQL
## 安装 MySQL 并初始化表
### 安装 MySQL
apiserver 用的是 MySQL，所以首先要确保服务器上安装有 MySQL，执行如下命令来检查是否安装了 MySQL（CentOS 7 上是`mariadb-server`，CentOS 6
上是`mysql-server`，这里以 CentOS 7 为例）：
```bash
$ rpm -q mariadb-server
```

如果提示`package mariadb-server is not installed`则说明没有安装 MySQL，需要手动安装。如果出现`mariadb-server-xxx.xxx.xx.el7.x86_64`则说明已经安装。