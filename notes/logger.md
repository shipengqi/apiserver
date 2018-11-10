# 记录和管理 API 日志
本小节源码下载路径：[demo03](https://github.com/lexkong/apiserver_demos/tree/master/demo03)

apiserver 所采用的日志包[lexkong/log](https://github.com/lexkong/log)，特点：
- 支持日志输出流配置，可以输出到 stdout 或 file，也可以同时输出到 stdout 和 file
- 支持输出为 JSON 或 plaintext 格式
- 支持彩色输出
- 支持 log rotate 功能
- 高性能

## 初始化日志包
在`conf/config.yaml`中添加`log`配置：
```yaml
runmode: debug                 # 开发模式, debug, release, test
addr: :8080                  # HTTP绑定端口
name: apiserver              # API Server的名字
url: http://127.0.0.1:8080   # pingServer函数请求的API服务器的ip:port
max_ping_count: 10           # pingServer函数try的次数
log:
  writers: file,stdout
  logger_level: DEBUG
  logger_file: log/apiserver.log
  log_format_text: false
  rollingPolicy: size
  log_rotate_date: 1
  log_rotate_size: 1
  log_backup_count: 7
```

在`config/config.go`中添加日志初始化代码：
```go
package config

import (
    ....
    "github.com/lexkong/log"
    ....
)
....
func Init(cfg string) error {
    ....
    // 初始化配置文件
    if err := c.initConfig(); err != nil {
        return err
    }

    // 初始化日志包
    c.initLog()
    ....
}

func (c *Config) initConfig() error {
    ....
}

func (c *Config) initLog() {
    passLagerCfg := log.PassLagerCfg {
        Writers:        viper.GetString("log.writers"),
        LoggerLevel:    viper.GetString("log.logger_level"),
        LoggerFile:     viper.GetString("log.logger_file"),
        LogFormatText:  viper.GetBool("log.log_format_text"),
        RollingPolicy:  viper.GetString("log.rollingPolicy"),
        LogRotateDate:  viper.GetInt("log.log_rotate_date"),
        LogRotateSize:  viper.GetInt("log.log_rotate_size"),
        LogBackupCount: viper.GetInt("log.log_backup_count"),
    }

    log.InitWithConfig(&passLagerCfg)
}

// 监控配置文件变化并热加载程序
func (c *Config) watchConfig() {
    ....
}
```

这里要注意，日志初始化函数`c.initLog()`要放在配置初始化函数`c.initConfig()`之后，因为日志初始化函数要读取日志相关的配置。
`func (c *Config) initLog()`是日志初始化函数，会设置日志包的各项参数，参数为：
- `writers`：输出位置，有两个可选项`file`和`stdout`。选择`file`会将日志记录到`logger_file`指定的日志文件中，选择`stdout`会将日志输出到标准输出，
当然也可以两者同时选择
- `logger_level`：日志级别，DEBUG、INFO、WARN、ERROR、FATAL
- `logger_file`：日志文件
- `log_format_text`：日志的输出格式，JSON 或者 plaintext，`true`会输出成非 JSON 格式，`false`会输出成 JSON 格式
- `rollingPolicy`：rotate 依据，可选的有`daily`和`size`。如果选`daily`则根据天进行转存，如果是`size`则根据大小进行转存
- `log_rotate_date`：rotate 转存时间，配合`rollingPolicy: daily`使用
- `log_rotate_size`：rotate 转存大小，配合`rollingPolicy: size`使用
- `log_backup_count`：当日志文件达到转存标准时，log 系统会将该日志文件进行压缩备份，这里指定了备份文件的最大个数

用`lexkong/log`替换掉之前的 log 包。

## 管理日志文件
这里将日志转存策略设置为`size`，转存大小设置为`1 MB`。
```yaml
  rollingPolicy: size
  log_rotate_size: 1
```

并在`main`函数中加入测试代码：
```go
	// Set gin mode.
	gin.SetMode(viper.GetString("runmode"))

	for {
	    log.info("111111111111111111111111111111111111")
	    time.Sleep(100 * time.Millisecond)
	}
```
启动 apiserver 后发现，在当前目录下创建了`log/apiserver.log`日志文件：
```bash
$ ls log/
apiserver.log
```
程序运行一段时间后，发现又创建了 zip 文件：
```bash
$ ls log/
apiserver.log  apiserver.log.20180531134509631.zip
```
该 zip 文件就是当`apiserver.log`大小超过 1MB 后，日志系统将之前的日志压缩成 zip 文件后的文件。