# cron防止重复执行
```bash
#使用timeout设定超时
timeout 10 bash /tmp/t1.sh
#上述命令10秒会超时不会无限等待
#再加上文件互斥锁
flock -xn /tmp/t1.lock -c "timeout 10 bash /tmp/t1.sh >> /tmp/t1exec.log 2>&1"
```