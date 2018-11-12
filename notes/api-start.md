# 给 API 增加启动脚本
本小节源码下载路径：[demo13](https://github.com/lexkong/apiserver_demos/tree/master/demo13)

## 为什么要添加启动脚本
通过添加服务器启动脚本可以很方便地启动、重启、停止和查看服务的状态。一些监控系统、发布系统需要有方法告诉它怎么启停和查看服务状态，
这时候如果有个服务控制脚本就可以很方便地添加，要不然输入一堆启动参数不仅烦琐还容易出错。

## 添加启动脚本
apiserver 是通过`admin.sh`脚本来实现服务启动、重启、停止和查看服务状态操作的
（详见 [demo13/admin.sh](https://github.com/lexkong/apiserver_demos/tree/master/demo13/admin.sh)），源码为：
```bash
#!/bin/bash

SERVER="apiserver"
BASE_DIR=$PWD
INTERVAL=2

# 命令行参数，需要手动指定
ARGS=""

function start()
{
	if [ "`pgrep $SERVER -u $UID`" != "" ];then
		echo "$SERVER already running"
		exit 1
	fi

	nohup $BASE_DIR/$SERVER $ARGS  server &>/dev/null &

	echo "sleeping..." &&  sleep $INTERVAL

	# check status
	if [ "`pgrep $SERVER -u $UID`" == "" ];then
		echo "$SERVER start failed"
		exit 1
	fi
}

function status()
{
	if [ "`pgrep $SERVER -u $UID`" != "" ];then
		echo $SERVER is running
	else
		echo $SERVER is not running
	fi
}

function stop()
{
	if [ "`pgrep $SERVER -u $UID`" != "" ];then
		kill -9 `pgrep $SERVER -u $UID`
	fi

	echo "sleeping..." &&  sleep $INTERVAL

	if [ "`pgrep $SERVER -u $UID`" != "" ];then
		echo "$SERVER stop failed"
		exit 1
	fi
}

case "$1" in
	'start')
	start
	;;
	'stop')
	stop
	;;
	'status')
	status
	;;
	'restart')
	stop && start
	;;
	*)
	echo "usage: $0 {start|stop|restart|status}"
	exit 1
	;;
esac
```