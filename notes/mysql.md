# 使用 MySQL
## 安装 MySQL 并初始化表
安装 MySQL，不多说。

### 创建示例需要的数据库和表
1. 创建 db.sql，内容为：
```sql
/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

CREATE DATABASE /*!32312 IF NOT EXISTS*/ `db_apiserver` /*!40100 DEFAULT CHARACTER SET utf8 */;

USE `db_apiserver`;

--
-- Table structure for table `tb_users`
--

DROP TABLE IF EXISTS `tb_users`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `tb_users` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `username` varchar(255) NOT NULL,
  `password` varchar(255) NOT NULL,
  `createdAt` timestamp NULL DEFAULT NULL,
  `updatedAt` timestamp NULL DEFAULT NULL,
  `deletedAt` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `username` (`username`),
  KEY `idx_tb_users_deletedAt` (`deletedAt`)
) ENGINE=MyISAM AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `tb_users`
--

LOCK TABLES `tb_users` WRITE;
/*!40000 ALTER TABLE `tb_users` DISABLE KEYS */;
INSERT INTO `tb_users` VALUES (0,'admin','$2a$10$veGcArz47VGj7l9xN7g2iuT9TF21jLI1YGXarGzvARNdnt4inC9PG','2018-05-27 16:25:33','2018-05-27 16:25:33',NULL);
/*!40000 ALTER TABLE `tb_users` ENABLE KEYS */;
UNLOCK TABLES;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2018-05-28  0:25:41
```

2. 登录 MySQL:
```bash
$ mysql -uroot -p
```

3. `source db.sql`
```bash
mysql> source db.sql
```
可以看到，`db.sql`创建了`db_apiserver`数据库和`tb_users`表，并默认添加了一条记录（用户名：`admin`，密码：`admin`）：
```sql
mysql> select * from tb_users \G;
*************************** 1. row ***************************
       id: 0
 username: admin
 password: $2a$10$veGcArz47VGj7l9xN7g2iuT9TF21jLI1YGXarGzvARNdnt4inC9PG
createdAt: 2018-05-28 00:25:33
updatedAt: 2018-05-28 00:25:33
deletedAt: NULL
1 row in set (0.00 sec)
```

### 在配置文件中添加数据库配置
在配置文件`conf/config.yaml`中配置数据库的 IP、端口、用户名、密码和数据库名信息。
```yaml
log:
  writers: stdout
  logger_level: DEBUG
  logger_file: log/apiserver.log
  log_format_text: true
  rollingPolicy: size
  log_rotate_date: 1
  log_rotate_size: 1
  log_backup_count: 7
db:
  name: db_apiserver
  addr: 127.0.0.1:3306
  username: root
  password: root
docker_db:
  name: db_apiserver
  addr: 127.0.0.1:3306
  username: root
  password: root
```

## 初始化 MySQL 数据库并建立连接
apiserver 用的 ORM 是 GitHub 上 star 数最多的[`gorm`](https://github.com/jinzhu/gorm)，相较于其他 ORM，它用起来更方便，更稳定，社区也更活跃。

### 初始化数据库
#### 在`model/init.go`中添加数据初始化代码

一个 API 服务器可能需要同时访问多个数据库，为了对多个数据库进行初始化和连接管理，这里定义了一个叫`Database`的`struct`：
```go
type Database struct {
    Self   *gorm.DB
    Docker *gorm.DB
}
```
`Database`结构体有个`Init()`方法用来初始化连接：
```go
func (db *Database) Init() {
    DB = &Database {
        Self:   GetSelfDB(),
        Docker: GetDockerDB(),
    }
}
```

`Init()`函数会调用`GetSelfDB()`和`GetDockerDB()`方法来同时创建两个`Database`的数据库对象。这两个`Get`方法最终都会调
用`func openDB(username, password, addr, name string) *gorm.DB`方法来建立数据库连接，不同数据库实例传入不同的`username`、`password`、`addr`
和名字信息，从而建立不同的数据库连接。`openDB`函数为：
```go
func openDB(username, password, addr, name string) *gorm.DB {
    config := fmt.Sprintf("%s:%s@tcp(%s)/%s?charset=utf8&parseTime=%t&loc=%s",
        username,
        password,
        addr,
        name,
        true,
        //"Asia/Shanghai"),
        "Local")

    db, err := gorm.Open("mysql", config)
    if err != nil {
        log.Errorf(err, "Database connection failed. Database name: %s", name)
    }

    // set for db connection
    setupDB(db)

    return db
}
```
可以看到，`openDB()`最终调用`gorm.Open()`来建立一个数据库连接。[`model/init.go`](https://github.com/lexkong/apiserver_demos/tree/master/demo04)。

#### 主函数中增加数据库初始化入口
```go
package main

import (
    ...
    "apiserver/model"

    ...
)

...

func main() {
    ...

    // init db
    model.DB.Init()
    defer model.DB.Close()

    ...
}
```
通过`model.DB.Init()`来建立数据库连接，通过`defer model.DB.Close()`来关闭数据库连接。
