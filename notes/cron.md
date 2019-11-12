# 定时任务
定时任务是很常见的，可以使用 [cron](https://github.com/robfig/cron) 来实现定时任务。

## 安装
```sh
# 安装 v3 版本
go get github.com/robfig/cron/v3@v3.0.0
```
导入：
```go
import "github.com/robfig/cron/v3"
```

## 格式
Cron 和 Crontab 的表达格式几乎一样（如 `0 0 0 1 1 *`），但是多了一个秒级：

字段名   | 是否必填 | 允许的值  | 允许的特殊字符
----------   | ---------- | --------------  | --------------------------
秒（Seconds）      | Yes        | 0-59            | * / , -
分（Minutes）      | Yes        | 0-59            | * / , -
时（Hours）        | Yes        | 0-23            | * / , -
一个月中的某天（Day of month） | Yes        | 1-31            | * / , - ?
月（Month）        | Yes        | 1-12 or JAN-DEC | * / , -
星期几（Day of week）  | Yes        | 0-6 or SUN-SAT  | * / , - ?

### 特殊字符
- `*`：代表所有可能的值，例如 `month` 字段如果是星号，则表示在满足其它字段的制约条件后每月都执行该命令操作。
- `,`：可以用逗号隔开的值指定一个列表范围，例如，`1,2,5,7,8`
- `-`：可以用整数之间的中杠表示一个整数范围，如 `1-5` 表示 `1,2,3,4,5`
- `/`：可以用正斜线指定时间的间隔频率，例如 `0-23/2` 表示每两小时执行一次。同时正斜线可以和星号一起使用，如 `*/10`，如果
用在 `minute` 字段，表示每十分钟执行一次。
- `?` 不指定值，用于代替 `*`，类似 `_`

### 预定义的 Cron 时间表
输入                  | 简述                                | 相当于
-----                  | -----------                                | -------------
@yearly (or @annually) | 1月1日午夜运行一次      | 0 0 0 1 1 *
@monthly               | 每个月的午夜，每个月的第一个月运行一次 | 0 0 0 1 * *
@weekly                | 每周一次，周日午夜运行一次       | 0 0 0 * * 0
@daily (or @midnight)  | 每天午夜运行一次                  | 0 0 0 * * *
@hourly                | 每小时运行一次        | 0 0 * * * *

## 使用
```go
type conf struct {
    UpdateUserInfoCron       string
}

func main() {
    // 创建一个 Cron job runner
    c := cron.New()
    // 向 Cron job runner 添加一个 func ，以按给定的时间表运行
    // 这里的时间表格式如果有问题，会直接 err
    c.AddFunc("* * * * * *", func() {
        // Do something
    })
    // 启动 Cron 调度程序
    c.Start()

    // 阻塞主程序
    t1 := time.NewTimer(time.Second * 10)
    for {
        select {
        case <-t1.C:
            t1.Reset(time.Second * 10)
        }
    }
}
```
官方的示例：
```go
c := cron.New()
c.AddFunc("30 * * * *", func() { fmt.Println("Every hour on the half hour") })
c.AddFunc("30 3-6,20-23 * * *", func() { fmt.Println(".. in the range 3-6am, 8-11pm") })
c.AddFunc("CRON_TZ=Asia/Tokyo 30 04 * * * *", func() { fmt.Println("Runs at 04:30 Tokyo time every day") })
c.AddFunc("@hourly",      func() { fmt.Println("Every hour, starting an hour from now") })
c.AddFunc("@every 1h30m", func() { fmt.Println("Every hour thirty, starting an hour thirty from now") })
c.Start()

// 也可以给一个正在运行的 Cron 添加 Func
c.AddFunc("@daily", func() { fmt.Println("Every day") })

// 停止 Cron，但是已经执行的 job 不会停止
c.Stop() 
```

## crontab
`/etc/crontab` 文件是系统任务的配置文件：
```
SHELL=/bin/bash  # 指定系统要使用的shell 这里是bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin # 指定系统执行命令的路径
MAILTO="" # 指定 crond 的任务执行信息将通过电子邮件发送给 root 用户，为空则不发送
HOME=/ # 指定在执行命令或者脚本时使用的主目录。

0 1 * * * root /user/local/run.sh
```

在用户的 `crontab` 文件中，每一行就是一个任务，怎么定义用户任务，以上面的 `crontab` 文件中的任务
`0 1 * * * root /user/local/run.sh` 为例， 一行任务分为六段：

```bash
minute   hour   day   month   week   command
```

- `minute`： 分钟，从 `0` 到 `59` 之间的任何整数。
- `hour`：小时，从 `0` 到 `23` 之间的任何整数。
- `day`：日期，从 `1` 到 `31` 之间的任何整数。
- `month`：月份，从 `1` 到 `12` 之间的任何整数。
- `week`: 星期几，从 `0` 到 `7` 之间的任何整数，`0` 或 `7` 代表星期日。
- `command`: 行的命令，可以是系统命令，也可以是脚本文件。

各段中还可以使用的特殊字符：`*`，`*`，`-`，`/`。

**Example**
```bash
* * * * * command
```
每分钟执行一次`command`。


```bash
10,20 * * * * command
```
每小时的第`10`和第`20`分钟执行一次。


```bash
10,20 8-11 */2 * * command
```
每隔两天的上午`8`点到`11`点的第`10`和第`20`分钟执行。


```bash
10 1 * * 6,0 /etc/init.d/smb restart
```
每周六、周日的一点十分重启smb