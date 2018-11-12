# API 性能测试和调优

## API 性能测试指标
API 性能测试，大的方面包括 API 框架的性能和指定 API 的性能，因为指定 API 的性能跟该 API 具体的实现有关，比如有无数据库连接，
有无复杂的逻辑处理等，脱离了具体实现来探讨单个 API 的性能是毫无意义的，所以本小节只探讨 API 框架的性能。

衡量 API 性能的指标主要有 3 个：
1. 并发数（Concurrent）

并发数是指某个时间范围内，同时正在使用系统的用户个数。

广义上的并发数是指同时使用系统的用户个数，这些用户可能调用不同的 API。严格意义上的并发数是指同时请求同一个 API 的用户个数。本小节所讨论的并发数是严格意义上的并发数。

2. 每秒查询数（QPS）

每秒查询数 QPS 是对一个特定的查询服务器在规定时间内所处理流量多少的衡量标准。

QPS = 并发数 / 平均请求响应时间。

3. 请求响应时间（TTLB）

请求响应时间指的是从客户端发出请求到得到响应的整个时间。这个过程从客户端发起的一个请求开始，到客户端收到服务器端的响应结束。在一些工具中，请求响应时间通常会被
称为 TTLB（Time to last byte，意思是从发送一个请求开始，到客户端收到最后一个字节的响应为止所消费的时间）。请求响应时间的单位一般为"秒”或“毫秒”。

衡量 API 性能的最主要指标是 QPS，但是在说明 QPS 时，需要指明是多少并发数下的 QPS，否则毫无意义，因为不同并发数下的 QPS 是不同的。比如单用户 100 QPS 和 100 用户
 100 QPS 是两个不同的概念，前者说明 API 可以在一秒内串行执行 100 个请求，而后者说明在并发数为 100 的情况下，API 可以在一秒内处理 100 个请求。当 QPS 相同时，
并发数越大，说明 API 性能越好，并发处理能力越强。

在并发数设置过大时，API 同时要处理很多请求，会频繁切换进程，而真正用于处理请求的时间变少，使得 QPS 反而会降低。并发数设置过大时，请求响应时间也会变大。
API 会有一个合适的并发数，在该并发数下，API 的 QPS 可以达到最大，但该并发数不一定是最佳并发数，还要参考该并发数下的平均请求响应时间。


## API 性能测试方法
Linux 下有很多 Web 性能测试工具，常用的有 Jmeter、AB、Webbench 和 Wrk。每个工具都有自己的特点，本小节用 Wrk 来对 API 进行性能测试。Wrk 非常简单，安装方便，
测试结果也相对专业些，并且可以支持 Lua 脚本来创建更复杂的测试场景。

### Wrk 安装
安装步骤如下（需要切换到 root 用户）：

1. Clone wrk repo
```bash
git clone https://github.com/wg/wrk
```

2. 执行`make`和`make install`安装
```bash
make
cp ./wrk /usr/bin
```

### Wrk 使用简介
Wrk 使用起来不复杂，执行`wrk --help`可以看到 wrk 的所有运行参数：
```bash
$ wrk --help
Usage: wrk <options> <url>
  Options:
    -c, --connections <N>  Connections to keep open
    -d, --duration    <T>  Duration of test
    -t, --threads     <N>  Number of threads to use

    -s, --script      <S>  Load Lua script file
    -H, --header      <H>  Add header to request
        --latency          Print latency statistics
        --timeout     <T>  Socket/request timeout
    -v, --version          Print version details

  Numeric arguments may include a SI unit (1k, 1M, 1G)
  Time arguments may include a time unit (2s, 2m, 2h)
```

常用的参数为：

`-t`: 线程数（线程数不要太多，是核数的 2 到 4 倍即可，多了反而会因为线程切换过多造成效率降低）
`-c`: 并发数
`-d`: 测试的持续时间，默认为 10s
`-T`: 请求超时时间
`-H`: 指定请求的 HTTP Header，有些 API 需要传入一些 Header，可通过 Wrk 的 -H 参数来传入
`--latency`: 打印响应时间分布
`-s`: 指定 Lua 脚本，Lua 脚本可以实现更复杂的请求


### Wrk 结果解析
一个简单的测试如下：
```bash
$ wrk -t144 -c3000 -d30s -T30s --latency http://127.0.0.1:8080/sd/health
Running 30s test @ http://127.0.0.1:8088/sd/health
  144 threads and 3000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    32.01ms   39.32ms 488.62ms   87.93%
    Req/Sec     1.00k   251.79     3.35k    69.00%
  Latency Distribution
     50%   25.05ms
     75%   55.36ms
     90%   78.45ms
     99%  166.76ms
  4329733 requests in 30.10s, 1.81GB read
  Socket errors: connect 0, read 5, write 0, timeout 64
Requests/sec: 143850.26
Transfer/sec:     61.46MB
```

- `144 threads and 3000 connections`: 用 144 个线程模拟 3000 个连接，分别对应`-t`和`-c`参数
- `Thread Stats`： 线程统计
- `Latency`: 响应时间，有平均值、标准偏差、最大值、正负一个标准差占比
- `Req/Sec`: 每个线程每秒完成的请求数, 同样有平均值、标准偏差、最大值、正负一个标准差占比
- `Latency Distribution`: 响应时间分布
  - `50%`: 50% 的响应时间为：4.74ms
  - `75%`: 75% 的响应时间为：23.42ms
  - `90%`: 90% 的响应时间为：82.88ms
  - `99%`: 99% 的响应时间为：236.39ms
- `19373531 requests in 30.10s, 1.35GB read`: 30s 完成的总请求数（19373531）和数据读取量（1.35GB）
- `Socket errors`: connect 0, read 5, write 0, timeout 64: 错误统计
- `Requests/sec`: QPS
- `Transfer/sec`: TPS


本小节的性能测试脚本请参考最终源码目录下的 [wrktest.sh](https://github.com/lexkong/apiserver_demos/blob/master/demo17/wrktest.sh) 脚本。