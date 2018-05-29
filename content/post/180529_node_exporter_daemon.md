---
title: "Prometheusのnode_exporterをdaemon化するスクリプト"
description: "Prometheusのnode_exporterをdaemon化するスクリプト"
date: 2018-05-29T22:33:23+09:00
categories:
  - 小ネタ
tags:
  - Prometheus
draft: false
---

Prometheusのnode_exporter(に限った話ではないが)のデーモン化について。
Systemdが使える環境ならリポジトリ内にサンプル設定が記載されているのだが、Amazon Linux 1環境などSystemdが使えない環境でdaemon化する方法はなかなか見つからなかったので組んだ。

```
#!/bin/bash
#
#   /etc/rc.d/init.d/node_exporter
#
# chkconfig: 2345 70 30
#
# pidfile: /var/run/node_exporter.pid

# Source function library.
. /etc/init.d/functions


RETVAL=0
ARGS=""
PROG="node_exporter"
DAEMON=/opt/node_exporter/${PROG}
PID_FILE=/var/run/${PROG}.pid
LOG_FILE=/var/log/node_exporter.log
LOCK_FILE=/var/lock/subsys/${PROG}
GOMAXPROCS=$(grep -c ^processor /proc/cpuinfo)

start() {
    if check_status > /dev/null; then
        echo "node_exporter is already running"
        exit 0
    fi

    echo -n $"Starting node_exporter: "
    ${DAEMON} ${ARGS} 1>>${LOG_FILE} 2>&1 &
    echo $! > ${PID_FILE}
    RETVAL=$?
    [ $RETVAL -eq 0 ] && touch ${LOCK_FILE}
    echo ""
    return $RETVAL
}

stop() {
    if check_status > /dev/null; then
        echo -n $"Stopping node_exporter: "
        kill -9 "$(cat ${PID_FILE})"
        RETVAL=$?
        [ $RETVAL -eq 0 ] && rm -f ${LOCK_FILE} ${PID_FILE}
        echo ""
        return $RETVAL
    else
        echo "node_exporter is not running"
        rm -f ${LOCK_FILE} ${PID_FILE}
        return 0
    fi
}  

check_status() {
    status -p ${PID_FILE} ${DAEMON}
    RETVAL=$?
    return $RETVAL
}

case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    restart)
        stop
        start
        ;;
    *)
        N=/etc/init.d/${NAME}
        echo "Usage: $N {start|stop|restart}" >&2
        RETVAL=2
        ;;
esac

exit ${RETVAL}
```

上記を `/etc/rc.d/init.d/node_exporter` として配置し、 `sudo chkconfig --add node_exporter` する。
